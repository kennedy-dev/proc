# Gov Opportunity Scout — Deployment Plan (Docker Compose)

A self-hosted, fully open-source stack that runs the two-stream n8n workflow:
**Kenya government software-development opportunities** and **US state & city reconciliation solicitations**.

---

## 1. Architecture

```
                          ┌─────────────────────────── host / VPS ───────────────────────────┐
   Internet ──443/80──►   │  caddy (TLS, optional)  ──►  n8n ──┬── postgres   (workflow + creds) │
   (you, browser)         │                                    ├── searxng ── valkey  (search)   │
                          │                                    └── SMTP ──► your mail provider    │
                          └───────────────────────────────────────────────────────────────────┘
```

- **n8n** (community edition) runs the workflow and the daily schedules.
- **postgres** is n8n's database (durable, production-grade — not SQLite).
- **searxng** is the open-source search backend; n8n queries it privately at `http://searxng:8080`.
- **valkey** is a Redis-compatible cache SearXNG can use (only needed if you enable its limiter).
- **caddy** *(optional, `proxy` profile)* terminates HTTPS and reverse-proxies n8n.
- **mailpit** *(optional, `dev` profile)* is a local mailbox for testing the email digests before wiring real SMTP.
- **SMTP** for real alerts is your own mail provider (configured as an n8n credential, not a container).

Everything talks over a private Docker network; only the proxy (or, locally, `127.0.0.1` ports) is exposed.

---

## 2. Prerequisites

- A Linux host with **Docker Engine 24+** and the **Docker Compose v2** plugin (`docker compose version`).
- 2 vCPU / 4 GB RAM / 20 GB disk is comfortable for this workload.
- For production HTTPS: a **domain name** with a DNS record pointing at the host, and ports **80/443** open.
- SMTP credentials (host, port, user, password) for sending the digest emails — or use the `dev` mailbox first.

---

## 3. Bundle contents

```
gov-scout-stack/
├── docker-compose.yml          # the stack
├── .env.example                # copy to .env and fill in
├── Caddyfile                   # TLS reverse proxy (proxy profile)
├── searxng/
│   └── settings.yml            # SearXNG config (JSON API enabled, limiter off)
└── workflows/
    └── Gov_Opportunity_Scout.n8n.json   # the workflow to import
```

---

## 4. Quick start (local / staging)

```bash
cd gov-scout-stack

# 1) Secrets
cp .env.example .env
sed -i "s/CHANGE_ME_openssl_rand_hex_32/$(openssl rand -hex 32)/" .env          # N8N_ENCRYPTION_KEY
sed -i "s/CHANGE_ME_strong_db_password/$(openssl rand -hex 16)/" .env           # POSTGRES_PASSWORD
sed -i "s/CHANGE_ME_openssl_rand_hex_32/$(openssl rand -hex 32)/" searxng/settings.yml  # SearXNG secret

# 2) Start core services (+ local mailbox for testing)
docker compose --profile dev up -d

# 3) Watch them come up
docker compose ps
docker compose logs -f n8n        # Ctrl-C to stop tailing
```

Then:

1. Open **http://localhost:5678** and create the n8n owner account.
2. **Workflows → Import from File →** `workflows/Gov_Opportunity_Scout.n8n.json`.
3. Create the email credential:
   - **Testing:** Credentials → **SMTP** → host `mailpit`, port `1025`, no SSL, no auth. View mail at **http://localhost:8025**.
   - **Real:** your provider's SMTP host/port/user/password.
4. Open both **"email digest"** nodes and select that credential; set the `from`/`to` addresses.
5. (Optional) Confirm SearXNG works: `curl "http://localhost:8080/search?q=test&format=json"` should return JSON.
6. **Save**, then flip the workflow to **Active** (top-right). The dedup memory only persists while active.
7. To verify immediately, open a stream's first node and **Execute Workflow** once — relevant, de-duplicated hits should arrive by email (or land in Mailpit).

---

## 5. Production deployment (HTTPS)

