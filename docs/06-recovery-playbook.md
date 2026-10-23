# 06 · 恢复手册（按"现象 → 判据 → 处置"排）

## 6.1 GB10 时钟锁存：唯一有效的是拔 AC

**现象**：serving 态某台（本站两台）SM 时钟钉在 500~900 MHz，`nvidia-smi --query-gpu=...` 的
**Clocks Event Reasons 全部 Not Active**，功耗只有 8~10 W，温度 36–41 °C。

```
dgx1  611–923 MHz（均值 642） / 峰 7.99 W
dgx2  884–890 MHz（钉死）      / 峰 10.46 W
dgx3  2177–2184 MHz            / 峰 16.78 W     ← 健康的那台
```

**先排除三条假因**（我们都排过，都不是）：

| 假因 | 证据 |
|---|---|
| 锁频没下发 | 三台 `dspark-gpu-clock.service`（`-lgc 0,2200`）都 `active`，同一条 unit 在 dgx3 上稳在 2184 |
| 功耗墙 / 温度墙 | `Current/Requested Power Limit` 全 N/A；SW Power Cap / HW Slowdown / Thermal 全 Not Active；温度 36–41 °C |
| 手动加锁能救 | `-lgc 2200,2200` 执行成功但**仍 611 / 890 MHz** |

**判据必须在起服务之后取。** 冷启动后立刻读时钟会看到 **208 MHz**，那是空载 DVFS 的正常值，
不是锁存；反过来说，"重启过了所以应该好了"是错觉——**热重启（`reboot`）对本故障无效**，
我们两台各重启过两次，数值一点没动。

**处置**（按配方的 recovery 文档顺序，无一条可跳过）：

```bash
python3 bin/spark cluster stop --remove --apply --parallel   # ① 先停，防对端 RoCE GID 漂移
# ② 复核：容器 0、nvidia-smi compute apps 0、/dev/nvidia-uvm 持有者 0、memguard unit 0
systemctl poweroff                                            # ③ 关机
# ④ 拔 AC 至少 1 分钟（1~5 分钟）                            ← 只有这一步能解锁
```

复核解锁：上电、起服务，**服务态**读三台时钟回到 ~2180–2200 才算解开（本站 2190 / 2184 / 2184）。
没解开就是拔电时间不够，重来。解锁后 bench 前置从"两台 ERROR 拒跑"变成 0 ERROR。

对现网的影响要知道两件事：

- 配方自己的 `bin/spark bench` 是 **fail-closed** 的：`clock_latch_problems()` 在服务态读到
  时钟低于阈值就报 **ERROR**，`bench` 停在 `refusing to benchmark a cluster that differs from
  its configuration`。所以**你不会取到慢 45% 的数**，但会取不到数 —— 先去解锁，别绕这条保护。
- 如果 head 就是那台要拔电的机器：拔电会带走你当前的 SSH 会话和一切 `nohup` 后台协调进程，
  上电后第一件事是重新 arm 心跳/协调，再去确认集群。

## 6.2 启动失败：显示预留 / 桌面

| 现象 | 根因 | 处置 |
|---|---|---|
| rank 0 `RuntimeError: SPARK3_DISPLAY_CARVEOUT_WEIGHTS=1 but the display carve-out cannot hold a 421 MiB weight (/dev/dri/card0: [Errno 12] Cannot allocate memory)` | GNOME 占着 `/dev/dri/card0`，显示预留里放不进权重 | 三台 headless：`set-default multi-user.target` + 配方 `scripts/host-recovery apply` |
| `cluster start` 报 `refusing to benchmark/start a cluster that differs from its configuration`，触发项是时钟 | 见 6.1 | 拔 AC |
| 卡在 post-load barrier 数分钟然后超时 | 三台冷热不一致，冷 page cache 那台读盘慢 | 要么全冷起要么全预热再起；必要时先在三台各顺序读一遍权重 |
| 容器死在 collective、exit 247 | 重启机器前没停服务 | 重启前 `cluster stop`；日志留在 `results/private/failed-starts/<时间戳>-<容器>/{dgx1,dgx2,dgx3}.log` |
| 一台 `nvidia-smi` 温度/功耗 `ERR!`、`Memory-Usage: Not Supported`、`-lgc` 返回 `GPU requires reset` | GPU 进入需 reset 状态 | 一次干净重启即可（本站自己好了，`dspark-gpu-clock` 从 failed 变 active）；重启后必须复验 `clocks.sm`/`pstate` 出数 |

## 6.3 head 黑屏：`gdm3.service` 是 alias，alias 不支持 `start`

跑配方的 `host-recovery apply` 会 `systemctl disable --now gdm3.service`（它要 headless），
而如果本机还留着 `nvidia_drm.modeset=N`（`apt purge nvidia-drm-options-modeset0` 只在下辈子生效）、
又没有 fbcon 文本控制台 ⇒ **接的显示器全黑**。

