---
name: golang-app-project-directory-structure
description: Standard folder and file layout for Go application projects (web services, CLI tools). Covers migrations, seeding, API routes, models, stores, and internal package organization.
---

# Go Application Project Structure

A practical, production-ready project layout for Go applications including web services and CLI tools. Prefers `internal/` for encapsulation and uses "stores" (not repositories) for data access.

## When to Use This Skill

Uses cases:

- Setting up a new Go web service or CLI project.
- Adding new modules to an existing project.
- Refactoring an unorganized codebase into a more maintainable structure.

## Project Layout Overview

```
myapp/
├── .cast                       # Cast project files
├── cmd/                        # Main applications
│   ├── api/                    # API server
│   │   └── main.go
│   ├── cli/                    # CLI application
│   │   └── main.go
│   └── migrate/                # Migration tool
│       └── main.go
├── eng/                        # scripts, ci/cd, Dockerfiles, etc.
|      scripts/                 # Build and utility scripts
│      ├── build.sh      
│      └── migrate.sh
├── internal/                   # Private application code
│   ├── api/                    # HTTP handlers and routes
│   │   ├── handler.go
│   │   ├── middleware.go
│   │   ├── routes.go
│   │   └── user_handler.go
│   ├── crypto/                 # Cryptographic utilities
│   │   └── crypto.go
│   ├── stores/                  # Data access layer (stores, not repositories)
│   │   └── users/
│   │       ├── store.go            # UserStore interface
│   │       ├── sqlite.go          # store implementation
│   │       └── models.go           # User model definition
│   ├── svc/                # Business logic layer
│   │   ├── user_service.go
│   │   └── auth_service.go
│   ├── seed/                   # Database seeding
│   │   └── seed.go
│   └── config/                 # Configuration
│   │   └── config.go
│   ├── migrations/              # Database migrations
│   │   ├── 001_create_users.up.sql
│   │   ├── 001_create_users.down.sql
│   │   ├── 002_create_products.up.sql
│   │   └── 002_create_products.down.sql
│   ├── seeds/                      # Seed data files
│   │   ├── users.sql
│   │   └── products.sql
│   └── telemetry/                  # Logging, metrics, tracing such as otel
│       └── telemetry.go
├── pkg/                        # Public packages (only if reusable)
│   └── validator/
│       └── validator.go
├── test/                       # Integration tests and test fixtures
│   ├── integration/
│   └── fixtures/
├── go.mod
├── go.sum
├── castfile                   # Cast task definitions
└── README.md
```

If the project uses supports multiple database providers, the `store` directory can be organized by provider:

```
    stores
      users
         mssql
           store.go # mssql implementation
         mysql
           store.go  # mysql implementation
         pg
            store.go # postgres implementation
         sqlite
            store.go # sqlite implementation
         store.go # UserStore interface
         models.go # User model definition

    migrations
        pg
            001_create_users.up.sql
            001_create_users.down.sql
        mssql
            001_create_users.up.sql
            001_create_users.down.sql
        mysql
            001_create_users.up.sql
            001_create_users.down.sql
        sqlite
            001_create_users.up.sql
            001_create_users.down.sql
    seeds
        pg
            users.sql
        mssql
            users.sql
        mysql
            users.sql
        sqlite
            users.sql
```

Prefer short well known abbrivations for folders over long verbose names. For example, `svc` is more concise than `service` and still clear in context. 

Avoid unnecessary nesting - keep the structure as flat as possible while maintaining clarity.



## Directory Descriptions

### `/cmd` - Main Applications

Entry points for each application. Keep these minimal - just initialization and wiring.

```go
// cmd/api/main.go
package main

import (
    "context"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    
    "myapp/internal/api"
    "myapp/internal/config"
    "myapp/internal/store/postgres"
)

func main() {
    cfg := config.Load()
    
    db, err := postgres.New(cfg.DatabaseURL)
    if err != nil {
        slog.Error("failed to connect to database", "err", err)
        os.Exit(1)
    }
    defer db.Close()
    
    srv := api.NewServer(db, cfg)
    
    go func() {
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            slog.Error("server error", "err", err)
        }
    }()
    
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    srv.Shutdown(ctx)
}
```

### `/internal` - Private Application Code

The `internal` directory is special in Go - packages here can only be imported by code in the parent directory tree. Use this for all application logic by default. Only create `pkg/` for packages that might be imported by other projects. Ask the user if they have a specific reason to make something public.

### `/internal/api` - HTTP Layer

Handles HTTP-specific concerns: routing, middleware, request/response handling.

