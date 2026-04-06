# Compose Watch Backend Recreate Repro

This is a minimal reproduction for a Docker Compose watch reconcile issue:

- `backend` is a one-line build wrapper around the off-the-shelf `hashicorp/http-echo` image
- `frontend` is a tiny `caddy` reverse proxy to `http://backend:8000`
- `frontend` depends on `backend`
- only `frontend` has a `develop.watch` rebuild rule
- touching `frontend/trigger.txt` is enough to trigger the issue
- using `backend` as a pure `image:` service did not reproduce the same behavior in my local tests

## Run

```bash
cd repros/compose-watch-backend-recreate
docker compose down
docker compose up --build --watch
```

In another shell:

```bash
cd repros/compose-watch-backend-recreate
docker compose ps
docker compose logs -f -t frontend backend
while true; do
  printf '%s ' "$(date -u +%Y-%m-%dT%H:%M:%S.%3NZ)"
  curl -s -o /dev/null -w 'status=%{http_code} total=%{time_total}\n' \
    http://localhost:3002/ || echo curl_failed
  sleep 0.2
done
```

Trigger the rebuild:

```bash
cd repros/compose-watch-backend-recreate
touch frontend/trigger.txt
```

## Expected

The watch session should report that it is rebuilding `frontend`, then unexpectedly recreate `backend`.

Typical log sequence:

```text
Rebuilding service(s) ["frontend"] after changes were detected...
backend-1 has been recreated
backend-1 exited after receiving an interrupt
backend-1 has been recreated
frontend-1 ... lookup backend on 127.0.0.11:53: no such host
```

After the event, `docker compose ps` may show that `backend` is gone while `frontend` remains up and keeps returning `500`.
