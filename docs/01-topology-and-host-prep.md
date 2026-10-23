# 01 · 三机勘察与主机预置

## 1.1 起点状态（只读实测，三台一致的部分只写一次）

| | dgx1（head） | dgx2 | dgx3 |
|---|---|---|---|
| 芯片 / 架构 | GB10 / ARM64 | 同 | 同 |
| 内核 / 页 | `7.0.0-1019-nvidia` / 4096 | 同 | 同 |
| GPU 驱动 | 580.178.04 | 同 | 同 |
| DGX OTA | 7.6.0（镜像年份分别 2026-03 / 2025-11 / 2025-09，SWBUILD 不同但关键栈一致） | | |
| 内存 total / avail | 121.7 / 115.0 GiB | 121.7 / 116.0 | 121.7 / 117.6 |
| 根盘空闲 | 3.0 TB | 2.5 TB | 3.5 TB |
| ConnectX-7 | 4 口全 `ACTIVE 200 Gb/sec (4X HDR)` | 同 | 同 |
| Secure Boot | 开，且 `mokutil --list-enrolled` = **0 个密钥** | 同 | 同 |
| 免密 sudo / docker 组 | 都没有 | 都没有 | 都没有 |
| 启动目标 | `graphical.target`（GNOME 常驻） | 同 | 同 |

**结论：硬件和 fabric 已经就绪，卡点全是 host 配置。** 唯一真正要人工的是 Secure Boot 下的签名模块。

## 1.2 网络：三角已经现成，不要改成环

每台 4 个 200G 口，接线已经构成**每对节点各两条独立链路的全互连三角**：

```
dgx1(enp1s0f0np0)   10.100.152.2 ↔ dgx2 10.100.152.1
dgx1(enP2p1s0f0np0) 10.100.153.2 ↔ dgx2 10.100.153.1
dgx1(enp1s0f1np1)   10.100.156.1 ↔ dgx3 10.100.156.2
dgx1(enP2p1s0f1np1) 10.100.157.1 ↔ dgx3 10.100.157.2
dgx2 ↔ dgx3         走 10.100.154 / 10.100.155 两条
```

连通性实测：head→4 个对端 IP 全 0% 丢包（RTT 0.82~1.16 ms），第三边也 0% 丢包。**无需重新布线。**

管理面：三台的校园网上行都走 WiFi，1G 铜口 `enP7s7` 是 DOWN，三角里每个 /24 只连两台，
所以 `distributed.master_addr` 与 `GLOO/TP_SOCKET_IFNAME` 用**跨三台可达的那条覆盖网接口**
（本站是 `tailscale0`，实测 1~2 ms），SSH 仍走 RoCE 别名。配方默认的 `NCCL_SOCKET_IFNAME=enP7s7`
在本机不存在，必须改。

> 拓扑判定：作者原话「交换机从来不是瓶颈，引擎和调优才是」，其同硬件环 vs 交换差距 2~5%。
> 三台三角比 4 节点环少一跳、且不需要 opposite-node 硬件转发补丁。**不要为了照抄某个多节点帖
> 把三角改成环。**

## 1.3 差距清单（配方要求 → 现状 → 最终动作）

