## [返回](/Doce/ZH-TW/Doce.md)

# 🗄️ 數據庫設計

> **數據庫**：PostgreSQL 16  
> **驅動**：pgx v5（Go）  
> **字符集**：UTF-8  
> **時區**：所有時間字段使用 `TIMESTAMPTZ`（帶時區）  

---

## 完整 Schema（001_init.sql）

```sql
-- ══════════════════════════════════════════════════════════
-- Novalano 數據庫完整初始化腳本
-- 執行順序：從上到下依次執行
-- ══════════════════════════════════════════════════════════

-- 啟用 UUID 擴展
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- ══════════════════════════════════════════════════════════
-- 用戶組（需要在 users 之前創建，因為 users 有外鍵引用）
-- ══════════════════════════════════════════════════════════
CREATE TABLE groups (
    id                    UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    name                  VARCHAR(64)  NOT NULL UNIQUE,
    storage_limit         BIGINT       NOT NULL DEFAULT 10737418240, -- 10GB，0 表示不限
    upload_speed_limit    BIGINT       NOT NULL DEFAULT 0,           -- bytes/s，0 表示不限
    download_speed_limit  BIGINT       NOT NULL DEFAULT 0,
    max_file_size         BIGINT       NOT NULL DEFAULT 0,           -- bytes，0 表示不限
    max_share_links       INT          NOT NULL DEFAULT 0,           -- 0 表示不限
    permissions           JSONB        NOT NULL DEFAULT '{
        "can_share": true,
        "can_webdav": true,
        "can_zip_download": true
    }',
    created_at            TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- 默認用戶組
INSERT INTO groups (id, name, storage_limit) 
VALUES ('00000000-0000-0000-0000-000000000001', 'Default', 10737418240);

-- ══════════════════════════════════════════════════════════
-- 用戶帳號
-- ══════════════════════════════════════════════════════════
CREATE TABLE users (
    id              UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    username        VARCHAR(64)  NOT NULL UNIQUE,
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    role            VARCHAR(16)  NOT NULL DEFAULT 'user'
                    CHECK (role IN ('admin', 'user')),
    status          VARCHAR(16)  NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'blocked')),
    email_verified  BOOLEAN      NOT NULL DEFAULT FALSE,
    group_id        UUID         REFERENCES groups(id) ON DELETE SET NULL
                    DEFAULT '00000000-0000-0000-0000-000000000001',
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    last_login_at   TIMESTAMPTZ,
    last_login_ip   INET
);

CREATE INDEX idx_users_email    ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_group_id ON users(group_id);

-- ══════════════════════════════════════════════════════════
-- 用戶設置
-- ══════════════════════════════════════════════════════════
CREATE TABLE user_settings (
    user_id                UUID         PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    language               VARCHAR(16),                          -- NULL = 跟隨瀏覽器
    theme                  VARCHAR(16)  NOT NULL DEFAULT 'system'
                           CHECK (theme IN ('light', 'dark', 'system')),
    show_stats             BOOLEAN      NOT NULL DEFAULT TRUE,   -- 是否在網盤顯示統計
    webdav_token           VARCHAR(255) NOT NULL DEFAULT encode(gen_random_bytes(32), 'hex'),
    display_name           VARCHAR(64),
    avatar_path            VARCHAR(512),
    default_upload_path    VARCHAR(512) NOT NULL DEFAULT '/',
    notifications_enabled  BOOLEAN      NOT NULL DEFAULT TRUE,
    two_factor_enabled     BOOLEAN      NOT NULL DEFAULT FALSE,
    two_factor_secret      VARCHAR(255)                          -- AES-256-GCM 加密存儲
);

-- 每個用戶創建時自動插入默認設置
CREATE OR REPLACE FUNCTION create_user_settings()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO user_settings (user_id) VALUES (NEW.id);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER after_user_create
    AFTER INSERT ON users
    FOR EACH ROW EXECUTE FUNCTION create_user_settings();

-- ══════════════════════════════════════════════════════════
-- 用戶配額
-- ══════════════════════════════════════════════════════════
CREATE TABLE user_quotas (
    user_id               UUID    PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    storage_limit         BIGINT  NOT NULL DEFAULT 0,   -- 0 = 使用用戶組配置
    storage_used          BIGINT  NOT NULL DEFAULT 0,
    upload_speed_limit    BIGINT  NOT NULL DEFAULT 0,   -- 0 = 使用用戶組配置
    download_speed_limit  BIGINT  NOT NULL DEFAULT 0,
    max_file_size         BIGINT  NOT NULL DEFAULT 0,
    max_share_links       INT     NOT NULL DEFAULT 0
);

-- 每個用戶創建時自動插入配額記錄
CREATE OR REPLACE FUNCTION create_user_quota()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO user_quotas (user_id) VALUES (NEW.id);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER after_user_create_quota
    AFTER INSERT ON users
    FOR EACH ROW EXECUTE FUNCTION create_user_quota();

-- ══════════════════════════════════════════════════════════
-- 節點配置
-- ══════════════════════════════════════════════════════════
CREATE TABLE nodes (
    id                UUID         PRIMARY KEY,           -- 部署時生成，永久不變
    display_name      VARCHAR(64)  NOT NULL,
    type              VARCHAR(16)  NOT NULL DEFAULT 'capability'
                      CHECK (type IN ('capability', 'backup')),
    address           VARCHAR(255) NOT NULL,              -- 對外訪問地址
    port              INT          NOT NULL DEFAULT 443,
    latitude          DECIMAL(9,6),                       -- GeoIP 或管理員手動設置
    longitude         DECIMAL(9,6),

    -- 功能模塊開關
    relay_enabled     BOOLEAN      NOT NULL DEFAULT TRUE,  -- 中轉功能（能力節點強制 TRUE）
    storage_enabled   BOOLEAN      NOT NULL DEFAULT FALSE,
    storage_limit     BIGINT       NOT NULL DEFAULT 0,     -- bytes，0 = 不限
    storage_used      BIGINT       NOT NULL DEFAULT 0,
    cache_enabled     BOOLEAN      NOT NULL DEFAULT FALSE,
    cache_limit       BIGINT       NOT NULL DEFAULT 0,
    frontend_enabled  BOOLEAN      NOT NULL DEFAULT TRUE,

    -- SSL 配置
    ssl_mode          VARCHAR(16)  NOT NULL DEFAULT 'auto'
                      CHECK (ssl_mode IN ('auto', 'dns', 'system_ca', 'manual')),
    ssl_domain        VARCHAR(255),
    ssl_expires_at    TIMESTAMPTZ,

    -- Raft
    leader_candidate  BOOLEAN      NOT NULL DEFAULT FALSE,
    leader_priority   INT          NOT NULL DEFAULT 100,   -- 越小越優先

    -- 安全
    tls_public_key    TEXT,                               -- 節點 TLS 公鑰（mTLS 用）

    -- 狀態
    last_heartbeat_at TIMESTAMPTZ,
    status            VARCHAR(16)  NOT NULL DEFAULT 'online'
                      CHECK (status IN ('online', 'offline')),
    joined_at         TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_nodes_status           ON nodes(status);
CREATE INDEX idx_nodes_leader_candidate ON nodes(leader_candidate, leader_priority);
CREATE INDEX idx_nodes_storage_enabled  ON nodes(storage_enabled, status);

-- ══════════════════════════════════════════════════════════
-- 目錄結構
-- ══════════════════════════════════════════════════════════
CREATE TABLE directories (
    id          UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    parent_id   UUID         REFERENCES directories(id) ON DELETE CASCADE,  -- NULL = 根目錄
    owner_id    UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    -- 同一目錄下不允許同名子目錄
    UNIQUE (owner_id, parent_id, name)
);

CREATE INDEX idx_directories_owner_id  ON directories(owner_id);
CREATE INDEX idx_directories_parent_id ON directories(parent_id);

-- ══════════════════════════════════════════════════════════
-- 文件記錄
-- ══════════════════════════════════════════════════════════
CREATE TABLE files (
    id               UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    name             VARCHAR(255) NOT NULL,
    parent_id        UUID         REFERENCES directories(id) ON DELETE CASCADE,  -- NULL = 根目錄
    owner_id         UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- 文件物理信息
    size             BIGINT       NOT NULL,
    mime_type        VARCHAR(128) NOT NULL DEFAULT 'application/octet-stream',
    hash             VARCHAR(64)  NOT NULL,           -- SHA-256 hex
    storage_node_id  UUID         REFERENCES nodes(id) ON DELETE RESTRICT,
    storage_path     VARCHAR(512),                    -- 存儲節點上的相對路徑

    -- 去重引用計數
    -- ref_count > 1 表示多個文件記錄共享同一物理文件
    ref_count        INT          NOT NULL DEFAULT 1,

    -- 附加信息
    thumbnail_path   VARCHAR(512),
    download_count   BIGINT       NOT NULL DEFAULT 0,

    -- 上傳狀態
    upload_status    VARCHAR(16)  NOT NULL DEFAULT 'uploading'
                     CHECK (upload_status IN ('uploading', 'completed', 'failed')),

    created_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    -- 同一目錄下不允許同名文件
    UNIQUE (owner_id, parent_id, name)
);

CREATE INDEX idx_files_owner_id        ON files(owner_id);
CREATE INDEX idx_files_parent_id       ON files(parent_id);
CREATE INDEX idx_files_hash            ON files(hash);            -- 去重查詢
CREATE INDEX idx_files_storage_node_id ON files(storage_node_id);
CREATE INDEX idx_files_upload_status   ON files(upload_status);

-- ══════════════════════════════════════════════════════════
-- 上傳分塊記錄（斷點續傳）
-- ══════════════════════════════════════════════════════════
CREATE TABLE upload_chunks (
    id                UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    upload_session_id UUID        NOT NULL,                -- 上傳會話 UUID
    file_id           UUID        REFERENCES files(id) ON DELETE CASCADE,
    owner_id          UUID        NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- 文件基本信息（上傳完成前 files 記錄可能不存在）
    filename          VARCHAR(255) NOT NULL,
    total_size        BIGINT       NOT NULL,
    file_hash         VARCHAR(64)  NOT NULL,               -- 整個文件的 SHA-256
    parent_dir_id     UUID         REFERENCES directories(id),

    -- 分塊信息
    chunks_count      INT          NOT NULL,
    chunk_size        BIGINT       NOT NULL,               -- 每塊大小（bytes）
    chunk_index       INT          NOT NULL DEFAULT -1,    -- -1 表示會話級別記錄

    -- 節點信息
    relay_node_id     UUID         REFERENCES nodes(id),  -- 中轉節點
    storage_node_id   UUID         REFERENCES nodes(id),  -- 目標存儲節點

    -- 狀態
    status            VARCHAR(16)  NOT NULL DEFAULT 'pending'
                      CHECK (status IN ('pending', 'uploaded', 'completed', 'failed')),
    error_message     TEXT,

    created_at        TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    expires_at        TIMESTAMPTZ  NOT NULL DEFAULT NOW() + INTERVAL '24 hours'
);

CREATE INDEX idx_upload_chunks_session_id ON upload_chunks(upload_session_id);
CREATE INDEX idx_upload_chunks_owner_id   ON upload_chunks(owner_id);
CREATE INDEX idx_upload_chunks_expires_at ON upload_chunks(expires_at);

-- 定期清理過期的上傳會話（可以設置 pg_cron 執行）
-- DELETE FROM upload_chunks WHERE expires_at < NOW();

-- ══════════════════════════════════════════════════════════
-- 分享連結
-- ══════════════════════════════════════════════════════════
CREATE TABLE shares (
    id               UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    slug             VARCHAR(32)  NOT NULL UNIQUE,        -- 最少 10 字符，URL 友好
    target_id        UUID         NOT NULL,               -- 文件或目錄 UUID
    target_type      VARCHAR(16)  NOT NULL
                     CHECK (target_type IN ('file', 'directory')),
    owner_id         UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    permission       VARCHAR(16)  NOT NULL DEFAULT 'view'
                     CHECK (permission IN ('view', 'edit', 'upload')),
    password_hash    VARCHAR(255),                        -- NULL = 無密碼
    expires_at       TIMESTAMPTZ,                         -- NULL = 永不過期
    download_count   BIGINT       NOT NULL DEFAULT 0,
    unique_visitors  BIGINT       NOT NULL DEFAULT 0,     -- 瀏覽器指紋去重
    enabled          BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_shares_slug     ON shares(slug);
CREATE INDEX idx_shares_owner_id ON shares(owner_id);
CREATE INDEX idx_shares_target   ON shares(target_id, target_type);

-- ══════════════════════════════════════════════════════════
-- 分享訪問統計
-- ══════════════════════════════════════════════════════════
CREATE TABLE share_visits (
    id                UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    share_id          UUID         NOT NULL REFERENCES shares(id) ON DELETE CASCADE,
    fingerprint_hash  VARCHAR(64)  NOT NULL,              -- 瀏覽器指紋 SHA-256
    downloaded        BOOLEAN      NOT NULL DEFAULT FALSE,
    file_id           UUID         REFERENCES files(id) ON DELETE SET NULL,
    visited_at        TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_share_visits_share_id         ON share_visits(share_id);
CREATE INDEX idx_share_visits_fingerprint_hash ON share_visits(share_id, fingerprint_hash);

-- ══════════════════════════════════════════════════════════
-- 郵件驗證記錄
-- ══════════════════════════════════════════════════════════
CREATE TABLE email_verifications (
    id          UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    purpose     VARCHAR(32)  NOT NULL
                CHECK (purpose IN ('register', 'change_email', 'reset_password')),
    token       VARCHAR(128) NOT NULL UNIQUE,             -- URL Token
    code        VARCHAR(6),                               -- 數字驗證碼（重置密碼）
    new_email   VARCHAR(255),                             -- 更改郵箱時的新郵箱
    expires_at  TIMESTAMPTZ  NOT NULL,                    -- 30 分鐘後過期
    used        BOOLEAN      NOT NULL DEFAULT FALSE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_email_verifications_token   ON email_verifications(token);
CREATE INDEX idx_email_verifications_user_id ON email_verifications(user_id);

-- ══════════════════════════════════════════════════════════
-- 郵件任務隊列（由 Leader 節點掃描發送）
-- ══════════════════════════════════════════════════════════
CREATE TABLE mail_queue (
    id               UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    "to"             VARCHAR(255) NOT NULL,
    subject          VARCHAR(512) NOT NULL,
    body_html        TEXT         NOT NULL,
    status           VARCHAR(16)  NOT NULL DEFAULT 'pending'
                     CHECK (status IN ('pending', 'sent', 'failed')),
    attempts         INT          NOT NULL DEFAULT 0,
    last_attempt_at  TIMESTAMPTZ,
    error_message    TEXT,
    created_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_mail_queue_status ON mail_queue(status, attempts, created_at);

-- ══════════════════════════════════════════════════════════
-- 邀請碼
-- ══════════════════════════════════════════════════════════
CREATE TABLE invite_codes (
    id          UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    code        VARCHAR(64)  NOT NULL UNIQUE,
    created_by  UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    used_by     UUID         REFERENCES users(id) ON DELETE SET NULL,
    used_at     TIMESTAMPTZ,
    expires_at  TIMESTAMPTZ,              -- NULL = 永不過期
    revoked     BOOLEAN      NOT NULL DEFAULT FALSE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_invite_codes_code       ON invite_codes(code);
CREATE INDEX idx_invite_codes_created_by ON invite_codes(created_by);

-- ══════════════════════════════════════════════════════════
-- IP 訪問日誌
-- ══════════════════════════════════════════════════════════
CREATE TABLE access_logs (
    id          UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    ip          INET         NOT NULL,
    ip_method   VARCHAR(16)  NOT NULL DEFAULT 'connection'
                CHECK (ip_method IN ('webrtc', 'connection')),
    country     VARCHAR(4),
    city        VARCHAR(128),
    latitude    DECIMAL(9,6),
    longitude   DECIMAL(9,6),
    user_id     UUID         REFERENCES users(id) ON DELETE SET NULL,
    node_id     UUID         REFERENCES nodes(id) ON DELETE SET NULL,
    action      VARCHAR(32)  NOT NULL
                CHECK (action IN ('upload', 'download', 'browse', 'share_view', 'login')),
    file_id     UUID         REFERENCES files(id) ON DELETE SET NULL,
    blocked     BOOLEAN      NOT NULL DEFAULT FALSE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- 分區表（按月份分區，便於管理大量日誌）
-- 實際實現可根據需求決定是否分區

CREATE INDEX idx_access_logs_ip         ON access_logs(ip);
CREATE INDEX idx_access_logs_user_id    ON access_logs(user_id);
CREATE INDEX idx_access_logs_created_at ON access_logs(created_at DESC);

-- IP 封鎖列表
CREATE TABLE blocked_ips (
    ip          INET         PRIMARY KEY,
    reason      TEXT,
    blocked_by  UUID         REFERENCES users(id) ON DELETE SET NULL,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- ══════════════════════════════════════════════════════════
-- Raft 選舉狀態（每個節點本地記錄）
-- ══════════════════════════════════════════════════════════
CREATE TABLE raft_state (
    node_id            UUID         PRIMARY KEY,          -- 本節點 UUID
    current_term       BIGINT       NOT NULL DEFAULT 0,
    current_leader_id  UUID,
    voted_for          UUID,                              -- 本任期投票給誰
    missed_heartbeats  INT          NOT NULL DEFAULT 0,
    last_election_at   TIMESTAMPTZ
);

-- ══════════════════════════════════════════════════════════
-- 節點配對 Token
-- ══════════════════════════════════════════════════════════
CREATE TABLE pairing_tokens (
    id          UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    token       VARCHAR(128) NOT NULL UNIQUE,
    created_by  UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    used        BOOLEAN      NOT NULL DEFAULT FALSE,
    expires_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW() + INTERVAL '30 minutes',
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_pairing_tokens_token ON pairing_tokens(token);

-- ══════════════════════════════════════════════════════════
-- 系統全局設置（KV 結構）
-- ══════════════════════════════════════════════════════════
CREATE TABLE system_settings (
    key         VARCHAR(64)  PRIMARY KEY,
    value       TEXT         NOT NULL,
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- 初始化默認設置
INSERT INTO system_settings (key, value) VALUES
    ('site_name',                    'Novalano'),
    ('default_language',             'en'),
    ('allow_registration',           'true'),
    ('require_email_verification',   'false'),
    ('require_captcha',              'false'),
    ('require_invite_code',          'false'),
    ('dedup_enabled',                'true'),
    ('default_group_id',             '00000000-0000-0000-0000-000000000001'),
    ('chunk_size_mb',                '16'),
    ('smtp_host',                    ''),
    ('smtp_port',                    '465'),
    ('smtp_user',                    ''),
    ('smtp_password',                ''),           -- AES-256-GCM 加密存儲
    ('smtp_from',                    ''),
    ('smtp_tls',                     'true'),
    ('backup_interval_seconds',      '86400'),      -- 24 小時
    ('geoip_update_interval_seconds','7200'),        -- 2 小時
    ('mode',                         'single'),
    ('version',                      '1.0.0'),
    ('master_key',                   ''),           -- 系統主密鑰
    ('system_ca_cert',               ''),           -- 系統根 CA 證書
    ('system_ca_key',                '');           -- 系統根 CA 私鑰（加密存儲）

-- ══════════════════════════════════════════════════════════
-- 實用函數和觸發器
-- ══════════════════════════════════════════════════════════

-- 自動更新 updated_at 字段的觸發器函數
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 為需要的表創建觸發器
CREATE TRIGGER update_files_updated_at
    BEFORE UPDATE ON files
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER update_directories_updated_at
    BEFORE UPDATE ON directories
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER update_upload_chunks_updated_at
    BEFORE UPDATE ON upload_chunks
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- 文件刪除時減少引用計數，歸零時刪除物理文件記錄
CREATE OR REPLACE FUNCTION on_file_delete()
RETURNS TRIGGER AS $$
BEGIN
    -- 找到同 hash 的其他文件記錄
    UPDATE files
    SET ref_count = ref_count - 1
    WHERE hash = OLD.hash AND id != OLD.id;

    -- 如果引用計數歸零，需要在應用層刪除物理文件
    -- （數據庫無法直接操作文件系統，這裡只記錄需要清理的文件）
    IF (SELECT COUNT(*) FROM files WHERE hash = OLD.hash AND upload_status = 'completed') = 0 THEN
        -- 可以插入一條待清理記錄，由後台任務處理
        INSERT INTO storage_cleanup_queue (hash, storage_node_id, storage_path)
        SELECT OLD.hash, OLD.storage_node_id, OLD.storage_path
        WHERE OLD.storage_path IS NOT NULL;
    END IF;

    RETURN OLD;
END;
$$ LANGUAGE plpgsql;

-- 存儲清理隊列（文件物理刪除任務）
CREATE TABLE storage_cleanup_queue (
    id               UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    hash             VARCHAR(64)  NOT NULL,
    storage_node_id  UUID         REFERENCES nodes(id) ON DELETE SET NULL,
    storage_path     VARCHAR(512) NOT NULL,
    created_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    processed        BOOLEAN      NOT NULL DEFAULT FALSE
);

CREATE TRIGGER on_file_delete_trigger
    AFTER DELETE ON files
    FOR EACH ROW EXECUTE FUNCTION on_file_delete();

-- 更新用戶已用存儲量（文件完成上傳時）
CREATE OR REPLACE FUNCTION update_user_storage_used()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.upload_status = 'completed' AND OLD.upload_status != 'completed' THEN
        UPDATE user_quotas
        SET storage_used = storage_used + NEW.size
        WHERE user_id = NEW.owner_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER on_file_complete
    AFTER UPDATE ON files
    FOR EACH ROW EXECUTE FUNCTION update_user_storage_used();

-- 文件刪除時恢復存儲量
CREATE OR REPLACE FUNCTION restore_user_storage_on_delete()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.upload_status = 'completed' THEN
        UPDATE user_quotas
        SET storage_used = GREATEST(0, storage_used - OLD.size)
        WHERE user_id = OLD.owner_id;
    END IF;
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER on_file_delete_restore_quota
    AFTER DELETE ON files
    FOR EACH ROW EXECUTE FUNCTION restore_user_storage_on_delete();
```