1. Point DNS (e.g. `n8n.example.org`) at the host; open ports 80 and 443.
2. In `.env` set:
   ```
   N8N_DOMAIN=n8n.example.org
   N8N_HOST=n8n.example.org
   N8N_PROTOCOL=https
   N8N_PUBLIC_URL=https://n8n.example.org/
   N8N_SECURE_COOKIE=true
   N8N_PROXY_HOPS=1
   ```
3. Bring the stack up **with the proxy**:
   ```bash
   docker compose --profile proxy up -d
   ```
   Caddy fetches a TLS certificate automatically. The editor is now at `https://n8n.example.org`.
4. The `127.0.0.1:5678` / `127.0.0.1:8080` published ports stay bound to localhost only; Caddy reaches n8n over the internal network. You can remove those `ports:` lines entirely in production if you don't need local access.

---

## 6. Operations

**Backups** (cron these):
```bash
# database (workflow + encrypted credentials)
docker compose exec -T postgres pg_dump -U n8n n8n | gzip > backup-db-$(date +%F).sql.gz
# n8n home (encryption key lives here too) and searxng config
docker run --rm -v gov-scout_n8n_data:/d -v "$PWD":/b alpine tar czf /b/backup-n8n-$(date +%F).tgz -C /d .
```
Keep `N8N_ENCRYPTION_KEY` safe and unchanged — without it, restored credentials can't be decrypted.

**Updates:**
```bash
docker compose pull && docker compose up -d        # add --profile proxy/dev as used
docker image prune -f
```
Pin image tags (e.g. `n8nio/n8n:1._._`) if you prefer controlled upgrades over `latest`.

**Logs / health:**
```bash
docker compose ps
docker compose logs -f n8n searxng
```

**Cadence & dedup:** schedules are daily (07:00 KE, 07:30 US) via cron in the trigger nodes — edit there. Each stream remembers seen URLs for 90 days, so you only get genuinely new hits.

---

## 7. Hardening checklist

- [ ] Strong, unique `POSTGRES_PASSWORD` and a 32-byte `N8N_ENCRYPTION_KEY`; real `.env` never committed.
- [ ] Production uses HTTPS (`proxy` profile) with `N8N_SECURE_COOKIE=true` and `N8N_PROXY_HOPS=1`.
- [ ] Host firewall allows only 80/443 (and SSH); the `5678`/`8080` ports stay on `127.0.0.1` or are removed.
- [ ] SearXNG stays private (no public port); its `secret_key` is set.
- [ ] Automated daily DB + volume backups, stored off-host; restore tested once.
- [ ] n8n owner account uses a strong password; consider enabling MFA.

---

## 8. Troubleshooting

| Symptom | Fix |
|---|---|
| SearXNG returns **403** to n8n | JSON format not enabled — confirm `search.formats` includes `json` in `searxng/settings.yml`, and `server.limiter: false`. Restart: `docker compose restart searxng`. |
| Workflow runs but finds **nothing** | Test the query directly: `curl "http://localhost:8080/search?q=software+tender+site:go.ke&format=json"`. Then widen/adjust the strings in the **build queries** nodes and the keyword lists in **normalize & filter**. |
| `$env.SEARXNG_URL` is **undefined** in Code nodes | Ensure `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` (set in compose) and the `SEARXNG_URL` env is present on the n8n service; restart n8n. |
| **No emails** arrive | Check the SMTP credential; test with the `dev` profile + Mailpit (`http://localhost:8025`); look at `docker compose logs n8n`. |
| Dedup never resets / or sends repeats on manual test | Static-data memory persists only when the workflow is **Active** and fired by its trigger; manual test executions don't update it. |
| Schedules fire at the wrong time | Set `TZ` in `.env` to your IANA zone and `docker compose up -d` to apply. |
| Caddy can't get a cert | DNS must resolve to this host and 80/443 must be open; check `docker compose logs caddy`. |

---

## 9. Extending

The **"add Slack/Sheets here"** No-Op node after each digest is the extension point. Append, for example:
- **Slack / Telegram** node for chat alerts.
- **Google Sheets**, or the open-source **NocoDB / Baserow** nodes, to log every hit to a tracker.
- A second discovery path: an **HTTP Request + HTML Extract** branch that scrapes a specific portal (e.g. `tenders.go.ke`) directly, feeding the same normalize → dedup → digest chain.
