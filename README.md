# DigitalHuman · 康养数字人

四个上游仓库 + 一套 Docker 集成。所有可运行的服务都封装进 `containerd/`，
由宿主上的 Ollama 提供 LLM/嵌入；上游仓库工作树始终零改动（补丁/覆盖都落在 `containerd/`）。

> 权威细节、实测数字、每个坑的分析都在 **[`containerd/README.md`](containerd/README.md)**。
> 本文只讲仓库构成与怎么起。

## 仓库构成

| 目录 | 是什么 | 容器化 |
|---|---|---|
| `fay/` | Fay 数字人框架（fork，Python/Flask + WebSocket） | ✅ `dh-fay` |
| `origin_fay/` | Fay 上游对照（同镜像、同端口，换模型来源验证） | ✅ `dh-origin-fay` |
| `service/` | 康养后端（FastAPI + MySQL + Redis，含 pytest 套件） | ✅ `dh-backend` + `dh-adapter` |
| `ue/` | Unreal Engine 5.1 数字人模型工程（前端） | ❌ 见下方边界 |
| `containerd/` | 全部 Docker 封装、补丁、覆盖、探针、测试 | —— 唯一入口 |

## 快速开始

```bash
cd containerd
./run.sh up        # 构建 + 起栈（首次约 3~6 分钟，pip 走阿里云镜像）
./run.sh smoke     # 端到端：后端 → adapter → Fay → Ollama → 落库
./run.sh test      # 九组测试件（backend-test · backend-probe · adapter-test · probe-selftest
                   #            · probe-fay-lite · ue-audit · fay-probe · probe-origin-fay · probe-yueshen）
./run.sh test fay-probe   # 只跑其中一组
./run.sh audit     # 核账：四个上游仓库是否仍零改动、与上游不分叉（非零退出可当断言）
./run.sh logs fay  # 看某个服务日志
```

`run.sh` 首次执行会把 `.env.example` 复制成 `.env` 并填入随机密钥。

## 服务与端口

全部发布端口**只绑 `127.0.0.1`**（外部不可达）；下表是宿主侧端口。

| 服务 | 宿主端口 | 说明 |
|---|---|---|
| `dh-backend` | `:8000` | FastAPI；OpenAPI `/docs`，健康 `/api/v1/health` |
| `dh-adapter` | `:8010` | 后端 `/api/chat` ↔ Fay `/api/send`+`get-msg` 的适配层 |
| `dh-fay` | `:5000` `:10002` `:10003` | Fay（fork）HTTP + 两条 WS |
| `dh-origin-fay` | `:5100` `:10012` `:10013` | 上游那份，错开端口并存 |
| `dh-mysql` | `:13306` | 业务库 `care_echo_rehab`（另有 pytest 独立库） |
| `dh-redis` | `:16379` | 缓存 |

`dh-yueshen-rag`（chromadb 知识库）与 Fay 的 `:5010`/`:8765` MCP 口只在 compose 内网，不发布到宿主。
LLM 与嵌入走宿主 Ollama（`http://host.docker.internal:11434`），不在本 compose 内。

```
   UE5 模型(桌面/Windows) --WS 10002--> Fay <--HTTP-- adapter <-- /api/chat -- backend --> MySQL/Redis
                                            └-- MCP :8765/:5010 --(tools / 知识库 / yueshen-rag)
```

## `ue` 的边界（为什么不进容器）

`ue/` 是 **UE5.1 的蓝图内容工程**（`shuziren.uproject`），实测：不含引擎本体、
`PlatformAllowList` 多为 `Win64`、全树 0 个预编译二进制、5 个插件要 Epic Marketplace 授权，
且**端点级引用 Fay 的 10002 协议为 0 处** —— 这个工程从未实现过与 Fay 的对接。

在 Linux 无头容器里跑起来需要 Windows + UE5.1 + 授权插件，与本机环境不兼容。因此 `ue/`
不进 compose，改由 `containerd` 里一个只读的 `ue-audit` 容器做**构建完整性体检**
（描述符可解析 + 每个插件模块的 `Source/` 目录都在），把"为什么不能容器化"变成可复核的数字。

## 密钥与安全

- `containerd/.env` 是本地实值密钥（DB 口令、JWT 签名密钥），已被 `containerd/.gitignore` 忽略，不入库。
- 本文档不再明文打印任何口令。
- ⚠️ 主机遗留项：`~/.gitconfig` 里有一条全局改写 `url."https://<用户>:gho_…@github.com/".insteadOf = "https://github.com/"`，把 `gh auth` 的 OAuth token 以明文写死，并让**所有** GitHub remote 在 `git remote -v` 里显示成带 token 的形式（fay/origin_fay/service/ue 各自 `.git/config` 里存的其实是干净的 URL）。建议删掉这条 insteadOf，改用已装好的 `gh auth git-credential` helper；主机若共享还应**轮换该 token**。此为宿主机全局 git 配置，未代为修改。
