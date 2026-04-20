---
tags:
  - docker
  - dockerfile
---

# 📄 Dockerfile

## Dockerfile 是什麼？

Dockerfile 是建立 Image 的**食譜**。
每一行指令，都會在 Image 裡產生一層（Layer）。

---

## 基本結構範例

```dockerfile
# 1. 從哪個 base image 開始
FROM node:20-alpine

# 2. 設定工作目錄
WORKDIR /app

# 3. 複製 package.json 並安裝依賴
COPY package*.json ./
RUN npm install

# 4. 複製所有原始碼
COPY . .

# 5. 對外開放的 port
EXPOSE 3000

# 6. 容器啟動時執行的指令
CMD ["node", "index.js"]
```

---

## 常用指令說明

| 指令 | 說明 | 範例 |
|------|------|------|
| `FROM` | 指定 base image | `FROM python:3.11-slim` |
| `WORKDIR` | 設定工作目錄 | `WORKDIR /app` |
| `COPY` | 複製檔案進 image | `COPY . .` |
| `ADD` | 複製（支援 URL 和解壓縮） | `ADD app.tar.gz /app` |
| `RUN` | 建置時執行指令 | `RUN pip install -r requirements.txt` |
| `CMD` | 容器啟動時的預設指令 | `CMD ["python", "app.py"]` |
| `ENTRYPOINT` | 固定的啟動指令 | `ENTRYPOINT ["gunicorn"]` |
| `ENV` | 設定環境變數 | `ENV PORT=8080` |
| `ARG` | 建置時的參數 | `ARG VERSION=1.0` |
| `EXPOSE` | 聲明對外 port（文件用） | `EXPOSE 8080` |
| `VOLUME` | 掛載點聲明 | `VOLUME /data` |
| `USER` | 切換執行使用者 | `USER node` |

---

## CMD vs ENTRYPOINT

```dockerfile
# CMD：可以被 docker run 後面的指令覆蓋
CMD ["python", "app.py"]
# docker run my-image python other.py  ← 會覆蓋 CMD

# ENTRYPOINT：固定不變，後面的指令變成參數
ENTRYPOINT ["python"]
CMD ["app.py"]
# docker run my-image other.py  ← 等於執行 python other.py
```

---

## 實際範例

### Python FastAPI

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Node.js

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

---

## ⚡ 最佳實踐

### 1. 善用 Layer 快取

```dockerfile
# ✅ 好：先複製 package.json，依賴不變就不重新安裝
COPY package*.json ./
RUN npm install
COPY . .

# ❌ 差：每次改任何檔案都要重新 npm install
COPY . .
RUN npm install
```

### 2. 用 .dockerignore 排除不必要的檔案

```
# .dockerignore
node_modules
.git
.env
*.log
__pycache__
```

### 3. 使用輕量 base image

```dockerfile
# ✅ 輕量（alpine）
FROM python:3.11-slim
FROM node:20-alpine

# ❌ 太肥
FROM ubuntu
```

### 4. Multi-stage Build（減少 image 大小）

```dockerfile
# Stage 1：建置
FROM node:20 AS builder
WORKDIR /app
COPY . .
RUN npm install && npm run build

# Stage 2：只取建置結果
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```
