# URL Shortener

A lightweight URL shortening service written in Go. Create short links, redirect through them with a Redis-backed fast path, and track click analytics without slowing down the redirect.

Built as a learning project to practice caching, concurrency, and schema design in a real (but small) backend.

## Features

- Shorten any URL into a random 7-character base62 code
- Optional expiry (TTL) per link
- Fast redirects: Redis cache first, PostgreSQL as the source of truth
- Asynchronous click logging that never blocks the redirect
- Per-link stats: total clicks, creation time, last click

## Tech Stack

| Layer | Choice |
|---|---|
| Language | Go 1.22 |
| Router | [chi](https://github.com/go-chi/chi) |
| Database | PostgreSQL via [pgx](https://github.com/jackc/pgx) |
| Cache | Redis via [go-redis](https://github.com/redis/go-redis) |

## Getting Started

**Prerequisites:** Go 1.22+, Docker

```bash
# 1. Start Postgres and Redis
docker compose up -d

# 2. Apply the schema
psql postgres://postgres:postgres@localhost:5432/urlshortener -f migrations/001_init.sql

# 3. Run the server
go mod tidy
go run ./cmd/server
```

The server runs on `:8080` by default.

### Configuration

| Variable | Default |
|---|---|
| `PORT` | `8080` |
| `DATABASE_URL` | `postgres://postgres:postgres@localhost:5432/urlshortener?sslmode=disable` |
| `REDIS_ADDR` | `localhost:6379` |
| `BASE_URL` | `http://localhost:8080` |

## API

### `POST /shorten`

```bash
curl -X POST http://localhost:8080/shorten \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/some/long/path", "ttl_seconds": 3600}'
```

`ttl_seconds` is optional. Response:

```json
{
  "code": "aB3xY9z",
  "short_url": "http://localhost:8080/aB3xY9z",
  "long_url": "https://example.com/some/long/path",
  "created_at": "2026-09-28T10:00:00Z"
}
```

### `GET /{code}`

Redirects (302) to the original URL. Returns `404` if the code doesn't exist or has expired.

### `GET /{code}/stats`

```json
{
  "code": "aB3xY9z",
  "long_url": "https://example.com/some/long/path",
  "total_clicks": 42,
  "created_at": "2026-09-28T10:00:00Z",
  "last_click": "2026-09-28T12:30:00Z"
}
```

## Project Structure

```
cmd/server/         Entry point and server wiring
internal/config/    Environment-based configuration
internal/db/        PostgreSQL connection pool
internal/cache/     Redis wrapper
internal/store/     Database queries
internal/shortcode/ Base62 code generation
internal/handlers/  HTTP handlers
internal/models/    Shared types
migrations/         SQL schema
```

## Roadmap

- [ ] Background job to purge expired links
- [ ] Custom aliases
- [ ] API key authentication
- [ ] Rate limiting
- [ ] Worker pool for click logging with graceful drain on shutdown
- [ ] QR code generation