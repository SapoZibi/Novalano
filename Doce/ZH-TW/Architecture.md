## [返回](/Doce/ZH-TW/Doce.md)

# 🏗️ Novalano 系統架構設計

---

## 整體架構圖

```
┌─────────────────────────────────────────────────────────────────────┐
                            用戶端                                    
   瀏覽器（React SPA）  /  S3Drive App（WebDAV）  /  任意 WebDAV 客戶端   
└──────────┬──────────────────────────────────────┬───────────────────┘
           │                                      │
    ① HTTP/2 HTTPS                        ② HTTP/2 HTTPS
    頁面訪問 / API 請求                   文件上傳 / 下載（直連）
           │                                      │
           ▼                                      ▼
┌──────────────────────┐             ┌────────────────────────────────┐
    探測頁（< 5KB）                            最近能力節點              
                                        由 GeoIP + WebRTC 計算選出     
   1. WebRTC 獲取真實IP                  流量完全不過前端伺服器          
   2. 查詢最佳前端節點                 └─────────────┬─────────────────┘
   3. 302 跳轉（≤1次）                              
└──────────┬───────────┘                   文件推送至存儲節點
          跳轉後                                 
           ▼                                       ▼
┌──────────────────────┐          ┌────────────────────────────────────┐
     最佳前端節點                              存儲節點                   
   完整 React 界面       ◄────────      開啟了存儲功能的能力節點           
   管理面板                 API         文件實際存放在此                    
   WebDAV 端點             請求    └────────────────────────────────────┘
└──────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
                     能力節點集群（Raft 共識）                           
                                                                        
    每個能力節點運行以下進程：                                            
    ┌─────────────────────────────────────────────────────────────┐    
    │  Caddy（反向代理 + HTTPS）                                    
    │  ↓                                                           
    │  Novalano Node（單一 Go 二進制）                              
    │    ├── HTTP API Server（Gin）    ← 處理所有 REST API           
    │    ├── gRPC Server              ← 節點間通信                   
    │    ├── Raft Module              ← 共識算法                     
    │    ├── Relay Module             ← 文件中轉                    
    │    ├── Storage Module           ← 文件存儲（可選開啟）          
    │    ├── Cache Module（LRU）      ← 熱門文件緩存（可選）          
    │    ├── GeoIP Module             ← 就近路由計算                  
    │    ├── WebDAV Server            ← WebDAV 協議                 
    │    ├── SSL Manager              ← 證書申請和續期               
    │    └── SMTP Worker（Leader）    ← 郵件發送（僅 Leader）        
    │  PostgreSQL 16（從庫，實時同步主庫）                            
    └─────────────────────────────────────────────────────────────┘     
                                                                        
   節點間通信：gRPC + mTLS（雙向 TLS 認證）                             
   數據同步：PostgreSQL Streaming Replication                          
└──────────────────────────────┬──────────────────────────────────────┘
                               │
              定時備份（增量文件 + PostgreSQL dump）
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
                     備份節點（可選，可多台）                             
   不對外提供服務，不運行前端，不參與 Raft                              
└─────────────────────────────────────────────────────────────────────┘
```

---

## 目錄結構（代碼倉庫）

