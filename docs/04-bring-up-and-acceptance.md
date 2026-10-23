# 04 · 起服务与验收

## 4.1 启动序列与两道内存门

```bash
python3 bin/spark cluster sync --apply                        # 三台对齐到同一站点提交
python3 bin/spark site shape tp3-triangle-4k --apply          # 4 KiB 阶段的形状
python3 bin/spark cluster start --apply --unqualified-shape   # 4k 形状需要，见下
python3 bin/spark status
```

- `tp3-triangle-4k` 的 `benchmark_reference` 解析出来是 `None`，所以 `cluster start` **必须带
  `--unqualified-shape`**（自证门，不是缺陷）。切到 `tp3-triangle` 后自带
  `manifests/suites/2026-10-19-karmic-kraken-r6x-tp3` 清单，**不用再带**。
- 启动是**从机先 launch、head 最后**（本站顺序 dgx2 → dgx3 → dgx1），每台起来先挂
  **startup memguard（阈值 5 GiB、0.25 s 轮询、prelaunch）**，API 就绪后三台一起切到
  **steady memguard（阈值 3 GiB、2 s、running）**。`status` 应该是三行 `Up` + `memguard=active`。
- **启动纪律：三台要么全冷起、要么全预热再起。** 冷 page cache 的那台读盘会让其余节点在
  post-load barrier 等到超时（同类多节点实测约 8 分钟）。
- 停服务：`cluster stop --remove --apply --parallel`。**重启机器前一定先停**，
  否则容器死在 collective 里（exit 247），而且对端的 RoCE GID 会漂移。

## 4.2 就绪语义：盯 `/v1/models`，不要盯 `/health`

`deployment.ready_url` 本站钉在 `/v1/models`。理由：

- `/health` 在 API server 起来后立刻返回 200，但那时 **TileLang/JIT autotune 还没跑完**，
  拿它当就绪门会让第一次推理慢得离谱、还会让基准取到无意义的数；
- `/v1/models` 真正代表引擎可用（首次启动里 JIT warmup 单独就占 196.8 s / 185 个调用）。

实测就绪时间：

| 口径 | API ready | 备注 |
|---|---|---|
| 4 KiB / 262K 冷 | **+305.0 s** | 其中 JIT warmup 196.8 s |
| 64 KiB / 1M 冷 | **+214.8 s**（cluster ready +215.6 s） | 配方 boot-test 参考约 121 s |
| 64 KiB / 1M，JIT 缓存热 | **+84.1 s**（走自启 unit 复测 +79.9 s） | 同机重启/重起的正常口径 |

⚠ 另外：**`/health` 与 `/metrics` 在带 `--api-key` 时仍然免鉴权返回 200**（vLLM 自身的豁免），
所以自启 wrapper 用 `/health` 判「是否已有引擎在应答」是安全的，而 `wait_for_api()` 必须继续用
`/v1/models`。详见 `05-…`。

## 4.3 第一次起不来的真实原因：显示预留

```
RuntimeError: SPARK3_DISPLAY_CARVEOUT_WEIGHTS=1 but the display carve-out cannot hold a 421 MiB
weight (/dev/dri/card0: [Errno 12] Cannot allocate memory)
```

配方会自动回滚并把三台日志留在 `results/private/failed-starts/<时间戳>-<容器>/{dgx1,dgx2,dgx3}.log`。
根因是 GNOME 占着 `/dev/dri/card0`，显示预留里放不进权重。⇒ **headless 不是"洁癖"，是硬前提**：
`systemctl set-default multi-user.target` + 配方的 `scripts/host-recovery apply`。要还原桌面：
`systemctl start gdm`（注意 `gdm3.service` 只是 alias，alias 不支持 `start`，见 `06-…`）。

## 4.4 验收：第一轮（4 KiB / 262K，`tp3-triangle-4k`）

```bash
python3 bin/spark bench                      # 默认门 + decode
python3 bin/spark bench --suites prefill
```

对线口径：配方给的 4KiB/262K 列。

| 指标 | 线 | 本站实测 | 判定 |
|---|---|---|---|
| quality gate | 5/5 | **LRU 5/5** | 过 |
| 单流 prose c1 | ~52 tok/s | **54.5 ±1.2%**（step 38.28 ms） | +4.8% |
| 单流 code c1 | ~63 tok/s | **67.4 ±0.1%**（step 41.21 ms） | +7.0% |
| c8 prose / code | 198.6 / 240.3 | **253.1 ±5.0% / 247.5 ±0.1%** | +27% / +3.0% |
| prefill 32K | ~4,300 tok/s | **5,366 ±0.5%** | +25% |
| prefill 65K / 128K | — | 5,275 ±0.5% / 5,078 ±1.1% | 未塌陷 |
| KV 容量 | 1,348,708 tok（5.1 窗） | **1,348,708 tok、并发 5.14x**（引擎日志原话） | 逐字节相等 |
| 内存红线 head | ≥ 6.38 GiB | bench 期间最低 **6.94 GiB**，从机 ≥ 8.63；swap growth **0.0 GiB** | 过 |
| 权重占用 | 94.26 GiB/机 | 同 | 过 |

**七项全过线**（2026-10-22 19:4x）。

## 4.5 验收：第二轮（64 KiB / 1,048,576，`tp3-triangle`）—— 阶段终判

对线口径是配方的 **r6x/1M** 列。除默认门 + decode + prefill，另跑 `--suites admission`
（4 × 500,000 token 长上下文准入）。

