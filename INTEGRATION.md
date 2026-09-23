# 集成预览：现在接到哪一步、模块之间怎么说话

这份是**进度与通讯快照**，给"先看一眼这套东西拼到什么程度"用；
每一条判据的成因、踩过的坑和完整实测数字在
**[`containerd/README.md`](containerd/README.md)**（权威细节），架构与端口口径在
**[`README.md`](README.md)**。本文不重复它们的论证，只回答两个问题：
**每个模块接到哪了**、**跨模块的每一跳走的是什么、慢在哪、坏了长什么样**。

> 本文所有数字都带日期，最近一次复核是 **2026-09-23 12:14 (CST)**：
> `./run.sh smoke` 6/6 全绿、`adapter-test` 18/18、`probe-selftest` 10/10、
> prod 真跑端口逐条 `docker inspect` 过。这组数字是在**栈外那台 26B 对话服务**（本机起的
> OpenAI 兼容推理服务，端口 `11432`，不进 compose）跑 LLM、宿主 ollama 跑嵌入这套配置下
> 量的 —— 端点、模型名与 token 都是按机器变的事，只存在于 `containerd/.env`，仓库里只有示例值。

## 一页结论

| 模块 | 接到了哪一步 | 证据（可复跑） | 还差什么 |
|---|---|---|---|
| `fay/` 数字人框架 | **容器化 + 改动全在补丁层**，问答/MCP/知识库/TTS/音频/远程音频都在跑 | 9 份补丁对 `upstream/main` 全部 `--fuzz=0` 可贴（2026-09-23 复跑）；`probe-fay-lite` 42 条判据 **40 PASS / 2 SKIP / 0 FAIL**（2026-09-22 复跑，那条链上唯一不靠显存也能给出正面证据的一组） | 真 UE 客户端从未拨进来过（见「数字人那一跳」） |
| `service/` 业务后端 | **原生进容器 + 自带测试全绿**，会话/用户/落库/调度器都在链路上 | `backend-test` **39 passed**（每轮重建空测试库）、`backend-probe` **11 PASS / 0 SKIP / 0 FAIL**（路由、迁移、schema、鉴权、活体写路径、调度器） | 没有真实业务数据（种子语料是自造的）；`dev-login` 是 DEBUG 口才给外壳用 |
| `frontend/` CareEcho H5 | **伙伴方产物 + 本层同源外壳**，聊天与麦克风都从 `:5173` 那一口进 | `frontend-test` **19/19**、`ws-relay-test` **14/14**（含 3 条负面自检）、`asr-test` **13/13** | 外壳只实现了 `/api/chat/send` 与 `/api/health` 两个口，H5 其余 `chatAPI` 方法未接；Xmov 密钥默认留空 → 数字人区域是占位提示 |
| `ue/` 数字人模型 | **不进容器**，改为构建完整性体检 | `ue-audit` **2 PASS**（描述符可解析 + 每个插件模块 `Source/` 在位）；实测 0 个预编译二进制、5 个插件要 Marketplace 授权、端点级引用 `10002` 为 **0 处** | 对接本身：那个工程从未实现过与 Fay 的 WS 协议，缺的是对端实现而不是本栈端口 |
| `containerd/` 集成层 | **唯一入口**，14 组测试件（+ 一组点名的 `kb-fay`）、8 常驻服务 + `fay-lite` + `kb-ingest` | `./run.sh test`（完整一轮见 run #32 那节）；`./run.sh audit` 报四个上游仓库 `0/0` 且 `0 dirty` | 完整一轮里 `frontend-probe` 那一条仍在 90s 墙上红（下面「慢在哪」有账） |

知识库这一格单独拎出来，因为它是这轮新验的：外部语料 14 份 .docx → **516 片段** →
入库 **576 向量**，12 问真实问法 **recall@3 = 12/12**（`./run.sh kb`，2026-09-22），
且 `top_k` 扫 3/5/8 三个档位是平的 —— 所以维持 L2 距离、不加阈值、不动切片粒度，
`patches/yueshen_rag/0003` 那 cosine+阈值 **不打**（没有 MISS 就没证据）。
上面那些都是**直调工具**验的库侧；业务侧另有一组 `./run.sh kbq`（`probes/kb_fay_probe.py`）
从 `/api/send` 问一句、判 Fay 那一行的原始回帧，2026-09-23 首跑 **10/10**：注入块在、
`query=` 逐字等于问句、片段带 `〔出处〕`、正文答完、`<dh-end>` 收尾，摘掉 prestart 注册的
对照轮里注入块随即消失。同日复跑 **9/10 + 1 SKIP**，跳的是「正文逐字复现知识库那个词」那条
软证据 —— 这条差别值得记进跨模块文档：**注入是确定发生的，模型引用不引用原话不是**，
所以那一行只报不判红（判据设计见 containerd/README「业务侧」那一节）。
它同时钉死一件事：**这一跳查不查库不是模型决定的** ——
`query_yueshen` 注册成 prestart（`{"query":"{{question}}","top_k":3}`、
`allow_function_call:false`），拼 prompt 之前无条件跑一次；工具的「禁用」开关掐不断这条路
（预启动调用带 `skip_enabled_check=True`），能掐断的只有摘注册。