---

## PostgreSQL 主從複製配置

### 主庫配置（`postgresql.conf`）

```ini
# 複製配置
wal_level = replica              # 啟用複製所需的 WAL 級別
max_wal_senders = 10             # 最多支持 10 個從庫連接
wal_keep_size = 1GB              # 保留足夠的 WAL 文件供從庫追趕

# 性能調優
shared_buffers = 256MB           # 推薦為物理記憶體的 25%
work_mem = 16MB
maintenance_work_mem = 128MB
checkpoint_completion_target = 0.9
```

### 主庫認證配置（`pg_hba.conf`）

```
# 允許從庫連接複製
host    replication     replicator    10.0.0.0/8    scram-sha-256
host    replication     replicator    172.16.0.0/12 scram-sha-256
```

### 創建複製用戶

```sql
-- 在主庫執行
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'strong_password_here';
```

### 從庫配置（`postgresql.conf`）

```ini
# 從庫連接到主庫
primary_conninfo = 'host=主庫IP port=5432 user=replicator password=strong_password_here'
hot_standby = on               # 允許從庫處理只讀查詢
```

### Go 應用層讀寫分離

```go
package db

// 讀寫分離連接池
type Pool struct {
    primary  *pgxpool.Pool  // 主庫：所有寫操作
    replicas []*pgxpool.Pool // 從庫：所有讀操作
    rr       int            // 輪詢計數器（Round-Robin 負載均衡）
}

// Writer 返回主庫連接（寫操作使用）
func (p *Pool) Writer() *pgxpool.Pool {
    return p.primary
}

// Reader 返回從庫連接（讀操作使用，Round-Robin 負載均衡）
func (p *Pool) Reader() *pgxpool.Pool {
    if len(p.replicas) == 0 {
        return p.primary // 單節點模式，讀寫都走主庫
    }
    p.rr = (p.rr + 1) % len(p.replicas)
    return p.replicas[p.rr]
}
```

