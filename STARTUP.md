# NexaBot 开发启动指南

> NexaBot 是 [HKUDS/nanobot](https://github.com/HKUDS/nanobot) 的个人二开分支（GitHub: `LEI9387/NexaBot`）。
> 本文件记录如何在本地把开发环境跑起来。官方完整说明见 [README.md](./README.md) 与 [docs/](./docs/)。

## 环境要求

- Python ≥ 3.11（用 **uv** 管理）
- Node.js（bun 依赖它）
- **bun**（前端包管理器）

## 一、安装依赖

### 1. Python 依赖

```bash
uv sync --all-extras --dev
```

- 创建 `.venv` 并安装运行 + 开发依赖（pytest / ruff / basedpyright 等）。
- 注意：`uv.lock` 被 `.gitignore` 忽略，首次 sync 会在本地生成它，后续复用。

### 2. 前端依赖

```bash
npm install -g bun   # 仅首次，bun 没装时执行
cd webui
bun install
```

## 二、启动（推荐：一条命令）

```bash
uv run nanobot webui --dev
```

这条命令会自动完成所有接线并打开浏览器：

| 组件 | 端口 |
|---|---|
| gateway（OpenAI 兼容 API、健康检查） | 18790 |
| WebUI 频道（前端真正的后端） | 8069 |
| Vite 前端开发服务器 | 5173 |

- 自动设置 `NANOBOT_API_URL=http://127.0.0.1:8069`
- 自动打开 `http://127.0.0.1:5173`，URL 里自带 bootstrapSecret，**免输密码**
- 一个进程管理全部，`Ctrl+C` 一起停止
- `--dev` 不能和 `--background` 同时使用

> ⚠️ 启动前先停掉旧的 `nanobot gateway` / `bun run dev` 进程，否则端口被占会报冲突。

## 三、启动（可选：手动分开跑）

```bash
# 终端 A：后端
uv run nanobot gateway

# 终端 B：前端（注意环境变量要指向 8069）
$env:NANOBOT_API_URL="http://127.0.0.1:8069"; bun run dev
```

浏览器手动访问需带鉴权 secret：

```
http://127.0.0.1:5173/#/?bootstrapSecret=<密码>
```

## 四、端口说明（重要）

| 端口 | 用途 |
|---|---|
| **8069** | WebUI 频道：`/webui/bootstrap`、会话、设置都走这里（WS 复用协议） |
| **18790** | gateway：OpenAI 兼容 API、`/health` 健康检查 |
| **5173** | Vite 前端开发服务器 |

> ⚠️ 仓库 `AGENTS.md` 里写的 `8765` 是旧版本默认端口，**已过时，不要用**。

## 五、WebUI 登录密码

- 密码 = `~/.nanobot/config.json` 中 `channels.websocket.tokenIssueSecret`（或 `token`）的值。
- Windows 上配置文件在 `C:\Users\<用户名>\.nanobot\config.json`。
- 用 `nanobot webui --dev` 启动时自动带 secret，**不会**看到登录框。
- 想改密码：编辑该字段后重启 gateway。

## 六、测试与检查

```bash
uv run pytest                  # Python 测试
uv run ruff check nanobot/     # lint
uv run basedpyright            # 严格类型检查（CI 同款）
cd webui && bun run test       # 前端测试
cd webui && bun run build      # 构建前端产物 → nanobot/web/dist（打包 wheel 用）
```

## 七、Git 工作流（本 fork）

直接在主分支 `main` 上开发：

```bash
git add .
git commit -m "做了什么"
git push origin main
```

同步官方（HKUDS/nanobot）更新：

```bash
git fetch upstream
git log main..upstream/main --oneline   # 先看上游多了哪些提交
git pull upstream main                   # 确认后全量合并
```

## 常见问题

**1. pytest 大量报 `PermissionError`（指向 Temp\pytest-of-LEI）**
临时目录权限损坏。先杀掉残留的 python/pytest 进程，再删除该目录后重跑：

```powershell
taskkill /PID <残留python进程ID> /F
Remove-Item -Recurse -Force "$env:TEMP\pytest-of-LEI"
```

**2. Vite 报 `ECONNREFUSED 127.0.0.1:8765`**
gateway 没跑在 8765（这版本默认 18790）。用 `nanobot webui --dev` 启动，或手动启动时把 `NANOBOT_API_URL` 指向 `8069`。

**3. WebUI 弹登录框要密码**
见上文"WebUI 登录密码"一节。
