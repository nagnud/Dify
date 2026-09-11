# 本地开发使用说明：Python 虚拟环境

本文介绍如何在当前电脑上，通过 WSL2（Ubuntu 20.04）运行 Dify 源码，适合修改前后端代码、调试工作流和知识库功能。后端依赖由 `uv` 管理，前端使用 Node.js 和 pnpm，数据库等依赖服务通过 Docker Compose 启动。

这是本地开发配置。希望直接部署整套 Dify 时，请参考[中文 README 的快速启动](README.md#快速启动)。

截至 2026-09-08，已完成的环境配置如下。后面的依赖安装和项目迁移步骤是待执行操作，不表示已经完成。

| 项目 | 当前状态 |
| --- | --- |
| WSL 发行版 | `Ubuntu-20.04`，使用 WSL2，已设为默认发行版 |
| Ubuntu 版本 | 安装镜像为 Ubuntu 20.04.3 LTS |
| Linux 用户 | `nagnud`，已设置密码，属于 `sudo` 用户组 |
| Ubuntu 虚拟磁盘 | `E:\wsl2\Ubuntu20.04\ext4.vhdx` |
| 安装下载文件 | 保留在 `E:\wsl2\downloads` |
| 已通过的检查 | 系统启动、默认用户、文件读写、Ubuntu 仓库网络连接、访问 Dify 项目 |
| 当前项目位置 | Windows 的 `E:\vscode\dify`，对应 WSL 的 `/mnt/e/vscode/dify` |
| 推荐的开发位置 | WSL 的 `~/projects/dify`；尚未迁移到这里 |
| Dify 开发环境 | Python 3.12 虚拟环境及其他项目依赖尚未安装，Docker Desktop 安装及集成状态待确认 |

## 1. 运行方式和目录

| 组件 | 运行方式 | 对应目录 |
| --- | --- | --- |
| API 后端 | Python 3.12 虚拟环境 | `api/`，虚拟环境为 `api/.venv/` |
| Celery Worker 和 Beat | 与 API 共用虚拟环境，分别启动进程 | `api/` |
| Web 前端 | Node.js + pnpm | `web/`，依赖从仓库根目录安装 |
| PostgreSQL、Redis、Weaviate | Docker 容器 | `docker/` |
| 插件服务、代码沙箱和本地 Agent 沙箱 | Docker 容器 | `docker/` |
| 新版 Agent 后端 | 独立 Python 服务，使用新版 Agent 时启动 | `dify-agent/` |

Python 虚拟环境负责隔离 Python 依赖，不能代替数据库、Redis 或前端运行环境。本方案仍需要 Docker 来运行依赖服务。

仓库的 `dev/` 脚本使用 Bash，且 [Celery 官方不支持原生 Windows](https://docs.celeryq.dev/en/main/faq.html#does-celery-support-windows)，因此本文使用 WSL2。本文只提供 Bash 命令，并明确区分 Windows 的 Git Bash 和 Ubuntu 内的 Bash。

### `docker/` 文件夹的用途

项目内的 `docker/` 保存部署配置和部分运行数据，不是 Docker 软件的安装目录。

| 文件或目录 | 用途 |
| --- | --- |
| `docker/docker-compose.yaml` | 启动整套 Dify，默认使用预构建的 API 和 Web 镜像 |
| `docker/docker-compose.middleware.yaml` | 为源码开发启动数据库、Redis、插件服务和沙箱等依赖 |
| `docker/envs/`、`docker/.env.example` | 服务配置示例 |
| `docker/nginx/`、`docker/ssrf_proxy/` | 代理服务配置 |
| `docker/volumes/` | 部分容器的持久化数据，以及配套配置和初始化文件 |

本文使用开发依赖 Compose 配置。Docker Desktop 的镜像和其他容器数据存储位置由 Docker 自身设置管理，不会因为 Ubuntu 装在 E 盘就自动存入这个项目文件夹。

## 2. 从 Git Bash 进入已安装的 Ubuntu

WSL2 和 Ubuntu 20.04 已安装，无须再次安装或注销发行版。

在 Windows 的 **Git Bash** 中执行：

```bash
wsl.exe --list --verbose
wsl.exe -d Ubuntu-20.04
```

第一条命令应显示 `Ubuntu-20.04`，`VERSION` 为 `2`。第二条命令进入 Ubuntu。进入后，在 **Ubuntu Bash** 中执行：

```bash
whoami
echo "$HOME"
uname -r
```

用户名应为 `nagnud`，主目录为 `/home/nagnud`，内核名称包含 `WSL2`。在 Ubuntu 中执行 `exit` 可以返回 Git Bash。

Git Bash 是 Windows 上的终端环境，打开它并不等于已经进入 Ubuntu。两边的 `~`、Python、Node.js 和已安装软件也不同；后续开发命令应在进入 Ubuntu 后执行。

### 准备 Docker Desktop

检查 Windows 是否已安装 Docker Desktop。若未安装，先安装；启动后启用 WSL2 引擎，并在 `Settings → Resources → WSL Integration` 中启用 **Ubuntu-20.04**，使用 Linux 容器模式。在 Ubuntu 中执行 `docker compose version` 确认集成生效。

参考：[Docker Desktop 的 WSL2 集成](https://docs.docker.com/desktop/features/wsl/)。

## 3. 安装开发工具

本节及后续未单独标明终端的命令，都在 **Ubuntu Bash** 中执行。当前 Ubuntu 自带 Python 3.8.10；项目要求 Python 3.12，使用 `uv` 单独安装和管理，不修改系统 Python。

```bash
sudo apt update
sudo apt install -y curl git build-essential openssl rsync

curl -LsSf https://astral.sh/uv/install.sh | sh
source "$HOME/.local/bin/env"
uv python install 3.12
```

`uv` 的安装命令来自[官方安装说明](https://docs.astral.sh/uv/getting-started/installation/)。以后打开终端若找不到 `uv`，检查 `$HOME/.local/bin` 是否在 `PATH` 中。

当前仓库要求：

| 工具 | 当前版本要求 | 依据 |
| --- | --- | --- |
| Python | 3.12.x | [`api/pyproject.toml`](../../api/pyproject.toml) |
| Node.js | 固定运行时版本 24.20.0，`engines` 为 `^24.20.0` | [`package.json`](../../package.json) |
| pnpm | 11.25.0 | 根目录 `package.json` 的 `packageManager` |

在 Ubuntu 内按照 [Node.js 下载页](https://nodejs.org/en/download)的 Linux 安装方式安装对应版本，再执行：

```bash
npm install -g pnpm@11.25.0

node --version
pnpm --version
uv --version
docker compose version
```

更新项目后，以仓库配置中的版本为准。不要把 Windows 安装的 Python 虚拟环境或 `node_modules` 与 Ubuntu 环境混用。

## 4. 选择项目位置并创建配置

### 当前目录和推荐目录

| 位置 | 实际含义 |
| --- | --- |
| Git Bash 中的 `/e/vscode/dify` | Windows 的 `E:\vscode\dify` |
| Ubuntu 中的 `/mnt/e/vscode/dify` | 同一份 Windows 项目文件，无须同步 |
| Ubuntu 中的 `~/projects/dify` | `/home/nagnud/projects/dify`，位于 Linux 文件系统内 |

推荐在 WSL 的 `~/projects/dify` 中长期开发。Python、Node.js 和 Git 在 Linux 文件系统中访问大量依赖文件时，可以减少跨 Windows 文件系统的开销。[微软文件存储建议](https://learn.microsoft.com/en-us/windows/wsl/filesystems)

这里的 `~` 必须是在 Ubuntu 中使用。文件实际保存在 `E:\wsl2\Ubuntu20.04\ext4.vhdx` 内，仍占用 E 盘空间。

### 方式 A：复制到 WSL 后开发（推荐，尚未执行）

决定采用此方式后，在 Ubuntu 中执行下面的复制命令。它保留原项目，包含 `.git` 等隐藏文件，但跳过应在 Linux 中重新生成的依赖和缓存。若目标已存在，则停止复制，先检查目标内容。

```bash
if test -e "$HOME/projects/dify"; then
  printf '%s\n' '目标目录已存在，请先检查，不要覆盖。'
else
  mkdir -p "$HOME/projects" &&
    rsync -a \
      --exclude='.venv/' \
      --exclude='venv/' \
      --exclude='node_modules/' \
      --exclude='__pycache__/' \
      --exclude='.next/' \
      --exclude='.vinext/' \
      --exclude='.pytest_cache/' \
      --exclude='.mypy_cache/' \
      --exclude='.ruff_cache/' \
      /mnt/e/vscode/dify/ "$HOME/projects/dify/"
fi
```

确认复制成功后进入新目录：

```bash
cd ~/projects/dify
git status --short
```

复制后是两份独立项目，它们不会自动同步。确认新目录完整后，日常只在 WSL 这份项目中开发；原目录先保留，是否删除另行决定。如果以后再迁移已经启动的环境，应先停止相关服务，避免在运行中复制数据库文件。

Windows 可以通过以下路径访问 WSL 内的同一份文件：

```text
\\wsl.localhost\Ubuntu-20.04\home\nagnud\projects\dify
```

建议使用 Windows 上 VS Code 的 WSL 扩展打开项目，让编辑界面留在 Windows，终端、Git、Python 和 Node.js 在 Ubuntu 中运行。安装好 VS Code 及 WSL 扩展后，在 Ubuntu 的项目目录中执行：

```bash
code .
```

确认 VS Code 左下角显示 WSL 连接。不要通过直接打开或修改 `ext4.vhdx` 来访问项目文件。

### 方式 B：暂时使用当前 Windows 项目目录

如果尚未复制，仍可在 Ubuntu 中使用现有项目：

```bash
cd /mnt/e/vscode/dify
```

它与 Windows 的 `E:\vscode\dify` 是同一份文件，任何一边的修改都会反映到另一边。源码可以共用，Windows 与 Linux 的虚拟环境和 `node_modules` 不应混用。

两种方式选择一种。后续所有命令都从选定的仓库根目录执行，每次打开新终端先进入 Ubuntu，再进入对应项目目录。

### 创建本地配置

创建配置文件，仅在目标文件不存在时复制：

```bash
test -f api/.env || cp api/.env.example api/.env
test -f web/.env.local || cp web/.env.example web/.env.local
test -f docker/middleware.env || cp docker/envs/middleware.env.example docker/middleware.env
```

| 配置文件 | 用途 |
| --- | --- |
| `api/.env` | API、Worker 和 Beat 的数据库连接、密钥等配置 |
| `web/.env.local` | 前端访问 API 的地址及功能配置 |
| `docker/middleware.env` | 开发依赖服务的密码、端口和启用的数据库类型 |

仓库也提供 `bash dev/setup`，但它会直接覆盖这三个配置文件。本文采用分别复制和安装依赖的方式；完成配置后，无须重新运行该脚本。

## 5. 安装项目依赖

在仓库根目录执行：

```bash
uv sync --project api --python 3.12 --group dev
pnpm install
```

`uv sync` 会创建 `api/.venv/` 并安装后端依赖，不需要另建根目录 `.venv`。仓库启动脚本会通过 `uv` 使用此环境，无须手动执行 `activate`。

可检查实际使用的 Python：

```bash
uv run --project api python -c "import sys; print(sys.version); print(sys.executable)"
```

解释器路径应位于当前项目的 `api/.venv/` 中。编辑器调试后端时，也应选择这个解释器。

## 6. 配置密钥和服务地址

生成应用密钥：

```bash
openssl rand -base64 42
```

把输出填写到 `api/.env` 中现有的 `SECRET_KEY=` 后面。生成一次并保存，日常启动不用重新生成。

检查 `api/.env` 中以下配置，修改现有项，避免重复添加同名变量：

```dotenv
DB_HOST=localhost
DB_PORT=5432
DB_DATABASE=dify
REDIS_HOST=localhost
REDIS_PORT=6379
VECTOR_STORE=weaviate
WEAVIATE_ENDPOINT=http://localhost:8080
PLUGIN_DAEMON_URL=http://127.0.0.1:5002
CODE_EXECUTION_ENDPOINT=http://127.0.0.1:8194
```

示例中的密码和服务密钥已经按开发环境配对。修改时应保持以下对应关系：

| `api/.env` | `docker/middleware.env` |
| --- | --- |
| `DB_USERNAME`、`DB_PASSWORD`、`DB_DATABASE` | 对应的同名数据库配置 |
| `REDIS_PASSWORD`、`CELERY_BROKER_URL` 中的密码 | `REDIS_PASSWORD` |
| `WEAVIATE_API_KEY` | `WEAVIATE_AUTHENTICATION_APIKEY_ALLOWED_KEYS` 中的一个有效密钥 |
| `PLUGIN_DAEMON_KEY` | `PLUGIN_DAEMON_KEY` |
| `INNER_API_KEY_FOR_PLUGIN` | `PLUGIN_DIFY_INNER_API_KEY` |
| `CODE_EXECUTION_API_KEY` | `SANDBOX_API_KEY` |

本地 API 连接数据库时使用 `localhost`。`docker/middleware.env` 的 `DB_HOST=db_postgres` 用于容器之间通信，应保留容器服务名。

`web/.env.local` 的默认地址为：

```dotenv
NEXT_PUBLIC_API_PREFIX=http://localhost:5001/console/api
NEXT_PUBLIC_PUBLIC_API_PREFIX=http://localhost:5001/api
NEXT_PUBLIC_SOCKET_URL=ws://localhost:5001
```

使用本文默认端口时可以保留。修改前端环境变量后，需要重启前端进程。

## 7. 启动依赖服务

确保 Docker Desktop 正在运行，然后执行：

```bash
bash dev/start-docker-compose
```

该脚本使用 `docker/docker-compose.middleware.yaml` 和 `docker/middleware.env`，启动开发所需的依赖服务，不会启动预构建的 Web 和 API 容器。

查看服务状态：

```bash
docker compose --env-file docker/middleware.env \
  -f docker/docker-compose.middleware.yaml -p dify ps
```

等待服务启动，带健康检查的服务应达到 `healthy`。首次拉取镜像需要网络连接。

## 8. 启动 API、Worker 和前端

分别打开三个 Git Bash 窗口，在每个窗口执行 `wsl.exe -d Ubuntu-20.04` 进入 Ubuntu，然后进入第 4 节选定的项目目录：

- 已复制到 WSL：`cd ~/projects/dify`。
- 尚未复制：`cd /mnt/e/vscode/dify`。

也可以在已通过 WSL 扩展打开项目的 VS Code 中，新建三个 Ubuntu Bash 终端。所有服务使用同一份项目目录和配置。

### 终端一：API 后端

```bash
bash dev/start-api
```

脚本先执行数据库迁移，再启动 API，监听 `0.0.0.0:5001`。如果迁移失败，先根据日志排查数据库连接，不要跳过迁移。

### 终端二：后台任务

```bash
bash dev/start-worker
```

Worker 负责知识库文档处理、工作流后台任务等。保持该终端运行，只有前端和 API 启动时，异步任务仍可能一直等待。

### 终端三：Web 前端

```bash
pnpm dev
```

此命令使用仓库默认的 vinext 开发服务和本地 API 代理。浏览器访问：

```text
http://localhost:3000
```

按页面提示创建管理员账号，然后在设置中配置模型供应商。模型调用所需的 API Key 按实际使用的供应商填写。

### 可选终端：定时任务

需要定时任务时，再打开一个终端，在仓库根目录执行：

```bash
bash dev/start-beat
```

同一开发环境启动一个 Beat 实例即可，任务实际执行仍依赖 Worker。

## 9. 使用新版 Agent 时启动独立后端

基础 API、Worker 和前端启动步骤不会自动启动 `dify-agent/`。当前前端示例开启了新版 Agent 功能；使用相关页面时需要额外启动这个服务。

在仓库根目录准备配置和依赖：

```bash
test -f dify-agent/.env || cp dify-agent/.example.env dify-agent/.env
uv sync --project dify-agent --extra server
```

检查 `dify-agent/.env` 中 Redis 密码、插件密钥、内部 API 密钥与第 6 节的配置一致。默认本地沙箱地址为 `http://localhost:5004`，由开发 Compose 中的 `local_sandbox` 提供。

在单独终端从仓库根目录启动：

```bash
uv run --project dify-agent --extra server uvicorn dify_agent.server.app:app \
  --host 0.0.0.0 --port 5050 --reload
```

这里明确使用 `5050`，与 `api/.env` 的 `AGENT_BACKEND_BASE_URL=http://localhost:5050` 配对。`dify-agent/Makefile` 的 `make dev` 默认使用 `8000`，不能在不调整 API 地址的情况下直接替换上面的命令。

涉及沙箱内文件访问、Shell 和 Agent Stub 回调时，还需要确认容器能够访问 `DIFY_AGENT_STUB_API_BASE_URL`、`DIFY_AGENT_SANDBOX_FILES_BASE_URL` 指定的地址。详细配置见 [Agent 入门](../../dify-agent/docs/dify-agent/get-started/index.md)和[运行配置指南](../../dify-agent/docs/dify-agent/guide/index.md)。

## 10. 日常启动、停止和更新

日常开发只需启动 Docker Desktop，从 Git Bash 进入 Ubuntu，再进入第 4 节选定的项目目录，执行第 7 节和第 8 节。使用新版 Agent 时再执行第 9 节的启动命令。无需重复安装 WSL、复制项目或配置、生成密钥或安装全部依赖。

停止前端、API、Worker、Beat 和 Agent 服务：在各自终端按 `Ctrl+C`。

停止 Docker 依赖服务，保留容器和数据：

```bash
docker compose --env-file docker/middleware.env \
  -f docker/docker-compose.middleware.yaml -p dify stop
```

依赖声明或锁文件发生变化后，从仓库根目录重新安装依赖：

```bash
uv sync --project api --group dev
pnpm install
```

Agent 依赖有变化时，另外执行第 9 节的 `uv sync`。后端代码修改后，如服务没有自动重载，停止并重新启动相应进程。

## 11. 常见问题

| 现象 | 检查方法 |
| --- | --- |
| `cd ~/projects/dify` 提示目录不存在 | 项目尚未复制时，按第 4 节选择复制到 WSL，或使用当前的 `/mnt/e/vscode/dify` |
| `whoami` 或 `echo "$HOME"` 的结果不符 | 确认已在 Git Bash 中执行 `wsl.exe -d Ubuntu-20.04`，进入的是 Ubuntu Bash |
| `python3 --version` 仍显示 3.8 | 这是 Ubuntu 20.04 的系统 Python；用 `uv run --project api python --version` 检查项目解释器，不要替换系统 Python |
| Ubuntu 中找不到 `docker` 或无法连接 Docker | 确认 Docker Desktop 已启动，且已启用 Ubuntu-20.04 的 WSL Integration |
| 脚本出现 `$'\r'` 或 `bash\r` 错误 | 用编辑器将对应 `dev/` 脚本的换行符改为 LF；`bash` 执行不能消除 CRLF 问题 |
| 数据库连接失败 | 检查容器状态、`api/.env` 的地址和端口、两份配置中的密码；已有数据库卷不会因修改环境变量自动修改数据库密码 |
| Redis 认证失败或后台任务不执行 | 检查 `REDIS_PASSWORD`、`CELERY_BROKER_URL` 和容器密码是否一致，确认 Worker 正在运行 |
| 前端打开但 API 请求失败 | 确认 API 正在监听 `5001`，检查前端 API 地址，修改配置后重启前端 |
| Docker 启动报告端口占用 | 检查是否同时运行了完整 Dify 部署；若修改 `EXPOSE_*` 端口，同步调整本地 API 的连接配置 |
| 插件、沙箱回调连接失败 | 默认 `host.docker.internal` 地址需要从容器访问到 WSL 内的 API；根据实际 WSL 网络模式配置可达地址，不能用容器内的 `localhost` 代替 |
| 新版 Agent 请求失败 | 确认独立 Agent 后端已启动，API 的 `AGENT_BACKEND_BASE_URL` 与实际端口一致 |

查看依赖服务日志：

```bash
docker compose --env-file docker/middleware.env \
  -f docker/docker-compose.middleware.yaml -p dify logs --tail=100
```

WSL 网络下需特别区分两条路径：本地进程访问 Docker 映射端口，以及 Docker 容器回调本地 API。前者成功不代表后者已打通。回调地址涉及 `docker/middleware.env` 的 `PLUGIN_DIFY_INNER_API_URL`、`api/.env` 的 `INTERNAL_FILES_URL`，以及 Agent 的沙箱回调配置。调整 Compose 环境变量后重新执行第 7 节，使容器使用新配置。

## 12. 提交代码前检查

保留源码、环境变量示例、`pnpm-lock.yaml` 和 `uv.lock`。本机配置、虚拟环境、依赖目录和运行数据应排除在版本控制之外。

当前根目录 `.gitignore` 尚未统一覆盖所有 `.env.*` 变体，例如 `api/.env.local` 和 `docker/.env.backup`；不要仅凭文件名判断它已经被忽略。本文不修改忽略规则，首次提交前可检查：

```bash
git status --short
git check-ignore -v api/.env web/.env.local docker/middleware.env dify-agent/.env
```

更多实现细节参见[后端说明](../../api/README.md)、[前端说明](../../web/README.md)和[开发依赖 Compose 配置](../../docker/docker-compose.middleware.yaml)。