```
Novalano/
│
├── cmd/                          # 可執行程序入口
│   ├── node/
│   │   └── main.go               # 節點主程序入口
│   └── migrate/
│       └── main.go               # 數據庫遷移工具
│
├── internal/                     # 內部包（不對外暴露）
│   ├── api/                      # HTTP API 層
│   │   ├── router.go             # 路由注冊
│   │   ├── middleware/           # 中間件
│   │   │   ├── auth.go           # JWT 認證中間件
│   │   │   ├── ratelimit.go      # 限流中間件
│   │   │   ├── iplog.go          # IP 日誌中間件
│   │   │   └── cors.go           # CORS 中間件
│   │   └── handlers/             # 請求處理器
│   │       ├── auth.go           # 認證相關
│   │       ├── files.go          # 文件操作
│   │       ├── upload.go         # 文件上傳
│   │       ├── download.go       # 文件下載
│   │       ├── share.go          # 分享連結
│   │       ├── users.go          # 用戶管理
│   │       ├── admin.go          # 管理員操作
│   │       ├── nodes.go          # 節點管理
│   │       └── system.go         # 系統設置
│   │
│   ├── auth/                     # 認證授權
│   │   ├── jwt.go                # JWT 生成和驗證
│   │   ├── password.go           # 密碼 hash（bcrypt）
│   │   └── webdav.go             # WebDAV 認證
│   │
│   ├── db/                       # 數據庫層
│   │   ├── db.go                 # 連接池初始化
│   │   ├── migrations/           # 數據庫遷移文件
│   │   │   ├── 001_init.sql
│   │   │   ├── 002_users.sql
│   │   │   └── ...
│   │   └── queries/              # 數據庫查詢（每個表一個文件）
│   │       ├── users.go
│   │       ├── files.go
│   │       ├── shares.go
│   │       ├── nodes.go
│   │       └── ...
│   │
│   ├── raft/                     # Raft 共識算法
│   │   ├── raft.go               # 核心邏輯
│   │   ├── heartbeat.go          # 心跳廣播和接收
│   │   ├── election.go           # Leader 選舉
│   │   ├── replication.go        # 日誌複製
│   │   └── state.go              # 狀態管理
│   │
│   ├── geoip/                    # GeoIP 查詢
│   │   ├── geoip.go              # MaxMind 庫封裝
│   │   ├── updater.go            # 自動更新邏輯
│   │   └── distance.go           # Haversine 距離計算
│   │
│   ├── storage/                  # 文件存儲
│   │   ├── storage.go            # 存儲接口定義
│   │   ├── local.go              # 本地文件系統存儲
│   │   └── dedup.go              # 文件去重邏輯
│   │
│   ├── relay/                    # 文件中轉
│   │   ├── relay.go              # 中轉邏輯
│   │   ├── chunked.go            # 分塊傳輸
│   │   └── resume.go             # 斷點續傳
│   │
│   ├── cache/                    # LRU 緩存
│   │   ├── cache.go              # LRU 實現
│   │   └── evict.go              # 驅逐策略
│   │
│   ├── ssl/                      # SSL 證書管理
│   │   ├── ssl.go                # 入口和模式選擇
│   │   ├── auto.go               # 模式一：自動申請
│   │   ├── dns.go                # 模式二：DNS 驗證
│   │   ├── system_ca.go          # 模式三：系統 CA
│   │   └── manual.go             # 模式四：手動上傳
│   │
│   ├── smtp/                     # 郵件系統
│   │   ├── smtp.go               # SMTP 發送
│   │   ├── queue.go              # 郵件隊列（Leader 掃描）
│   │   └── templates/            # 郵件模板（各語言）
│   │       ├── welcome.html
│   │       ├── verify_email.html
│   │       └── reset_password.html
│   │
│   ├── webdav/                   # WebDAV 服務器
│   │   ├── webdav.go             # WebDAV 協議實現
│   │   └── lock.go               # WebDAV 鎖管理
│   │
│   ├── share/                    # 分享系統
│   │   ├── share.go              # 分享邏輯
│   │   ├── stats.go              # 訪問統計
│   │   └── zip.go                # 文件夾 ZIP 打包
│   │
│   ├── node/                     # 節點管理
│   │   ├── node.go               # 節點自身狀態
│   │   ├── discovery.go          # 節點發現
│   │   ├── pairing.go            # 節點配對（Token）
│   │   └── sync.go               # 數據同步（state.json、前端文件）
│   │
│   └── config/                   # 配置管理
│       ├── config.go             # 配置結構體
│       └── loader.go             # 從 node.json / 環境變量加載
│
├── proto/                        # gRPC Protocol Buffers 定義
│   ├── node.proto                # 節點間通信接口
│   └── generated/                # 自動生成的 Go 代碼
│
├── frontend/                     # React 前端
│   ├── src/
│   │   ├── main.tsx              # 入口
│   │   ├── App.tsx               # 根組件 + 路由
│   │   ├── pages/                # 頁面組件
│   │   │   ├── Probe.tsx         # 探測頁（WebRTC + 跳轉）
│   │   │   ├── Login.tsx
│   │   │   ├── Register.tsx
│   │   │   ├── Drive.tsx         # 文件管理器
│   │   │   ├── Share.tsx         # 分享頁面（訪客視角）
│   │   │   ├── Settings.tsx      # 個人設置
│   │   │   └── admin/            # 管理面板
│   │   │       ├── Dashboard.tsx
│   │   │       ├── Nodes.tsx
│   │   │       ├── Users.tsx
│   │   │       └── System.tsx
│   │   ├── components/           # 可復用組件
│   │   ├── api/                  # API 請求封裝（axios）
│   │   ├── hooks/                # React Hooks
│   │   ├── store/                # 狀態管理（Zustand）
│   │   └── i18n/                 # 國際化
│   │       ├── index.ts          # i18next 配置
│   │       └── locales/          # 語言包
│   │           ├── zh-TW.json
│   │           ├── zh-CN.json
│   │           ├── en.json
│   │           └── ...（共 12 種）
│   ├── public/
│   │   └── probe.html            # 探測頁靜態文件（極輕量）
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   └── tailwind.config.ts
│
├── scripts/
│   ├── install.sh                # 多語言部署腳本
│   └── update.sh                 # 更新腳本
│
├── docker/
│   ├── Dockerfile                # 生產環境鏡像
│   ├── Dockerfile.dev            # 開發環境鏡像
│   ├── docker-compose.yml        # 生產環境
│   ├── docker-compose.dev.yml    # 開發環境
│   └── Caddyfile                 # Caddy 配置模板
│
├── docs/                         # 工程文檔（本文件所在）
│
├── go.mod
├── go.sum
├── .golangci.yml                 # Go lint 配置
├── .eslintrc.json                # ESLint 配置
├── .prettierrc                   # Prettier 配置
└── Makefile                      # 常用命令
```

