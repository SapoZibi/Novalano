## [返回](/Doce/ZH-TW/Doce.md)

# 🔧 後端工程設計

> **語言**：Go 1.22+  
> **框架**：Gin v1.9+  
> **數據庫驅動**：pgx v5  
> **節點間通信**：gRPC + Protocol Buffers  

---

## 主程序入口

### `cmd/node/main.go`

```go
package main

import (
    "context"
    "log"
    "os"
    "os/signal"
    "syscall"

    "github.com/Sapozibi/Novalano/internal/api"
    "github.com/Sapozibi/Novalano/internal/config"
    "github.com/Sapozibi/Novalano/internal/db"
    "github.com/Sapozibi/Novalano/internal/geoip"
    "github.com/Sapozibi/Novalano/internal/node"
    "github.com/Sapozibi/Novalano/internal/raft"
    "github.com/Sapozibi/Novalano/internal/smtp"
    "github.com/Sapozibi/Novalano/internal/ssl"
)

func main() {
    // 1. 加載配置（從 /data/node.json 和環境變量）
    cfg, err := config.Load()
    if err != nil {
        log.Fatalf("配置加載失敗: %v", err)
    }

    // 2. 初始化數據庫連接池
    pool, err := db.NewPool(cfg.DatabaseURL)
    if err != nil {
        log.Fatalf("數據庫連接失敗: %v", err)
    }
    defer pool.Close()

    // 3. 運行數據庫遷移
    if err := db.Migrate(pool); err != nil {
        log.Fatalf("數據庫遷移失敗: %v", err)
    }

    // 4. 初始化 GeoIP
    geo, err := geoip.New(cfg.GeoIPDBPath)
    if err != nil {
        log.Fatalf("GeoIP 初始化失敗: %v", err)
    }

    // 5. 初始化 SSL 管理器
    sslManager, err := ssl.NewManager(cfg.SSLConfig)
    if err != nil {
        log.Fatalf("SSL 管理器初始化失敗: %v", err)
    }

    // 6. 初始化節點管理器
    nodeManager, err := node.NewManager(cfg, pool, geo)
    if err != nil {
        log.Fatalf("節點管理器初始化失敗: %v", err)
    }

    // 7. 初始化 Raft（多節點模式下）
    var raftNode *raft.Node
    if cfg.Mode == config.ModeMulti {
        raftNode, err = raft.NewNode(cfg, pool, nodeManager)
        if err != nil {
            log.Fatalf("Raft 初始化失敗: %v", err)
        }
        go raftNode.Run()
    }

    // 8. 初始化 SMTP 隊列（Leader 才啟動發送 goroutine）
    smtpWorker := smtp.NewWorker(pool, cfg.SMTPConfig)
    if raftNode == nil || raftNode.IsLeader() {
        go smtpWorker.Start()
    }
    // 監聽 Leader 變化，動態啟停
    if raftNode != nil {
        go smtpWorker.WatchLeadership(raftNode)
    }

    // 9. 啟動 HTTP API 服務器
    router := api.NewRouter(api.Dependencies{
        DB:          pool,
        GeoIP:       geo,
        NodeManager: nodeManager,
        Raft:        raftNode,
        SMTPWorker:  smtpWorker,
        Config:      cfg,
    })

    // 10. 優雅關閉
    ctx, stop := signal.NotifyContext(context.Background(),
        os.Interrupt, syscall.SIGTERM)
    defer stop()

    go func() {
        if err := router.Run(":8080"); err != nil {
            log.Printf("HTTP 服務器錯誤: %v", err)
        }
    }()

    <-ctx.Done()
    log.Println("正在優雅關閉...")
    // 清理資源
}
```

---

## 配置模塊

### `internal/config/config.go`

```go
package config

import (
    "encoding/json"
    "os"
)

type Mode string

const (
    ModeSingle Mode = "single"
    ModeMulti  Mode = "multi"
)

type SSLMode string

const (
    SSLAuto     SSLMode = "auto"
    SSLDNS      SSLMode = "dns"
    SSLSystemCA SSLMode = "system_ca"
    SSLManual   SSLMode = "manual"
)

// Config 是節點的完整配置結構體
type Config struct {
    // 節點基本信息（來自 /data/node.json，永久不變）
    NodeID   string `json:"id"`
    NodeType string `json:"type"` // "capability" | "backup"

    // 運行時配置（來自數據庫 system_settings 表）
    Mode        Mode   `json:"mode"`
    DatabaseURL string `json:"database_url"`
    GeoIPDBPath string `json:"geoip_db_path"`

    // 功能開關（來自數據庫 nodes 表）
    StorageEnabled  bool  `json:"storage_enabled"`
    StorageLimit    int64 `json:"storage_limit_bytes"`
    CacheEnabled    bool  `json:"cache_enabled"`
    CacheLimit      int64 `json:"cache_limit_bytes"`
    FrontendEnabled bool  `json:"frontend_enabled"`

    // SSL 配置
    SSLConfig SSLConfig `json:"ssl"`

    // SMTP 配置（Leader 使用）
    SMTPConfig SMTPConfig `json:"smtp"`
}

type SSLConfig struct {
    Mode       SSLMode `json:"mode"`
    Domain     string  `json:"domain"`
    Email      string  `json:"email"` // Let's Encrypt 通知郵箱
    DNSProvider string `json:"dns_provider"` // "cloudflare" | "dnspod" | ...
    DNSAPIToken string `json:"dns_api_token"`
    CertPath   string  `json:"cert_path"` // 手動模式
    KeyPath    string  `json:"key_path"`  // 手動模式
}

type SMTPConfig struct {
    Host     string `json:"host"`
    Port     int    `json:"port"`
    User     string `json:"user"`
    Password string `json:"password"` // 加密存儲，加載時解密
    From     string `json:"from"`
    TLS      bool   `json:"tls"`
}

// Load 從 /data/node.json 加載節點基本配置
// 其他配置在數據庫初始化後從數據庫讀取
func Load() (*Config, error) {
    data, err := os.ReadFile("/data/node.json")
    if err != nil {
        return nil, err
    }
    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, err
    }
    // 數據庫 URL 從環境變量讀取（Docker Compose 注入）
    cfg.DatabaseURL = os.Getenv("DATABASE_URL")
    return &cfg, nil
}
```

---

## 數據庫模塊

### `internal/db/db.go`

```go
package db

import (
    "context"

    "github.com/jackc/pgx/v5/pgxpool"
)

// NewPool 創建 PostgreSQL 連接池
// 連接池配置：最大 25 個連接，最小 5 個空閒連接
func NewPool(databaseURL string) (*pgxpool.Pool, error) {
    config, err := pgxpool.ParseConfig(databaseURL)
    if err != nil {
        return nil, err
    }

    // 連接池參數
    config.MaxConns = 25
    config.MinConns = 5

    pool, err := pgxpool.NewWithConfig(context.Background(), config)
    if err != nil {
        return nil, err
    }

    // 測試連接
    if err := pool.Ping(context.Background()); err != nil {
        return nil, err
    }

    return pool, nil
}
```

### `internal/db/queries/users.go`

