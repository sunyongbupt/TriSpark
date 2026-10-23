# 02 · 64 KiB 页内核 + memory-saver（512K/1M 上下文的前提）

`tp3-triangle-4k` 能在 4 KiB 页上跑 262K 上下文，但 512K/1M 需要
**`7.0.0-1019-nvidia-64k` + `dgx-spark-memory-saver`**。两者的价值不只是页大小：

```
MemTotal  121.69 GiB (4 KiB)  →  123.79 GiB (64 KiB)     # +2.10 GiB，纯页表开销下降
```

单机多出的 2.1 GiB 在每 rank 只剩个位数 GiB 余量的长上下文场景里是真金白银。

## 2.1 内核五件套与版本后缀

```bash
apt install linux-image-7.0.0-1019-nvidia-64k \
            linux-modules-7.0.0-1019-nvidia-64k \
            linux-headers-7.0.0-1019-nvidia-64k \
            linux-tools-7.0.0-1019-nvidia-64k \
            linux-modules-nvidia-580-open-7.0.0-1019-nvidia-64k
```

- 前四个是 `7.0.0-1019.19~24.04.2`；**`linux-modules-nvidia-580-open-…-64k` 是 `+2` 后缀**
  （档案里 `+1` 已被 Ubuntu 重同步挤掉）。`scripts/kernel_readiness.py` 明写接受 `version + "N"` 后缀，
  实测包内驱动仍是 `580.178.04`（`modinfo -k 7.0.0-1019-nvidia-64k -F version nvidia` 三台一致）。
  遇到 readiness 报版本不匹配先看它有没有按后缀匹配，别急着降级。
- `/boot/config-…-64k` 里必须有 `CONFIG_ARM64_64K_PAGES=y`；`grub-script-check /boot/grub/grub.cfg` 要 ok。
- **`linux-tools-…-64k` 一定要装**：作者踩过 `nv-cpu-governor.service` 在这个内核上失败，根因就是缺 tools。
  我们三台装齐后 failed units = 0。
- `experiments/2026-10-02-kernel-64k/prepare.sh` **别跑**：它末尾断言 `dgx-spark-fan-control`
  在该内核已 `installed`，而本站 `host.fan_control=false`，这条永远过不去。
  按 `docs/setup.md` 的 1–4 步手工走即可。

## 2.2 GRUB 默认项：显式钉，且要匹配 readiness 的字符串

`tp3-triangle` 的 `host.kernel.policy` 里 `"default": "7.0.0-1019-nvidia-64k"`，而
`scripts/kernel_readiness.py` 要求 **`grub.default_kernel == policy["default"]`**，
还会把残留的 `next_entry` / `initrdfail` 单独挑出来报错。所以：

```bash
cat > /etc/default/grub.d/zz-spark-kernel-trial.cfg <<'EOF'
GRUB_DEFAULT="gnulinux-advanced-<root-uuid>>gnulinux-7.0.0-1019-nvidia-64k-advanced-<root-uuid>"
EOF
update-grub
```

要点：

- `<root-uuid>` 逐台不同，别抄别人的；
- **4 KiB 内核留作 fallback entry**（`grub.cfg` 里仍在），回滚只要 GRUB 选回它，权重和镜像不用动；
- 试起阶段用**一次性**入口，别改默认：
  `grub-reboot "gnulinux-advanced-<uuid>>gnulinux-7.0.0-1019-nvidia-64k-advanced-<uuid>"`；
- 一个容易看错的地方：`update-grub` 之后 `grub.cfg` 的 **entry 0（`gnulinux-simple-<uuid>`）会指向 64K**
  （GRUB 认为它更新），真正把默认钉在 4K 的是上面那个 drop-in。只 `grep` entry 0 会误判。
- **试起可能无声失败**：我们有一台 `next_entry` 确实写进去了，那次开机却仍是 4K，
  `last` 里没有 64K 记录、grubenv 干净（无 `initrdfail` 残留），原因未定；原样重试第二次就上去了。
  ⇒ 判据必须是开机后回读，不能相信「命令成功了」。

## 2.3 上游 boot-support 脚本的一个真 bug（会让 64K 完全没有 swap）

`install-boot-support.sh` 把 helper 装成 `/usr/local/sbin/spark-kernel-memory`，
而 unit 的 `ExecStart` 写的是 `/usr/local/sbin/spark3-…`（差一个 `3`）：

```
# 上游原样
install-boot-support.sh:32   →  /usr/local/sbin/spark-kernel-memory
spark3-kernel-memory.service ExecStart → /usr/local/sbin/spark3-kernel-memory
```

`systemd-analyze verify` 对这种情况**只 WARN、不返回失败**，脚本照旧 exit 0，装完完全看不出来。
后果很实在：64K 开机时那个 oneshot 找不到文件，而 `/swap.img` 已被改成 `noauto`、
`/swap-64k.img` 根本不在 fstab（由 service 直接 `swapon`）⇒ **64 KiB 下完全没有 swap**，
THP 与 `vm.min_free_kbytes=45166` 也不生效。

本站修正路径并加了断言（站点提交 `4396075b`）：

```bash
test -x "$execstart" || { echo "ExecStart missing: $execstart"; exit 1; }
```

修后三台实测：`/swap-64k.img` 16 GiB `0600` 且偏移 65526 处 `SWAPSPACE2` 签名在、
`/swap.img` → `noauto`（改前的 fstab 备份留在 `/var/lib/spark3/kernel-64k-boot/`）、
`spark3-kernel-memory.service` `enabled`，而**当前活动 swap 仍是原 `/swap.img`**
（脚本故意不碰运行态，符合设计，重启后才切换）。