## 模块之间怎么说话

一次问答的完整路径。**箭头方向 = TCP 连接方向**（谁拨谁），
栈里没有任何服务外拨到 UE —— 数字人那一跳是 UE 拨进 `:10002`：

```
H5（浏览器）
 ├─ HTTP  POST /api/chat/send ─▶ dh-frontend :5173 ─▶ dh-backend :8000 ─▶ dh-adapter :8010 ─▶ dh-fay :5000
 ├─ WS    /funasr-ws ──────────▶ dh-frontend :5173 ──常量表转发─────────▶ dh-funasr :10095
 └─ WS    Xmov SDK ────────────▶ 魔珐云（浏览器里，不经过本栈任何容器）

dh-fay      ─▶ 栈外对话服务 :11432（OpenAI 兼容 /chat/completions，端点与模型名在 .env）
            ─▶ dh-yueshen-rag :8766/sse（MCP over SSE）─▶ 宿主 ollama :11434（嵌入）
            ─▶ dh-redis（Fay 的会话与记忆）
dh-backend  ─▶ dh-mysql（care_echo_rehab）  ·  dh-adapter ─▶ dh-fay :5000（/api/send + 轮询 /api/get-msg）
UE（另一台 Windows）─ WS 拨入 ─▶ dh-fay :10002          音频文件 URL 反向写在 Fay 的回帧文本里
```

| # | 一跳 | 端点与协议 | 认证 | 实测与失败表现 |
|---|---|---|---|---|
| 1 | H5 → 外壳 | `POST /api/chat/send`（**页面同源**，`baseURL=/api`，伙伴方 axios 30s 写死） | 无（外壳按 cookie 里的设备号代持身份） | 外壳只实现 `/api/chat/send` 与 `/api/health` 两个口，其余 `/api/*` 回 501；axios 那 30s 是这条链**真实**的用户侧上限 |
| 2 | 外壳 → 后端 | `POST /api/v1/chat/sessions/{id}/messages`，外壳替设备先走一次 `POST /auth/dev-login` mint JWT | DEBUG 口；后端再转 adapter | 后端不可达 → 用户看到 `后端 POST /chat/sessions/N/messages 不可达：timed out`（今天这条就是红的样子） |
| 3 | 后端 → adapter | `POST http://adapter:8010/api/chat`（同步一问一答） | compose 内网名，无鉴权 | 上游失败回 **502**，后端置 `fay_error`，不会静默成"数字人沉默" |
| 4 | adapter → Fay | 表单 `POST /api/send` 投问题，再**轮询** `POST /api/get-msg` 取回复行 | 同上 | 结束判据是 `<dh-end>` 哨兵（补丁 `fay/0008`），没哨兵才退回 8s 静默 + 必须有正文；假 Fay 下 18 条契约判据今天复跑 18/18 |
| 5 | Fay → 对话端点 | OpenAI 兼容 `POST {base}/chat/completions`，`base` 由 `FAY_GPT_BASE_URL` / `FAY_BIG_MODEL_BASE_URL` 覆盖（`fay/0009`） | 那个服务的 token 只在 `.env` | 现在这一跳打的是**栈外那台 26B**（`gemma-4-26b-a4b-nvfp4`，本机起的推理服务）：H5 连发四问 **5.6 / 9.4 / 17.8 / 18.2s** 各回一段干净正文（2026-09-22）；后面那句「同一句话 **172~301s**」量的是对话还落在宿主 ollama、`qwen3.5:9b` 只有 6% 权重进显存那阵子，端点搬走之后不再是这条链的数 |
| 6 | Fay → 知识库 | MCP over **SSE** `http://yueshen-rag:8766/sse`（`yueshen_rag/0001` 加的口；上游只有 stdio，容器里 stdin 那头没人） | 无 | 3 个工具在清单里；`query_yueshen` 是 **prestart** 工具，拼 prompt 之前无条件执行，结果以 `<prestart>` 注入当轮 —— **注意这一跳不经 :5010 的管理面**（`runtime_bridge` 进程内直调），所以 yueshen 那侧的日志数不出这一跳的请求，不能用它判断「查没查」。**这一跳坏了的症状曾经很难看**：工具压根不在清单 → `共 0 步` → 用户只拿到"我来帮你查一下，稍等…" |
| 7 | 知识库 → 嵌入 | OpenAI 兼容 `/embeddings`，`YUESHEN_EMBED_*` 与 Fay 那组**分开**（留空时上游会复用 `gpt_base_url`） | 同上 | 冷换入嵌模型实测 72.9s，所以超时开成 `YUESHEN_EMBED_TIMEOUT`；今天一发 0.6b 嵌入 **2.6s**（已换入） |
| 8 | H5 麦克风 → 外壳 → FunASR | `ws://<同源>/funasr-ws` 按**常量表** `CARECHO_WS_RELAY=/funasr-ws=funasr:10095/` 转发，客户端给的 path 永不进 `getaddrinfo` | 无 | 协议 `{"text","is_final"}`；`ws-relay-test` 14/14、`asr-test` 13/13、`asr-probe` 9/9 |
| 9 | Fay ↔ 远程音频 | TCP `:10001`（裸文本 + `{"vad_need":…}` 那套上游方言） | 用户名注册 | 与第 8 跳**是两套方言、未接通**，所以 `远程音频的 ASR 认出了文字` 那条判据恒 SKIP —— 记的是边界不是故障 |
| 10 | UE → Fay | `ws://:10002`，`Topic:"human"`，UE 拨入；Fay 回帧里带一个**音频文件 URL** | 无 | `probe-selftest` 10/10 假服务端验过协议侧；那个 URL 由 `fay_url` 拼（`fay/0007` 开成 `FAY_URL`），跨机时不设就是"UE 去取它自己的 127.0.0.1" |
| 11 | 后端/Fay → 存储 | `mysql:3306`（`care_echo_rehab`）/ `redis:6379` | 口令 / `REDIS_PASSWORD` | `backend-probe` 的活体写路径判据会真写一行再回读视图，防止"读得到写不进" |