```go
package queries

import (
    "context"
    "time"

    "github.com/google/uuid"
    "github.com/jackc/pgx/v5/pgxpool"
)

// User 對應 users 表的結構體
type User struct {
    ID            uuid.UUID  `db:"id"`
    Username      string     `db:"username"`
    Email         string     `db:"email"`
    PasswordHash  string     `db:"password_hash"`
    Role          string     `db:"role"`
    Status        string     `db:"status"`
    EmailVerified bool       `db:"email_verified"`
    GroupID       *uuid.UUID `db:"group_id"`
    CreatedAt     time.Time  `db:"created_at"`
    LastLoginAt   *time.Time `db:"last_login_at"`
    LastLoginIP   *string    `db:"last_login_ip"`
}

type UserQueries struct {
    pool *pgxpool.Pool
}

func NewUserQueries(pool *pgxpool.Pool) *UserQueries {
    return &UserQueries{pool: pool}
}

// CreateUser 創建新用戶
func (q *UserQueries) CreateUser(ctx context.Context, username, email, passwordHash, role string) (*User, error) {
    const sql = `
        INSERT INTO users (id, username, email, password_hash, role, status, email_verified, created_at)
        VALUES ($1, $2, $3, $4, $5, 'active', false, NOW())
        RETURNING *`

    user := &User{}
    row := q.pool.QueryRow(ctx, sql, uuid.New(), username, email, passwordHash, role)
    err := row.Scan(
        &user.ID, &user.Username, &user.Email, &user.PasswordHash,
        &user.Role, &user.Status, &user.EmailVerified, &user.GroupID,
        &user.CreatedAt, &user.LastLoginAt, &user.LastLoginIP,
    )
    return user, err
}

// GetUserByEmail 通過郵箱查找用戶
func (q *UserQueries) GetUserByEmail(ctx context.Context, email string) (*User, error) {
    const sql = `SELECT * FROM users WHERE email = $1 AND status = 'active'`
    user := &User{}
    row := q.pool.QueryRow(ctx, sql, email)
    err := row.Scan(
        &user.ID, &user.Username, &user.Email, &user.PasswordHash,
        &user.Role, &user.Status, &user.EmailVerified, &user.GroupID,
        &user.CreatedAt, &user.LastLoginAt, &user.LastLoginIP,
    )
    if err != nil {
        return nil, err
    }
    return user, nil
}

// CountUsers 統計用戶總數（用於判斷是否為第一個用戶）
func (q *UserQueries) CountUsers(ctx context.Context) (int64, error) {
    var count int64
    err := q.pool.QueryRow(ctx, `SELECT COUNT(*) FROM users`).Scan(&count)
    return count, err
}

// UpdateLastLogin 更新最後登錄信息
func (q *UserQueries) UpdateLastLogin(ctx context.Context, userID uuid.UUID, ip string) error {
    const sql = `
        UPDATE users 
        SET last_login_at = NOW(), last_login_ip = $2
        WHERE id = $1`
    _, err := q.pool.Exec(ctx, sql, userID, ip)
    return err
}
```

---

## API 路由設計

### `internal/api/router.go`

```go
package api

import (
    "github.com/gin-gonic/gin"
    "github.com/Sapozibi/Novalano/internal/api/handlers"
    "github.com/Sapozibi/Novalano/internal/api/middleware"
)

type Dependencies struct {
    DB          interface{} // *pgxpool.Pool
    GeoIP       interface{} // *geoip.GeoIP
    NodeManager interface{} // *node.Manager
    Raft        interface{} // *raft.Node（可為 nil）
    SMTPWorker  interface{} // *smtp.Worker
    Config      interface{} // *config.Config
}

func NewRouter(deps Dependencies) *gin.Engine {
    r := gin.New()

    // 全局中間件
    r.Use(gin.Recovery())
    r.Use(middleware.Logger())
    r.Use(middleware.CORS())
    r.Use(middleware.IPLog(deps.DB))

    // ═══════════════════════════════════════
    // 公開路由（無需認證）
    // ═══════════════════════════════════════
    public := r.Group("/api")
    {
        // 探測頁相關
        public.POST("/probe/best-node", handlers.BestNode(deps))
        public.GET("/probe/my-ip", handlers.MyIP())

        // 認證
        auth := public.Group("/auth")
        {
            auth.POST("/login", middleware.RateLimit(10), handlers.Login(deps))
            auth.POST("/register", middleware.RateLimit(5), handlers.Register(deps))
            auth.POST("/forgot-password", middleware.RateLimit(3), handlers.ForgotPassword(deps))
            auth.POST("/reset-password", handlers.ResetPassword(deps))
            auth.GET("/verify-email", handlers.VerifyEmail(deps))
            auth.POST("/refresh", handlers.RefreshToken(deps))
        }

        // 分享頁面（訪客訪問）
        share := public.Group("/share")
        {
            share.GET("/:slug", handlers.GetShare(deps))
            share.POST("/:slug/auth", handlers.AuthShare(deps))
            share.GET("/:slug/files", handlers.ListShareFiles(deps))
            share.GET("/:slug/download/:file_id", handlers.DownloadShareFile(deps))
            share.GET("/:slug/download-dir/:dir_id", handlers.DownloadShareDir(deps)) // ZIP
        }

        // 系統初始化（僅在用戶數為 0 時可用）
        public.POST("/init", handlers.InitSystem(deps))
    }

    // ═══════════════════════════════════════
    // 需要認證的路由
    // ═══════════════════════════════════════
    authed := r.Group("/api", middleware.Auth(deps))
    {
        // 文件系統
        files := authed.Group("/files")
        {
            files.GET("/", handlers.ListFiles(deps))
            files.GET("/:id", handlers.GetFile(deps))
            files.DELETE("/:id", handlers.DeleteFile(deps))
            files.PATCH("/:id", handlers.UpdateFile(deps))    // 重命名
            files.POST("/:id/move", handlers.MoveFile(deps))
            files.POST("/:id/copy", handlers.CopyFile(deps))
        }

        // 目錄
        dirs := authed.Group("/dirs")
        {
            dirs.GET("/", handlers.ListDir(deps))
            dirs.POST("/", handlers.CreateDir(deps))
            dirs.DELETE("/:id", handlers.DeleteDir(deps))
            dirs.PATCH("/:id", handlers.UpdateDir(deps))
        }

        // 上傳
        upload := authed.Group("/upload")
        {
            upload.POST("/init", handlers.InitUpload(deps))
            upload.POST("/complete", handlers.CompleteUpload(deps))
            upload.GET("/status/:session_id", handlers.UploadStatus(deps))
        }

        // 下載（獲取下載 URL，實際下載走中轉節點）
        authed.GET("/download/:file_id", handlers.GetDownloadURL(deps))

        // 分享管理
        shares := authed.Group("/shares")
        {
            shares.GET("/", handlers.ListShares(deps))
            shares.POST("/", handlers.CreateShare(deps))
            shares.GET("/:id", handlers.GetShareDetail(deps))
            shares.PATCH("/:id", handlers.UpdateShare(deps))
            shares.DELETE("/:id", handlers.DeleteShare(deps))
        }

        // 用戶設置
        settings := authed.Group("/user")
        {
            settings.GET("/profile", handlers.GetProfile(deps))
            settings.PATCH("/profile", handlers.UpdateProfile(deps))
            settings.PATCH("/password", handlers.ChangePassword(deps))
            settings.POST("/email/change", handlers.RequestEmailChange(deps))
            settings.POST("/email/verify", handlers.VerifyEmailChange(deps))
            settings.GET("/webdav-token", handlers.GetWebDAVToken(deps))
            settings.POST("/webdav-token/regenerate", handlers.RegenerateWebDAVToken(deps))
        }

        // 管理員路由
        admin := authed.Group("/admin", middleware.AdminOnly())
        {
            // 用戶管理
            adminUsers := admin.Group("/users")
            {
                adminUsers.GET("/", handlers.AdminListUsers(deps))
                adminUsers.POST("/", handlers.AdminCreateUser(deps))
                adminUsers.GET("/:id", handlers.AdminGetUser(deps))
                adminUsers.PATCH("/:id", handlers.AdminUpdateUser(deps))
                adminUsers.DELETE("/:id", handlers.AdminDeleteUser(deps))
                adminUsers.POST("/:id/block", handlers.AdminBlockUser(deps))
                adminUsers.POST("/:id/unblock", handlers.AdminUnblockUser(deps))
            }

            // 用戶組
            adminGroups := admin.Group("/groups")
            {
                adminGroups.GET("/", handlers.AdminListGroups(deps))
                adminGroups.POST("/", handlers.AdminCreateGroup(deps))
                adminGroups.PATCH("/:id", handlers.AdminUpdateGroup(deps))
                adminGroups.DELETE("/:id", handlers.AdminDeleteGroup(deps))
            }

            // 邀請碼
            adminInvites := admin.Group("/invites")
            {
                adminInvites.GET("/", handlers.AdminListInvites(deps))
                adminInvites.POST("/", handlers.AdminCreateInvite(deps))
                adminInvites.DELETE("/:id", handlers.AdminRevokeInvite(deps))
            }

            // 節點管理
            adminNodes := admin.Group("/nodes")
            {
                adminNodes.GET("/", handlers.AdminListNodes(deps))
                adminNodes.GET("/:id", handlers.AdminGetNode(deps))
                adminNodes.PATCH("/:id", handlers.AdminUpdateNode(deps))
                adminNodes.DELETE("/:id", handlers.AdminRemoveNode(deps))
                adminNodes.POST("/pairing-token", handlers.AdminGenPairingToken(deps))
                adminNodes.POST("/:id/transfer-leader", handlers.AdminTransferLeader(deps))
                adminNodes.PUT("/leader-queue", handlers.AdminUpdateLeaderQueue(deps))
                adminNodes.POST("/:id/migrate-storage", handlers.AdminMigrateStorage(deps))
            }

            // 系統設置
            admin.GET("/settings", handlers.AdminGetSettings(deps))
            admin.PUT("/settings", handlers.AdminUpdateSettings(deps))

            // IP 日誌
            adminLogs := admin.Group("/logs")
            {
                adminLogs.GET("/", handlers.AdminListLogs(deps))
                adminLogs.POST("/block-ip", handlers.AdminBlockIP(deps))
                adminLogs.POST("/unblock-ip", handlers.AdminUnblockIP(deps))
            }
        }
    }

    // ═══════════════════════════════════════
    // 中轉節點路由（節點間通信，需要節點密鑰）
    // ═══════════════════════════════════════
    relay := r.Group("/relay", middleware.NodeAuth())
    {
        relay.PUT("/upload/:session_id/chunk/:index", handlers.RelayReceiveChunk(deps))
        relay.GET("/download/:file_id", handlers.RelayServeFile(deps))
    }

    // WebDAV
    r.Any("/dav/*path", middleware.WebDAVAuth(deps), handlers.WebDAV(deps))

    // 前端靜態文件
    if true { // cfg.FrontendEnabled
        r.Static("/assets", "/data/frontend/app/assets")
        r.StaticFile("/", "/data/frontend/probe.html")        // 探測頁
        r.NoRoute(func(c *gin.Context) {
            // SPA 路由：所有未匹配的請求返回 app/index.html
            c.File("/data/frontend/app/index.html")
        })
    }

    return r
}
```

