# 05 · 运维：开机自启与端点鉴权

审核（`doctor --live` **0 ERROR / 19 WARN**）暴露两件收尾工作：
**服务不会自愈也不会自启**（拔电/重启后集群是死的，全靠人跑 `cluster start`），
以及 **8000 端口在校园网上无鉴权可达**（从另一台用 WiFi 侧地址实测 `/v1/models` 返回 200，
`ufw` 不活动、nft input policy 是 accept）。两条都已落地，设计取舍记在这里。

## 5.1 开机自启：一个 oneshot unit + 一个 wrapper

```ini
# /etc/systemd/system/dspark-serving.service
[Unit]
Description=DSpark TP3 serving lane (bring the engine up after boot)
# 锁频必须在任何 rank 起来之前生效，否则一台带着全场跑锁存时钟；
# 64 KiB 的 boot helper 必须在引擎要吃内存之前把 swap 选好。
After=network-online.target docker.service containerd.service dspark-gpu-clock.service dspark-roce-mtu.service spark3-kernel-memory.service
Wants=network-online.target docker.service

[Service]
Type=oneshot
User=<deploy-user>
WorkingDirectory=<site-path>
# 今天可选、端点带 --api-key 之后是必需
EnvironmentFile=-<site-path>/.env
ExecStart=<site-path>/scripts/serving-boot.sh
# 等 ssh/docker(300s) + 两次尝试，每次受 profile 的 ready_timeout_seconds(3600s) 约束
TimeoutStartSec=7800

[Install]
WantedBy=multi-user.target
```

`systemd` 表达不了 `cluster start` 的真实前提，所以判断都写在 wrapper 里
（`scripts/serving-boot.sh`，别名按本站 `peer-a` / `peer-b`）：

```bash
# 1) 等对端可达 + docker 就绪：三台同时上电时，network-online 触发那一刻从机还 ssh 不上去，
#    直接 launch 会得到半集群并卡在 post-load barrier
for attempt in $(seq 20); do
  missing=""
  for host in peer-a peer-b; do
    ssh -o BatchMode=yes -o ConnectTimeout=5 -o StrictHostKeyChecking=accept-new "$host" true \
        2>/dev/null || missing="$missing $host"
  done
  docker info >/dev/null 2>&1 || missing="$missing docker"
  [ -z "$missing" ] && break
  log "waiting on:$missing (attempt $attempt/20)"; sleep 15
done
[ -n "$missing" ] && { log "giving up, still missing:$missing"; exit 1; }

# 2) 已有引擎在应答就退让：cluster start 不幂等，boot 抢在人手工起之前不能把现网拆掉
if curl -sf --max-time 10 http://127.0.0.1:8000/health >/dev/null 2>&1; then
  log "engine already answering on 8000, nothing to do"; exit 0
fi

# 3) 起，失败重试一次，再失败就留着不动等人看
python3 bin/spark --cluster-config config/cluster.json cluster start --apply && exit 0
sleep 60
python3 bin/spark --cluster-config config/cluster.json cluster start --apply || {
  log "cluster start FAILED twice; leaving it down for a person to look at"; exit 1; }
```

⚠ **`Type=oneshot` 跑完就 `inactive (dead)`，服务活着 ≠ unit active。**
`systemctl is-active` 永远返回 dead，别拿它当健康判据；看
`systemctl status` 里的 `status=0/SUCCESS`、journal 末行的 `cluster ready`，或者干脆 `curl /v1/models`。
（要 `is-active` 有意义就加 `RemainAfterExit=yes`，本站没加，因为这活儿的语义就是"跑一次"。）

已验证：重复触发打出 `engine already answering on 8000, nothing to do`；
停掉集群后用 `systemctl start dspark-serving.service --wait` 走完整路径 →
`cluster ready; memory guards are active on all nodes (+79.9s)`、`status` 三行 `Up`。
**尚未由一次真实重启证明**——下次重启窗口顺手确认。

## 5.2 端点鉴权：key 不进树，`--api-key` 在 `expected_command()` 里注入

两个不显然但决定设计的点：

**① 这个构建里 `VLLM_API_KEY` 环境变量不被认**（`vllm/entrypoints/openai/api_server.py` 里 0 处引用），
只有 CLI `--api-key` 生效。所以 key 必须进容器的启动命令。

**② key 绝不能写进 `config/cluster.json` 的 `serve_args`。** `build_fingerprint(cluster)`
把容器标签、`/opt` overlay、`environment`（去掉 `NOT_TIMING`）**以及 `serve_args`** 都算进哈希，
改 `serve_args` 会轮转 `dspark_cost_dir`（`/cache/kkref/dspark-costs/build-<fp>`）——
等于凭空制造一次"新构建"，JIT/成本缓存全丢、还会让三台看起来配置漂移。
所以注入点在 `expected_command()`：`list(cluster["serve_args"])` 之后、`--node-rank` 之前追加
`["--api-key", api_key()]`，tracked 配置一行不动。

key 的来源是 node-local、被 git 忽略的 `.env`（`0600`），仓库里只留 `.env.example`：

