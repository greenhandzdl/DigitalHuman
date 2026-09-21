# DigitalHuman · 康养数字人

三个上游仓库 + 一套 Docker 集成。所有可运行的服务都封装进 `containerd/`，
由宿主上的 Ollama 提供 LLM/嵌入；上游仓库工作树始终零改动（补丁/覆盖都落在 `containerd/`）。

> 权威细节、实测数字、每个坑的分析都在 **[`containerd/README.md`](containerd/README.md)**。
> 本文只讲仓库构成与怎么起。

## 仓库构成

| 目录 | 是什么 | 容器化 |
|---|---|---|
| `fay/` | Fay 数字人框架，fork [`chuan918/Fay`](https://github.com/chuan918/Fay)（上游 v4.8.1 的直接后代） | ✅ `dh-fay` |
| `service/` | 康养后端（FastAPI + MySQL + Redis，含 pytest 套件） | ✅ `dh-backend` + `dh-adapter` |
| `ue/` | Unreal Engine 5.1 数字人模型工程（前端） | ❌ 见下方边界 |
| `containerd/` | 全部 Docker 封装、补丁、覆盖、探针、测试 | —— 唯一入口 |

`fay/ service/ ue/ containerd/` 以 **git submodule** 记录各自上游的精确 commit 作为溯源。

Fay 的**上游不是一份并排的拷贝，而是 `fay/` 仓库里的一个 remote**：
`origin` = fork `chuan918/Fay`（子模块记录的地址），`upstream` = `xszyou/Fay`。
`main` 只 track `origin/main`，跟上游走靠合并 —— `cd containerd && ./run.sh upstream`
会 fetch 一次、报落后几条，并把每份 fay 补丁对 `upstream/main` 干跑预检（贴不上就非零退出）。
曾经有一份 `origin_fay/` 上游参照实例（第二个镜像、第二个端口段、第九组测试件）。
fork 已经是上游的直接后代 —— 当前只落后一个只动 `requirements.txt` 的提交，且那个改动
我们自己那份 overlay 早就带着 —— 并排跑第二份的意义没了，2026-09-21 撤掉。

```bash
git clone --recursive https://github.com/greenhandzdl/DigitalHuman.git
# 已 clone 过则：git submodule update --init --recursive
# submodule 的远端换了地址（fay → chuan918/Fay）时：
git submodule sync -- fay && git submodule update --init --recursive
```

## 快速开始

```bash
cd containerd
./run.sh up        # 构建 + 起栈（首次约 3~6 分钟，pip 走阿里云镜像）
./run.sh smoke     # 端到端：后端 → adapter → Fay → Ollama → 落库
./run.sh test      # 八组测试件（backend-test · backend-probe · adapter-test · probe-selftest
                   #            · probe-fay-lite · ue-audit · fay-probe · probe-yueshen）
./run.sh test fay-probe   # 只跑其中一组
./run.sh audit     # 核账：三个上游仓库是否仍零改动、与上游不分叉（非零退出可当断言）
./run.sh upstream  # 跟上游对表：报 fork 落后 xszyou/Fay 几条 + 补丁可否照贴
./run.sh logs fay  # 看某个服务日志
```

`run.sh` 首次执行会调用 `containerd/tools/gen_keys.py` 把 `.env.example` 复制成 `.env` 并填入随机密钥（DB 口令与 JWT 签名密钥）。

## 服务与端口

全部发布端口**只绑 `127.0.0.1`**（外部不可达）；下表是宿主侧端口。

| 服务 | 宿主端口 | 说明 |
|---|---|---|
| `dh-backend` | `:8000` | FastAPI；OpenAPI `/docs`，健康 `/api/v1/health` |
| `dh-adapter` | `:8010` | 后端 `/api/chat` ↔ Fay `/api/send`+`get-msg` 的适配层 |
| `dh-fay` | `:5000` `:10002` `:10003` | Fay HTTP + 两条 WS（另有 test profile 下的 `dh-fay-lite`，同镜像换 1.5b 小模型，不发布宿主端口）|
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
- ⚠️ 主机遗留项：`~/.gitconfig` 里有一条全局改写 `url."https://<用户>:gho_…@github.com/".insteadOf = "https://github.com/"`，把 `gh auth` 的 OAuth token 以明文写死，并让**所有** GitHub remote 在 `git remote -v` 里显示成带 token 的形式（fay/service/ue 各自 `.git/config`、以及 `fay` 里 `upstream` 这个 remote 存的其实都是干净 URL）。建议删掉这条 insteadOf，改用已装好的 `gh auth git-credential` helper；主机若共享还应**轮换该 token**。此为宿主机全局 git 配置，未代为修改。