---

## 認證模塊

### `internal/api/handlers/auth.go`

```go
package handlers

import (
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
    "golang.org/x/crypto/bcrypt"
)

// POST /api/auth/login
// 請求體：{ "email": "user@example.com", "password": "..." }
// 成功返回：{ "access_token": "...", "refresh_token": "...", "user": {...} }
func Login(deps Dependencies) gin.HandlerFunc {
    return func(c *gin.Context) {
        var req struct {
            Email    string `json:"email" binding:"required,email"`
            Password string `json:"password" binding:"required"`
        }
        if err := c.ShouldBindJSON(&req); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": "參數無效"})
            return
        }

        // 查找用戶
        userQ := queries.NewUserQueries(deps.DB)
        user, err := userQ.GetUserByEmail(c.Request.Context(), req.Email)
        if err != nil {
            // 故意不區分「用戶不存在」和「密碼錯誤」，防止用戶枚舉
            c.JSON(http.StatusUnauthorized, gin.H{"error": "郵箱或密碼錯誤"})
            return
        }

        // 驗證密碼
        if err := bcrypt.CompareHashAndPassword(
            []byte(user.PasswordHash), []byte(req.Password)); err != nil {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "郵箱或密碼錯誤"})
            return
        }

        // 檢查郵箱是否已驗證（如果系統要求）
        if !user.EmailVerified && isEmailVerificationRequired(deps) {
            c.JSON(http.StatusForbidden, gin.H{"error": "請先驗證郵箱"})
            return
        }

        // 更新最後登錄信息
        userQ.UpdateLastLogin(c.Request.Context(), user.ID, c.ClientIP())

        // 生成 JWT
        accessToken, err := auth.GenerateAccessToken(user.ID, user.Role)
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "生成 Token 失敗"})
            return
        }
        refreshToken, err := auth.GenerateRefreshToken(user.ID)
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "生成 Token 失敗"})
            return
        }

        c.JSON(http.StatusOK, gin.H{
            "access_token":  accessToken,
            "refresh_token": refreshToken,
            "expires_in":    86400, // 24 小時
            "user": gin.H{
                "id":       user.ID,
                "username": user.Username,
                "email":    user.Email,
                "role":     user.Role,
            },
        })
    }
}

// POST /api/init
// 僅在 users 表為空時可用，創建第一個管理員帳號
func InitSystem(deps Dependencies) gin.HandlerFunc {
    return func(c *gin.Context) {
        userQ := queries.NewUserQueries(deps.DB)

        // 確認系統尚無用戶
        count, err := userQ.CountUsers(c.Request.Context())
        if err != nil || count > 0 {
            c.JSON(http.StatusForbidden, gin.H{"error": "系統已初始化"})
            return
        }

        var req struct {
            Username string `json:"username" binding:"required,min=3,max=32"`
            Email    string `json:"email" binding:"required,email"`
            Password string `json:"password" binding:"required,min=8"`
        }
        if err := c.ShouldBindJSON(&req); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }

        // 密碼 hash
        hash, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "密碼處理失敗"})
            return
        }

        // 創建管理員帳號（role = "admin"）
        user, err := userQ.CreateUser(c.Request.Context(),
            req.Username, req.Email, string(hash), "admin")
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "創建用戶失敗"})
            return
        }

        // 立即標記郵箱為已驗證（第一個用戶無需驗證）
        userQ.VerifyEmail(c.Request.Context(), user.ID)

        c.JSON(http.StatusCreated, gin.H{
            "message": "管理員帳號創建成功",
            "user":    user,
        })
    }
}
```

