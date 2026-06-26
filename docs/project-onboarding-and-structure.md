# AgentScope 本地启动与改造指南

这份记录总结了从克隆仓库到本地启动成功的完整过程，并整理了项目目录结构与二次开发的切入点。

## 1. 从 Clone 到启动，我做了哪些事

1. 从 GitHub 克隆仓库到本地工作目录 `/Users/zhangwen22/work/gitlab/agentscope`。
2. 先读了 `README_zh.md`、`pyproject.toml` 和 `examples/agent_service/main.py`，确认项目是 Python 3.11+ 的 Agent 框架，带有 FastAPI 后端和独立的 Web UI 示例。
3. 扫描了 `examples/web_ui` 的前端和后端配置，确认：
   - 前端是 Vite + React + TypeScript。
   - `examples/web_ui/backend` 是一个独立的 Node/Express 辅助后端。
   - 前端真正连接的 AgentScope 服务地址来自 `localStorage.server_url`。
4. 检查了本机工具链，确认：
   - `python3` 默认是 3.8，不满足项目要求。
   - `uv`、Node 20、`pnpm` 可用。
   - Docker daemon 没有运行，不能直接用容器起 Redis。
5. 用 `uv venv --python 3.11 .venv` 创建了 Python 3.11 虚拟环境。
6. 用 `uv pip install -e '.[full]'` 安装了项目完整依赖。
7. 验证 Python 包能正常导入，确认 `agentscope` 版本为 `2.0.3`。
8. 用 Homebrew 安装了 Redis，并通过 `brew services start redis` 启动本地 Redis 服务。
9. 在 `examples/web_ui` 下执行 `pnpm install` 安装了前端依赖。
10. 先后启动了：
   - AgentScope FastAPI 后端
   - Web UI backend
   - Web UI frontend
11. 用 `curl` 做了连通性检查，确认：
   - AgentScope API 可访问
   - Web UI backend `/api/health` 正常
   - Vite 前端 dev server 正常响应

## 2. 遇到的坑，以及我是怎么处理的

1. 默认 `python3` 版本太低
   - 现象：本机 `python3 --version` 是 `3.8.2`，而项目要求 `>= 3.11`。
   - 处理：使用 `uv` 单独创建 Python 3.11 虚拟环境，不依赖系统 Python。

2. `uv` 初次使用时访问用户缓存目录受限
   - 现象：`uv python list` 报 `Failed to initialize cache`，因为沙箱限制访问 `~/.cache/uv`。
   - 处理：请求授权后执行 `uv venv --python 3.11 .venv`，让它完成运行时下载和虚拟环境创建。

3. 项目依赖安装时间较长
   - 现象：`uv pip install -e '.[full]'` 安装和编译了大量依赖，耗时接近 5 分钟。
   - 处理：耐心等它完成，没有中途改写安装方案，因为这是最接近真实运行环境的方式。

4. Docker daemon 没运行，不能用容器起 Redis
   - 现象：`docker ps` 报无法连接 Docker daemon。
   - 处理：改走本机 Homebrew 安装 Redis。

5. Web UI 前端构建默认会写 `node_modules/.tmp`
   - 现象：`pnpm --filter frontend build` 最初报 `EPERM`，因为沙箱下普通执行不能写某些 node_modules 临时目录。
   - 处理：用授权方式重跑构建，让它拥有正常的写权限。

6. Redis 校验在普通权限下被拦
   - 现象：`redis-cli ping` 先报 `Operation not permitted`。
   - 处理：用授权方式访问本机 Redis socket，确认 `PONG`。

7. AgentScope 后端默认端口 `8000` 被别的进程占用
   - 现象：启动 `main.py` 时提示 `Address already in use`。
   - 处理：查到占用进程是另一个 `node` 项目，不是本仓库服务。
   - 进一步处理：改用 `uvicorn main:app --port 8001` 启动 AgentScope 后端，避开冲突端口。

8. `examples/agent_service/main.py` 里端口写死
   - 现象：直接设置 `PORT=8001 python main.py` 仍然起在 8000 相关逻辑上。
   - 处理：不改源码，直接用 `uvicorn` 命令行显式指定 `--port 8001`。

## 3. 目录结构总结

### 顶层

- `src/agentscope/`：AgentScope 核心库源码。
- `examples/`：示例项目和可运行样板。
- `docs/`：文档资源。
- `tests/`：测试。
- `scripts/`：辅助脚本。
- `assets/`：仓库资源文件。

### 核心库 `src/agentscope`

- `agent/`：Agent 核心实现。
- `app/`：面向服务化部署的 FastAPI 应用层。
- `credential/`：各类模型和服务凭证。
- `embedding/`：向量嵌入相关实现。
- `event/`：事件系统。
- `message/`：消息模型。
- `middleware/`：Agent 执行链路中的中间件。
- `model/`：模型适配层。
- `permission/`：权限控制。
- `skill/`：技能相关抽象。
- `state/`：状态管理。
- `tool/`：工具系统。
- `tts/`：语音合成相关适配。
- `workspace/`：工作区与沙箱。
- `mcp/`：MCP 相关封装。

### 服务层 `src/agentscope/app`