**routes.go** - Route definitions:
```go
package api

type Server struct {
    router     *chi.Mux
    users      *UserHandler
    products   *ProductHandler
}

func (s *Server) Routes() {
    r := s.router
    
    r.Group(func(r chi.Router) {
        r.Use(middleware.Logger)
        r.Use(middleware.Recoverer)
        
        r.Get("/health", s.Health)
        
        r.Route("/api/v1/users", func(r chi.Router) {
            r.Get("/", s.users.List)
            r.Post("/", s.users.Create)
            r.Get("/{id}", s.users.Get)
            r.Put("/{id}", s.users.Update)
            r.Delete("/{id}", s.users.Delete)
        })
    })
}
```

**handler.go** - Common handler patterns:
```go
package api

type Handler struct {
    store  store.Store
    logger *slog.Logger
}

func (h *Handler) respond(w http.ResponseWriter, r *http.Request, data any, status int) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    if data != nil {
        json.NewEncoder(w).Encode(data)
    }
}

func (h *Handler) error(w http.ResponseWriter, r *http.Request, err error, status int) {
    h.logger.Error("request failed", "err", err, "path", r.URL.Path)
    h.respond(w, r, map[string]string{"error": err.Error()}, status)
}
```

**user_handler.go** - Resource-specific handlers:
```go
package api

type UserHandler struct {
    store   store.UserStore
    service *service.UserService
}

func (h *UserHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        h.error(w, r, err, http.StatusBadRequest)
        return
    }
    
    user, err := h.service.Create(r.Context(), req)
    if err != nil {
        h.error(w, r, err, http.StatusInternalServerError)
        return
    }
    
    h.respond(w, r, user, http.StatusCreated)
}
```



### `/internal/stores` - Data Access Layer

Inside `stores/`, define interfaces at the top level and implementations in subdirectories. Use "store" naming convention instead of "repository".

#### Example: UserStore

```
myapp/
├── internal/
│   ├── stores/
│   │   └── users/
│   │       ├── store.go            # UserStore interface
│   │       ├── pg.go          # PostgreSQL implementation
│   │       └── models.go           # User model definition
```

Use "Store" naming convention instead of "Repository". Define interfaces at the top level, implementations in subdirectories.

**internal/stores/users/store.go** - Aggregated interface:
```go
package users

type Store interface {
    Users() UserStore
    Products() ProductStore
}

type UserStore interface {
    Get(ctx context.Context, id int64) (*model.User, error)
    List(ctx context.Context, filter model.UserFilter) ([]*model.User, error)
    Create(ctx context.Context, user *model.User) error
    Update(ctx context.Context, user *model.User) error
    Delete(ctx context.Context, id int64) error
}
```

**internal/stores/users/pg.go** - PostgreSQL implementation:
```go
package users

type Store struct {
    db      *sql.DB
    users   *UserStore
    products *ProductStore
}

func New(databaseURL string) (*Store, error) {
    db, err := sql.Open("postgres", databaseURL)
    if err != nil {
        return nil, err
    }
    
    if err := db.Ping(); err != nil {
        return nil, err
    }
    
    return &Store{
        db:       db,
        users:    &UserStore{db: db},
        products: &ProductStore{db: db},
    }, nil
}

func (s *Store) Users() store.UserStore   { return s.users }
func (s *Store) Products() store.ProductStore { return s.products }
func (s *Store) Close() error             { return s.db.Close() }
```

**internal/stores/users/models.go** - User model definition:

```go
package users

type User struct {
    ID             int64     `json:"id"`
    Email          string    `json:"email"`
    Name           string    `json:"name"`
    HashedPassword string    `json:"-"`
    CreatedAt      time.Time `json:"created_at"`
    UpdatedAt      time.Time `json:"updated_at"`
}
```

### `/internal/svc` - Business Logic

Domain logic that orchestrates stores and other services. Not just a passthrough to stores.

```go
package svc

type UserService struct {
    store   store.UserStore
    hasher  password.Hasher
    emailer EmailService
}

func (s *UserService) Create(ctx context.Context, req model.CreateUserRequest) (*model.User, error) {
    if err := validator.Validate(req); err != nil {
        return nil, err
    }
    
    hashedPassword, err := s.hasher.Hash(req.Password)
    if err != nil {
        return nil, err
    }
    
    user := &model.User{
        Email:          req.Email,
        Name:           req.Name,
        HashedPassword: hashedPassword,
    }
    
    if err := s.store.Create(ctx, user); err != nil {
        return nil, err
    }
    
    s.emailer.SendWelcome(ctx, user.Email)
    
    return user, nil
}
```

### `/migrations` - Database Migrations

Use versioned SQL migrations with `.up.sql` and `.down.sql` suffixes.

```
migrations/
├── 001_create_users.up.sql
├── 001_create_users.down.sql
├── 002_create_products.up.sql
├── 002_create_products.down.sql
├── 003_add_user_status.up.sql
└── 003_add_user_status.down.sql
```