### `internal/auth/jwt.go`

```go
package auth

import (
    "time"

    "github.com/golang-jwt/jwt/v5"
    "github.com/google/uuid"
)

var (
    accessSecret  = []byte(mustGetEnv("JWT_ACCESS_SECRET"))
    refreshSecret = []byte(mustGetEnv("JWT_REFRESH_SECRET"))
)

type Claims struct {
    UserID uuid.UUID `json:"user_id"`
    Role   string    `json:"role"`
    jwt.RegisteredClaims
}

// GenerateAccessToken 生成 24 小時有效的訪問 Token
func GenerateAccessToken(userID uuid.UUID, role string) (string, error) {
    claims := Claims{
        UserID: userID,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "Novalano",
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(accessSecret)
}

// ValidateAccessToken 驗證並解析 Token
func ValidateAccessToken(tokenStr string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenStr, &Claims{},
        func(token *jwt.Token) (interface{}, error) {
            if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, jwt.ErrSignatureInvalid
            }
            return accessSecret, nil
        })
    if err != nil {
        return nil, err
    }
    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, jwt.ErrTokenInvalid
    }
    return claims, nil
}
```

---

## 文件上傳模塊

### `internal/api/handlers/upload.go`

```go
package handlers

import (
    "crypto/sha256"
    "encoding/hex"
    "io"
    "net/http"

    "github.com/gin-gonic/gin"
)

// POST /api/upload/init
// 客戶端先計算文件 SHA-256 hash，發送文件信息
// 返回上傳會話 ID 和目標節點 URL
//
// 請求體：
// {
//   "filename": "photo.jpg",
//   "size": 10485760,
//   "hash": "abc123...",   // 客戶端預計算的 SHA-256
//   "mime_type": "image/jpeg",
//   "parent_dir_id": "uuid 或 null（根目錄）"
// }
//
// 響應：
// {
//   "session_id": "uuid",
//   "upload_node_url": "https://jp.example.com",
//   "chunk_size": 16777216,  // 16MB
//   "chunks_count": 1,
//   "deduplicated": false    // 如果為 true，文件已存在，無需上傳
// }
func InitUpload(deps Dependencies) gin.HandlerFunc {
    return func(c *gin.Context) {
        userID := c.MustGet("user_id").(uuid.UUID)

        var req struct {
            Filename    string     `json:"filename" binding:"required"`
            Size        int64      `json:"size" binding:"required,min=1"`
            Hash        string     `json:"hash" binding:"required,len=64"` // SHA-256 hex
            MimeType    string     `json:"mime_type"`
            ParentDirID *uuid.UUID `json:"parent_dir_id"`
        }
        if err := c.ShouldBindJSON(&req); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }

        ctx := c.Request.Context()
        fileQ := queries.NewFileQueries(deps.DB)

        // 1. 去重檢查（如果系統開啟去重）
        if deps.Config.DedupEnabled {
            existing, err := fileQ.GetFileByHash(ctx, req.Hash)
            if err == nil && existing != nil {
                // 文件已存在，直接創建引用記錄
                newFile, err := fileQ.CreateFileRef(ctx, queries.CreateFileRefParams{
                    OwnerID:     userID,
                    FileHash:    req.Hash,
                    Filename:    req.Filename,
                    ParentDirID: req.ParentDirID,
                    SourceFileID: existing.ID,
                })
                if err != nil {
                    c.JSON(http.StatusInternalServerError, gin.H{"error": "創建文件記錄失敗"})
                    return
                }
                c.JSON(http.StatusOK, gin.H{
                    "deduplicated": true,
                    "file": newFile,
                })
                return
            }
        }

        // 2. 配額檢查
        quotaQ := queries.NewQuotaQueries(deps.DB)
        quota, _ := quotaQ.GetUserQuota(ctx, userID)
        if quota.StorageLimit > 0 && quota.StorageUsed+req.Size > quota.StorageLimit {
            c.JSON(http.StatusForbidden, gin.H{"error": "存儲空間不足"})
            return
        }

        // 3. 計算分塊大小和數量
        chunkSize := int64(deps.Config.ChunkSizeMB) * 1024 * 1024 // 默認 16MB
        chunksCount := (req.Size + chunkSize - 1) / chunkSize

        // 4. 選擇最優中轉節點（GeoIP 就近）
        userIP := getClientIP(c)
        uploadNode, err := deps.NodeManager.GetBestRelayNode(ctx, userIP)
        if err != nil {
            c.JSON(http.StatusServiceUnavailable, gin.H{"error": "無可用節點"})
            return
        }

        // 5. 選擇目標存儲節點
        storageNode, err := deps.NodeManager.GetBestStorageNode(ctx, userIP)
        if err != nil {
            c.JSON(http.StatusInsufficientStorage, gin.H{
                "error": "所有存儲節點已滿，無法上傳文件",
                "code":  "STORAGE_FULL",
            })
            return
        }

        // 6. 創建上傳會話
        sessionID := uuid.New()
        err = fileQ.CreateUploadSession(ctx, queries.CreateUploadSessionParams{
            SessionID:     sessionID,
            UserID:        userID,
            Filename:      req.Filename,
            Size:          req.Size,
            Hash:          req.Hash,
            MimeType:      req.MimeType,
            ParentDirID:   req.ParentDirID,
            ChunksCount:   int(chunksCount),
            ChunkSize:     chunkSize,
            RelayNodeID:   uploadNode.ID,
            StorageNodeID: storageNode.ID,
        })
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "創建上傳會話失敗"})
            return
        }

        c.JSON(http.StatusOK, gin.H{
            "session_id":      sessionID,
            "upload_node_url": uploadNode.Address,
            "chunk_size":      chunkSize,
            "chunks_count":    chunksCount,
            "deduplicated":    false,
        })
    }
}

// PUT /relay/upload/:session_id/chunk/:index
// 這個端點在中轉節點上執行，接收文件分塊
// 注意：此路由需要節點密鑰認證（middleware.NodeAuth）
func RelayReceiveChunk(deps Dependencies) gin.HandlerFunc {
    return func(c *gin.Context) {
        sessionID, err := uuid.Parse(c.Param("session_id"))
        if err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": "無效的 session_id"})
            return
        }
        chunkIndex, err := strconv.Atoi(c.Param("index"))
        if err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": "無效的 chunk index"})
            return
        }

        // 讀取分塊數據（流式，不一次性加載到記憶體）
        // 限制單塊最大 256MB
        limitedReader := http.MaxBytesReader(c.Writer, c.Request.Body, 256*1024*1024)
        chunkData, err := io.ReadAll(limitedReader)
        if err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": "讀取數據失敗"})
            return
        }

        // 計算此塊的 SHA-256（驗證完整性）
        hash := sha256.Sum256(chunkData)
        chunkHash := hex.EncodeToString(hash[:])

        // 臨時存儲分塊（到本地臨時目錄）
        tmpPath := fmt.Sprintf("/tmp/nova/%s/chunk_%d", sessionID, chunkIndex)
        if err := os.MkdirAll(filepath.Dir(tmpPath), 0755); err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "存儲失敗"})
            return
        }
        if err := os.WriteFile(tmpPath, chunkData, 0644); err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "存儲失敗"})
            return
        }

        // 更新分塊狀態
        fileQ := queries.NewFileQueries(deps.DB)
        err = fileQ.UpdateChunkStatus(c.Request.Context(), queries.UpdateChunkParams{
            SessionID:  sessionID,
            ChunkIndex: chunkIndex,
            ChunkHash:  chunkHash,
            Status:     "uploaded",
        })
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "更新狀態失敗"})
            return
        }

        c.JSON(http.StatusOK, gin.H{
            "chunk_index": chunkIndex,
            "chunk_hash":  chunkHash,
            "status":      "uploaded",
        })
    }
}

// POST /api/upload/complete
// 所有分塊上傳完成後調用
func CompleteUpload(deps Dependencies) gin.HandlerFunc {
    return func(c *gin.Context) {
        var req struct {
            SessionID uuid.UUID `json:"session_id" binding:"required"`
        }
        if err := c.ShouldBindJSON(&req); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }

        ctx := c.Request.Context()

        // 1. 驗證所有分塊已上傳
        fileQ := queries.NewFileQueries(deps.DB)
        session, err := fileQ.GetUploadSession(ctx, req.SessionID)
        if err != nil {
            c.JSON(http.StatusNotFound, gin.H{"error": "上傳會話不存在"})
            return
        }

        chunks, err := fileQ.GetUploadedChunks(ctx, req.SessionID)
        if err != nil || len(chunks) != session.ChunksCount {
            c.JSON(http.StatusBadRequest, gin.H{
                "error":           "部分分塊未上傳完成",
                "uploaded_chunks": len(chunks),
                "total_chunks":    session.ChunksCount,
            })
            return
        }

        // 2. 在後台異步：合併分塊、推送至存儲節點、驗證完整文件 hash
        // 立即返回「處理中」，前端輪詢完成狀態
        go func() {
            if err := deps.Relay.MergeAndPushToStorage(ctx, session); err != nil {
                fileQ.UpdateSessionStatus(ctx, req.SessionID, "failed", err.Error())
                return
            }
            fileQ.UpdateSessionStatus(ctx, req.SessionID, "completed", "")
        }()

        c.JSON(http.StatusAccepted, gin.H{
            "status":     "processing",
            "session_id": req.SessionID,
            "message":    "文件合併中，請稍後查詢完成狀態",
        })
    }
}
```