---

## 常用查詢優化

### 查找用戶最近的存儲節點

```sql
-- 給定用戶位置（lat, lon），找到距離最近且未滿的存儲節點
-- 使用 Haversine 公式在 SQL 中計算距離
SELECT
    id,
    display_name,
    address,
    storage_limit - storage_used AS available_bytes,
    -- Haversine 近似距離計算（精度足夠）
    (6371 * acos(
        cos(radians($1)) * cos(radians(latitude)) *
        cos(radians(longitude) - radians($2)) +
        sin(radians($1)) * sin(radians(latitude))
    )) AS distance_km
FROM nodes
WHERE
    status = 'online'
    AND storage_enabled = TRUE
    AND (storage_limit = 0 OR storage_used < storage_limit)
ORDER BY distance_km ASC
LIMIT 5;
-- 參數：$1 = 用戶緯度，$2 = 用戶經度
```

### 去重查詢

```sql
-- 上傳前檢查文件是否已存在
SELECT
    f.id,
    f.hash,
    f.size,
    f.storage_node_id,
    f.storage_path
FROM files f
WHERE
    f.hash = $1
    AND f.upload_status = 'completed'
LIMIT 1;
-- 參數：$1 = 文件 SHA-256 hash
```

### 分享連結查詢（含過期和密碼檢查）