| # | 项 | 配方要求 | 实测现状 | 动作与结论 |
|---|---|---|---|---|
| G1 | DGX OS | 26.09.2+ | 三台 OTA 7.6.0、内核/驱动一致、apt 干净 | 以 `bin/spark doctor --live` 复核为准，通过 |
| G2 | 布线 | switchless 三角 | 已满足 | 只做寻址校验 |
| G3 | 链路 MTU | 9000 字节 DF ping 通 | 全部 **1500** | **必须写进 netplan**，见 1.4 |
| G4 | 页大小 | 512K/1M 需 64k 内核 | 4096 | 阶段 2，见 `02-…`；4 KiB 可先用 `tp3-triangle-4k` 绕过 |
| G5 | memory saver | 64 KiB profile 必需 | 未装 | DKMS + MOK 签名，见 `02-…` |
| G6 | 免密 sudo | head→各节点 passwordless SSH **and sudo** | 均无 | `/etc/sudoers.d/` 一行；**注意 1.5 的 stdin 坑** |
| G7 | docker nvidia runtime | Docker + NVIDIA runtime | 两台未注册；有一台的 `nvidia` 用户不在 docker 组 | `nvidia-ctk runtime configure` + `usermod -aG docker`（旧会话不生效，要重连） |
| G8 | 磁盘 | ~480 GiB/台 | 2.5~3.5 TB | 已满足；按 700 GiB 备更稳 |
| G9 | 权重 | 每台一份，约 476~510 GB | 都没有 | **只下一份再分发**，见 1.6 |
| G10 | headless | 纯命令行 | graphical.target | GNOME 白吃约 6 GiB，直接顶到内存红线 |
| G11 | 锁频 | 三台一致 | 一台 P0、两台 P8 | `nvidia-smi -lgc 0,2200` 做成 systemd unit；TP3 每层两次 all-reduce，一台降频全场空等 |
| G13 | GPUDirect RDMA | 未要求 | GB10 的 `libmlx5` 缺 dmabuf 注册，NCCL 自动回退 | **不用追、不用排查**，省一整轮弯路 |
| G14 | L2 weight prefetch | 只在 one-shot RoCE 路径有收益 | — | 走 NCCL 传输时**别开**（实测无效）；切 sparknet one-shot 再开，约 −1 ms/step |
| G15 | NVMe 中断合并 | 关 | 三台 enabled，每次开机自己打开 | `systemctl mask --now` + 跑一次 disable 脚本。Engram 的小磁盘读要等 8 完成/100 µs 定时器 |
| G16 | NVMe APST | 2000 / 60000 ms | 默认 **100 / 2000** | 运行时写 sysfs + `/etc/modprobe.d/` + `update-initramfs -u`（`nvme_core` 从 initramfs 加载，只写 modprobe.d 不重建等于没写）。作者实测 8 流时 GPU 为等 Engram 行耗过 **1.5 s**，保持唤醒吞吐高 3% |
| G17 | `vm.watermark_boost_factor` | 0 | 15000 | `/etc/sysctl.d/` 一行（配 `vm.swappiness=10`） |
| G18 | nvidia-drm modeset | `modeset=Y`，**对 4 KiB 也 required** | DGX 装了 `nvidia-drm-options-modeset0` 把它压成 0，doctor 报 ERROR | `apt purge nvidia-drm-options-modeset0`（那个 override 文件只属于这个包）。**不必重启也能收**：purge 后 `host-recovery apply` 会触发模块重载，三台复核 `modeset=Y` |
| G19 | headless 细化 | 关空闲服务 + 文本控制台 | gdm 在跑，蓝牙/CUPS/snapd/dgx-dashboard 常驻（DGX 仪表盘轮询更新峰值近 850 MiB） | 配方自带 `scripts/host-recovery apply` 一次搞定（含 watchdog、SSH/覆盖网保护、文本控制台） |
| G20 | fan-floor DKMS | doctor 报 ERROR；`bench` 没它会跳过降温曲线 | 三台都没装，且 0 个 MOK | 短期 `host.fan_control=false`（文档明说 "serving does not need it"），代价见 `04-…` 的散热段 |

`config/cluster.json` 里写**顶层 `host`** 就够了：`scripts/recipe.py` 用深合并把 shape 层的 `host`
覆盖并进 recipe 的 `host`，而 `tp3-triangle-4k` 那层只覆盖 `host.kernel`，不会吃掉 `fan_control` 这个键。

## 1.4 MTU 必须落在 netplan，否则每次开机被打回

`dspark-roce-mtu.service` 这类 oneshot 会在开机早期把 4 个口设成 9000，但 NetworkManager 随后
`activate` netplan 生成的 profile，按 profile 默认值改回 **1500**（journal 里两口
`state change: config → activated` 的时间戳都晚于 unit 的 `ActiveEnterTimestamp`）。
`nmcli connection modify … 802-3-ethernet.mtu 9000` 也无效：netplan 生成的 profile 落在
`/run/NetworkManager/system-connections/`，重启就重新生成。**唯一不被覆盖的地方是 netplan 本身。**

落地写法（先校验再应用，失败自动回滚）：