---

## 下載模塊

### `internal/api/handlers/download.go`

```go
package handlers

import "net/http"

// GET /api/download/:file_id
// 返回最優下載 URL（帶臨時簽名 Token）
// 前端拿到 URL 後直接請求，不再過 API 服務器
//
// 響應：
// {
//   "download_url": "https://node.example.com/relay/download/uuid?token=xxx",
//   "filename": "photo.jpg",
//   "size": 10485760,
//   "expires_in": 3600
// }
func GetDownloadURL(deps Dependencies) gin.HandlerFunc {
    return func(c *gin.Context) {
        fileID, err := uuid.Parse(c.Param("file_id"))
        if err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": "無效的文件 ID"})
            return
        }

        userID := c.MustGet("user_id").(uuid.UUID)
        ctx := c.Request.Context()

        // 1. 查詢文件信息和存儲節點
        fileQ := queries.NewFileQueries(deps.DB)
        file, err := fileQ.GetFileWithNode(ctx, fileID, userID)
        if err != nil {
            c.JSON(http.StatusNotFound, gin.H{"error": "文件不存在"})
            return
        }

        // 2. 選擇最優下載節點（GeoIP 計算）
        userIP := getClientIP(c)
        bestNode, err := deps.NodeManager.GetBestDownloadNode(ctx, userIP, file.StorageNodeID)
        if err != nil {
            c.JSON(http.StatusServiceUnavailable, gin.H{"error": "無可用下載節點"})
            return
        }

        // 3. 生成臨時下載 Token（1 小時有效）
        downloadToken, err := auth.GenerateDownloadToken(file.ID, userID)
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "生成下載鏈接失敗"})
            return
        }

        downloadURL := fmt.Sprintf("%s/relay/download/%s?token=%s",
            bestNode.Address, file.ID, downloadToken)

        // 4. 更新下載計數
        go fileQ.IncrementDownloadCount(context.Background(), file.ID)

        c.JSON(http.StatusOK, gin.H{
            "download_url": downloadURL,
            "filename":     file.Name,
            "size":         file.Size,
            "mime_type":    file.MimeType,
            "expires_in":   3600,
        })
    }
}

// GET /relay/download/:file_id?token=xxx
// 中轉節點實際服務文件下載
// 支持 HTTP Range 請求（斷點續傳）
func RelayServeFile(deps Dependencies) gin.HandlerFunc {
    return func(c *gin.Context) {
        fileID, _ := uuid.Parse(c.Param("file_id"))
        token := c.Query("token")

        // 驗證下載 Token
        claims, err := auth.ValidateDownloadToken(token)
        if err != nil || claims.FileID != fileID {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "無效或過期的下載鏈接"})
            return
        }

        ctx := c.Request.Context()
        fileQ := queries.NewFileQueries(deps.DB)
        file, err := fileQ.GetFileByID(ctx, fileID)
        if err != nil {
            c.JSON(http.StatusNotFound, gin.H{"error": "文件不存在"})
            return
        }

        // 1. 嘗試從本地緩存獲取
        if deps.Config.CacheEnabled {
            if cached, ok := deps.Cache.Get(file.Hash); ok {
                // 緩存命中，直接從緩存返回
                serveFileContent(c, cached, file)
                return
            }
        }

        // 2. 判斷文件是否在本節點
        localPath := fmt.Sprintf("/data/files/%s", file.StoragePath)
        if _, err := os.Stat(localPath); err == nil {
            // 文件在本節點，直接服務
            serveFileFromDisk(c, localPath, file)
            // 同時寫入緩存
            if deps.Config.CacheEnabled {
                go deps.Cache.Put(file.Hash, localPath, file.Size)
            }
            return
        }

        // 3. 從存儲節點拉取（流式代理）
        storageNode, err := deps.NodeManager.GetNodeByID(ctx, file.StorageNodeID)
        if err != nil {
            c.JSON(http.StatusServiceUnavailable, gin.H{"error": "存儲節點不可達"})
            return
        }

        proxyURL := fmt.Sprintf("%s/relay/storage/%s", storageNode.Address, file.StoragePath)
        proxyReq, _ := http.NewRequestWithContext(ctx, "GET", proxyURL, nil)
        // 轉發 Range 頭（支持斷點續傳）
        if rangeHeader := c.GetHeader("Range"); rangeHeader != "" {
            proxyReq.Header.Set("Range", rangeHeader)
        }

        resp, err := http.DefaultClient.Do(proxyReq)
        if err != nil {
            c.JSON(http.StatusBadGateway, gin.H{"error": "從存儲節點獲取文件失敗"})
            return
        }
        defer resp.Body.Close()

        // 設置響應頭
        c.Header("Content-Disposition", fmt.Sprintf(`attachment; filename="%s"`, file.Name))
        c.Header("Content-Type", file.MimeType)
        c.Header("Content-Length", strconv.FormatInt(file.Size, 10))
        c.Header("Accept-Ranges", "bytes")

        if resp.StatusCode == http.StatusPartialContent {
            c.Status(http.StatusPartialContent)
            c.Header("Content-Range", resp.Header.Get("Content-Range"))
        } else {
            c.Status(http.StatusOK)
        }

        // 流式傳輸給用戶（同時寫入緩存）
        if deps.Config.CacheEnabled {
            // 使用 io.TeeReader 同時傳輸和緩存
            // （實際實現需要緩存寫入完成後再確認）
        }
        io.Copy(c.Writer, resp.Body)
    }
}
```