```sql
-- 獲取分享連結詳情
SELECT
    s.*,
    CASE
        WHEN s.expires_at IS NULL THEN FALSE
        WHEN s.expires_at < NOW() THEN TRUE
        ELSE FALSE
    END AS is_expired
FROM shares s
WHERE
    s.slug = $1
    AND s.enabled = TRUE;
```

### 統計分享的不重複訪客數

```sql
-- 檢查瀏覽器指紋是否已存在（去重）
INSERT INTO share_visits (share_id, fingerprint_hash, downloaded, file_id)
VALUES ($1, $2, $3, $4)
ON CONFLICT DO NOTHING;  -- 同一指紋不重複插入

-- 更新分享的統計數字
UPDATE shares
SET
    download_count = download_count + 1,
    unique_visitors = (
        SELECT COUNT(DISTINCT fingerprint_hash)
        FROM share_visits
        WHERE share_id = $1
    )
WHERE id = $1;
```

---

## 數據庫遷移管理

### `internal/db/migrations/` 目錄結構

```
migrations/
├── 001_init.sql              # 初始化所有表（上面的完整腳本）
├── 002_add_storage_cleanup.sql
├── 003_add_two_factor.sql
└── ...
```

### 遷移執行邏輯（`internal/db/migrate.go`）

```go
package db

import (
    "context"
    "embed"
    "fmt"
    "sort"
)

//go:embed migrations/*.sql
var migrationFiles embed.FS

func Migrate(pool *pgxpool.Pool) error {
    ctx := context.Background()

    // 創建遷移版本表
    _, err := pool.Exec(ctx, `
        CREATE TABLE IF NOT EXISTS schema_migrations (
            version     VARCHAR(16) PRIMARY KEY,
            applied_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
        )
    `)
    if err != nil {
        return fmt.Errorf("創建遷移表失敗: %w", err)
    }

    // 獲取已執行的遷移
    rows, _ := pool.Query(ctx, `SELECT version FROM schema_migrations ORDER BY version`)
    applied := make(map[string]bool)
    for rows.Next() {
        var version string
        rows.Scan(&version)
        applied[version] = true
    }
    rows.Close()

    // 讀取並排序遷移文件
    entries, _ := migrationFiles.ReadDir("migrations")
    sort.Slice(entries, func(i, j int) bool {
        return entries[i].Name() < entries[j].Name()
    })

    // 執行未應用的遷移
    for _, entry := range entries {
        version := entry.Name()[:3] // 取前 3 位（如 "001"）
        if applied[version] {
            continue
        }

        content, _ := migrationFiles.ReadFile("migrations/" + entry.Name())

        // 在事務中執行遷移
        tx, err := pool.Begin(ctx)
        if err != nil {
            return err
        }

        if _, err := tx.Exec(ctx, string(content)); err != nil {
            tx.Rollback(ctx)
            return fmt.Errorf("遷移 %s 失敗: %w", entry.Name(), err)
        }

        tx.Exec(ctx, `INSERT INTO schema_migrations (version) VALUES ($1)`, version)

        if err := tx.Commit(ctx); err != nil {
            return err
        }

        fmt.Printf("✓ 遷移 %s 執行成功\n", entry.Name())
    }

    return nil
}
```

## [返回](/Doce/ZH-TW/Doce.md)