| 指标 | 线（r6x/1M） | 本站实测 | 判定 |
|---|---|---|---|
| quality gate | 5/5 | **LRU 5/5**，bench 前置 **0 ERROR** | 过 |
| 单流 prose c1 step | 38.21 ms | **36.72 ±0.6% ms** | 快 3.9% |
| 单流 code c1 step | 42.43 ms | **39.75 ±0.3% ms** | 快 6.3% |
| 单流 prose c1 decode | ~52 tok/s | **56.9 ±0.6%** | +9.4% |
| 单流 code c1 decode | ~63 tok/s | **69.9 ±0.4%** | +11.0% |
| c2 prose / code | — | 87.3 ±0.2% / 104.1 ±5.6% | — |
| c4 prose / code | — | 155.5 ±1.8% / 180.4 ±8.1% | — |
| c8 prose | 198.6 | **270.7 ±0.6%** | **+36.3%** |
| c8 code | 240.3 | **257.3 ±3.1%** | +7.1% |
| prefill 32K | 4,306 | **5,344.8 ±0.1%** | **+24.1%** |
| prefill 2K / 65K / 128K | — | 4,464.5 ±2.3% / 5,265.2 ±0.4% / **5,086.6 ±0.5%** | 未塌陷 |
| KV 容量 | 4,765,898 tok（4.55 窗）/ 4.91 GiB | **4,765,898 tok、并发 4.55x、每 rank 预留 4.91 GiB**（引擎日志原话） | 逐字节相等 |
| 长上下文准入 | — | **4 × 500,000 pass**，peak KV **37%**，单流 decode 24.65 tok/s | 过 |
| 权重占用 | 94.26 GiB/机 | 同（`Initial free memory 113.42 GiB` − KV 4.91 GiB 对得上） | 过 |
| swap growth | — | 三台全程 **0.0 GiB**（`/swap-64k.img` 挂着没动） | 过 |
| 启动时间 | ~121 s | 冷 **214.8 s** / JIT 热 **84.1 s** | 过 |
| **内存红线 head** | **≥ 5.93 GiB** | decode 5.51 / prefill 5.39 / admission **4.74 GiB** | **未过（最低 −20%）** |

引擎日志里与口径逐字节一致的三行，值得留档：

```
Initial free memory 113.42 GiB, reserved 4.91 GiB memory for KV Cache
GPU KV cache size: 4,765,898 tokens, Maximum concurrency for 1,048,576 tokens per request: 4.55x
init engine (profile, create kv cache, warmup model) took 74.87 s
```

## 4.6 唯一没过的那条线，归因是可查的

head（dgx1）比从机多扛 **`VLLM::EngineCore` 1.50 GiB + `vllm` 0.49 GiB 增量**（rank 0 固有），
另外还挂着**跑这套测量本身的交互式 CLI 会话 0.54 GiB** 和 docker/exec 探针。admission 结束时
三台容器 RSS 是 **head 7.41 / dgx2 6.29 / dgx3 6.31 GiB**，正好是那约 1.1 GiB 的差。

空载服务态 MemAvailable：head 6.42 / dgx2 7.26 / dgx3 7.40 GiB（红线 ≥ 5.93，过）；
压测后回到 5.09 / 7.16 / 6.60（head 没完全回弹，说明一部分是 KV/元数据水位而非纯 page cache）。

**离 steady memguard 的 3 GiB 闸还剩 1.7 GiB**，所以这不是"随时会被自己的守护杀掉"的状态，
但确实比配方可预期的余量薄。结论性运维规定：**长上下文压测期间不要在 head 上跑交互会话/采集**，
把测量挪到从机或外部机器上做。

## 4.7 散热与方差（决定你的数字能不能直接比）

- **散热不是瓶颈**：两轮 bench 三台 `thermal slowdown 0.0 ms`；GPU 峰值 70 / 69 / 68 °C
  （离各自限值还有 22–24 °C）；功率峰 45.6 / 46.1 / 44.2 W。
  但 **head 的板温峰值 94.9 °C**（第一轮 91.9 °C），原因就是站点 `fan_control=false` 让 bench
  **跳过降温曲线、不等预冷** —— 于是读数偏乐观而不是偏悲观，而板温余量变小。
  这是把 `dgx_ec_fan_control` 补到 64 KiB 内核的第一动机。
- **方差集中在 code 侧并发点**：c2 ±5.6%、c4 ±8.1%（round 2 曾到 ±38.85%/±66.06%，靠 round 3–4 收敛）；
  prose 全点 ≤1.8%。第一轮里那个 `code-c2 ±18.6%`（step ±21.2%）在 64 KiB 口径下没复现。
  被拒的采样会留在报告 JSON 的 `discarded_samples` 里，引用并发点前先看 CI 列。
- 基准前置是 **fail-closed** 的：`bench` 会先比对集群实际状态与配置，任何偏差（包括时钟锁存）
  直接报 `refusing to benchmark a cluster that differs from its configuration` 拒跑。
  这条保护让"慢 45% 的数白取"变成不可能 —— 但也意味着**报错先看 doctor 的 ERROR，不要绕**。

## 4.8 推理调用的一个坑

这个模型默认开思考。`max_tokens` 给小了会返回 `content: null`、`reasoning_tokens` 占满。
要短回答得显式关思考：

```json
{"model": "deepseek-v4.1-flash", "messages": [...], "thinking": {"type": "disabled"}}
```
