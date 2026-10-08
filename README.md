# LogPocket

LogPocket is a lightweight, self-hosted log collection, storage, search,
and viewing application.

> Status: Early development. v0.1.0 has not been released yet.

## Goals

- Simple deployment
- Low resource usage
- Local-first operation
- Persistent log history
- Fast full-text search
- Real-time log viewing
- No mandatory external database
- Open-source and community-driven

## Technology

- Go
- Svelte 5
- TypeScript
- SQLite + FTS5
- Docker Engine API
- Server-Sent Events

## Development

### Backend

From the repository root:

    go run ./cmd/logpocket

### Frontend

From the repository root:

    cd web
    pnpm install
    pnpm dev

### Build

Backend:

    go build ./...

Frontend:

    cd web
    pnpm build

### Tests

    go test ./...

## Architecture

See [architecture.md](architecture.md).

## Development roadmap

See [DEVELOPMENT_PLAN.md](DEVELOPMENT_PLAN.md).

## Implementation guide

See [IMPLEMENTATION_GUIDE.md](IMPLEMENTATION_GUIDE.md).

## License

Apache License 2.0.