## 慢在哪：预算是单调排下来的

同一件事在链上每一环都有自己的等待上限，**短的那个决定用户体验，长的决定谁先放弃**。
所以这些数不是各调各的，是排过序的：

```
axios 30s  <  外壳 CARECHO_UPSTREAM_TIMEOUT 90s  <  LLM 420s ≤ Fay 回复空闲 480s
             <  adapter 500s  <  后端 520s  <  smoke 600s
```

- **30s 那一条是伙伴方代码写死的**（`src/api/request.js:6`），我们不改它们的项目，
  所以它是**事实**不是参数。
- **90s 是今天仍在红的那条判据撞的墙**。2026-09-23 12:1x 复现：`frontend-probe` 第 3 条
  「我血压有点高，平时该注意什么？」**90.2s → 502**，而同一轮前两条（发产物、`/api/health`）PASS。
  这一问本该走第 6 跳（查知识库），但 `dh-yueshen-rag` 在那十分钟里**只记到 ListTools 心跳、
  一条 `CallToolRequest` 都没有** —— 也就是说 90s 烧在检索之前（Fay 的判断与 LLM 那一发），
  不是嵌入换入：宿主侧嵌模型是 0.6b，事后一发 `/embeddings` 只花 **2.6s**。
  这与 2026-09-22 记的「另一轮 90.1s 撞在外壳那 90s 上」同类。**结论没变**：90s 不该被当成够用的预算。
  同一天 `./run.sh kbq` 在业务口量到一个能把这条墙解释清楚的现象（那一轮打的是那台 26B，
  不是上面那一发 frontend-probe，但机制同一条）：模型有时会先回一句
  「我来帮你查一下，稍等…」然后**改走去调工具**，那一整段沉默实测 76.5s（回帧里是
  `共 0 步`，即工具没被认出来）才把真正的回答和 `<dh-end>` 补上。也就是说**光是这句开场白
  加一次失败的规划就能吃掉 90s 的 85%**，还没算嵌入换入 —— 前端那条 30s 与外壳那条 90s
  都不是余量。
- 420/480/500/520 那几条是给"模型慢慢想"留的，数字定在对话还落在宿主 ollama 的时候：显存被
  占满时 9b 一句话要 172~301s，600s 的 smoke 才等得起；显存充裕时同一句话冷启动 39~40s、暖态 4.8s。
  端点搬到那台 26B 之后单发实测 5.6~18.2s，预算却没跟着砍，两个原因：一问在 fork 的规划器链路里
  可能是**好几发**模型调用（run #2 实测那条链路单轮 180~240s），而这几条真正在保护的是它们的
  **单调顺序** —— 必须先放弃的是浏览器那 30s 和外壳那 90s，而不是链路中间某一环悄悄放弃。
- Fay 那边还有一条 `EMBEDDING_TIMEOUT=90`：仿生记忆检索的换入实测 72.9s，压到 20s 会把
  开机线程堵在重试后面，`:8765` 拖到第 61 秒才 bind（这个坑记在 containerd/README 的 0002 那节）。

## 麦克风与语音的口在哪里

