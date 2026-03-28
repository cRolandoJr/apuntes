# Autenticación y Autorización en Go

La seguridad en una API tiene dos capas distintas:

- **Autenticación** (AuthN): "¿Quién sos?" → JWT, sesiones
- **Autorización** (AuthZ): "¿Qué podés hacer?" → RBAC, permisos

---

## JWT — JSON Web Tokens

Un JWT es un token con tres partes: `header.payload.signature`

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJ1c2VySWQiOiI0MiIsInJvbGUiOiJhZG1pbiIsImV4cCI6MTcxMjAwMDAwMH0.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### Instalar

```bash
go get github.com/golang-jwt/jwt/v5
```

### Token Service — capa de dominio

```go
// internal/domain/token.go

package domain

import "time"

type Claims struct {
    UserID string
    Email  string
    Role   Role
}

type TokenPair struct {
    AccessToken  string
    RefreshToken string
    ExpiresAt    time.Time
}

// La interfaz vive en el dominio — desacoplada de la implementación concreta
type TokenService interface {
    Generate(claims Claims) (*TokenPair, error)
    Validate(token string) (*Claims, error)
    Refresh(refreshToken string) (*TokenPair, error)
}

type Role string
const (
    RoleAdmin  Role = "admin"
    RoleUser   Role = "user"
    RoleViewer Role = "viewer"
)

func (r Role) HasPermission(p Permission) bool {
    switch r {
    case RoleAdmin:
        return true
    case RoleUser:
        return p == PermRead || p == PermWrite
    case RoleViewer:
        return p == PermRead
    }
    return false
}

type Permission string
const (
    PermRead   Permission = "read"
    PermWrite  Permission = "write"
    PermDelete Permission = "delete"
)
```

### Implementación JWT

```go
// internal/infra/auth/jwt_token_service.go

package auth

import (
    "errors"
    "time"

    "github.com/golang-jwt/jwt/v5"
    "gestion_productos/internal/domain"
)

type JWTConfig struct {
    AccessSecret  string
    RefreshSecret string
    AccessTTL     time.Duration // típicamente 15 minutos
    RefreshTTL    time.Duration // típicamente 7-30 días
}

type jwtTokenService struct {
    cfg JWTConfig
}

func NewJWTTokenService(cfg JWTConfig) domain.TokenService {
    return &jwtTokenService{cfg: cfg}
}

// Claims internas del JWT (extienden jwt.RegisteredClaims)
type jwtClaims struct {
    UserID string      `json:"userId"`
    Email  string      `json:"email"`
    Role   domain.Role `json:"role"`
    jwt.RegisteredClaims
}

func (s *jwtTokenService) Generate(claims domain.Claims) (*domain.TokenPair, error) {
    now := time.Now()

    accessClaims := jwtClaims{
        UserID: claims.UserID,
        Email:  claims.Email,
        Role:   claims.Role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(now.Add(s.cfg.AccessTTL)),
            IssuedAt:  jwt.NewNumericDate(now),
            Subject:   claims.UserID,
        },
    }

    accessToken, err := jwt.NewWithClaims(jwt.SigningMethodHS256, accessClaims).
        SignedString([]byte(s.cfg.AccessSecret))
    if err != nil {
        return nil, fmt.Errorf("firmar access token: %w", err)
    }

    refreshToken, err := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.RegisteredClaims{
        ExpiresAt: jwt.NewNumericDate(now.Add(s.cfg.RefreshTTL)),
        IssuedAt:  jwt.NewNumericDate(now),
        Subject:   claims.UserID,
    }).SignedString([]byte(s.cfg.RefreshSecret))
    if err != nil {
        return nil, fmt.Errorf("firmar refresh token: %w", err)
    }

    return &domain.TokenPair{
        AccessToken:  accessToken,
        RefreshToken: refreshToken,
        ExpiresAt:    now.Add(s.cfg.AccessTTL),
    }, nil
}

func (s *jwtTokenService) Validate(tokenStr string) (*domain.Claims, error) {
    token, err := jwt.ParseWithClaims(tokenStr, &jwtClaims{}, func(t *jwt.Token) (interface{}, error) {
        // CRÍTICO: verificar el algoritmo esperado
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("algoritmo inesperado: %v", t.Header["alg"])
        }
        return []byte(s.cfg.AccessSecret), nil
    })

    if err != nil {
        if errors.Is(err, jwt.ErrTokenExpired) {
            return nil, domain.NewUnauthorizedError("token expirado")
        }
        return nil, domain.NewUnauthorizedError("token inválido")
    }

    claims, ok := token.Claims.(*jwtClaims)
    if !ok || !token.Valid {
        return nil, domain.NewUnauthorizedError("token malformado")
    }

    return &domain.Claims{
        UserID: claims.UserID,
        Email:  claims.Email,
        Role:   claims.Role,
    }, nil
}
```

---

## Middleware de autenticación

