# TriSpark：三台 DGX Spark 无交换机三角，TP3 原生精度跑 DeepSeek-V4.1-Flash-1017

本仓库是三台 NVIDIA DGX Spark（GB10 / ARM64 / 128 GiB 统一内存）以 **tensor parallel 3** 部署
`deepseek-ai/DeepSeek-V4.1-Flash-1017` **原生精度**、把上下文推到 **1,048,576 token** 的全套落地方案：
主机预置清单、64 KiB 页 + memory-saver 的内核路线、站点发布与镜像链、启动/验收口径、运维自启与鉴权，
以及踩过的坑和恢复手册。所有数字都是本站实测，不是引用他人。

上游配方：`christopherowen/spark-ds41f`，生产 baseline `2026-10-19-karmic-kraken-r6x`
（TileLang SM121 内核 + DeepSeek TileKernels + sparknet one-shot RoCE + DSpark 5 草稿）。
本站形状：`tp3-triangle`，模型 revision 原生权重，单机权重占用 94.26 GiB。

## 结论（可以直接抄的部分）

| 项 | 值 |
|---|---|
| 拓扑 | 每台 4 个 ConnectX-7 (200 Gb/s HDR)，构成每对节点两条独立链路的全互连三角，**无交换机、无需改布线** |
| 内核 | `7.0.0-1019-nvidia-64k`（`CONFIG_ARM64_64K_PAGES=y`），4 KiB 内核留作 GRUB fallback |
| 内存 | `MemTotal` 121.69 GiB (4K) → **123.79 GiB (64K)**，+2.10 GiB 纯粹来自页表开销下降 |
| 上下文 | 1,048,576 token；KV 池 4,765,898 token = 4.55 个整窗，每 rank 预留 4.91 GiB |
| 单流 decode | prose **56.9 tok/s**（step 36.72 ms）/ code **69.9 tok/s**（step 39.75 ms），比配方给出的线快 9~11% |
| 8 流聚合 | prose **270.7** / code **257.3 tok/s** |
| prefill | 32K **5,344.8 tok/s**；128K 5,086.6，全程未塌陷 |
| 长上下文准入 | 4 × 500,000 token 并发通过，peak KV 37% |
| 质量门 | 配方自带 quality gate **5/5** |
| 就绪时间 | 冷 214.8 s / JIT 缓存热后 **84.1 s**（配方 boot-test 参考 ~121 s） |
| 开机自启 | `dspark-serving.service`（oneshot，等 docker + 等对端 SSH，已服务则退让） |
| 端点鉴权 | `--api-key` 保护 `/v1/*`，key 存在 node-local 且被 git 忽略的 `.env` 里 |

## 值得先读的两段

- **三台里至少一台会得 GB10 时钟锁存**（serving 态钉在 500~900 MHz、`nvidia-smi -lgc` 无效、
  热重启无效、Clocks Event Reasons 全部 Not Active）。唯一确认有效的解法是**拔 AC 1~5 分钟**。
  见 `docs/06-recovery-playbook.md`。不知道这一点会白跑一整轮基准（慢约 45%）并误判成配置问题。
- **`tp3-triangle`（64 KiB）需要 Secure Boot 下签名加载的 DKMS 模块**（`dgx-spark-memory-saver`）。
  三台出厂都没有注册任何 MOK 密钥，所以这条路必然要求**每台一次控制台人工操作**，而且顺序是
  「人先站到屏幕前，再 reboot」——MokManager 的注册请求会被下一次上电消费掉且不保留。

## 目录

| 文件 | 内容 |
|---|---|
| `docs/01-topology-and-host-prep.md` | 三机勘察、20 条差距清单（配方要求 → 现状 → 动作）、host 预置的持久化写法（netplan MTU / 锁频 / NVMe / sysctl / headless / docker runtime / 免密 sudo） |
| `docs/02-kernel-64k-and-memory-saver.md` | 64 KiB 内核五件套与版本后缀、GRUB 默认项、boot-support 脚本的上游 bug、swap 选择、MOK + DKMS 全流程、`srcversion` pin 校验、一次性试起判据 |
| `docs/03-site-and-image-chain.md` | 站点发布链（本地裸仓当 `deployment.repository`、`origin` 用别名而非 IP、`cluster sync` 会把 head 自己 detach 的坑、浅克隆、dirty 硬门）、基镜像 digest 与镜像站不忠实、构建与 `save\|load` 改 image ID |
| `docs/04-bring-up-and-acceptance.md` | 启动顺序与就绪语义（为什么盯 `/v1/models` 而不是 `/health`）、启动门/稳态门、两轮验收实测全表、唯一未过线的归因 |
| `docs/05-operations.md` | 开机自启 unit 与 wrapper 设计、bearer key 注入方式（以及为什么不能写进 `serve_args`）、bench/采集侧的鉴权头、`doctor` 审计与遗留项 |
| `docs/06-recovery-playbook.md` | 时钟锁存恢复、显示预留导致的启动失败、`gdm3` alias 导致 head 黑屏、MokManager 时序、GitHub edge 与 `/etc/hosts` 权宜、远端后台任务的正确姿势 |

## 本文档刻意不写的内容

节点序列号/主机名、管理网与校园网 IP、Tailscale 地址、API key 与任何凭据、MOK 注册口令、
WiFi 802.1X 身份口令、心跳协调文件。它们属于 node-local，写进仓库就再也收不回来。
文档里出现的 `<head-mgmt>`、`peer-a` / `peer-b`、`<site-bare-repo>` 都是这类占位。

## 复现的代价预期

- 一次重启窗口（modeset + headless，配方对 4 KiB 也标 required），加两次逐台 64 KiB 上电；
- 每台一次控制台的 MokManager 注册（无人可代）；
- 475.3 GiB / 48 个 safetensors 权重只需下载一次，再从 RoCE 并行 push（直连约 1 GB/s/链路，
  每台 8~10 分钟）；**别三台各自走 WAN 下载**，出口是按 IP 共享限流的；
- 盘上按每台 700 GiB 备着（权重 476 + Engram 落表约 200 + build/wheel 缓存）。