更坑的是补救命令本身写错：`systemctl start gdm3` 看着像生效，实际
`gdm3.service` **只是 alias**（真名 `gdm.service`，状态 indirect/enabled），而
**alias 不支持 `start`**，报错信息还是假的。正确姿势：

```bash
systemctl start gdm                       # 不是 gdm3
systemctl status display-manager.service  # 确认符号链指向 gdm.service
# 本镜像里 gdm.service / display-manager.service 两个 unit 都没有 [Install] 段，
# `systemctl enable` 必然失败；这条链是靠 graphical.target 的 Wants= 建立的。
```

复核到会话层：`default=graphical.target ; display-manager=active`，gnome-shell 在跑、
`seat0` 的 tty2 会话正常。**注意：再跑一次 recovery 仍会有一小段黑屏窗口**（关 gdm → 再拉起），属预期。

我们的运维结论相反：**桌面不恢复**。head 正是内存最紧的一台（服务态 5.00 GiB），
图形栈那约 1 GiB 恰好是没过的内存红线需要的量。

## 6.4 出不了 GitHub：edge 时通时不通，一次性 `/etc/hosts` 权宜

**症状**：三台到 GitHub 的可达性完全不同 —— 一个节点完全不通；一个节点注册前通、
**重启后换了 DHCP 地址就不通**；head 网页端点 200 但 **git 端点直接挂 60 秒**。

真相：DNS 把 `github.com` 指到一个**地区 edge**（本站是新加坡），校园网到它时通时不通；
而另一组 `140.82.113.x / 140.82.114.x` 三个地址三台都通、也能 clone。

现场权宜（DKMS 的 `git fetch origin` 依赖它）：

```bash
echo '140.82.113.3 github.com' >> /etc/hosts        # 每台
```

⚠ 两条必须记住：

1. **`install-dkms.sh:14` 断言 `git remote get-url origin` 逐字符等于 pin 里的 GitHub URL**，
   所以**不能**用 `insteadOf` 把 GitHub 重写成本地裸仓来绕（`get-url` 会应用重写，断言必挂）。
   只能让 GitHub 真的可达。
2. 这是权宜，不是配置。DKMS 装完就该删（我们收工时删掉了：整行精确匹配 `sed -i '/^…$/d'`，
   diff 确认唯一改动就是那一行，权限 `644 root:root` 不变）。
   **但将来重装内核五件套、或补 fan-control 的 DKMS，要先重新钉一次。**
   顺带：到不了 GitHub 的那台两次 `rc=128` 都死在这一步的 fetch 上，**都在 `dkms add` 之前，没有半装状态** ——
   遇到 fetch 失败直接重试，不用担心脏状态。

同类网络事实（省一整轮排查）：`raw.githubusercontent.com` 在本网络被封但配方不依赖它；
`git ls-remote` 那几个大仓要给 >30 s 超时否则会假报失败；docker registry 的 401 是未鉴权 ping 的正常响应。

## 6.5 MokManager：人先到屏幕前，再 reboot

已在 `02-…` §2.4 详述，这里只留作恢复清单里的硬提醒：
`mokutil --import` 排的队列由 shim 在**下一次上电**消费，**没人操作就会被消耗掉且不保留**。
SSH 补不了这一步，必须现场键盘。所以任何"要注册 MOK"的窗口，先安排人，再按重启按钮。

## 6.6 远程批量操作的两个技术坑

1. **别把脚本喂 stdin 给 `sudo bash -s`**（`-t` 会把脚本正文当密码读掉，rc 还是 0）。
   密码走 tty、脚本走磁盘，见 `01-…` §1.5。
2. **SSH 上后台化要用 `setsid`/`nohup`，别用 `( … ) &`。**
   `ssh host "( long-running ) &"` 在远端**什么都不留下**：我们做一次跨三台的负载采样，
   只有 head 落出了 trace 文件，两台从机的采样进程静默消失，白跑一轮。

## 6.7 回滚点（每一层都能单独退）

| 层级 | 回滚动作 | 代价 |
|---|---|---|
| 服务 | `cluster stop --remove --apply --parallel` | 秒级，随时可 |
| 形状 / 上下文 | `site shape tp3-triangle-4k --apply`（回到 4K/262K） | 需重启服务 |
| 页大小 | GRUB 选回 `7.0.0-1019-nvidia`（4 KiB），**权重和镜像不用动** | 一次重启；`/swap.img` 会重新成为活动 swap |
| 内核五件套 | `apt remove` 掉 `-64k` 五件套 + `update-grub` | 别在 DKMS 还引用它时做 |
| DKMS | `dkms remove dgx-spark-memory-saver/0.4.1 --all` | MOK 注册是一次性的，**撤不掉**（要撤得进 UEFI） |
| MTU | 删掉 netplan 里的 `mtu: 9000` + `netplan apply` | 1~2 s 链路抖动，管理面不受影响 |
| 站点配置 | `git checkout <上一个站点提交>` + `cluster sync --apply` | 三台一起，别单台改 |