安装完顺手复核这四条，别只看服务 enabled：

```bash
systemd-analyze verify /etc/systemd/system/spark3-kernel-memory.service
test -x /usr/local/sbin/spark3-kernel-memory && echo ok
tail -c +65527 /swap-64k.img | head -c 10 | od -c | head -1   # SWAPSPACE2
grep -n swap /etc/fstab
```

## 2.4 memory-saver：源码、签名、MOK

**源码就位（不用重启）**

```bash
git clone https://github.com/<owner>/dgx-spark-memory-saver.git   # 本站 @ ec1b17a，detached + clean
cd dgx-spark-memory-saver && ./scripts/check                       # 45 tests OK（纯单测 + bash -n，不碰内核）
```

三个必须逐字符对齐的地方：

1. `/usr/src/dgx-spark-memory-saver-0.4.1` 必须**不存在**，`install-dkms.sh` 里有 `test ! -e`。
2. **`git remote get-url origin` 的返回值必须逐字符等于 pin 里的 URL**（含 `.git` 后缀）。
   clone 时省掉 `.git`，断言就必挂。
3. 因此**不能**用 `insteadOf` 把 GitHub 重写成本地裸仓来绕过网络问题：`get-url` 会应用重写规则，
   断言照样挂。origin 就是得是真的 GitHub URL（见 `06-…` 关于 edge 不通时的一次性 `/etc/hosts` 权宜）。

**fleet 签名身份（三台同一个 key，便于一次注册）**

```
/root/.local/share/dgx-spark-fan-control/keys/MOK.priv   # 0600，root
/root/.local/share/dgx-spark-fan-control/keys/MOK.der    # 0644
/etc/dkms/framework.conf.d/zz-spark-mok.conf             # 指向上面那对
```

`mokutil --test-key MOK.der` 的状态要读懂：
`already in the enrollment request` = 已排队、**尚未注册**；`is not enrolled` = 队列已被消费掉。

**MokManager 的时序（我们因此白死一次）**

`mokutil --import` 只是把证书排进待注册队列，队列由 shim 在**下一次上电**时消费。
**如果那次开机没人操作，请求会被消耗掉而不会保留。** 正确顺序是：

> **人先到屏幕前，再 reboot。** 别反过来。

顺带：本站有一次 `shutdown -r +0` 之前机器就自己重启了（原因未查），
等人到屏幕前时队列已经空了，只能重新 `--import` 再走一次。

**DKMS 安装**

前置条件是先**停掉服务**（无运行容器、没有 GPU 客户端持有 `/dev/nvidia-uvm`），
`install-dkms.sh` 会检查。逐台跑，`rc=0` 后复核：

```bash
dkms status        # dgx-spark-memory-saver/0.4.1, 7.0.0-1019-nvidia-64k, aarch64: installed (Original modules exist)
cat /sys/module/nvidia_uvm/srcversion                          # 必须逐字符等于 config/kernel-trial.json 的 pin
modinfo -k 7.0.0-1019-nvidia-64k nvidia-uvm -F signer          # DGX Spark local module signing
cat /sys/module/nvidia_uvm/parameters/uvm_pack_sysmem_leaf_tables   # Y
```

⚠ 快照时序坑：如果在 DKMS 模块真正加载之前拍了状态，会读到旧的 `srcversion` 和
`packing no-parameter`，看起来像安装失败。判据要在**起进 64K 且模块加载之后**取。

## 2.5 64 KiB 试起判据（逐台）

上电后一次性核对，本站两台从机 + head 全绿：

| 检查 | 期望 |
|---|---|
| `uname -r` | `7.0.0-1019-nvidia-64k` |
| `getconf PAGESIZE` | `65536` |
| `MemTotal` | **123.79 GiB**（4K 下 121.69） |
| `swapon --show` | 活动的是 `/swap-64k.img` |
| THP | `never` |
| `vm.min_free_kbytes` | `45166` |
| `nvidia_uvm` | `srcversion` 匹配 pin、`uvm_pack_sysmem_leaf_tables=Y` |
| 驱动 | `580.178.04` |
| `systemctl --failed` | 0 条 |
| memory-saver 状态 | `state: packing-enabled`、`qualification: validated` |

GPU 侧再跑配方的 `experiments/2026-10-02-driver-cuda-refresh/cuda-check.sh vllm-ds41f-kkref:<tag>`，
`rc=0` 判据：`allocation_stress` 3584 allocations × 12 轮 pass、
`cuda_smoke` 4,831,838,208 B pinned **跨 4 GiB 边界** pass（这条正是 64K 下最容易翻车的分配路径）。

三台都过了再把 64K 升为**默认**（2.2 的 drop-in），然后 `bin/spark site shape tp3-triangle --apply`
切形状。注意 `tp3-triangle` 自带 `benchmark_reference` 清单，切过去之后
`cluster start` **不需要** `--unqualified-shape`。

## 2.6 遗留：fan-control 在 64K 上没有包

`dgx_ec_fan_control` / `dgx-spark-fan-control` 的 DKMS 在 `7.0.0-1019-nvidia-64k` 上三台都装不了
（没有对应版本），doctor 只报 WARN、不影响 bench，但站点 `fan_control=false` 意味着
**bench 跳过降温曲线、不等预冷**，读数偏乐观。实测代价见 `04-…`：head 的板温峰值 94.9 °C。
要补的话是「取源码 + fleet key 签名 + DKMS 安装 + 翻 `host.fan_control=true`」一整条链，
且翻完要重启服务。
