# Platforma Consumer App Project Structure

Use this layout for applications that consume `github.com/platforma-dev/platforma`.

## Recommended Layout

```text
myapp/
├── cmd/
│   └── myapp/
│       └── main.go
├── internal/
│   ├── config/
│   │   └── config.go
│   ├── app/
│   │   └── build.go
│   ├── platforma/
│   │   ├── database.go
│   │   └── http.go
│   └── domains/
│       ├── auth/
│       │   ├── domain.go
│       │   ├── service.go
│       │   ├── repository.go
│       │   ├── handlers.go
│       │   ├── middleware.go
│       │   └── migrations/
│       │       └── 001_create_auth_tables.sql
│       ├── users/
│       │   ├── domain.go
│       │   ├── service.go
│       │   ├── repository.go
│       │   ├── handlers.go
│       │   └── migrations/
│       │       └── 001_create_users_table.sql
│       ├── jobs/
│       │   ├── queue.go
│       │   └── handlers.go
│       └── scheduler/
│           └── tasks.go
├── go.mod
└── go.sum
```

## Domain-First Rule

Treat each business domain as a vertical slice. Put everything that domain needs in one package:

- `repository.go`: DB reads/writes for that domain.
- `service.go`: business logic.
- `handlers.go`: HTTP endpoints and request/response mapping.
- `middleware.go` (optional): auth or domain-specific request guards.
- `migrations/`: SQL migrations owned by this domain.
- `domain.go`: domain wiring entrypoint used by app bootstrap.

Avoid shared cross-domain repository/service folders by default.

## File Responsibilities (Top Level)

- `cmd/myapp/main.go`
  - Parse config and startup context.
  - Call a single app builder.
  - Execute `app.Run(ctx)`.
- `internal/app/build.go`
  - Compose all domains into `application.Application`.
  - Register database, repositories/domains, and service runners.
  - Keep registration order explicit and centralized.
- `internal/platforma/database.go`
  - Construct `database.New(...)`.
  - Provide shared DB bootstrap helpers.
- `internal/platforma/http.go`
  - Construct `httpserver.New(...)`.
  - Register shared middleware and mount domain handlers.

## Build and Runtime Shape

1. Build an `application.Application` instance.
2. Register database(s).
3. Register each domain (which brings its repository/service/handlers/migrations).
4. Register service runners (`httpserver`, `queue`, `scheduler`, custom runners).
5. Run migrations via `myapp migrate`.
6. Start services via `myapp run`.

## Domain Wiring Pattern

Each domain should expose one constructor/wiring function that returns the pieces app bootstrap needs.

```go
type Domain struct {
    Name       string
    Repository any
    Routes     http.Handler
}

func New(db *sqlx.DB) *Domain {
    repo := NewRepository(db)
    svc := NewService(repo)
    h := NewHandlers(svc)
    return &Domain{
        Name:       "users",
        Repository: repo,
        Routes:     h,
    }
}
```

In app bootstrap:

1. Create domain via `domains/users.New(...)`.
2. Register repository/domain for migrations.
3. Mount routes to HTTP server.
4. Register services (`httpserver`, `queue`, `scheduler`, and custom runners).

## Practical Conventions

- Keep domain boundaries strict; avoid leaking repository/service objects across domains.
- Keep SQL migrations versioned and immutable after release, inside the owning domain.
- Keep registration/wiring in one place to avoid scattered order bugs.
- Name services with stable identifiers (`api`, `queue`, `scheduler`) for readable logs.
- Use `context.Context` end-to-end for graceful shutdown and cancellation.
