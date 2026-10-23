# 03 · 站点发布链与镜像链

配方的构建/启动路径上硬编码了两道门，绕不过去，只能把发布基建先搭好：

```python
repository_dirty()   # 硬门 1：git status --porcelain --untracked-files=all 必须为空
                     #         未跟踪文件也算脏 ⇒ .env / cluster.json 这类 node-local 文件必须被 gitignore
"commit 必须发布在 origin/<branch> 上"   # 硬门 2：本地提交没 push 就等于不存在
```

## 3.1 在 head 自建裸仓当 `deployment.repository`

不需要公网 git 服务，也不需要把实验室配置推到别人看得见地方：

```bash
git init --bare --initial-branch=main /home/<user>/git/spark-ds41f.git   # 路径自选，记进 cluster.json
```

作者的 GitHub 仓库保留在每台检出里，只是**改名为 `github` 当上游参照**；`origin` 指向裸仓，
于是官方那两处 `git fetch origin`（`build image`、`cluster sync`）都打到本机。

**`origin` URL 用一个 SSH 别名，而不是 IP。** 因为 `normalized_repository_url()` 只做字符串归一化，
而三台的 `origin` 字面值必须逐字符一致；用别名的话每台 `~/.ssh/config` 里的 `Host trispark-head`
各自解释：

```
# 两台从机
Host trispark-head
    HostName <head-mgmt-ip>       # 从机眼里的 head
# head 自己
Host trispark-head
    HostName 127.0.0.1            # head 对自己的 tailnet/管理 IP 做 host key 校验在本机过不去，回环那条本来就被信任
```

为此要落的鉴权（都可回滚，动手前每个文件留 `.bak`）：

- head 的 `~/.ssh/authorized_keys` 追加：每台从机自己的集群 key 公钥（没有 `.pub` 就
  `ssh-keygen -y -f <私钥>` 导出来）+ head 自己的公钥（让它 ssh 到自己）；
- 三台 `~/.ssh/known_hosts` 加入 head 的 host key —— **从 head 的 `/etc/ssh/ssh_host_ed25519_key.pub` 取，
  不要 TOFU 首连信任**；
- 三台 `~/.ssh/config` 追加 `Host trispark-head` 段。

Git 身份（姓名/邮箱）只写进三台的 `.git/config`，不要动 `--global`。

## 3.2 浅克隆是隐藏雷

三台的配方检出原本是 `--depth 1`，裸仓直接拒收浅历史：

```
remote rejected: shallow update not allowed
```

补齐：能从 GitHub 走的 `git fetch --unshallow github main`（本站 1368 个提交）；
到不了 GitHub 的那台，改从裸仓 `git fetch --unshallow origin main`。
**今后铺新节点别再 `--depth 1`。**

## 3.3 `cluster sync` 会把 head 自己的 HEAD detach —— 于是 push 会假装成功

本站真的踩过：`cluster sync --apply` 之后 head 上又落了新提交，
`git push origin main` 报 **`Everything up-to-date`**，而裸仓还停在上一个提交。
原因是 **`cluster sync` 对 head 自己也执行 detach checkout**，新提交挂在游离头上，
`main` 这个 ref 根本没动。

正确发布顺序：

```bash
git log --oneline -1                      # 确认游离头就是你要发布的那个提交
git branch -f main <sha>                  # 把分支 ref 拨回来
git push origin main
python3 bin/spark cluster sync --apply    # 三台 detach 到同一个提交
```

复核（每台）：`git status --porcelain` 空、`git rev-parse HEAD` 三台一致、
`dirty=0`，且 `config/site.json`（gitignored）由 sync 复制到从机。
顺带清出库的杂物挪到部署目录下另存，不要留在工作树里——`--untracked-files=all` 会把它算成脏。

本站站点提交链（父提交是上游 baseline）：

```
<upstream>  配方 baseline
   ↓ e562eee8  三节点 tp3-triangle-4k 的站点配置
   ↓ 3fc87ba0  JIT cache 挂载改到本站部署路径
   ↓ 4396075b  修正 boot-support 的 ExecStart 路径（见 02-… §2.3）
   ↓ 9362abb7  端点 bearer key + 开机自启（见 05-…）
```