---

## 數據流向圖

### 文件上傳數據流

```
用戶瀏覽器
    │
    │ ① POST /api/upload/init（含文件 hash、大小）
    ▼
最近能力節點（前端節點）
    │
    │ 後端計算：選擇最優中轉節點和存儲節點
    │ 返回：{ upload_node_url, session_id, chunk_size }
    │
    │ ② PUT {upload_node_url}/relay/upload/{session_id}/chunk/{n}
    ▼
最近能力節點（中轉節點，可能是同一台）
    │
    │ 接收每個分塊，記錄進度
    │
    │ ③ 所有塊接收完成後，推送至存儲節點
    ▼
存儲節點（開啟存儲功能的能力節點）
    │
    │ 驗證文件完整性（SHA-256）
    │
    │ ④ gRPC：通知 Leader 更新元數據
    ▼
Leader 節點
    │
    │ 寫入 PostgreSQL 主庫
    │ 廣播同步至所有 Follower
    ▼
所有能力節點的 PostgreSQL 從庫
```

### 文件下載數據流

```
用戶瀏覽器
    │
    │ ① GET /api/download/{file_id}（獲取下載 URL）
    ▼
任意能力節點（API）
    │
    │ 查詢元數據：文件在哪個存儲節點？
    │ GeoIP 計算：哪個中轉節點最近？
    │ 返回：{ download_url }（可能是存儲節點或緩存節點）
    │
    │ ② GET {download_url}（帶 Range 頭支持斷點續傳）
    ▼
最優能力節點（存儲節點或緩存節點）
    │
    │ 如果是緩存節點且緩存未命中：
    │   從存儲節點拉取 → 緩存 → 返回用戶
    │ 如果是存儲節點：
    │   直接從磁碟讀取返回
    ▼
用戶瀏覽器（直接接收，流量不過前端）
```

---

## 進程間通信

### HTTP REST API（用戶 ↔ 節點）

```
協議：HTTPS（HTTP/2）
認證：JWT Bearer Token
格式：JSON
端口：443（Caddy 代理到後端 8080）
```

### gRPC（節點 ↔ 節點）