```go
// internal/delivery/http/middleware/auth.go

package middleware

type contextKey string
const claimsContextKey contextKey = "claims"

func AuthMiddleware(tokenSvc domain.TokenService) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            authHeader := r.Header.Get("Authorization")
            if authHeader == "" {
                http.Error(w, `{"error":"autenticación requerida"}`, http.StatusUnauthorized)
                return
            }

            // Formato esperado: "Bearer token"
            parts := strings.SplitN(authHeader, " ", 2)
            if len(parts) != 2 || !strings.EqualFold(parts[0], "bearer") {
                http.Error(w, `{"error":"formato de token inválido"}`, http.StatusUnauthorized)
                return
            }

            claims, err := tokenSvc.Validate(parts[1])
            if err != nil {
                var appErr *domain.AppError
                if errors.As(err, &appErr) {
                    http.Error(w, appErr.JSON(), appErr.HTTPStatus())
                } else {
                    http.Error(w, `{"error":"token inválido"}`, http.StatusUnauthorized)
                }
                return
            }

            ctx := context.WithValue(r.Context(), claimsContextKey, claims)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// OptionalAuth — inyecta claims si hay token, pero no falla si no hay
func OptionalAuthMiddleware(tokenSvc domain.TokenService) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if authHeader := r.Header.Get("Authorization"); authHeader != "" {
                parts := strings.SplitN(authHeader, " ", 2)
                if len(parts) == 2 {
                    if claims, err := tokenSvc.Validate(parts[1]); err == nil {
                        r = r.WithContext(context.WithValue(r.Context(), claimsContextKey, claims))
                    }
                }
            }
            next.ServeHTTP(w, r)
        })
    }
}

// Helper para obtener claims en handlers
func GetClaims(ctx context.Context) (*domain.Claims, bool) {
    claims, ok := ctx.Value(claimsContextKey).(*domain.Claims)
    return claims, ok
}

// MustGetClaims — sólo usar después de AuthMiddleware
func MustGetClaims(ctx context.Context) *domain.Claims {
    claims, ok := GetClaims(ctx)
    if !ok {
        panic("MustGetClaims: no hay claims en el contexto (¿falta AuthMiddleware?)")
    }
    return claims
}
```

---

## Autorización — RBAC middleware

```go
// RequirePermission — middleware para rutas específicas
func RequirePermission(perm domain.Permission) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            claims, ok := GetClaims(r.Context())
            if !ok {
                http.Error(w, `{"error":"no autenticado"}`, http.StatusUnauthorized)
                return
            }

            if !claims.Role.HasPermission(perm) {
                http.Error(w, `{"error":"permisos insuficientes"}`, http.StatusForbidden)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}

// Uso en rutas
func setupRoutes(r chi.Router, authMid, ...) {
    r.Group(func(r chi.Router) {
        r.Use(AuthMiddleware(tokenSvc))

        r.Get("/products", handler.ListProducts)    // solo autenticado

        r.Group(func(r chi.Router) {
            r.Use(RequirePermission(domain.PermWrite))
            r.Post("/products", handler.CreateProduct)    // autenticado + write
            r.Put("/products/{id}", handler.UpdateProduct)
        })

        r.Group(func(r chi.Router) {
            r.Use(RequirePermission(domain.PermDelete))
            r.Delete("/products/{id}", handler.DeleteProduct) // autenticado + delete
        })
    })
}
```

---

## Password hashing con bcrypt

```go
import "golang.org/x/crypto/bcrypt"

const bcryptCost = 12 // recomendado: 12 en producción, bcrypt.MinCost en tests

func HashPassword(password string) (string, error) {
    // bcrypt incluye el salt internamente — nunca hashes sin salt
    hash, err := bcrypt.GenerateFromPassword([]byte(password), bcryptCost)
    if err != nil {
        return "", fmt.Errorf("hashear password: %w", err)
    }
    return string(hash), nil
}

func CheckPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}
```

---

## Refresh tokens — flujo completo

```go
// Guardar refresh tokens en base de datos (no solo en el cliente)
type RefreshTokenRepository interface {
    Save(ctx context.Context, userID string, token string, expiresAt time.Time) error
    Revoke(ctx context.Context, token string) error
    IsValid(ctx context.Context, token string) (string, error) // retorna userID
}

func (uc *authUseCase) Refresh(ctx context.Context, refreshToken string) (*domain.TokenPair, error) {
    // 1. Verificar firma del token
    userID, err := uc.tokenSvc.Refresh(refreshToken)
    if err != nil {
        return nil, domain.NewUnauthorizedError("refresh token inválido")
    }

    // 2. Verificar que no fue revocado (si tenés blacklist/whitelist)
    storedUserID, err := uc.refreshRepo.IsValid(ctx, refreshToken)
    if err != nil || storedUserID != userID {
        return nil, domain.NewUnauthorizedError("refresh token revocado")
    }

    // 3. Revocar el refresh token usado (rotación)
    _ = uc.refreshRepo.Revoke(ctx, refreshToken)

    // 4. Generar nuevo par de tokens
    user, err := uc.userRepo.GetByID(ctx, userID)
    if err != nil {
        return nil, err
    }

    return uc.tokenSvc.Generate(domain.Claims{
        UserID: user.ID,
        Email:  user.Email,
        Role:   user.Role,
    })
}
```

---

## Práctica: Novato vs Profesional

### Novato

```go
// Verificar el token en cada handler — repetición + no reutilizable
func (h *Handler) CreateProduct(w http.ResponseWriter, r *http.Request) {
    token := r.Header.Get("Authorization")
    if token != "mi-token-secreto" {  // hardcoded, sin firma
        http.Error(w, "no autorizado", 401)
        return
    }
    // ...
}
```

### Profesional

```go
// Token validado una vez en middleware — handlers reciben claims ya verificados
func (h *Handler) CreateProduct(w http.ResponseWriter, r *http.Request) {
    claims := middleware.MustGetClaims(r.Context())  // ya validado por middleware

    // Auditoría: saber quién hizo qué
    ctx := context.WithValue(r.Context(), "actorId", claims.UserID)

    result, err := h.productUC.Create(ctx, input)
    // ...
}
```