- `_app.py`：应用工厂 `create_app`，是服务化启动的主入口。
- `_router/`：HTTP 路由定义。
- `_service/`：业务服务实现。
- `_manager/`：后台调度、任务、唤醒等管理器。
- `storage/`：存储抽象与 Redis 实现。
- `message_bus/`：消息总线抽象与实现。
- `workspace_manager/`：工作区管理实现。
- `middleware/`：服务级中间件。
- `_tool/` 和 `_tools/`：服务内置工具。

### 示例项目 `examples`

- `examples/agent_service/`
  - `main.py`：AgentScope 服务示例入口。
  - 这里负责装配 storage、message bus、workspace manager、MCP 和子智能体模板。
- `examples/web_ui/`
  - `frontend/`：React + Vite 的主界面。
  - `backend/`：Node/Express 的轻量辅助后端。
  - `pnpm-workspace.yaml`：前端 workspace 配置。

## 4. 快速上手改造建议

1. 先改服务入口，再改 UI。
   - 服务入口是 `examples/agent_service/main.py`。
   - 这里适合接入新的 MCP、workspace manager、middleware、权限策略和 sub-agent 模板。

2. UI 改造从 `examples/web_ui/frontend/src` 开始。
   - 页面级内容在 `src/pages/`。
   - 组合组件在 `src/components/`。
   - API 调用封装在 `src/api/`。
   - 登录或服务地址配置逻辑在 `src/pages/setup/index.tsx` 和 `src/api/client.ts`。

3. 连接后端时注意服务地址来源。
   - Web UI 不硬编码 Agent 后端地址。
   - 它从 `localStorage.server_url` 读取地址。
   - 启动后在 UI 的 setup 页面填入 `http://localhost:8001` 即可连到本地后端。

4. 如果你要改服务能力，优先看这些位置。
   - `src/agentscope/app/_app.py`：整体应用装配。
   - `src/agentscope/app/_router/`：对外 API。
   - `src/agentscope/app/_service/`：业务实现。
   - `src/agentscope/app/storage/`：数据持久化。
   - `src/agentscope/app/workspace_manager/`：工具执行环境。

5. 如果你要改 UI 交互或视觉风格，优先看这些位置。
   - `examples/web_ui/frontend/src/App.tsx`
   - `examples/web_ui/frontend/src/pages/chat/`
   - `examples/web_ui/frontend/src/components/layout/`
   - `examples/web_ui/frontend/src/components/chat/`
   - `examples/web_ui/frontend/src/index.css`

## 5. 本地启动命令

以下命令均在仓库根目录 `/Users/zhangwen22/work/gitlab/agentscope` 下执行。四个服务需分别开终端启动（Redis 可用 `brew services` 后台常驻）。

### 1. Redis

```bash
brew services start redis

# 验证
redis-cli ping   # 应返回 PONG
```

- 地址：`localhost:6379`

### 2. AgentScope API

```bash
source .venv/bin/activate
cd examples/agent_service
uvicorn main:app --host 0.0.0.0 --port 8001 --reload
```

- 地址：`http://localhost:8001`
- API 文档：`http://localhost:8001/docs`
- 说明：示例 `main.py` 默认端口为 `8000`，若被占用请用 `--port 8001` 显式指定。

### 3. Web UI backend

```bash
cd examples/web_ui
pnpm dev:backend
```

- 地址：`http://localhost:3000`
- 健康检查：`curl http://localhost:3000/api/health`

### 4. Web UI frontend

```bash
cd examples/web_ui
pnpm dev:frontend
```

- 地址：`http://localhost:5173`
- 首次进入 Setup 页面，填入 AgentScope 后端地址：`http://localhost:8001`

### 快捷方式：同时启动 Web UI 前后端

若不需要分开调试，可在 `examples/web_ui` 下一条命令同时起前端和后端：

```bash
cd examples/web_ui
pnpm dev
```

### 本地服务地址汇总

| 服务 | 地址 |
|------|------|
| Redis | `localhost:6379` |
| AgentScope API | `http://localhost:8001` |
| Web UI backend | `http://localhost:3000` |
| Web UI frontend | `http://localhost:5173` |

### 停止服务

#### 方式一：在启动终端按 `Ctrl+C`

适用于 AgentScope API、Web UI frontend/backend，以及 `pnpm dev` 同时启动的前后端。在对应终端窗口按 `Ctrl+C` 即可停止。

#### 方式二：按服务分别停止

**1. Redis**

```bash
brew services stop redis
```

**2. AgentScope API**

```bash
# 查找占用 8001 端口的进程
lsof -ti :8001 | xargs kill
```

**3. Web UI backend**

```bash
lsof -ti :3000 | xargs kill
```

**4. Web UI frontend**

```bash
lsof -ti :5173 | xargs kill
```

#### 方式三：一次性停止所有本地开发服务

```bash
lsof -ti :8001,:3000,:5173 | xargs kill
brew services stop redis
```

> 若 `kill` 无效，可改用 `kill -9` 强制结束。关闭 Cursor 后台终端不会自动停止已启动的进程，需手动执行上述命令。

## 6. 当前结论

这个仓库已经完成了本地运行链路的验证，后续二次开发可以直接在：

- `examples/agent_service/main.py`
- `examples/web_ui/frontend/src`
- `src/agentscope/app`

这三层展开。
