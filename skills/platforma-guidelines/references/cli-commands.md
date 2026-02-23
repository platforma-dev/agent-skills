# Platforma Framework-Native CLI Commands

Use this file for backend operator command guidance.

## Command Map

| Command | Purpose | When to run |
|---|---|---|
| `go run ./cmd/myapp migrate` | Run migrations without prebuilding a binary | Local development and quick checks |
| `go run ./cmd/myapp run` | Start app directly from source | Local development |
| `go run ./cmd/myapp --help` | Show runtime command usage from source | Verify command availability |
| `./myapp migrate` | Run registered DB migrations and exit | Before service startup, deploy prep |
| `./myapp run` | Start startup tasks and services | Normal runtime |
| `./myapp --help` | Show app command usage | Discover/verify commands |

## Runtime Command Behavior (`application` package)

- `run`
  - Execute startup tasks in registration order.
  - Start services concurrently.
  - Block until context cancellation.
- `migrate`
  - Run migrations on registered databases/repositories.
  - Exit after migration completes.
- `--help` and unknown/no command
  - Show command usage.
  - Unknown commands return an explicit command error.

## Standard Operator Sequences

### Local development

```bash
go run ./cmd/myapp migrate
go run ./cmd/myapp run
```

### Deployment startup

```bash
./myapp migrate && ./myapp run
```
