# Compose Watch Backend Recreate Repro

This is a minimal reproduction for a Docker Compose watch reconcile issue:

- `backend` is a one-line build wrapper around the off-the-shelf `hashicorp/http-echo` image
- `frontend` is a tiny `caddy` reverse proxy to `http://backend:8000`
- `frontend` depends on `backend`
- both `frontend` and `backend` have `develop.watch` config
- touching `frontend/trigger.txt` is enough to trigger the issue
- on Docker 29.3.1 / Compose v5.1.1, the original repo only reproduced reliably after adding a `backend.develop.watch` entry

## Run

```bash
cd /Users/jack/Code/compose-watch-backend-recreate-repro
docker compose down
docker compose up -d --build
docker compose watch --quiet frontend
```

In another shell:

```bash
cd /Users/jack/Code/compose-watch-backend-recreate-repro
while true; do
  printf '%s ' "$(date -u +%Y-%m-%dT%H:%M:%S.%3NZ)"
  curl -s -o /dev/null -w 'status=%{http_code} total=%{time_total}\n' \
    http://localhost:3002/ || echo curl_failed
  sleep 0.2
done
```

Trigger the rebuild:

```bash
cd /Users/jack/Code/compose-watch-backend-recreate-repro
touch frontend/trigger.txt
```

## Expected

The watch session should report that it is rebuilding `frontend`, then unexpectedly recreate `backend`.

Typical log sequence:

```text
Rebuilding service(s) ["frontend"] after changes were detected...
compose-watch-backend-recreate-repro-backend-1 Recreate
compose-watch-backend-recreate-repro-backend-1 Recreated
frontend-1 ... lookup backend on 127.0.0.11:53: no such host
```

After the event:

- `docker compose ps` no longer shows `backend`
- `docker compose ps -a` shows `backend` stuck in `Created`
- `curl http://localhost:3002/` returns `502`
- `docker compose logs frontend` shows `lookup backend on 127.0.0.11:53: no such host`

## Minimal Config

The smallest config I found that still reproduces the bug is:

- `backend` is a built service, not a pure `image:` service
- `frontend` depends on `backend`
- `frontend` has a `develop.watch` rebuild rule for `frontend/trigger.txt`
- `backend` has any `develop.watch` rule at all; in this repro it uses `action: restart` on `backend/Dockerfile`
