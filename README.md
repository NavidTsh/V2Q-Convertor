# V2Q Conertor

This web app converts OVA/OVF files to QCOW2 images for use with OpenStack. It can also install VirtIO drivers on QCOW2 images.

## Architecture

```
Browser
  │  HTTPS (put nginx/TLS in front for real use)
  ▼
web (FastAPI, login + UI + API)  ─┐
                                   ├─▶ Redis (job status/events, Celery broker)
worker (Celery)  ─────────────────┘
  │  HTTP only (no docker.sock, no data volume)
  ▼
agent (FastAPI)  ── docker.sock + data volume ──▶ Docker host
  │
  ▼  docker run/exec
v2q:3.1.0 container (virt-v2v, ovftool, ...)
```

**Security boundary:** only `agent` has `/var/run/docker.sock` and the
shared data folder mounted. `web` and `worker` never touch either
directly - they only make HTTP calls to `agent`'s narrow API (`/browse`,
`/jobs`). Even a full compromise of `web`/`worker` does not hand over
raw Docker access.

`agent` itself validates every path against `CONTAINER_DATA_ROOT` and
only ever starts the fixed, configured `V2V_DOCKER_IMAGE` - it does not
accept arbitrary image names or extra docker flags from callers.

## Deploy

1. `cp .env.example .env` and fill in `HOST_DATA_ROOT` (the real host
   path holding your VM export folders), a strong `LOGIN_PASSWORD`, and
   a random `SESSION_SECRET`.
2. Make sure the image named in `V2V_DOCKER_IMAGE` (default
   `v2q:3.1.0`) is already built on this host (`docker images | grep v2q`).
3. `docker compose up -d --build`
4. The UI listens on `127.0.0.1:8080` only (not exposed externally by
   itself). Put nginx (or similar) in front with TLS for real access:

   ```nginx
   server {
       listen 443 ssl;
       server_name v2v.example.internal;
       ssl_certificate     /etc/ssl/certs/v2v.crt;
       ssl_certificate_key /etc/ssl/private/v2v.key;

       location / {
           proxy_pass http://127.0.0.1:8080;
           proxy_set_header Host $host;
           proxy_set_header Upgrade $http_upgrade;      # needed for the WebSocket log stream
           proxy_set_header Connection "upgrade";
       }
   }
   ```

5. Restrict network access to this host/port to your internal network
   (firewall, VPN, security group, etc) - the built-in login is a
   convenience, not a substitute for network-level restriction.

## Managing users

The `LOGIN_USERNAME`/`LOGIN_PASSWORD` in `.env` are a "bootstrap" account
that always works, even with an empty user store - so you can never lock
yourself out. Beyond that, add as many named accounts as you want,
stored (bcrypt-hashed) in a file on a persistent volume, no rebuild
needed:

```bash
docker compose exec web python manage_users.py add alice
# prompts for a password (hidden input)

docker compose exec web python manage_users.py add bob supersecretpw123
# or pass the password directly (careful: ends up in shell history)

docker compose exec web python manage_users.py list
docker compose exec web python manage_users.py remove alice
```

## Known limitations / next hardening steps

- The login is a single fixed username/password by default, no rate
  limiting on `/api/login` - use `manage_users.py` to add named
  accounts with their own passwords; still fine for a small internal
  team, not for anything broader without adding rate limiting/lockout.
- `agent`'s job list/events are kept in memory - restarting the `agent`
  container loses in-flight job history (Redis keeps it on the `web`
  side, but the underlying container run is gone too).
- No TLS termination is included - it MUST sit behind a reverse proxy
  for anything beyond local testing.
- Consider adding a request size / rate limit in front (nginx) to
  reduce abuse surface on `/api/login` and `/api/jobs`.
