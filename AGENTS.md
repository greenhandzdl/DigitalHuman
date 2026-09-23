# AGENTS.md —— 改这个仓库之前必须遵守的规矩

这份只管**约束和为什么**。"这东西是什么、怎么起来"看 [`README.md`](README.md)，
"接到哪一步、每一跳走什么协议"看 [`INTEGRATION.md`](INTEGRATION.md)，
能照着敲的细节在 [`containerd/README.md`](containerd/README.md)，
决策来龙去脉与每次失败的复盘在 [`containerd/AGENTS.md`](containerd/AGENTS.md)。

## 1. 上游工作树永远零改动

`fay/` `service/` `frontend/` `ue/` 四个 submodule 的工作树必须干净、`main`（前端是 `master`）
必须 == `@{u}`。要改的行为写成 `containerd/patches/*.patch`（构建期）或
`containerd/overlay/` 里的文件（运行期挂载）。

**为什么**：直接改别人代码的代价是每次上游更新都要重做一遍；收进补丁层之后那个代价变成一个
可复核的目录，`./run.sh audit` 把它打成输出，`./run.sh upstream` 顺带预检每份补丁还贴不贴得上
（贴不上就非零退出）。

## 2. 按机器变的事只进 `.env`，仓库里只给示例值

端点、模型名、token、对端真能取到的本机地址、随机口令 —— 一律只出现在 `containerd/.env`
（gitignored）。被 git 跟踪的 `.env.example` / `system.conf` / `overlay/*/*.json.example`
只放"在这台机器上跑得起来"的示例值。

**为什么**：否则换一次部署就得改一次被跟踪的文件，而 git 历史里会留下上一台机器的地址。
运行期会被程序整份回写的文件（Fay 那三份 json）同理：跟踪模板，实值由
`tools/gen_overlay.py` 首跑复制，`tools/gen_keys.py` 管 `.env`。

## 3. `dev` 档位就是绑 `0.0.0.0`，不要再"挑地址"

跨机联调用 `./run.sh dev`：14 条端口全开，含 `13306` / `16379` 与 Fay 的管理面。
`prod`（缺省）由 `run.sh` 硬拒非 loopback 的 `BIND_ADDR`，渲染出 8 条全 `127.0.0.1`。

**为什么**：曾经有过一个"dev 只放开某一个地址"的档位（`DH_EXTRA_BIND` + 一份 extra yml），
2026-09-22 撤掉了 —— 它能表达的事绑 `0.0.0.0` 都能表达，而它多出来的"该填哪张网卡"会让每次
联调先做一次判断。暴露的代价写在档位说明里，不藏起来；用完 `./run.sh down`。

## 4. 能力必须被测试件证明，否则不算存在

判据只有三态：PASS / FAIL / 明确 SKIP + 原因。**没有证据的"通过"不是通过** ——
所以每条判据还配一个"故意改坏它必须变红"的负面自检，所以探针不许读它没接线的函数。

知识库这条链路有额外一条：**必须在 Fay 的业务口测**（`/api/send` + `/api/get-msg`，
即 `./run.sh kbq`），判那一问落库的原始回帧。宿主侧 `docker logs dh-yueshen-rag` 不算证据 ——
检索那一跳走 `faymcp/runtime_bridge` 进程内直调，根本不经过 yueshen 的管理面，
它的日志数不出这一跳。

## 5. 文档里的数字必须带日期，且区分"判据"与"当时的实测"

同一句话在显存被占满时 172~301s、暖态 4.8s，换端点又是另一组数。所以实测值一律写成
「（2026-09-xx 复跑）」，换配置后它们只是历史证据。判据设计要跟着最坏情况，不是跟着最好看的那次。

## 6. 部署相关的第三方凭据：记现象，不擅自"修"宿主

本机 `~/.gitconfig` 有一条把 https remote 改写成带明文 token 形式的 `insteadOf`。
所有者已定过口径：**不轮换、不改全局配置**，推送一律走显式 SSH URL
（`git@github.com:greenhandzdl/…`）。细节与理由见
[`containerd/AGENTS.md`](containerd/AGENTS.md)「宿主机遗留：git 的 insteadOf 里躺着明文 token」。

## 7. 推送是显式决定，不是收尾动作

任何 push / 发布 / PR 之前先确认改动已提交，并过安全审查闸门；不要在一次任务里"顺手"推。
一次授权不代表以后都授权。