```
協議：gRPC over TLS（mTLS 雙向認證）
認證：客戶端證書（系統 CA 或 Let's Encrypt）
格式：Protocol Buffers
端口：9090（內部端口，不對外暴露）

主要 RPC 方法：
- Heartbeat：Leader 廣播心跳
- SyncState：同步系統狀態
- RelayFile：轉發文件至存儲節點
- NotifyFrontendUpdate：通知前端文件更新
- RequestVote：Raft 投票請求（強制移交 Leader 時）
```

### PostgreSQL Streaming Replication（元數據同步）

```
主庫（Leader 節點）→ 從庫（所有 Follower 能力節點）
協議：PostgreSQL Replication Protocol
延遲：毫秒級（局域網）/ 秒級（跨地域）
模式：異步複製（性能優先）/ 同步複製（一致性優先，可配置）
```

---

## 狀態機

### 節點狀態

```
              部署腳本執行
                   │
                   ▼
            ┌───────────────┐
            │  INITIALIZING │  初始化
            └──────┬────────┘
                   │ 初始化完成
                   ▼
            ┌─────────────┐
            │   FOLLOWER  │  Follower 狀態
            └──────┬──────┘
                   │
        ┌──────────┴──────────┐
        │                     │
        │ 連續3次              │ 被選為 Leader
        │ 心跳未到             │ 或強制移交
        ▼                     ▼
 ┌─────────────┐         ┌──────────────┐
 │  CANDIDATE  │         │   LEADER     │
 │  候選中      │──────▶ │  Leader 狀態 │
 └─────────────┘ 當選     └──────┬──────┘
                                │  連續 3 次
                                │  心跳未被收到
                                │  或強制移交leader
                                ▼
                        ┌─────────────┐
                        │   FOLLOWER  │
                        └─────────────┘
```

### 系統模式狀態

```
       第一個節點部署完成
              │
              ▼
       ┌────────────┐
       │ SINGLE_NODE│  單節點模式
       └─────┬──────┘
             │ 管理員在面板成功
             │ 添加第二個節點
             ▼
       ┌────────────┐
       │ MULTI_NODE │  多節點模式
       └─────┬──────┘
             │ 節點數量減少至 1
             ▼
       ┌────────────┐
       │ SINGLE_NODE│  單節點模式
       └────────────┘
```

---

## 安全設計

### 認證體系

```
用戶認證：
├── 登錄：用戶名/郵箱 + 密碼（bcrypt 哈希）→ JWT Token
│         JWT 有效期：24 小時（訪問 Token）+ 7 天（刷新 Token）
├── WebDAV：用戶名 + WebDAV 專用密鑰（非登錄密碼）
└── 雙因素：TOTP（RFC 6238），兼容 Google Authenticator

節點間認證：
└── mTLS（雙向 TLS），使用系統 CA 簽發的節點證書
    每個節點有唯一的客戶端證書，用於 gRPC 通信
```

### 數據加密

```
傳輸加密：
├── 用戶 ↔ 節點：TLS 1.3
└── 節點 ↔ 節點：mTLS 1.3

靜態加密（存儲在數據庫中的敏感數據）：
├── SMTP 密碼：AES-256-GCM，密鑰派生自系統主密鑰
├── 系統根 CA 私鑰：AES-256-GCM
├── 雙因素驗證密鑰：AES-256-GCM
└── WebDAV 密鑰：bcrypt（與用戶密碼相同處理方式）
```

### 防護措施

```
├── 速率限制：登錄接口 10 次/分鐘/Browser，上傳接口按用戶組設置
├── IP 封鎖：管理員手動封鎖，封鎖後 403 拒絕所有請求
├── SQL 注入：使用參數化查詢（pgx 庫），禁止拼接 SQL
├── XSS：前端所有用戶輸入 HTML 轉義，使用 CSP 頭
├── CSRF：API 使用 JWT 而非 Cookie，天然防 CSRF
└── 文件類型：不執行任何上傳文件，僅存儲和傳輸
```

## [返回](/Doce/ZH-TW/Doce.md)