---

## GeoIP 模塊

### `internal/geoip/geoip.go`

```go
package geoip

import (
    "math"
    "net"
    "time"

    "github.com/oschwald/geoip2-golang"
)

type GeoIP struct {
    db      *geoip2.Reader
    dbPath  string
    updater *Updater
}

type Location struct {
    Latitude  float64
    Longitude float64
    Country   string
    City      string
}

func New(dbPath string) (*GeoIP, error) {
    db, err := geoip2.Open(dbPath)
    if err != nil {
        return nil, err
    }
    g := &GeoIP{db: db, dbPath: dbPath}
    // 每 2 小時自動更新
    g.updater = NewUpdater(dbPath, 2*time.Hour, g.reload)
    go g.updater.Start()
    return g, nil
}

// Lookup 查詢 IP 地理位置
func (g *GeoIP) Lookup(ipStr string) (*Location, error) {
    ip := net.ParseIP(ipStr)
    if ip == nil {
        return nil, fmt.Errorf("無效的 IP 地址: %s", ipStr)
    }

    record, err := g.db.City(ip)
    if err != nil {
        return nil, err
    }

    return &Location{
        Latitude:  record.Location.Latitude,
        Longitude: record.Location.Longitude,
        Country:   record.Country.IsoCode,
        City:      record.City.Names["en"],
    }, nil
}

// reload 熱重載 GeoIP 數據庫（更新後調用）
func (g *GeoIP) reload(newPath string) error {
    newDB, err := geoip2.Open(newPath)
    if err != nil {
        return err
    }
    old := g.db
    g.db = newDB
    old.Close()
    return nil
}
```

### `internal/geoip/distance.go`

```go
package geoip

import "math"

// Haversine 計算地球上兩點之間的距離（公里）
// 公式參考：https://en.wikipedia.org/wiki/Haversine_formula
func Haversine(lat1, lon1, lat2, lon2 float64) float64 {
    const R = 6371 // 地球半徑（公里）

    lat1Rad := lat1 * math.Pi / 180
    lat2Rad := lat2 * math.Pi / 180
    deltaLat := (lat2 - lat1) * math.Pi / 180
    deltaLon := (lon2 - lon1) * math.Pi / 180

    a := math.Sin(deltaLat/2)*math.Sin(deltaLat/2) +
        math.Cos(lat1Rad)*math.Cos(lat2Rad)*
            math.Sin(deltaLon/2)*math.Sin(deltaLon/2)

    c := 2 * math.Atan2(math.Sqrt(a), math.Sqrt(1-a))
    return R * c
}

// FindNearestNode 從節點列表中找到離給定位置最近的節點
func FindNearestNode(userLat, userLon float64, nodes []NodeLocation) *NodeLocation {
    if len(nodes) == 0 {
        return nil
    }

    nearest := &nodes[0]
    minDist := Haversine(userLat, userLon, nodes[0].Lat, nodes[0].Lon)

    for i := 1; i < len(nodes); i++ {
        dist := Haversine(userLat, userLon, nodes[i].Lat, nodes[i].Lon)
        if dist < minDist {
            minDist = dist
            nearest = &nodes[i]
        }
    }

    return nearest
}

type NodeLocation struct {
    ID  string
    Lat float64
    Lon float64
    URL string
}
```

---

## Raft 共識模塊

### `internal/raft/raft.go`

