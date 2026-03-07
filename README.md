# services-healthcheck

A Go library that adds liveness (`/healthz`) and readiness (`/readyz`) endpoints to your service. Checkers run concurrently with a configurable timeout, and panics inside checkers are safely recovered and reported.

## Installation

```bash
go get github.com/jamillosantos/services-healthcheck
```

## Usage

### 1. Create a Healthcheck instance

```go
import svchealthcheck "github.com/jamillosantos/services-healthcheck"

hc := svchealthcheck.NewHealthcheck(
    svchealthcheck.WithTimeout(10 * time.Second), // default: 15s
    svchealthcheck.WithCheck("database", dbChecker),
    svchealthcheck.WithReadyCheck("cache", cacheChecker),
)
```

Checkers can also be added dynamically at any time:

```go
hc.AddHealthCheck("database", dbChecker)
hc.AddReadyCheck("cache", cacheChecker)
```

### 2. Implement a Checker

```go
type Checker interface {
    Check(ctx context.Context) error
}
```

Use `CheckerFunc` to wrap a function:

```go
checker := svchealthcheck.CheckerFunc(func(ctx context.Context) error {
    return db.PingContext(ctx)
})
```

For dependencies that expose a `Ping(ctx context.Context) error` method, use the built-in helper from the `checkers` package:

```go
import "github.com/jamillosantos/services-healthcheck/checkers"

dbChecker := checkers.PingerChecker(db)
```

### 3. Register routes with your HTTP framework

**`net/http`**

```go
import "github.com/jamillosantos/services-healthcheck/hchttp"

mux := http.NewServeMux()
hchttp.HttpInitialize(hc, mux)
```

**GoFiber v2**

```go
import "github.com/jamillosantos/services-healthcheck/hcfiber"

app := fiber.New()
hcfiber.FiberInitialize(hc, app)
```

**GoFiber v3**

```go
import "github.com/jamillosantos/services-healthcheck/hcfiberv3"

app := fiber.New()
hcfiberv3.FiberInitialize(hc, app)
```

## Endpoints

| Path | Description |
|---|---|
| `GET /healthz` | Liveness check — is the service alive? |
| `GET /readyz` | Readiness check — is the service ready to serve traffic? |

### Response format

HTTP `200 OK` when all checks pass, `503 Service Unavailable` when any check fails, `500 Internal Server Error` when a checker panics.

```json
{
  "status": "OK",
  "checks": {
    "database": {
      "duration": "1.23ms"
    },
    "cache": {
      "duration": "500µs",
      "error": "connection refused"
    }
  }
}
```

## Options

| Option | Default | Description |
|---|---|---|
| `WithTimeout(d)` | `15s` | Global timeout applied to all checkers per request |
| `WithCheck(name, checker)` | — | Register a liveness checker at construction time |
| `WithReadyCheck(name, checker)` | — | Register a readiness checker at construction time |
| `WithBindAddress(addr)` | `localhost:8082` | Bind address (used with server-fiber integration) |
| `WithInitializer(fn)` | — | Fiber app initializer (used with server-fiber integration) |
