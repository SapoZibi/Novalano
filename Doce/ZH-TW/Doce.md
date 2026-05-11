
> ## Novalano
> 
> **版本**：v1.0.0-design  
> **狀態**：設計定稿，開發中  

---
 
## ⚡ 這份文檔是給開發者的
 
本文檔包含**Novalano**的完整工程設計，包括：
- 每個模塊用什麼語言和框架實現
- 每個 API 端點的詳細設計
- 每張數據庫表的完整定義
- 每個算法的實現思路和偽代碼
- 前後端如何交互
- 如何部署和測試
**你不需要從頭讀完**。找到你負責的模塊，直接跳過去即可。
 
---
 
## 📁 Doce結構
 
```
docs/
├── Doce.md                      # 本文件：總覽和快速導航
├── ARCHITECTURE.md              # 系統架構總覽
├── BACKEND.md                   # 後端完整設計
├── FRONTEND.md                  # 前端完整設計
├── DATABASE.md                  # 數據庫完整設計
├── DEPLOYMENT.md                # 部署腳本設計
├── API.md                       # 完整 API 文檔
└── Modules/
    ├── RAFT.md                  # Raft 共識算法實現
    ├── GEOIP.md                 # GeoIP 就近路由
    ├── SSL.md                   # SSL 證書管理
    ├── UPLOAD.md                # 文件上傳模塊
    ├── DOWNLOAD.md              # 文件下載模塊
    ├── WEBDAV.md                # WebDAV 服務器
    ├── SMTP.md                  # 郵件系統
    └── SHARE.md                 # 分享系統
```
 
---
 
## 🗺️ 快速導航
 
| 你想做什麼       | 看哪個文件                                          |
|-------------|------------------------------------------------|
| 了解整體架構      | [Architecture.md](/Doce/ZH-TW/Architecture.md) |
| 開發後端 API    | [Backend.md](/Doce/ZH-TW/Backend.md)                       |
| 開發前端界面（暫無）    | [Frontend.md](/Doce/ZH-TW/Frontend.md)                     |
| 設計/修改數據庫    | [Database.md](/Doce/ZH-TW/Database.md)                     |
| 實現 Raft 算法（暫無）  | [Modules/Raft.md](/Doce/ZH-TW/Modules/Raft.md)             |
| 實現文件上傳（暫無）      | [Modules/Upload.md](/Doce/ZH-TW/Modules/Upload.md)         |
| 實現文件下載（暫無）      | [Modules/Download.md](/Doce/ZH-TW/Modules/Download.md)     |
| 實現 GeoIP 路由（暫無） | [Modules/GEOIP.md](/Doce/ZH-TW/Modules/GEOIP.md)           |
| 實現 SSL 管理（暫無）   | [Modules/SSL.md](/Doce/ZH-TW/Modules/SSL.md)               |
| 實現 WebDAV（暫無）   | [Modules/WebDAV.md](/Doce/ZH-TW/Modules/WebDAV.md)         |
| 實現郵件系統（暫無）      | [Modules/SMTP.md](/Doce/ZH-TW/Modules/SMTP.md)             |
| 實現分享系統（暫無）      | [Modules/Share.md](/Doce/ZH-TW/Modules/Share.md)           |
| 寫部署腳本（暫無）       | [Deployment.md](/Doce/ZH-TW/Deployment.md)                 |
| 查詢 API 接口（暫無）   | [API.md](/Doce/ZH-TW/API.md)                               |
 
---
 
## 🏗️ 技術棧一覽
 
```
後端：   Go 1.22+  +  Gin v1.9+  +  gRPC
前端：   React 18  +  TypeScript 5  +  Tailwind CSS v3
數據庫： PostgreSQL 16（主從複製）
緩存：   本地 LRU（純 Go 實現，不依賴 Redis）
代理：   Caddy v2（自動 HTTPS）
通信：   gRPC + TLS（節點間）  |  HTTP/2 + TLS（用戶傳輸）
GeoIP：  MaxMind GeoLite2（免費，每 2 小時更新）
部署：   Docker Compose v2  +  Bash 腳本
行動端： WebDAV 協議（兼容 S3Drive 等現有客戶端）
```
 
---
 
## 🧩 模塊分工建議
 
| 模塊 | 難度 |  依賴 |
|---|---|---|
| 數據庫 Schema | ⭐⭐ | 無 |
| 用戶認證系統 | ⭐⭐⭐ | 數據庫 |
| 文件上傳/下載 | ⭐⭐⭐⭐ | 用戶認證、數據庫 |
| Raft 共識算法 | ⭐⭐⭐⭐⭐ | 數據庫 |
| GeoIP 路由 | ⭐⭐⭐ | 數據庫、節點系統 |
| SSL 管理 | ⭐⭐⭐ | 節點系統 |
| SMTP 郵件 | ⭐⭐ | 數據庫 |
| 分享系統 | ⭐⭐⭐ | 文件系統、用戶認證 |
| WebDAV | ⭐⭐⭐⭐ | 文件系統、用戶認證 |
| 前端（管理面板） | ⭐⭐⭐ | 後端 API |
| 前端（用戶界面） | ⭐⭐⭐ | 後端 API |
| 部署腳本 | ⭐⭐⭐ | 所有後端模塊 |
 
---
 

 
### Git 分支規範
```
main          # 穩定版本
dev           # 開發主分支
feature/*     # 功能開發（如 feature/raft-consensus）
fix/*         # Bug 修復
docs/*        # 文檔更新
```
 
### 提交信息規範（Conventional Commits）
```
feat: 新增文件去重功能
fix: 修復 Leader 切換時郵件丟失問題
docs: 更新 API 文檔
refactor: 重構 GeoIP 查詢邏輯
test: 新增 Raft 選舉單元測試
chore: 更新依賴版本
```
 
### 代碼規範
- **Go**：`gofmt` + `golangci-lint`，包名小寫，錯誤必須處理
- **TypeScript**：ESLint + Prettier，嚴格模式，禁止 `any`
- **SQL**：表名複數小寫下劃線，字段名小寫下劃線
---
 
## 🚀 本地開發環境搭建
 
```bash
# 1. 克隆倉庫
git clone https://github.com/Novalano/Novacloud.git
cd Novalano
 
# 2. 啟動開發依賴（PostgreSQL）
docker compose -f docker/docker-compose.dev.yml up -d
 
# 3. 初始化數據庫
go run ./cmd/migrate/main.go
 
# 4. 啟動後端（開發模式）
go run ./cmd/node/main.go --dev
 
# 5. 啟動前端（另一個終端）
cd frontend
npm install
npm run dev
 
# 前端開發服務器：http://localhost:5173
# 後端 API：http://localhost:8080
```