```bash
# <site-path>/.env（每台各有一份，mode 0600，绝不进树）
DSPARK_API_KEY=<你的 key>
```

```python
# bin/spark
API_KEY_VARIABLE = "DSPARK_API_KEY"
def load_local_env() -> None: ...   # 在 __main__ 入口解析 .env，os.environ.setdefault（不覆盖显式导出）
def api_key() -> str: return os.environ.get(API_KEY_VARIABLE, "").strip()
def auth_headers(headers=None) -> dict: ...   # 有 key 时加 Authorization: Bearer <key>
```

**缺 `.env` 的节点会解析出不同的 expected command**，于是 `doctor` 报漂移，
而不是悄悄起一个无鉴权的 rank —— 这是刻意选的行为：宁可启动失败，不要静默降级成开放端点。

客户端侧全部要跟着改，否则一上 key 采集面就 401、看起来像服务坏了：

| 位置 | 改动 |
|---|---|
| `wait_for_api()` | curl 探针加 `--header "Authorization: Bearer <key>"`；走 `run_ssh`（内部 `shlex.join`），引号安全 |
| `scrape_metrics` / `scrape_metric_series` | `urlopen(Request(f"{base}/metrics", headers=auth_headers()))` |
| `stream_request` | `headers=auth_headers({"Content-Type": "application/json"})` |
| `servebench/auth.py` | 新建，读同一个环境变量；8 个 HTTP 调用点全部过 `auth.headers(...)` |

就绪语义仍然钉在 `/v1/models`（见 `04-…` §4.2），**没有**跟着改成 `/health`。

## 5.3 鉴权上线后的实际暴露面

| 路径 | 无 key | 错 key | 带 key |
|---|---|---|---|
| `/v1/models` | 401 | 401 | 200 |
| `/v1/chat/completions` | 401 | 401 | 200 |
| `/health` | **200** | **200** | 200 |
| `/metrics` | **200** | **200** | 200 |

`/health` 与 `/metrics` 的豁免来自 vLLM 自身，不是漏配。后果要知道：绑定仍是 `--host 0.0.0.0 --port 8000`
（按需求保持公网可达），因此校园网内任何人都能从 `/metrics` 读到模型名、token 计数、KV 用量这类运维信息。
要彻底收只剩防火墙一条路（把监听收到管理网段或加 `ufw` 白名单），本站选择不做。
**如果部署到不可信网络，这是必须改的第一件事。**

## 5.4 一次失败的先例（别照抄第一次尝试）

本站历史上第一次加 bearer token 的提交在 18 分钟后被整条 revert，**提交信息没写原因**。
它和本次实现的三处差异，恰好可能就是它被撤的原因：

1. key 与 `--api-key` 写在 **tracked 的 `config/cluster.json`** ⇒ `build_fingerprint` 轮转、
   成本/JIT 缓存目录像凭空换了一次构建；
2. `deployment.ready_url` 从 `/v1/models` 改成 `/health` ⇒ 就绪判定变松，JIT 没跑完就报 ready，
   下一轮基准取到无意义的数；
3. 只给 `stream_request` 加了头，`/metrics` 抓取和 bench 路径没加 ⇒ 采集侧直接 401，
   看起来像服务故障，容易被误判成"鉴权这功能有问题"而回滚。

教训：**给端点加鉴权是一个横切改动**，注入点、就绪探针、指标抓取、基准驱动、客户端五处必须一起改完，
且不能碰 tracked 配置里的哈希输入。

## 5.5 审核里仍然开着的项目

| 项 | 状态 / 建议 |
|---|---|
| 无值守的 apt 升级会动内核/驱动五件套 | `apt-mark showhold` 是空的 ⇒ 要么 `apt-mark hold linux-image-… linux-modules-nvidia-580-open-…`（含 `-64k` 全套），要么 `systemctl mask apt-daily-upgrade.timer`。64K + 签名 DKMS 的组合对"自动换内核"零抵抗 |
| `dgx_ec_fan_control` 在 64K 上没有包 | 见 `02-…` §2.6；动机是 head 板温 94.9 °C |
| head 的 5.93 GiB 内存线未过 | 见 `04-…` §4.6；运维规定是别在 head 跑交互会话，或把它当已接受偏离写进 §决策 |
| 部署记录单份在 head | 部署目录（勘察+计划+脚本）与站点裸仓需要一份异地备份，拔电/换盘时才会后悔 |
| 协调用的心跳靠 `nohup` 脚本 | 进程一死就静默失效 ⇒ systemd timer 化 |
| `/etc/hosts` 的 GitHub pin | 已删（DKMS 装完后它的唯一用途耗尽）。**将来重装内核五件套或补 fan DKMS，要先重新钉一次**，见 `06-…` §6.4 |
| 桌面未恢复 | 有意维持 headless：head 正是内存最紧的一台，图形栈那约 1 GiB 恰好是没过的红线需要的量 |
| `postfix@-.service` 自 8 月起 failed | 与内核无关；`snapd`、权重 extent 残留 WARN 同类，接受 |