```bash
cp /etc/netplan/99-nvidia-sync-cluster.yaml{,.bak}
# 每个 RoCE 口插入   mtu: 9000
netplan generate            # 语法/字段校验，失败就 restore .bak 并退出
netplan apply
systemctl enable dspark-roce-mtu.service   # 保留当保险，不是主力
```

复核证据（三台一致）：4 口 `mtu=9000`；`ibv_devinfo` 四个设备 `active_mtu=4096`；
`ping -M do -s 8972` 三角三条边各自 0% 丢包（head↔dgx2 0.43 ms、head↔dgx3 0.85 ms、dgx2↔dgx3 0.89 ms）。
`netplan apply` 只让这 4 个口重新激活（1~2 s 抖动），管理口走覆盖网，实测地址全部保留。

## 1.5 批量改远端主机的三条硬规则（我们因此静默半程失败过一次）

同一条预置命令，实际只有 head 被配置，两台从机**一项都没落**（`sudo_nopasswd=NO`、四口仍 1500、
IB 仍 1024、coalescing 仍 enabled、modeset0 仍 INSTALLED、用户仍不在 docker 组）。三个独立成因：

1. **别把脚本喂进 `ssh -t host "sudo bash -s" < 脚本`。** `-t` 分配 tty 后，sudo 会把从 stdin 灌进来的
   **脚本正文当密码读掉**（结尾 `1 incorrect password attempt`，但 `bash -s` 已经开始执行、退出码还是 0）。
   远端 sudoers 静默不装，之后每个 `sudo -n` 全部空转。
   修法：**密码走 tty，脚本走磁盘** —— 先 `scp` 到远端，再 `ssh -t host "sudo bash <磁盘路径> <子命令>"`，
   装完立刻 `ssh host 'sudo -n true'` 验证，并用一个 `OK_<node>` 门拦住后续步骤。
2. **不需要密码的那几处同样有隐患。** `ssh host "sudo -n bash -s" < 脚本` 里若含 `apt-get purge`，
   dpkg 维护脚本也读 stdin，同样能把还没执行的下半截吞掉。
   修法：所有远端调用统一走一个 helper，一律按磁盘路径调用，不再喂 stdin。
3. **别把「退出码 0」当成「配置生效」。** 上面的失败全程 rc=0。每一项都要读远端状态复核
   （`sudo -n true` / `ip link show` / `ibv_devinfo` / `getent group docker`），复核脚本本身才算验收。

## 1.6 权重分发：一份下载 + 邻居并行 push

- 1017 原生权重 = **475.3 GiB / 48 个 safetensors**；仓库里没有独立 Engram 表文件，
  Engram 是 `table_memory:disk` 在本地 NVMe 落表。
- **限流是按出口 IP 共享的**：head 独占时 59 MB/s，另一台同时开 4 路会把 head 挤到 3 MB/s。
  所以「三台各下 1/3」不提速，只会把 WAN 流量从 476 GB 变成 1.4 TB。
- 分发用**对端自己的邻居并行 push**，直连链路实测 ~1 GB/s/链路，480 GiB 每台约 8~10 分钟。
  **不要用 `ssh -J` 经中继**：中间节点重新加密，掉到 ~300 MB/s。
- 磁盘预算按每台 **700 GiB** 备（权重 476 + 落盘 Engram 表约 200 + build/wheel 缓存）。
- 下载与 `build prepare` **不要并行**：clone + pip 会和权重抢同一条被限流的出口，把 7 小时拖成十几小时。

## 1.7 ext4 extent 碎片（可选，但便宜）

doctor 会 WARN 权重文件的 extent 数远高于最佳值（实测 2721 / 494 / 1327 → 最佳 265）。
`e4defrag` 三台并行跑，每台约 2 分钟降到 **294 / 317 / 317**。两个坑：这个版本 `e4defrag`
**不认 `-q`**（带上就每个文件都失败、只打 usage），而且它的进度条是同一行 `\r` 刷新，
不吞掉 stdout 日志会变成几万字符一行。Engram 落表和权重加载都吃顺序大块读，残留 21~30 个
高分文件的 WARN 接受即可。