```go
package raft

import (
    "context"
    "sync"
    "time"
)

// 核心常量
const (
    HeartbeatInterval    = 60 * time.Second  // Leader 廣播心跳間隔
    HeartbeatTimeout     = 65 * time.Second  // 單次心跳超時時間
    MaxMissedHeartbeats  = 3                 // 連續幾次未收到才切換
    ElectionCooldown     = 180 * time.Second // 選舉冷卻時間（150-300 秒隨機）
)

type NodeState string

const (
    StateFollower  NodeState = "follower"
    StateLeader    NodeState = "leader"
)

type RaftNode struct {
    mu sync.RWMutex

    nodeID   string
    state    NodeState
    term     int64
    leaderID string

    missedHeartbeats int
    lastHeartbeatAt  time.Time

    db          *pgxpool.Pool
    nodeManager *node.Manager
    grpcClients map[string]proto.NodeServiceClient

    leaderCallbacks   []func() // Leader 上任時調用
    followerCallbacks []func() // 降級為 Follower 時調用
}

// Run 啟動 Raft 節點（阻塞）
func (r *RaftNode) Run() {
    // 啟動 gRPC 服務器（接收其他節點的請求）
    go r.startGRPCServer()

    // 啟動心跳監聽
    go r.watchHeartbeat()

    // 如果是 Leader，啟動心跳廣播
    if r.IsLeader() {
        go r.broadcastHeartbeat()
    }
}

// IsLeader 返回當前節點是否為 Leader
func (r *RaftNode) IsLeader() bool {
    r.mu.RLock()
    defer r.mu.RUnlock()
    return r.state == StateLeader
}

// watchHeartbeat 監聽心跳，連續 3 次未收到則切換 Leader
func (r *RaftNode) watchHeartbeat() {
    ticker := time.NewTicker(HeartbeatTimeout)
    defer ticker.Stop()

    for range ticker.C {
        r.mu.Lock()
        if r.state == StateLeader {
            r.mu.Unlock()
            continue // Leader 不需要監聽自己的心跳
        }

        timeSinceLast := time.Since(r.lastHeartbeatAt)
        if timeSinceLast > HeartbeatTimeout {
            r.missedHeartbeats++
            log.Printf("[Raft] 未收到心跳，連續 %d 次（閾值: %d）",
                r.missedHeartbeats, MaxMissedHeartbeats)

            if r.missedHeartbeats >= MaxMissedHeartbeats {
                r.mu.Unlock()
                r.triggerLeaderSwitch()
                continue
            }
        } else {
            r.missedHeartbeats = 0 // 重置計數器
        }
        r.mu.Unlock()
    }
}

// triggerLeaderSwitch 觸發 Leader 切換
func (r *RaftNode) triggerLeaderSwitch() {
    log.Printf("[Raft] 觸發 Leader 切換（連續 %d 次未收到心跳）", MaxMissedHeartbeats)

    ctx := context.Background()

    // 從數據庫獲取候選節點隊列（按 leader_priority 排序）
    candidates, err := r.getLeaderCandidates(ctx)
    if err != nil {
        log.Printf("[Raft] 獲取候選節點失敗: %v", err)
        return
    }

    // 找到當前 Leader 在隊列中的位置，從下一個開始嘗試
    currentIdx := -1
    for i, c := range candidates {
        if c.ID == r.leaderID {
            currentIdx = i
            break
        }
    }

    // 嘗試每個候選節點
    for i := 1; i <= len(candidates); i++ {
        nextIdx := (currentIdx + i) % len(candidates)
        candidate := candidates[nextIdx]

        // 跳過當前掛掉的 Leader
        if candidate.ID == r.leaderID {
            continue
        }

        // 如果候選節點是當前節點自己
        if candidate.ID == r.nodeID {
            r.becomeLeader()
            return
        }

        // 嘗試通知候選節點接管
        if err := r.requestNodeTakeLeadership(ctx, candidate.ID); err != nil {
            log.Printf("[Raft] 節點 %s 無法接管 Leader: %v", candidate.ID, err)
            continue
        }
        return
    }

    log.Printf("[Raft] 所有候選節點均不可用，系統進入只讀模式")
}

// becomeLeader 當前節點成為 Leader
func (r *RaftNode) becomeLeader() {
    r.mu.Lock()
    r.state = StateLeader
    r.term++
    r.leaderID = r.nodeID
    r.missedHeartbeats = 0
    r.mu.Unlock()

    log.Printf("[Raft] 節點 %s 成為 Leader（任期 %d）", r.nodeID, r.term)

    // 更新數據庫
    ctx := context.Background()
    r.db.Exec(ctx,
        `UPDATE raft_state SET current_leader_id = $1, current_term = $2 WHERE node_id = $3`,
        r.nodeID, r.term, r.nodeID)

    // 廣播新 Leader 心跳
    go r.broadcastHeartbeat()

    // 調用 Leader 上任回調（啟動 SMTP Worker 等）
    for _, cb := range r.leaderCallbacks {
        go cb()
    }
}

// broadcastHeartbeat Leader 定期廣播心跳
func (r *RaftNode) broadcastHeartbeat() {
    ticker := time.NewTicker(HeartbeatInterval)
    defer ticker.Stop()

    for {
        if !r.IsLeader() {
            return // 不再是 Leader，停止廣播
        }

        r.mu.RLock()
        term := r.term
        leaderID := r.nodeID
        r.mu.RUnlock()

        // 向所有 Follower 廣播心跳
        nodes := r.nodeManager.GetAllOnlineNodes()
        for _, n := range nodes {
            if n.ID == r.nodeID {
                continue
            }
            go func(nodeID string) {
                client := r.grpcClients[nodeID]
                if client == nil {
                    return
                }
                ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
                defer cancel()
                client.Heartbeat(ctx, &proto.HeartbeatRequest{
                    Term:     term,
                    LeaderID: leaderID,
                })
            }(n.ID)
        }

        <-ticker.C
    }
}

// OnHeartbeatReceived 接收到心跳時調用（gRPC Handler）
func (r *RaftNode) OnHeartbeatReceived(term int64, leaderID string) {
    r.mu.Lock()
    defer r.mu.Unlock()

    r.lastHeartbeatAt = time.Now()
    r.missedHeartbeats = 0
    r.leaderID = leaderID

    // 如果收到更高任期的心跳，自動降級
    if term > r.term {
        r.term = term
        if r.state == StateLeader {
            log.Printf("[Raft] 收到更高任期心跳，從 Leader 降級為 Follower")
            r.state = StateFollower
            for _, cb := range r.followerCallbacks {
                go cb()
            }
        }
    }
}
```

---

## LRU 緩存模塊

### `internal/cache/cache.go`

```go
package cache

import (
    "container/list"
    "sync"
)

// LRUCache 線程安全的 LRU 緩存實現
type LRUCache struct {
    mu       sync.RWMutex
    capacity int64         // 最大容量（bytes）
    used     int64         // 已使用容量（bytes）
    list     *list.List    // 雙向鏈表（LRU 順序）
    items    map[string]*list.Element
}

type cacheEntry struct {
    key      string
    filePath string // 緩存文件在磁碟的路徑
    size     int64
}

func NewLRUCache(capacityBytes int64) *LRUCache {
    return &LRUCache{
        capacity: capacityBytes,
        list:     list.New(),
        items:    make(map[string]*list.Element),
    }
}

// Get 獲取緩存項（命中時移到鏈表頭部）
func (c *LRUCache) Get(key string) (string, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()

    elem, ok := c.items[key]
    if !ok {
        return "", false
    }

    c.list.MoveToFront(elem)
    return elem.Value.(*cacheEntry).filePath, true
}

// Put 插入緩存項，必要時驅逐最舊的項目
func (c *LRUCache) Put(key, filePath string, size int64) {
    c.mu.Lock()
    defer c.mu.Unlock()

    // 如果已存在，更新並移到頭部
    if elem, ok := c.items[key]; ok {
        c.list.MoveToFront(elem)
        return
    }

    // 驅逐舊項目，直到有足夠空間
    for c.used+size > c.capacity && c.list.Len() > 0 {
        c.evictOldest()
    }

    // 插入新項目到頭部
    entry := &cacheEntry{key: key, filePath: filePath, size: size}
    elem := c.list.PushFront(entry)
    c.items[key] = elem
    c.used += size
}

// evictOldest 驅逐最久未使用的項目（從鏈表尾部移除）
func (c *LRUCache) evictOldest() {
    back := c.list.Back()
    if back == nil {
        return
    }
    entry := back.Value.(*cacheEntry)
    c.list.Remove(back)
    delete(c.items, entry.key)
    c.used -= entry.size
    // 刪除磁碟上的緩存文件
    os.Remove(entry.filePath)
}
```

---

## 郵件系統

### `internal/smtp/queue.go`

