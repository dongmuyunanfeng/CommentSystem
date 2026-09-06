# 留言板（Guestbook）

一个用于演示**全栈项目线上部署工作流**的留言板应用：代码托管在 GitHub，通过 GitHub Actions 自动触发，后端和前端部署在 Render，数据存储在 Neon 云数据库，全程无需本地服务器。

## 架构

```
用户浏览器
    ↓ 访问前端页面
Render 前端 (Static Site)
    ↓ 调用后端 API
Render 后端 (Web Service)
    ↓ 读写数据
Neon 云数据库 (PostgreSQL)
```

三层分离：

- **前端**：React + Vite，打包为静态站点，部署在 Render Static Site
- **后端**：Node.js + Express，部署在 Render Web Service
- **数据库**：PostgreSQL 云端托管（Neon），数据不随服务器重启丢失

## 技术栈

| 层 | 技术 |
|----|------|
| 前端 | React 18 · TypeScript · Vite |
| 后端 | Node.js · Express · TypeScript · pg |
| 数据库 | PostgreSQL（Neon 云数据库） |
| CI/CD | GitHub Actions · Render Deploy Hook |

## 项目结构

```
.
├── backend/                  # 后端服务（Render Web Service）
│   └── src/
│       ├── index.ts          # Express 入口
│       ├── routes.ts         # API 路由
│       └── db.ts             # 数据库连接与建表
├── frontend/                 # 前端应用（Render Static Site）
│   └── src/
│       ├── App.tsx           # 主组件
│       └── api.ts            # 后端接口封装
└── .github/workflows/ci.yml  # CI 检查 + 自动部署
```

## 部署工作流

```
git push → GitHub
              ↓
        GitHub Actions 触发
        ① 后端：依赖安装 → 类型检查 → 编译
        ② 前端：依赖安装 → 类型检查 → 打包
              ↓
        检查全部通过 ✅
              ↓
        触发 Render Deploy Hook
              ↓
   Render 拉取最新代码并重新部署
        ↓                ↓
   后端（连 Neon）    前端（页面可访问）
```

**核心自动化逻辑**（`.github/workflows/ci.yml`）：

- `push` / `pull_request` 到 `master` 或 `main` 分支时触发
- `backend`、`frontend` 两个 job 分别执行 `npm ci`、`npm run typecheck`、`npm run build`
- `deploy` job 依赖前两者，全部通过后调用 `RENDER_BACKEND_HOOK` 与 `RENDER_FRONTEND_HOOK` 触发 Render 重新部署

日常更新只需：

```bash
git add . && git commit -m "改了什么" && git push
```

等几分钟即自动上线。

## 配置要点

### 后端环境变量（Render Web Service）

| Key | Value |
|-----|-------|
| `DATABASE_URL` | Neon 提供的连接字符串（`postgresql://...`） |

代码通过 `process.env.DATABASE_URL` 读取，密码不写入代码仓库。

### 前端环境变量（Render Static Site）

| Key | Value |
|-----|-------|
| `VITE_API_URL` | `https://你的后端地址/api` |

前端代码 `import.meta.env.VITE_API_URL || '/api'` 决定请求后端地址。

### GitHub Actions Secrets

| Secret | 说明 |
|--------|------|
| `RENDER_BACKEND_HOOK` | 后端 Deploy Hook URL |
| `RENDER_FRONTEND_HOOK` | 前端 Deploy Hook URL |

## API

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/api/messages` | 获取留言列表（按时间倒序） |
| `POST` | `/api/messages` | 创建留言，请求体 `{ author, content }` |

## 功能

- 发布留言（昵称 + 内容）
- 留言列表按时间倒序展示
- 字段校验（昵称非空、内容非空且不超过 500 字）
- 首次启动自动创建 `messages` 表

## 更多

完整的从零到上线部署教程（含每一步原因解释与常见问题排查）见 [github前后端项目部署.md](github前后端项目部署.md)。