站点提交里改了哪些东西（全部是「本站事实」而非配方缺陷）：
`host.home`、`host.fan_control=false`、`deployment.path`、`deployment.repository`、
`distributed.master_addr=<head-mgmt-ip>`、`GLOO/TP_SOCKET_IFNAME`、`NCCL_SOCKET_IFNAME`。

> `NCCL_SOCKET_IFNAME` 那条注意继承：两个 shape 的 `NCCL_SOCKET_IFNAME` 原值指向一个在本机 DOWN 的口，
> 而 `tp3-triangle-4k` 是从 `tp3-triangle` 继承的，所以**只需要改 `tp3-triangle` 那处**，
> 改两层会造出无谓的 diff。

## 3.4 基镜像：digest 固定 + 镜像站不忠实

Dockerfile 默认基镜像是**摘要固定**的
`docker.io/vllm/vllm-openai@sha256:3cf14c52…`（37 层 / 压缩 9.7 GB）。
head 直连 `registry-1.docker.io` 超时，但 daemon 里配的镜像站在鉴权挑战后能取到同一份 manifest。

**踩到的坑**：`docker pull` 两次都在**同一层** `719d6bc8…` 失败，且两次返回的摘要不同
（`96865e08…`、`e31f0927…`）——镜像站会重压缩/改写层内容，digest 校验必挂。

对策（本站有效）：**绕开 daemon 的 `registry-mirrors`，直接按摘要从单个镜像站拉**：

```bash
docker pull docker.<mirror>.run/vllm/vllm-openai@sha256:3cf14c52…
```

内容 store 是按摘要寻址的，只要有一次把正确的层落进本地，再拉 `docker.io/…@sha256:3cf14c52…`
就会复用这些层。注意 `Dockerfile` 里的 `ARG BASE_IMAGE` **没有** `bin/spark build image` 的覆盖口子，
所以只能让本地存在这个摘要引用，**不能靠改 Dockerfile 或 lock 文件绕过**。

> 从机现成的 `vllm/vllm-openai:latest` 是另一个 digest，**不能拿它顶替**——
> 摘要固定这一条存在的意义就是可复现。

## 3.5 构建

```bash
python3 bin/spark build prepare
python3 bin/spark build check
python3 bin/spark build image --apply
```

- build host 需要 64 GiB 空闲内存；本站 `build image` 用时 **743 s**（8 jobs × 2 NVCC threads），
  全程最低 MemAvailable **74.6 GiB**（配方门 `BUILD_RESERVE_GIB = 24`），三段 GPU/JIT smoke 全过。
- 产物 `vllm-ds41f-kkref:19f2c20ed4d6-r6x`（34.9 GB），收据里记录的 deployment commit
  必须等于三台实际 checkout 的那个。
- **别在权重下完之前跑 `build prepare`**：clone + pip 下载会和 475 GiB 权重抢同一条被限流的出口
  （三台共享约 60 MB/s 总闸），把 7 小时拖成十几小时。
- 核验 git 源可达性时 `git ls-remote` 那几个大仓（vllm / nccl / flashinfer / cutlass / tilelang /
  TileKernels / sparknet / b12x）**要给 >30 s 超时**，否则会假报失败。
  `raw.githubusercontent.com` 在本网络被封，但配方不依赖它；registry v2 ping 返回 401 是正常的未鉴权响应。

## 3.6 分发：`save | load` 会改 image ID，但内容一致性是可以证的

```bash
docker save vllm-ds41f-kkref:19f2c20ed4d6-r6x | ssh peer-a docker load     # 走 RoCE 直连 ~560 MB/s
```

一台 5 分钟、另一台 9 分钟。**⚠ `save|load` 不保留 image ID**：head 是 `1cb149665f7b`，
两台从机是 `3be262a48eda`。这个 docker（29.6.2，containerd store）的 `save` **根本没有保留摘要的格式选项**
（`--format oci-*` 不存在），config 重新序列化必然换 ID。

内容一致**是证出来的不是猜的**，三条证据：

```bash
docker image inspect -f '{{json .RootFS.Layers}}' <ref> | sha256sum   # 三台一致（diffID 哈希一致）
docker image inspect -f '{{json .Config.Labels}}' <ref>               # local.spark3.*.tree 标签逐台一致
```

launch preflight 查的正是**标签 + `docker image inspect` 可达**，不是 image ID。
所以别因为三台 image ID 不同就重构建——**一次构建、同一份内容推到每个 rank，绝不原地覆盖 tag**。