```go
package smtp

import (
    "context"
    "time"
)

// Worker 郵件發送工作器（只在 Leader 節點啟動）
type Worker struct {
    pool       *pgxpool.Pool
    smtpConfig *config.SMTPConfig
    stopCh     chan struct{}
}

func NewWorker(pool *pgxpool.Pool, cfg *config.SMTPConfig) *Worker {
    return &Worker{
        pool:       pool,
        smtpConfig: cfg,
        stopCh:     make(chan struct{}),
    }
}

// Start 開始掃描並發送郵件隊列（Leader 調用）
func (w *Worker) Start() {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    log.Println("[SMTP] 郵件工作器已啟動（本節點為 Leader）")

    for {
        select {
        case <-ticker.C:
            w.processPendingMails()
        case <-w.stopCh:
            log.Println("[SMTP] 郵件工作器已停止")
            return
        }
    }
}

// Stop 停止工作器（節點降級為 Follower 時調用）
func (w *Worker) Stop() {
    close(w.stopCh)
    w.stopCh = make(chan struct{}) // 重置，以便再次啟動
}

// WatchLeadership 監聽 Leader 狀態變化，自動啟停
func (w *Worker) WatchLeadership(raftNode *raft.RaftNode) {
    raftNode.OnBecomeLeader(func() {
        go w.Start()
    })
    raftNode.OnBecomeFollower(func() {
        w.Stop()
    })
}

func (w *Worker) processPendingMails() {
    ctx := context.Background()

    // 查詢待發送郵件（最多 50 條，防止一次處理太多）
    rows, err := w.pool.Query(ctx, `
        SELECT id, "to", subject, body_html, attempts
        FROM mail_queue
        WHERE status = 'pending' AND attempts < 3
        ORDER BY created_at ASC
        LIMIT 50
    `)
    if err != nil {
        log.Printf("[SMTP] 查詢郵件隊列失敗: %v", err)
        return
    }
    defer rows.Close()

    for rows.Next() {
        var mail struct {
            ID       uuid.UUID
            To       string
            Subject  string
            BodyHTML string
            Attempts int
        }
        if err := rows.Scan(&mail.ID, &mail.To, &mail.Subject,
            &mail.BodyHTML, &mail.Attempts); err != nil {
            continue
        }

        // 嘗試發送
        if err := w.sendMail(mail.To, mail.Subject, mail.BodyHTML); err != nil {
            // 發送失敗，更新重試計數
            w.pool.Exec(ctx, `
                UPDATE mail_queue
                SET attempts = attempts + 1,
                    last_attempt_at = NOW(),
                    error_message = $2,
                    status = CASE WHEN attempts + 1 >= 3 THEN 'failed' ELSE 'pending' END
                WHERE id = $1
            `, mail.ID, err.Error())
            log.Printf("[SMTP] 郵件發送失敗（第 %d 次）: %v", mail.Attempts+1, err)
        } else {
            // 發送成功
            w.pool.Exec(ctx, `
                UPDATE mail_queue SET status = 'sent', last_attempt_at = NOW()
                WHERE id = $1
            `, mail.ID)
        }
    }
}

func (w *Worker) sendMail(to, subject, bodyHTML string) error {
    // 使用 gomail 庫發送
    m := gomail.NewMessage()
    m.SetHeader("From", w.smtpConfig.From)
    m.SetHeader("To", to)
    m.SetHeader("Subject", subject)
    m.SetBody("text/html", bodyHTML)

    d := gomail.NewDialer(
        w.smtpConfig.Host,
        w.smtpConfig.Port,
        w.smtpConfig.User,
        w.smtpConfig.Password,
    )
    d.TLS = &tls.Config{ServerName: w.smtpConfig.Host}

    return d.DialAndSend(m)
}
```

---

## gRPC 協議定義

### `proto/node.proto`

```protobuf
syntax = "proto3";

package Novalano;

option go_package = "github.com/Sapozibi/Novalano/proto/generated";

// 節點間通信服務
service NodeService {
    // Raft 心跳
    rpc Heartbeat(HeartbeatRequest) returns (HeartbeatResponse);

    // 請求某節點接管 Leader
    rpc TakeLeadership(TakeLeadershipRequest) returns (TakeLeadershipResponse);

    // 同步系統狀態
    rpc SyncState(SyncStateRequest) returns (SyncStateResponse);

    // 通知前端文件更新
    rpc NotifyFrontendUpdate(FrontendUpdateRequest) returns (FrontendUpdateResponse);

    // 文件中轉：從存儲節點獲取文件（節點間）
    rpc FetchFile(FetchFileRequest) returns (stream FileChunk);

    // UUID 衝突檢查
    rpc CheckUUIDConflict(CheckUUIDRequest) returns (CheckUUIDResponse);
}

message HeartbeatRequest {
    int64  term      = 1;
    string leader_id = 2;
    int64  timestamp = 3;
}

message HeartbeatResponse {
    bool   accepted = 1;
    int64  term     = 2;
}

message TakeLeadershipRequest {
    string requesting_node_id = 1;
    int64  new_term           = 2;
}

message TakeLeadershipResponse {
    bool   accepted = 1;
    string reason   = 2;
}

message SyncStateRequest {
    string state_json = 1; // 序列化的 state.json 內容
}

message SyncStateResponse {
    bool success = 1;
}

message FrontendUpdateRequest {
    string version    = 1;
    string update_url = 2; // 從哪裡下載新前端文件
}

message FrontendUpdateResponse {
    bool success = 1;
}

message FetchFileRequest {
    string file_id      = 1;
    string storage_path = 2;
    int64  offset       = 3; // 支持 Range 請求
    int64  length       = 4; // 0 表示到文件末尾
}

message FileChunk {
    bytes data      = 1;
    int64 offset    = 2;
    bool  is_last   = 3;
}

message CheckUUIDRequest {
    string uuid = 1;
}

message CheckUUIDResponse {
    bool conflict = 1;
}
```

---

## 中間件

### `internal/api/middleware/auth.go`

```go
package middleware

import (
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"
    "github.com/Sapozibi/Novalano/internal/auth"
)

// Auth JWT 認證中間件
func Auth(deps interface{}) gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" || !strings.HasPrefix(authHeader, "Bearer ") {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "需要認證",
                "code":  "UNAUTHORIZED",
            })
            return
        }

        token := strings.TrimPrefix(authHeader, "Bearer ")
        claims, err := auth.ValidateAccessToken(token)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Token 無效或已過期",
                "code":  "TOKEN_INVALID",
            })
            return
        }

        // 將用戶信息注入 Context
        c.Set("user_id", claims.UserID)
        c.Set("user_role", claims.Role)
        c.Next()
    }
}

// AdminOnly 管理員權限中間件
func AdminOnly() gin.HandlerFunc {
    return func(c *gin.Context) {
        role, exists := c.Get("user_role")
        if !exists || role.(string) != "admin" {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
                "error": "需要管理員權限",
                "code":  "FORBIDDEN",
            })
            return
        }
        c.Next()
    }
}

// NodeAuth 節點間通信認證中間件
// 使用預共享密鑰（從系統主密鑰派生）
func NodeAuth() gin.HandlerFunc {
    return func(c *gin.Context) {
        nodeKey := c.GetHeader("X-Node-Key")
        expectedKey := config.Get().NodeInternetKey // 從系統主密鑰派生
        if !hmac.Equal([]byte(nodeKey), []byte(expectedKey)) {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "節點認證失敗",
            })
            return
        }
        c.Next()
    }
}
```

### `internal/api/middleware/iplog.go`

```go
package middleware

import (
    "github.com/gin-gonic/gin"
    "github.com/Sapozibi/Novalano/internal/db/queries"
)

// IPLog IP 訪問日誌中間件
func IPLog(pool interface{}) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next() // 先處理請求

        // 異步記錄日誌，不阻塞響應
        go func() {
            logQ := queries.NewLogQueries(pool.(*pgxpool.Pool))
            logQ.CreateAccessLog(context.Background(), queries.CreateAccessLogParams{
                IP:        c.ClientIP(),
                UserAgent: c.GetHeader("User-Agent"),
                Path:      c.Request.URL.Path,
                Method:    c.Request.Method,
                Status:    c.Writer.Status(),
                UserID:    getUserIDFromContext(c), // 可能為 nil
            })
        }()
    }
}
```

## [返回](/Doce/ZH-TW/Doce.md)