**001_create_users.up.sql**:
```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    password_hash TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

**001_create_users.down.sql**:
```sql
DROP INDEX IF EXISTS idx_users_email;
DROP TABLE IF EXISTS users;
```

Recommended migration tools:
- [goose](https://github.com/pressly/goose)
- [migrate](https://github.com/golang-migrate/migrate)

### `/seeds` - Database Seeding

SQL or Go-based seed data for development and testing.

**seeds/users.sql**:
```sql
INSERT INTO users (email, name, password_hash) VALUES
    ('admin@example.com', 'Admin User', '$2a$10$...'),
    ('user1@example.com', 'Test User', '$2a$10$...');
```

**internal/seed/seed.go**:
```go
package seed

type Seeder struct {
    db *sql.DB
}

func (s *Seeder) Run(ctx context.Context) error {
    if err := s.seedUsers(ctx); err != nil {
        return err
    }
    return s.seedProducts(ctx)
}

func (s *Seeder) seedUsers(ctx context.Context) error {
    users := []struct {
        email, name, password string
    }{
        {"admin@example.com", "Admin", "password123"},
        {"user@example.com", "User", "password123"},
    }
    
    for _, u := range users {
        hash, _ := bcrypt.GenerateFromPassword([]byte(u.password), bcrypt.DefaultCost)
        _, err := s.db.ExecContext(ctx,
            "INSERT INTO users (email, name, password_hash) VALUES ($1, $2, $3)",
            u.email, u.name, hash,
        )
        if err != nil {
            return err
        }
    }
    return nil
}
```

### `/internal/config` - Configuration

Centralized configuration loading from environment, files, or flags.

Unless otherwise stated, use viper.

```go
package config

type Config struct {
    DatabaseURL string
    Port        int
    LogLevel    string
    JWTSecret   string
}

func Load() *Config {
    return &Config{
        DatabaseURL: getEnv("DATABASE_URL", "postgres://localhost/myapp?sslmode=disable"),
        Port:        getEnvInt("PORT", 8080),
        LogLevel:    getEnv("LOG_LEVEL", "info"),
        JWTSecret:   getEnv("JWT_SECRET", ""),
    }
}

func getEnv(key, defaultValue string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return defaultValue
}
```

### `/pkg` - Public Packages (Optional)

Only create `pkg/` if you have packages that might be imported by other projects. Most applications don't need this - prefer `internal/`.

```
pkg/
└── validator/
    └── validator.go    # Reusable validation helpers
```

## Dependency Injection Patterns

### Constructor Injection

```go
type Server struct {
    store   store.Store
    service *service.UserService
    logger  *slog.Logger
}

func NewServer(st store.Store, svc *service.UserService, logger *slog.Logger) *Server {
    return &Server{
        store:   st,
        service: svc,
        logger:  logger,
    }
}
```

### Wire-up in main

```go
func main() {
    cfg := config.Load()
    
    db, _ := postgres.New(cfg.DatabaseURL)
    defer db.Close()
    
    userSvc := service.NewUserService(db.Users())
    productSvc := service.NewProductService(db.Products())
    
    srv := api.NewServer(db, userSvc, productSvc, slog.Default())
    
    http.ListenAndServe(":"+strconv.Itoa(cfg.Port), srv)
}
```

## Testing Structure

```
internal/
├── store/
│   ├── user_store_test.go       # Unit tests with mocks
│   └── postgres/
│       └── user_store_test.go   # Integration tests
└── service/
    └── user_service_test.go

test/
├── integration/
│   └── api_test.go              # Full stack tests
└── fixtures/
    └── users.json               # Test data
```

## Castfile Example

```yaml
tasks:
  build:
    desc: "Build the project"
    uses: bash
    run: |
      go build -o bin/api ./cmd/api

  run:
    desc: "Run the API server"
    uses: bash
    run: go run ./cmd/api

  test:
    desc: "Run tests"
    uses: bash
    run: go test -v ./...

  migrate-up:
    desc: "Run database migrations up"
    uses: bash
    env:
      DATABASE_URL: ${DATABASE_URL}
    run: |
      goose -dir internal/migrations postgres $DATABASE_URL up

  migrate-down:
    desc: "Run database migrations down"
    uses: bash
    env:
      DATABASE_URL: ${DATABASE_URL}
    run: |
      goose -dir internal/migrations postgres $DATABASE_URL down

  seed:
    desc: "Seed the database with initial data"
    uses: bash
    env:
      DATABASE_URL: ${DATABASE_URL}
    run: go run ./cmd/seed
```

## Key Principles

1. **Prefer `internal/`** - Keep implementation details private
2. **Use stores, not repositories** - Simpler naming, same concept
3. **Flat is better than nested** - Don't over-organize
4. **Interfaces at the consumer** - Define interfaces where they're used
5. **SQL migrations** - Use versioned `.up.sql` and `.down.sql` files
6. **Minimal cmd/** - Entry points should only wire dependencies
7. **Configuration via environment** - 12-factor app principles
