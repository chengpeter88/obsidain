---
tags:
  - docker
  - compose
---

# 🧩 Docker Compose

## 是什麼？

當你的專案需要**多個容器**同時運作（例如：API + 資料庫 + Redis），
用 Docker Compose 可以用一個 `compose.yaml` 檔統一管理。

> 一個指令啟動全部：`docker compose up -d`

---

## 基本結構

```yaml
# compose.yaml
services:
  app:                          # 服務名稱（自訂）
    build: .                    # 從當前目錄的 Dockerfile 建立
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      - db

  db:
    image: postgres:16          # 直接用現有 image
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:                      # 宣告 named volume
```

---

## 常用指令

```bash
# 啟動
docker compose up           # 前景啟動（看得到 log）
docker compose up -d        # 背景啟動

# 停止
docker compose down         # 停止並移除容器
docker compose down -v      # 連 volume 一起刪

# 查看狀態
docker compose ps           # 查看所有服務狀態
docker compose logs         # 查看所有 log
docker compose logs -f app  # 追蹤特定服務的 log

# 重建
docker compose build        # 重新建立 image
docker compose up --build   # 重建並啟動

# 執行指令
docker compose exec app bash         # 進入 app 容器
docker compose exec db psql -U user  # 進入資料庫
```

---

## 完整範例：Python API + PostgreSQL + Redis

```yaml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/appdb
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - .:/app                  # 掛載本地程式碼（開發用）

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: appdb
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  db_data:
  redis_data:
```

---

## 常用設定說明

### ports — 對外開放 port

```yaml
ports:
  - "8080:80"       # 本機 8080 對應容器 80
  - "127.0.0.1:3000:3000"   # 只允許本機存取
```

### volumes — 掛載資料

```yaml
volumes:
  - ./data:/app/data          # 本地目錄掛載
  - db_data:/var/lib/data     # named volume（持久化）
  - /tmp:/tmp                 # 絕對路徑
```

### environment — 環境變數

```yaml
environment:
  - NODE_ENV=production
  - PORT=3000

# 或用 env_file
env_file:
  - .env
```

### depends_on — 啟動順序

```yaml
depends_on:
  db:
    condition: service_healthy    # 等 db healthy 才啟動
  redis:
    condition: service_started    # 等 redis 啟動就好
```

---

## 多環境設定技巧

```bash
# 開發環境
docker compose -f compose.yaml -f compose.dev.yaml up

# 正式環境
docker compose -f compose.yaml -f compose.prod.yaml up
```

```yaml
# compose.dev.yaml（覆蓋開發設定）
services:
  api:
    volumes:
      - .:/app          # 開發才掛載本地程式碼
    command: uvicorn main:app --reload
```