- 生产**只有一个入口**：`ws://<页面同源>/funasr-ws` → 外壳转发 → `dh-funasr:10095`。
  `:10095` 在 prod 不发布宿主端口（今天 `docker inspect` 核过：`null`）。
- 首次启动要下 ~1.3GB 模型（paraformer-large + fsmn-vad + ct-punc），健康检查给了 300s
  `start_period`；那个卷是**缓存不是状态**，删了重拷不影响任何判据。
- **一条必须写下来的质量边界**：15.26s 的真实语音只认出 0.5 字/秒（final 是
  `飞。Ai.Ai.Ai.飞。…`），2.64s 的合成句是 3.4 字/秒 —— 成因是 `CHUNK_BYTES=32000`
  每发一次无上下文 `generate`。判据放的是 0.5~15 字/秒这个量级区间，所以它绿着，但
  **短语音能识别不等于长语音能识别**。
- 还有浏览器那一刀：`getUserMedia` 只在 **https 或 localhost** 是安全上下文，
  所以从手机用 `http://<局域网 IP>:5173` 打开时麦克风按钮必红，那不是本栈的缺陷。

## 端口：默认只在内机上，全开是一次显式决定

| | prod `./run.sh up` | dev `./run.sh dev` |
|---|---|---|
| 渲染出的 `host_ip` | **8 条，全 `127.0.0.1`** | **14 条，全 `0.0.0.0`** |
| 不发布的口 | yueshen 8766、funasr 10095 | ——（这两个口 dev 才有） |
| 保证方式 | `run.sh` 硬拒非 loopback 的 `BIND_ADDR`（直接非零退出） | 覆盖 `BIND_ADDR=0.0.0.0`，含管理口 13306 / 16379 |

dev 是**按设计全开**：同网段任何机器都敲得到 13306，进去的门槛只剩那一发口令，
`:5000` 离了 loopback 就同时暴露 Fay 的管理台和无鉴权的 OpenAI 兼容 façade。
代价写在档位说明里，用完 `./run.sh down`。
复核命令（渲染 + 真跑两份都要看，`config` 绿不等于容器按那份跑着）：

```bash
docker compose --env-file .env -f docker-compose.yml -f docker-compose.dev.yml config | grep -E 'host_ip: ' | sort -u
docker ps --format '{{.Names}}\t{{.Ports}}'; ss -ltn
```

## 上游对表（2026-09-23 12:2x）

| submodule | 钉在 | 对 origin 落后 | 备注 |
|---|---|---|---|
| `fay` | `f702528` | **0** | `upstream`(=`xszyou/Fay`) 领先 1 条：`d49f476 修复websocket与uvicorn版本不兼容问题` |
| `service` | `987f3c3` | 0 | 自有后端，无新提交 |
| `frontend` | `4493fb9` | 0 | gitee 那份，无新提交 |
| `ue` | `5696072` | 0 | 无新提交 |

`./run.sh upstream` 今天复跑：9 份补丁对 `upstream/main` **全部可贴**（`--fuzz=0`，逐份独立预检）。
`d49f476` **本轮没有合**，理由有三条，凑在一起就是"收益为零、代价可见"：

1. 它只动 `fay/requirements.txt`（加 `uvicorn<0.35`），而 `images/fay.Dockerfile` 装的是
   `overlay/fay/requirements-docker.txt`（第 22 行早就钉着同一条）—— **镜像内容一点不变**。
2. `./run.sh audit` 的断言是"四个上游仓库 `main` == `@{u}` 且 `0 dirty`"。
   本地合一次就分叉，那条能进 CI 的断言立刻变红。
3. 合出来的 SHA 不存在于任何远端（`fay` 的 `origin` 是 `chuan918/Fay`，别人的 fork，不推），
   根仓库记这个指针 = 别人 `clone --recursive` 时取不到东西。

要合就一条命令，然后按规矩复跑：`git -C fay merge upstream/main && ./run.sh build && ./run.sh test`
（代价是第 2、3 条，得先决定 audit 那条不变量怎么改口径）。

## 现在还不能说"通过"的事

- **真 UE 对接**：第 10 跳只有假服务端证据 + `ue/` 那 0 处端点引用，缺的是对端实现。
- **`frontend-probe` 第 3 条**：90s 墙上重跑仍红（2026-09-23 又一次），它是**结构**问题不是抖动。
- **完整一轮 `./run.sh test`**：最新一次完整的是 run #32（2026-09-21 22:53~23:27，十三组），
  十四组的完整一轮**还没在有显存的机器上跑过** —— 本轮只复跑了不受显存影响的那几组。
- **`远程音频的 ASR`**（第 9 跳）与 **H5 其余 `chatAPI` 方法**：明写的未覆盖，不是待查。
- 微信小游戏壳要**已备案 https 域名**，是发布资质问题；Xmov 密钥要 `--build-arg` 自己给
  （注意 ARG 值会留在镜像 history）。
