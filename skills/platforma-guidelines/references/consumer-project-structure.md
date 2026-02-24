# Platforma Consumer App Project Structure

Use this layout for applications that consume `github.com/platforma-dev/platforma`.

## Recommended Layout

```text
myapp/
├── cmd/
│   └── myapp/
│       └── main.go
├── internal/
│   ├── app/
│   │   └── build.go
│   ├── config/
│   │   └── config.go
│   ├── auth/
│   │   ├── domain.go
│   │   ├── service.go
│   │   ├── repository.go
│   │   ├── handlers.go
│   │   ├── middleware.go
│   │   └── migrations/
│   │       └── 0001_create_auth_tables.sql
│   └── users/
│       ├── domain.go
│       ├── service.go
│       ├── repository.go
│       ├── handlers.go
│       └── migrations/
│           └── 0001_create_users_table.sql
├── go.mod
└── go.sum
```

## Domain-First Rule

Treat each business domain as a vertical slice. Put everything that domain needs in one package directly under `internal/`:

- `repository.go`: DB reads/writes for that domain.
- `service.go`: business logic.
- `handlers.go`: HTTP endpoints and request/response mapping.
- `middleware.go` (optional): auth or domain-specific request guards.
- `migrations/`: SQL migrations owned by this domain.
- `domain.go`: domain wiring entrypoint used by app bootstrap.

Avoid shared cross-domain repository/service folders by default. Each domain lives in its own `internal/<domain>/` package.

## File Responsibilities (Top Level)

- `cmd/myapp/main.go`
  - Parse config and startup context.
  - Call `app.Build(...)` from `myapp/internal/app`.
  - Execute `app.Run(ctx)`.
- `internal/app/build.go`
  - Compose all domains into `application.Application`.
  - Register database, repositories/domains, and service runners.
  - Keep registration order explicit and centralized.

## Build and Runtime Shape

1. Build an `application.Application` instance.
2. Register database(s).
3. Register each domain (which brings its repository/service/handlers/migrations).
4. Register service runners (`httpserver`, `queue`, `scheduler`, custom runners).
5. Run migrations via `myapp migrate`.
6. Start services via `myapp run`.

## CLI Commands (Built-in)

**Important:** Platforma's `Application.Run()` parses CLI arguments automatically. Do not implement command parsing in your code.

The application supports these commands out of the box:

- `myapp run` - Start all services
- `myapp migrate` - Run database migrations and exit
- `myapp --help` - Show usage

Your `main()` should be simple: create the app, register components, and call `app.Run(ctx)`. No flag parsing, no switch statements.

**Do NOT do this:**
```go
// ❌ Wrong - don't parse commands yourself
if len(os.Args) > 1 && os.Args[1] == "migrate" {
    db.Migrate(ctx)
} else {
    app.Run(ctx)
}
```

**Do this instead:**
```go
// ✅ Correct - let platforma handle it
app.Run(ctx)  // parses run/migrate/--help automatically
```

## Domain Wiring Pattern

Each domain should expose one constructor/wiring function that returns the pieces app bootstrap needs.

```go
type Domain struct {
    Name       string
    Repository *Repository
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

func (d *Domain) GetRepository() any {
    return d.Repository
}
```

In app bootstrap:

1. Create domain via `internal/users.New(...)`.
2. Register repository/domain for migrations.
3. Mount routes to HTTP server.
4. Register services (`httpserver`, `queue`, `scheduler`, and custom runners).

## Practical Conventions

- Keep domain boundaries strict; avoid leaking repository/service objects across domains.
- Keep SQL migrations versioned and immutable after release, inside the owning domain.
- Keep registration/wiring in one place to avoid scattered order bugs.
- Name services with stable identifiers (`api`, `queue`, `scheduler`) for readable logs.
- Use `context.Context` end-to-end for graceful shutdown and cancellation.

## Simple Application Example

Here's a complete, straightforward app bootstrap pattern. No builder chains, no abstraction layers:

```go
// internal/app/build.go
package app

import (
    "context"

    "github.com/jmoiron/sqlx"
    "github.com/platforma-dev/platforma/application"
    "github.com/platforma-dev/platforma/database"
    "github.com/platforma-dev/platforma/httpserver"

    "myapp/internal/auth"
    "myapp/internal/users"
)

func Build(ctx context.Context, cfg Config) *application.Application {
    app := application.New()

    // Database
    db := database.New(cfg.DatabaseURL)
    app.RegisterDatabase("main", db)

    // Domains (they register their own repositories)
    authDomain := auth.New(db.Connection())
    usersDomain := users.New(db.Connection())

    // Register domains
    app.RegisterDomain("auth", "main" authDomain)
    app.RegisterDomain("users", "main", usersDomain)

    // HTTP server
    api := httpserver.New(cfg.Port, cfg.Timeout)
    api.Handle("/auth", authDomain.Handler)
    api.HandleGroup("/users", usersDomain.HandlerGroup)
    app.RegisterService("api", api)

    return app
}
```

**Key points:**
- Linear code flow from top to bottom
- Each step clearly labeled
- No builder patterns or fluent chains
- All registration happens in one file
- Easy to read and debug

That's it. Call `Build()` from main and you're done.
