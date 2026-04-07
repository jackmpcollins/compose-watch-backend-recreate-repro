# Compose Watch Backend Recreate Repro

This repo is a minimal reproduction for a Docker Compose watch bug where rebuilding `frontend` also recreates `backend`, and `backend` can then be left stuck in `Created`.

## What This Repo Contains

The setup is intentionally small:

- `backend` is a built service based on `hashicorp/http-echo`
- `frontend` is a `caddy` reverse proxy to `backend:8000`
- `frontend` depends on `backend`
- both services have `develop.watch` configured
- touching `frontend/trigger.txt` triggers a `frontend` rebuild

The checked-in compose file is:

```yaml
services:
  backend:
    build: ./backend
    develop:
      watch:
        - action: restart
          path: ./backend/Dockerfile

  frontend:
    depends_on:
      - backend
    build: ./frontend
    ports:
      - "3002:3000"
    develop:
      watch:
        - action: rebuild
          path: ./frontend/trigger.txt
```

Supporting files:

- `backend/Dockerfile`

```Dockerfile
FROM hashicorp/http-echo:1.0.0

CMD ["-text=backend ok", "-listen=:8000"]
```

- `frontend/Dockerfile`

```Dockerfile
FROM caddy:2.10.2-alpine

COPY Caddyfile /etc/caddy/Caddyfile
```

- `frontend/Caddyfile`

```text
:3000 {
    reverse_proxy backend:8000
}
```

- `frontend/trigger.txt`

```text
touch me to trigger a frontend rebuild
```

## Repro

You can reproduce it either by starting `watch` separately:

```bash
docker compose down
docker compose up -d --build
docker compose watch frontend
```

Or by using `up --watch`:

```bash
docker compose down
docker compose up --build --watch
```

In another shell, trigger the `frontend` rebuild and inspect the resulting state:

```bash
touch frontend/trigger.txt
docker compose ps
docker compose ps -a
curl -i http://localhost:3002/
docker compose logs --since=1m frontend
```

## Expected Behavior

Touching `frontend/trigger.txt` should rebuild or recreate `frontend` only, while `backend` stays healthy and reachable through `http://localhost:3002/`.

## Actual Behavior

Watch output includes `backend` being recreated even though the change only targets `frontend`:

```text
Rebuilding service(s) ["frontend"] after changes were detected...
Container <project>-backend-1 Recreate
Container <project>-backend-1 Recreated
Container <project>-frontend-1 Recreate
Container <project>-frontend-1 Recreated
```

With `docker compose up --build --watch`, there is also:

```text
backend-1 has been recreated
backend-1 ... received interrupt, shutting down...
backend-1 exited with code 2
```

After that, the stack is broken:

- `docker compose ps` no longer lists `backend`
- `docker compose ps -a` shows `backend` in `Created`
- `curl http://localhost:3002/` returns `502`
- `frontend` logs show `lookup backend on 127.0.0.11:53: no such host`

## Ablations

The bug disappears if any of the following are changed:

- remove `backend.develop.watch`
- remove `frontend.depends_on`
- remove `frontend.develop.watch`
- change `backend` to a pure `image:` service

The `backend` watch action does not need to be `rebuild`; `restart` is enough.
