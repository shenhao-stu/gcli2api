# gcli2api Render 部署指南

## 概述

本文档描述如何将 gcli2api 部署到 [Render](https://render.com) 平台，并配置 Neon PostgreSQL 作为持久化存储，以避免 Render 免费版重启后数据丢失。

> **安全提示：** 所有密钥、连接串、密码均不得写入此文件或任何代码文件中。请在 Render Dashboard 的 Environment 页面手动填写。

---

## 分支说明

| 分支 | 用途 |
|---|---|
| `master` | 跟踪上游 [su-kaka/gcli2api](https://github.com/su-kaka/gcli2api)，不直接修改 |
| `render-deploy` | Render 部署分支，包含本文档和 render.yaml 修改 |

**上游更新流程：** Sync Fork 更新 master → 将 master 变更 rebase 到 render-deploy → Render 自动重新部署。

```bash
git checkout master && git pull
git checkout render-deploy
git rebase master
git push origin render-deploy --force-with-lease
```

---

## render.yaml 说明

`render.yaml` 已配置以下内容：

```yaml
services:
  - type: web
    name: gcli2api
    runtime: docker
    dockerfilePath: ./Dockerfile
    dockerContext: .
    plan: free
    region: singapore
    healthCheckPath: /
    envVars:
      - key: HOST
        value: 0.0.0.0
      - key: PORT
        value: "10000"
      - key: API_PASSWORD
        sync: false
      - key: PANEL_PASSWORD
        sync: false
      - key: POSTGRESQL_URI
        sync: false
      - key: KEEPALIVE_URL
        sync: false
      - key: KEEPALIVE_INTERVAL
        value: "300"
```

`sync: false` 的变量不会写入代码库，需在 Render Dashboard 手动填写。

---

## 部署步骤

### 第一步：连接仓库

1. 登录 [render.com](https://render.com) → **New** → **Blueprint**
2. 连接 GitHub，授权访问你的 fork 仓库
3. **重要：选择 `render-deploy` 分支**（不是 master）
4. Render 自动读取 `render.yaml`，识别出 `gcli2api` 服务

### 第二步：填写环境变量

进入服务 → **Environment** → 添加以下变量（值在 Render Dashboard 中填写，不要写入代码）：

| 变量名 | 说明 |
|---|---|
| `API_PASSWORD` | API 调用时的 Bearer Token，自行设定 |
| `PANEL_PASSWORD` | Web 控制面板登录密码，自行设定 |
| `POSTGRESQL_URI` | Neon PostgreSQL 连接串，从 Neon Dashboard 获取 |
| `KEEPALIVE_URL` | 部署完成后填写（见第四步） |

**POSTGRESQL_URI 格式参考（从 Neon Dashboard 复制，不要使用示例值）：**
```
postgresql://<user>:<password>@<host>/<dbname>?sslmode=require
```

### 第三步：部署

点击 **Apply** 或 **Deploy**，等待 Docker 构建完成（首次约 5-10 分钟）。

### 第四步：配置保活

部署成功后，Render 会分配 URL，如 `https://gcli2api.onrender.com`。

回到 **Environment** 补填：

| 变量名 | 值 |
|---|---|
| `KEEPALIVE_URL` | `https://<your-service>.onrender.com/keepalive` |

保存后服务每 300 秒 ping 自身，防止 Render 免费版 15 分钟无流量休眠。

---

## 存储说明

系统按以下优先级自动选择存储后端：

```
PostgreSQL (POSTGRESQL_URI) > MongoDB (MONGODB_URI) > 本地 SQLite (默认)
```

设置 `POSTGRESQL_URI` 后自动启用 PostgreSQL 模式，所有 Google OAuth 凭证和配置持久化到 Neon，Render 重启不丢失数据。

---

## 密码说明

| 变量名 | 用途 | 优先级 |
|---|---|---|
| `PASSWORD` | 通用密码，同时覆盖 API 和 Panel 密码 | 最高 |
| `API_PASSWORD` | 仅用于 API 端点认证 | 次高 |
| `PANEL_PASSWORD` | 仅用于 Web 控制面板登录 | 次高 |

本部署使用 `API_PASSWORD` + `PANEL_PASSWORD` 分离配置，不设置 `PASSWORD`。

---

## 使用方式

### 调用 API（OpenAI 格式）

```bash
curl https://<your-service>.onrender.com/v1/chat/completions \
  -H "Authorization: Bearer <API_PASSWORD>" \
  -H "Content-Type: application/json" \
  -d '{"model": "gemini-2.5-pro", "messages": [{"role": "user", "content": "Hello"}]}'
```

### 登录控制面板

```
https://<your-service>.onrender.com/panel
密码: <PANEL_PASSWORD>
```

控制面板支持：上传 / 管理 Google OAuth 凭证、查看凭证状态、修改服务配置、查看日志。

---

## 模型说明

### GeminiCLI 路由（默认，`/v1/`）

基础模型：

| 模型 ID | 系列 | 定位 |
|---|---|---|
| `gemini-2.5-pro` | 2.5 | 最强推理，支持思考预算 |
| `gemini-2.5-flash` | 2.5 | 速度质量平衡，支持思考预算 |
| `gemini-3-pro-preview` | 3 | 新一代旗舰 |
| `gemini-3-flash-preview` | 3 | 新一代快速 |
| `gemini-3.1-pro-preview` | 3.1 | 旗舰 |
| `gemini-3.1-flash-lite-preview` | 3.1 | 轻量版 |

### Antigravity 路由（`/antigravity/v1/`）

模型列表从 Google Antigravity API 动态获取，同样支持功能前缀。

---

## 三种流式模式

| 前缀 | 模式 | 原理 | 适用场景 |
|---|---|---|---|
| 无前缀 | **真流式** | token 生成即推送，实时输出 | 正常使用，延迟最低 |
| `假流式/` | **假流式** | 先等完整响应，再在服务端切块模拟 SSE 推送 | 客户端不兼容真流式、网络不稳定 |
| `流式抗截断/` | **抗截断流** | 真流式 + 检测截断 + 自动续传（最多 3 次） | Gemini 输出被提前截断时 |

### 思考后缀

**Gemini 2.5 系列：** `-max` / `-high` / `-medium` / `-low` / `-minimal`

**Gemini 3 系列：**
- `gemini-3-flash-preview`：`-high` / `-medium` / `-low` / `-minimal`
- `gemini-3-pro-preview`：`-low`

### 搜索后缀

所有模型支持 `-search` 后缀，启用 Google Search 工具，可与思考后缀组合：

```
gemini-2.5-pro-high-search
假流式/gemini-2.5-flash-medium-search
流式抗截断/gemini-3-flash-preview-high-search
```
