# rsync 文件同步从入门到使用

## 写在前面

如果说 firewalld 决定了一个端口"能不能进来"，那 rsync 决定的是一件更朴素也更要命的事：**数据到底有没有在两台机器之间对上**。

它是 Linux 下最常用的文件同步工具。10GB 的目录里改了 1MB，scp 会把 10GB 重传一遍，rsync 只传那 1MB。网站代码发布、跨机房备份、日志归集、服务器搬家、磁盘克隆，几乎每一样都能用它解决。

这份文档延续上一份 firewalld 教程的写法：**每一节先讲清楚"为什么"，再给出"怎么敲"，最后落到"本节考点"**。全文按认知、地基规则、参数体系、三种工作模式、备份方案设计、自动化、实战、排错、速记的顺序推进。

约定说明：

- 文中 IP（`192.168.1.100`）、主机名（`remote`）、目录（`/data/src`）均为占位演示值，实际使用请替换。
- 标注为「示意输出」的代码块是命令的典型返回格式，不是你机器的真实结果。
- 需要提权的命令统一以 `sudo` 开头。
- 密码、密钥等敏感内容一律用占位符表示，请勿把真实凭据写入聊天或文档。

---

## 第一章 入门认知：rsync 为什么快

### 1.1 它只做一件事：把差异传过去

rsync 名字里的 r 就是 remote。与 cp、scp 这类"从头复制一遍"的工具不同，rsync 会先**比对发送方和接收方已有的文件**，只把发生变动的部分传过去。

默认用哪种方式判断"变动"？官方称为 **quick check（快速检查）算法**：比较文件的**大小**和**最后修改时间**。这两个字段任何一个不同，就认为需要传输；两个都相同，就跳过。

这就是为什么 rsync 第二次同步几乎瞬间完成。也是为什么它会带来一个反直觉的坑：如果你只改了文件内容、但没改到大小和时间戳（比如用工具重设了 mtime），rsync 默认会认为"没变"而跳过。要治这个，得用 `-c`/`--checksum` 强制按校验和判断（第九章展开）。

### 1.2 差分传输的思路

在文件级别只传"变化的文件"，很多工具都能做到。rsync 真正的看家本领是**文件内部只传变化的块**：

```
接收端：对已有文件按块计算「滚动弱校验 + 强校验」的指纹，形成签名表
    ↓
发送端：用签名表在源文件里滑动匹配，找出「接收端没有的那些数据块」
    ↓
只把这些差异块 + 一个重建指令发给接收端
    ↓
接收端：用旧文件 + 差异块拼装出新文件
```

好处极其明确：一个 5GB 的日志文件只追加了 200MB，实际走的网络流量就是那 200MB 量级，而不是 5GB。

代价也要清楚：这套算法靠 CPU 换带宽。局域网千兆以上、数据本身可压缩性差、或者机器算力吃紧时，用 `-W`/`--whole-file` 关闭差分直接整文件复制反而更快。

### 1.3 三种工作模式

| 模式 | 写法特征 | 适用场景 | 认证方式 |
| --- | --- | --- | --- |
| 本地模式 | `rsync ... /src/ /dst/` | 本机跨目录、跨挂载点复制，可当智能版 cp | 文件系统权限 |
| 远程 Shell 模式（SSH） | `rsync ... user@host:/path` | 少量机器之间的同步推送拉取，配置最简单 | 系统用户 + SSH 密钥/密码 |
| 守护进程模式（daemon） | `rsync ... user@host::module` 或 `rsync://user@host/module` | 备份服务器、多客户端批量分发、需要模块级权限隔离 | rsyncd.conf 里的**虚拟用户** |

第三种模式的"服务端口"是 873。

一句话选择建议：**本地或有 SSH 权限的少量远程同步 → 用普通/SSH 模式；没有 SSH 权限、要接很多客户端、要按模块隔离目录 → 用 daemon 模式。**

### 1.4 几个必须提前建立的边界认知

1. **收发两端都必须装 rsync**。远程模式下接收端跑的是同一个 rsync 进程，缺了就报错。
2. **传统上不支持两台远程机器之间直接同步**（source 和 destination 不能同时是 `host:path` 形式）。要 A→B 又不想经过本机，就登录 A 去执行。
3. **rsync 本身不是备份工具**。它只负责"把两边对齐"，没有版本历史、没有去重、没有加密存储。所谓"rsync 增量备份"是你用 `--link-dest`、日期目录、`--backup-dir` 这些机制**搭出来的**方案（第十一章）。带 `--delete` 的 rsync 尤其危险——它会忠实地把你在源端误删的东西，从备份端也删掉。
4. **走 SSH 模式的流量本身是加密的**，不需要额外处理；daemon 模式默认明文（除非自己套 TLS/VPN 或改走 SSH），生产上要谨慎。

### 本章考点

- rsync 默认 quick check 依据是"文件大小 + 修改时间"，不是内容校验和。
- 差分算法的价值在"只传变化块"，代价是 CPU 换带宽，可用 `-W` 关闭。
- 三种模式：本地 / SSH / daemon；daemon 端口 873，用虚拟用户认证。
- 两端都要装 rsync；不能直接 remote→remote；rsync 只是同步器，不等于备份系统。

---

## 第二章 两条地基规则：搞错这两点，参数再熟也白搭

### 2.1 语法骨架

```bash
rsync [选项] 源路径 目标路径
```

方向由参数位置决定：**左边是源，右边是目标**。要从远程往本地拉，就把远程写在左边。

### 2.2 结尾斜杠：rsync 第一大坑

**源路径带不带结尾斜杠，结果完全不同。**

```bash
# 不带斜杠：把 source 目录本身拷进去
rsync -av /data/source /backup/
# 结果 → /backup/source/...

# 带斜杠：只把 source 目录里的内容拷进去
rsync -av /data/source/ /backup/
# 结果 → /backup/...（没有 source 这一层）
```

记忆方法：**斜杠表示"这个目录里面的东西"，没有斜杠表示"这个目录本身"。** 阿里云 NAS 迁移文档甚至直接写明"rsync 命令中的源路径结尾必须带有正斜线，否则同步后数据路径不匹配"，可见这个坑在真实项目里踩得有多频繁。

再多一句：**目标路径的结尾斜杠不影响这个语义**，只对源路径敏感。所以每次正式同步前，先 `-n` 干跑看一遍路径（第三章）。

### 2.3 `-a` 到底展开了什么

`-a`（archive，归档模式）是最常用的参数，它本身是**一组参数的简写**：

```
-a  ==  -r -l -p -t -g -o -D
```

| 参数 | 含义 | 备注 |
| --- | --- | --- |
| `-r` | recursive，递归子目录 | 不给 `-r`，rsync 根本不处理目录 |
| `-l` | 保留符号链接（照原样建成链接） | |
| `-p` | 保留权限位 | |
| `-t` | 保留修改时间 | 这条直接决定下次 quick check 的判定结果 |
| `-g` | 保留属组 | 仅超级用户有效 |
| `-o` | 保留属主 | 仅超级用户有效 |
| `-D` | 保留设备文件等特殊文件 | 等价于 `--devices --specials` |

**`-a` 不包含的三个重要属性：**

| 参数 | 含义 | 为什么 -a 不含 |
| --- | --- | --- |
| `-H` | 保留硬链接关系 | 需要额外内存与 inode 操作，按需加 |
| `-A` | 保留 ACL | `-a` 只保留传统 Unix 权限 |
| `-X` | 保留扩展属性（xattr/chattr） | 同上 |

所以想做到尽可能逐位一致的镜像，社区常用的完整组合是：

```bash
rsync -avHAX --progress /源目录/ /目标目录/
```

做整盘/跨磁盘克隆时还会再加 `-x`（不跨越文件系统边界），典型的磁盘搬家写法：

```bash
rsync -avxHAX --info=progress2 / /新磁盘挂载点/
```

### 2.4 安装与确认

```bash
# 查看是否已装、版本几何
rsync --version
rpm -q rsync                    # RHEL/CentOS/Rocky/Alma
dpkg -l rsync                   # Debian/Ubuntu

# 安装
sudo yum install rsync -y       # CentOS/RHEL
sudo apt-get install rsync -y   # Debian/Ubuntu
# macOS：brew install rsync（系统自带的版本较老，建议 brew 版）
```

顺带一条：RHEL 系里 daemon 模式的 systemd 服务不是 rsync 主包带的，Rocky/CentOS 8+ 需要额外装 `rsync-daemon`，然后才可以用 `systemctl start rsyncd`（第六章）。

### 本章考点

- 源路径带斜杠 = 传目录内容；不带斜杠 = 连目录本身一起传。目标路径的斜杠不影响该语义。
- `-a` = `-rlptgoD`，不含 `-H`（硬链接）、`-A`（ACL）、`-X`（扩展属性）。
- `-g`/`-o` 属主属组只在以 root 运行时真正生效。
- 大文件/局域网场景可用 `-W` 关差分；整盘克隆加 `-x`。

---

## 第三章 参数体系：分七类记，别当字典背

rsync 参数上百个，日常高频的只有十几个。按"用途分类"来记，比按字母顺序背高效得多。

### 3.1 观察类（先看，别急着动手）

```bash
-v                  # 详细输出，列出传输了哪些文件
-vv                 # 更详细，调试排错时用
-q                  # quiet，抑制非必要输出（脚本里常用）
-h                  # human-readable，K/M/G 显示
-n, --dry-run       # ★ 模拟运行，不真传、不真删，只看会做什么
-i, --itemize-changes  # 逐文件列出「变更类型」明细
--stats             # 结束后输出统计摘要（文件数、传输字节、加速比）
--progress          # 显示每个文件的传输进度
--info=progress2    # 显示整体进度（含总百分比与速率），比 --progress 好用
```

**`-n` 是 rsync 最重要的安全阀。**任何带 `--delete`、覆盖生产数据的命令，第一遍都必须加 `-n`。

### 3.2 传输与性能类

```bash
-z                  # 传输时压缩，慢网络省带宽；本地同步无意义，反而费 CPU
-P                  # = --partial --progress，断点续传 + 进度条，大文件必备
--partial           # 保留中断的半截文件，下次续传
--bwlimit=1000      # 限速，单位 KB/s，别把业务带宽吃满
--timeout=300       # I/O 超时秒数
--contimeout=10     # daemon 模式连接超时
-W, --whole-file    # 关闭差分算法，整文件复制
-c, --checksum      # 用校验和判断差异，不看 mtime（慢但准）
--size-only         # 只看大小不看时间
```

### 3.3 删除与镜像类

```bash
--delete                        # 目标端有、源端没有的 → 删除（做严格镜像）
--delete-before                 # 传输前删
--delete-during                 # 传输过程中删（较新版本常为默认）
--delete-after                  # 传输结束后删
--delete-excluded               # 连被 --exclude 排除掉的也一并从目标删除
--max-delete=NUM                # 最多删 N 个，护栏，防止源端异常导致备份被清空
--force                         # 允许删除目标端非空目录
--ignore-errors                 # 有 I/O 错误也继续执行删除
```

### 3.4 过滤选择类

```bash
--exclude=PATTERN          # 排除匹配项
--include=PATTERN          # 强制包含（优先级高于后面的 exclude）
--exclude-from=FILE        # 从文件读排除规则，一行一个
--include-from=FILE        # 从文件读包含规则
--files-from=FILE          # 反过来：只传这个文件里列出的清单
-C, --cvs-exclude          # 用 CVS 风格自动忽略一批生成文件
--filter='...'             # 更细粒度的规则（merge/remove/protect 等）
```

### 3.5 属性与安全类

```bash
-H  -A  -X                 # 硬链接 / ACL / 扩展属性（-a 不含，按需加）
-L, --copy-links           # 把符号链接指向的真实文件传过去，而不是建链接
--safe-links               # 忽略指向源目录树之外的软链接
--numeric-ids              # 按 UID/GID 数字原样传，不做名字映射
--chown=[USER][:GROUP]     # 目标端强制改写属主属组
--chmod=MODE               # 目标端强制改权限，如 --chmod=D0755,F0644（数字写法各版本略有差异，用前 --help 确认）
--owner --group            # 单独控制是否保留属主属组
```

### 3.6 远程连接类

```bash
-e "ssh -p 2222"           # 指定远程 shell，可带端口、密钥、算法等 SSH 参数
--rsync-path=/usr/bin/rsync  # 指定远端 rsync 可执行文件路径（非标准安装/多版本时救急）
--rsh=COMMAND              # 同 -e
--address=                 # 绑定本机源地址（多网卡）
```

### 3.7 备份与落盘类

```bash
-b, --backup               # 目标端同名文件被覆盖前，先重命名保留（默认后缀 ~）
--suffix=STRING            # 自定义备份后缀
--backup-dir=DIR           # 把被覆盖的旧版本统一放进 DIR（增量历史的关键）
--link-dest=DIR            # 未变化文件硬链接到 DIR，做「看似全量、实为增量」的快照
--compare-dest=DIR         # 额外参照 DIR 判断是否需要传输
--relative, -R             # 保留命令行中给定的完整相对路径层级
--inplace                  # 直接写目标文件（不先写临时文件再改名）
--append                   # 追加到已存在的较短文件末尾
-S, --sparse               # 稀疏文件优化，节省目标空间
-x, --one-file-system      # 不跨越文件系统边界（克隆整盘时用）
--remove-source-files      # 传成功后删除源端的非目录文件（是搬运，不是备份）
--temp-dir=DIR             # 指定临时文件目录（目标盘满时救命）
--log-file=FILE            # 输出日志到文件，脚本化必配
```

### 3.8 命令套路：三步走

```bash
# 第 1 步：干跑，确认要传什么、要删什么
rsync -avn --delete --stats /data/src/ user@remote:/data/dst/

# 第 2 步：确认无误后去掉 -n，正式执行
rsync -avz --delete --bwlimit=5000 /data/src/ user@remote:/data/dst/

# 第 3 步：验证一致性（比大小 + 抽样校验和）
rsync -avnc /data/src/ user@remote:/data/dst/     # -n + -c，只报告差异不传输
```

### 本章考点

- `-n` 干跑 + `-v` 输出，是任何破坏性同步（尤其带 `--delete`）的强制前置动作。
- `-z` 只对网络传输有意义；本地同步加 `-z` 纯属浪费 CPU。
- `-P` = `--partial --progress`，大文件传输首选。
- 判断差异的三种口径：默认 quick check（大小+mtime）、`--size-only`（只看大小）、`-c`（看校验和，最准最慢）。
- `--max-delete` 是给 `--delete` 上的保险丝。

---

## 第四章 本地同步实操

本机之间（含不同挂载点之间）的复制，rsync 完全可以替代 cp，而且更聪明。

```bash
# 基本目录同步（注意源目录的斜杠）
sudo rsync -av /data/source/ /data/backup/

# 多个源 → 一个目标
rsync -av dir1 dir2 file.txt /destination/

# 单文件复制
rsync -av backup.tar.gz /tmp/data/

# 大文件：带进度、支持断点
rsync -avP bigfile.iso /mnt/nas/iso/

# 结束后看统计（传输量、加速比、文件数）
rsync -av --stats --human-readable /data/src/ /data/dst/

# 目标目录不存在会自动创建（-a 语义下）
rsync -av /etc/nginx/ /backup/nginx-conf/
```

### 4.1 看懂变更明细：`-i`（itemize-changes）

```bash
rsync -avi /data/src/ /data/dst/
```

示意输出：

```
<f..t...... file.txt
>f.st...... new.txt
cd+++++++++ subdir/
.d..t...... existing-dir/
```

读法要点：

- **第 1 个字符**是方向：`>` 表示从本地发往对端（更新/新建），`<` 表示从对端接收，`.` 表示这个文件本身没传数据、只动了元信息。
- **第 2 个字符**区分类型：`f` 文件、`d` 目录、`L` 符号链接、`d`+`c` 里 `c` 表示创建。
- 后面的字母位分别对应 checksum/大小、权限、属主、属组、时间等属性是否变化。`s` 大小不同、`t` 时间不同、`p` 权限不同、`o` 属主不同、`g` 属组不同；没变化的位置显示 `.`。
- 全 `+`（如 `cd+++++++++`）表示目标端该项是新建的。

各字段的权威定义在 `man rsync` 的 **ITEMIZE CHANGE STANZAS** 一节，那里有完整对照表，值得专门查一次。

`-i` 与 `-n` 组合尤其好用——能列出"谁会被传、谁会被删"的精确清单，是审 `--delete` 的最佳工具。

### 4.2 整盘/新磁盘克隆

```bash
# 先把新盘挂载到 /mnt/newdisk，然后：
sudo rsync -avxHAX --info=progress2 / /mnt/newdisk/
```

`-x` 保证不跨文件系统（避免把 /proc、/sys、/run 这些虚拟文件系统"搬"过去），`-HAX` 把硬链接、ACL、扩展属性一并保住。

顺带一个进度条的坑：rsync 用"增量递归"边扫描边传，**在扫完全部文件之前，百分比是不准的**（`--info=progress2` 输出里的 `ir-chk` 字段就是扫描进度的体现）。想让进度条老实一点，加 `--no-inc-recursive`：先扫完全部文件再开始同步，总大小才算得准。

### 本章考点

- 本地同步加 `-z` 无收益，直接 `-av` 或 `-avP`。
- `-i` 输出第一字符是方向，`>` 发送 / `<` 接收 / `.` 未传数据；与 `-n` 组合可审删除清单。
- 整盘克隆用 `-avxHAX`，`-x` 防止搬走虚拟文件系统。
- `--info=progress2` 的百分比在增量递归扫描完成前不准。

---

## 第五章 远程同步：SSH 模式（最常用）

远程模式默认走 SSH，天然加密、天然复用你已有的登录凭据，**服务端不需要额外起任何 rsync 服务**。这是它的最大优点。

### 5.1 两个方向

```bash
# 推送：本地 → 远程
rsync -avz /var/www/html/ deploy@192.168.1.100:/var/www/html/

# 拉取：远程 → 本地
rsync -avz deploy@192.168.1.100:/var/log/nginx/ ./nginx-backup/

# 只写主机不写用户：默认用当前登录用户名
rsync -avz ./backup 192.168.1.100:/data/

# 省略目标路径：落到远端该用户的 home 目录
rsync -avz testfile user@remote:
```

普通用户也能推送，但**只能推到该用户有写权限的目录**。想推 /data 就得先让 /data 对该用户可写，或用 root（注意安全与合规）。

### 5.2 指定端口、密钥、跳板

```bash
# 非标准 SSH 端口
rsync -avz -e "ssh -p 2222" /data/src/ user@remote:/data/dst/

# 指定私钥
rsync -avz -e "ssh -i ~/.ssh/deploy_key -p 2222" /data/src/ user@remote:/data/dst/

# 首次连接跳过 host key 确认（自动化脚本里常用，但降低了安全性）
rsync -avz -e "ssh -o StrictHostKeyChecking=no" /data/src/ user@remote:/data/dst/

# 远端 rsync 不在 PATH 里（比如自编译装在 /usr/local）
rsync -avz --rsync-path=/usr/local/bin/rsync /data/src/ user@remote:/data/dst/

# 走跳板机（借用 ~/.ssh/config 里配置好的 ProxyJump 即可）
rsync -avz /data/src/ bastion-target:/data/dst/
```

### 5.3 配免密：定时任务的前提

rsync 挂到 crontab 之前必须先把 SSH 免密做好，否则任务会因为等不到密码输入而失败：

```bash
ssh-keygen -t ed25519 -f ~/.ssh/rsync_deploy -N ''
ssh-copy-id -i ~/.ssh/rsync_deploy.pub deploy@192.168.1.100

# 验证（能免密登录即可）
ssh -i ~/.ssh/rsync_deploy deploy@192.168.1.100 'echo ok'
```

生产上建议为同步任务单独建低权限账号，并在远端 `authorized_keys` 里限制来源 IP（`from="..."`）。

至于 `sshpass`：能跑，但把密码写进命令行/文件等于裸奔，只建议在临时测试时用，长期方案一律走密钥或 daemon 模式。

### 5.4 实战推荐组合

```bash
# 发布代码：排除依赖和日志，先干跑，限速 5MB/s
rsync -avzn --delete --exclude='.git/' --exclude='node_modules/' --exclude='*.log' \
  ./dist/ deploy@192.168.1.100:/var/www/html/

# 确认输出无误后，去掉 n 执行
rsync -avz --delete --exclude='.git/' --exclude='node_modules/' --exclude='*.log' \
  --bwlimit=5000 --stats ./dist/ deploy@192.168.1.100:/var/www/html/
```

### 本章考点

- SSH 模式无需服务端起服务，方向由源/目标位置决定，流量天然加密。
- `-e` 可传完整 SSH 参数串（端口、密钥、算法、跳板）。
- 挂 crontab 前必须先完成免密；`sshpass` 不适合长期使用。
- `--rsync-path` 处理"远端装了 rsync 但不在 PATH"这一类疑难。

---

## 第六章 守护进程模式：搭一台备份服务器

### 6.1 为什么要有 daemon 模式

SSH 模式有两个绕不开的痛点：

1. **要用系统用户**——每接入一台机器就得开一个 Linux 账号，账号越多攻击面越大，不符合运维最小权限原则。
2. **权限难给**——低权限用户推送到 /backup 会 Permission denied，给 root 又太危险。

daemon 模式的解法是引入**虚拟用户**：`auth users` 里定义的用户**不需要在系统里存在**，只用于 rsync 协议认证；实际落盘身份由 `uid`/`gid` 指定的系统用户承担。再加上"模块（module）"这一层，可以对每个目录单独配读写、IP 白名单、压缩策略。

### 6.2 配置文件：/etc/rsyncd.conf

这个文件默认不存在，需要自己创建（RHEL/Rocky 8+ 系如此）。格式规则：**一行一个参数，`name = value`；模块用 `[模块名]` 开头，直到下一个模块为止；注释行独立成行**（写在配置行末尾的行内注释可能被当成值的一部分，导致诡异故障）。

```bash
# ===== 全局参数（对所有模块生效）=====
uid = rsync                          # 传输进程运行用户，文件落盘后的属主
gid = rsync                          # 运行属组
use chroot = no                      # true=传输前 chroot 到模块 path（更安全，但需 root，且无法同步 path 之外的软链接目标）
max connections = 200                # 最大并发连接数，0 为不限制，负值关闭该模块
timeout = 300                        # I/O 超时秒数，0 为永不超时；建议 300~600
port = 873                           # 监听端口（默认 873）
address = 192.168.100.10             # 监听地址
pid file = /var/run/rsyncd.pid       # PID 文件
lock file = /var/run/rsyncd.lock     # 支撑 max connections 的锁文件
log file = /var/log/rsyncd.log       # 日志；不设或设错则走 syslog
strict modes = yes                   # 是否检查密码文件权限（默认 true）
dont compress = *.gz *.tgz *.zip *.z *.Z *.rpm *.deb *.bz2   # 已压缩的不再压，省 CPU
motd file = /etc/rsyncd.motd         # 客户端连接时展示的消息（可选）

# ===== 模块：daily_backup =====
[daily_backup]
comment = daily backup dir
path = /backup                       # 模块对应的真实路径，启动服务前该目录必须存在
read only = no                       # yes=只读（客户端只能拉），no=可写（可推送）
list = no                            # 是否允许客户端列出该模块
ignore errors                        # 忽略无关 I/O 错误
hosts allow = 192.168.100.0/24       # 允许的来源 IP/网段，多个用空格分隔
hosts deny = *                       # 拒绝其余全部
auth users = rsync_backup            # 虚拟用户（与系统用户无关），空格或逗号分隔
secrets file = /etc/rsyncd.secrets   # 密码文件，格式 user:password，权限必须 600
fake super = yes                     # 允许非特权进程用 xattr 保存文件属性（root 之外想保住属性时开）
```

要点补充：

- **`use chroot`**：默认 true。设 true 更安全（把客户端限制在 path 内），但需要 root 权限，且同步指向 path 外部的符号链接时只会保留链接本身、不会带内容。内网环境下设 no 也常见。
- **`read only`**：默认 true（只读），想让客户端推送必须显式改 `no`。
- **`list = no`** 配合 `hosts allow/deny`：被拒绝的主机请求时直接返回"模块不存在"而不是"权限拒绝"，减少信息泄露。
- **`fake super = yes`**：以非 root 身份跑 daemon 时，用它把属主属组/特殊位存进扩展属性，从而仍能保留文件属性。
- 修改配置后建议**重启服务**（虽然文档层面全局参数可即时生效、模块参数需重启，但实践上统一重启更稳）。

### 6.3 系统用户、目录属主、密码文件

```bash
# 1. 建专用于 rsync 落盘的系统用户（不可登录）
sudo useradd -M -s /sbin/nologin rsync

# 2. 建备份目录并把属主给它
sudo mkdir -p /backup
sudo chown -R rsync:rsync /backup

# 3. 服务端密码文件：用户名:密码，权限 600
echo 'rsync_backup:<此处填密码占位>' | sudo tee /etc/rsyncd.secrets
sudo chmod 600 /etc/rsyncd.secrets
```

**权限不是 600 会直接被拒绝连接**——`strict modes` 默认开启，rsync 会检查密码文件是否"不可被其他用户读"，不合规就报错并拒绝认证。

### 6.4 启动服务

```bash
# 方式一：手工前台/后台启动（临时验证用）
sudo rsync --daemon
sudo rsync --daemon --config=/etc/rsyncd.conf   # 指定非默认配置路径

# 方式二：systemd（推荐，生产用）
sudo dnf install rsync-daemon -y     # RHEL/Rocky 8+ 需要这个包才带 rsyncd.service
sudo systemctl enable --now rsyncd
sudo systemctl status rsyncd

# 确认监听
sudo ss -tlnp | grep 873
```

停止：

```bash
sudo systemctl stop rsyncd
# 手工方式起的：
sudo kill $(cat /var/run/rsyncd.pid) && sudo rm -f /var/run/rsyncd.pid
```

### 6.5 客户端侧配置与操作

**客户端不需要写 rsyncd.conf，也不需要启动服务**，只要装 rsync + 一个密码文件。

关键差异，务必记住：

```bash
# 客户端密码文件：只写密码，不写用户名！
echo '<此处填密码占位>' | sudo tee /etc/rsync.passwd
sudo chmod 600 /etc/rsync.passwd
```

如果客户端也写成 `user:password` 形式，认证会失败——这是 daemon 模式最常见的"配置看起来都对但连不上"的原因之一。

**语法（双冒号是 daemon 模式的标志）：**

```bash
# 拉取 Pull：rsync [OPTION...] [USER@]HOST::SRC... [DEST]
rsync -avz --password-file=/etc/rsync.passwd rsync_backup@192.168.1.100::daily_backup/ /local/restore/

# 推送 Push：rsync [OPTION...] SRC... [USER@]HOST::DEST
rsync -avz --password-file=/etc/rsync.passwd /data/src/ rsync_backup@192.168.1.100::daily_backup

# 等价的 URL 写法
rsync -avz rsync://rsync_backup@192.168.1.100/daily_backup/ /local/restore/

# 列出服务端可见模块
rsync --list-only rsync_backup@192.168.1.100::
rsync rsync://192.168.1.100/
```

注意 `::模块名` 引用的是 rsyncd.conf 里的**模块**，不是文件系统路径；而模块的 `path` 才对应真实目录。源目录结尾同样要遵守斜杠规则。

### 6.6 别忘了防火墙

daemon 模式监听 873，如果服务器开了 firewalld，端口不放行客户端永远连不上。放行方式（与上一份 firewalld 文档衔接）：

```bash
sudo firewall-cmd --permanent --add-port=873/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports          # 验证
```

SSH 模式则要确认 22（或自定义端口）已放行。

### 6.7 两个高频报错及解法

**报错一：推送时报 `rsync: [sender] read error: Connection reset by peer (104)`**

原因基本是模块 `read only = yes`。改成 `no` 并重启服务。

**报错二：改成 no 之后报 `rsync: mkstemp "/.fedora.txt.xxxxx" (in share) failed: Permission denied (13)`**

这是**落盘权限**问题：`auth users` 里的虚拟用户默认映射为系统用户 `nobody`，而 nobody 对模块 `path` 目录没有写权限。三种解法：

```bash
# 解法 A（推荐）：在 rsyncd.conf 里显式指定有权限的 uid/gid，并给目录属主
sudo chown -R rsync:rsync /backup

# 解法 B：用 ACL 单独授权虚拟用户映射的系统用户
sudo setfacl -m u:nobody:rwx /rsync/
sudo getfacl /rsync/

# 解法 C：非 root 跑但想保留属性，加 fake super = yes
```

报错里的 `mkstemp` 很说明问题——rsync 写文件时是"先建临时文件再改名"，所以它对目标目录**必须有 w 和 x 权限**，光有 r 是不行的。

另外，开启 SELinux 的环境里，daemon 访问非标准目录可能被内核策略拦截。此时 `Permission denied` 未必来自文件权限，要查 `/var/log/audit/audit.log`（或 `ausearch -m avc -ts recent`）确认是不是被拒绝，再决定是调整目录标签还是启用相应布尔值。这类问题属于 SELinux 范畴，不要和文件权限混为一谈。

### 本章考点

- daemon 模式的价值：虚拟用户认证（`auth users` 不必是系统用户）+ 模块级隔离 + 批量客户端。
- `read only` 默认 yes，推送必须改 no；`list = no` + `hosts allow/deny` 降低信息暴露。
- 密码文件权限必须 600，`strict modes` 会强制检查；服务端写 `user:password`，客户端只写 `password`。
- `mkstemp Permission denied` 的本质是虚拟用户映射的系统用户（默认 nobody）对 path 目录无写权限。
- 默认端口 873，需防火墙放行；systemd 起服务在 RHEL 8+ 需要 `rsync-daemon` 包。

---

## 第七章 过滤与选择：精确决定传什么

### 7.1 排除

```bash
# 单个模式
rsync -av --exclude='*.log' /data/src/ /data/dst/

# 多个模式，写多个 --exclude
rsync -av --exclude='file1.txt' --exclude='dir1/*' /data/src/ /data/dst/

# 目录整体排除
rsync -av --exclude='node_modules' /data/src/ /data/dst/

# 借 shell 大括号一次给多个（注意这是 shell 展开，不是 rsync 语法）
rsync -av --exclude={'*.log','cache/','.git/'} /data/src/ /data/dst/

# 规则多就放文件里，一行一个模式
rsync -av --exclude-from='/etc/rsync_exclude.list' /data/src/ /data/dst/
```

`exclude-list` 内容示例：

```
*.log
cache/
.git/
node_modules/
tmp/
```

### 7.2 三个必须知道的细节

1. **隐藏文件默认会同步**。想排除所有隐藏文件，写 `--exclude=".*"`。
2. **排除目录里的内容、但保留目录本身**，要写成 `--exclude='dir1/*'`，而不是 `--exclude='dir1'`。
3. **`--include` 与 `--exclude` 是按顺序匹配、命中即停**，写在前面者优先。典型"只传 txt、其余都不要"要这样写：

```bash
rsync -av --include='*.txt' --exclude='*' /data/src/ /data/dst/
```

（先 include 放行 txt，再 exclude 兜底排除其余。）

### 7.3 只传清单里的文件

```bash
# files.txt 每行一个相对路径
rsync -av --files-from=/tmp/files.txt /data/src/ /data/dst/
```

这是配合 `find` 做"按时间/条件挑选同步"的标准做法：

```bash
find /data/src -type f -newermt '2026-10-01' -printf '%P\n' > /tmp/files.txt
rsync -av --files-from=/tmp/files.txt /data/src/ /backup/dst/
```

### 7.4 按文件大小筛（先确认版本支持）

较新的 rsync 支持 `--max-size` / `--min-size`（如 `--max-size=100M`）过滤大文件，老版本没有。别直接抄进脚本，先自证：

```bash
rsync --help 2>&1 | grep -E 'max-size|min-size'
```

有输出再用；没有就用 `--exclude` 规则或 `--files-from` + `find -size` 组合替代。

### 本章考点

- 排除目录内容保留目录本身 → `--exclude='dir/*'`。
- 隐藏文件默认会被同步，需显式 `--exclude='.*'`。
- include/exclude 顺序敏感、命中即停，`--include='*.txt' --exclude='*'` 是白名单模式。
- `--files-from` 配合 find 可实现"按条件挑选同步"。

---

## 第八章 删除同步：最强大，也最危险

### 8.1 --delete 的语义

rsync 默认**只保证源端的内容在目标端都存在**，它不会让两边完全相同，也不会删除任何文件。想要"目标就是源的镜像"，必须加 `--delete`：

```bash
rsync -av --delete /data/src/ /data/dst/
```

它会删除目标端有、源端没有的文件。用于发布和镜像非常合适——源端删掉的文件，目标端也消失，不会留下历史垃圾。

### 8.2 删除时机

```bash
--delete-before      # 传输开始前先清空目标多余文件（安全，但需两次遍历）
--delete-during      # 边传边删（较新版本默认，更快）
--delete-after       # 传完再删（传输期间目标端新旧文件并存）
--delete-excluded    # 把被 exclude 规则排除的文件也从目标删掉（危险，见下）
```

`--delete-excluded` 要特别小心：你以为只是"不同步日志"，加了它实际变成"把目标端的日志全删了"。

### 8.3 护栏三件套

```bash
rsync -avz --delete \
  --max-delete=1000 \            # 一次最多删 1000 个，超过就中止
  --exclude='*.db' \
  -n \                           # 先看清单
  /data/src/ backup@192.168.1.100::daily_backup
```

配合审清单的精确写法：

```bash
# 只列「会被删除」的文件，不列其他
rsync -avn --delete --itemize-changes /data/src/ /data/dst/ | grep -E '^\*?deleting'
```

（说明：配合 `-i` 时删除项输出以 `*deleting` 开头，只用 `-v` 干跑时则为 `deleting ` 开头。）

### 8.4 误删是怎么发生的

不是 rsync 出问题，而是**它太忠实地执行了"镜像"这个语义**：

- 源目录选错了（选了空的 / 选了半迁移的）→ 目标端被清空
- 上游程序异常把源文件删了 → 下一轮定时任务把备份也删了
- 源路径忘了写结尾斜杠 → 镜像了错误的层级

防线（务必形成习惯）：

1. **`--max-delete` 永远配上**，给一个业务可容忍的上限。
2. **镜像类备份至少保留一份"不带 --delete"的副本**，或用第十一章的 `--link-dest` 快照方案——每个日期目录都是独立完整视图，删不掉历史。
3. **正式跑之前必跑 `-n`**，且用 `-i` 审删除清单。
4. 定时任务里不要直接把生产目录 `--delete` 同步到唯一的一份备份。

### 本章考点

- 默认 rsync 不删文件；镜像一致必须 `--delete`。
- `--delete-excluded` 会删掉目标端被排除规则命中的文件，语义最容易误解。
- `--max-delete` 是唯一硬保险丝，生产脚本必须配。
- 防误删的本质：不要让"带 --delete 的单向同步"成为你唯一的备份路径。

---

## 第九章 传输控制与性能调优

### 9.1 限速与超时

```bash
--bwlimit=5000      # 单位 KB/s，即约 5MB/s。白天同步大目录必配
--bwlimit=0         # 不限速
--timeout=300       # I/O 静默超过 300 秒则断开，防止挂死
--contimeout=10     # daemon 模式连接阶段的超时
```

线上服务器同步时把带宽吃满是事故高发原因，`--bwlimit` 是最简单有效的礼貌。

### 9.2 断点续传的正确姿势

```bash
rsync -avP /bigfile.tar.gz user@remote:/backup/
# -P = --partial（保留半截文件）+ --progress（显示进度）
```

rsync 默认会把未传完的临时文件删掉，`--partial` 让它保留，下次接着传。对 GB 级文件、不稳定链路来说这是必备项。

`--append` 是更极端的情形：确认目标文件只是"末尾被追加"（如日志），直接续写而不是重新校验整文件。用错会造成数据错乱，只在明确的追加型文件上使用。

### 9.3 判断差异的三种口径

| 口径 | 参数 | 何时用 |
| --- | --- | --- |
| 快速检查（默认） | 无 | 常规场景，快 |
| 只看大小 | `--size-only` | mtime 不可信（如跨系统时区/时钟漂移），但内容改动必然变大小 |
| 看校验和 | `-c` / `--checksum` | mtime 和大小都可能骗人（改内容没改时间、原地更新数据库文件等），最准最慢 |

`-c` 会让两端都算校验和，数据量大时耗时明显。折中办法：日常同步用默认，定期（比如每周）跑一次 `-c` 做完整性核对。

### 9.4 落盘行为微调

```bash
-W, --whole-file    # 关闭差分算法：局域网高带宽、或超大单文件时反而更快
--inplace           # 直接改目标文件，不生成临时文件（省空间，但中断后目标文件是坏的；数据库大文件场景常用）
-S, --sparse        # 稀疏文件高效处理，节省目标磁盘
--temp-dir=/data/tmp  # 临时文件放别处，防止目标分区写满导致失败
-x, --one-file-system # 不跨文件系统（整盘克隆必加）
-R, --relative      # 保留完整路径层级：
                    #   rsync foo/bar/foo.c remote:/tmp/          → /tmp/foo.c
                    #   rsync -R foo/bar/foo.c remote:/tmp/       → /tmp/foo/bar/foo.c
```

### 9.5 为什么"大目录第一次很慢"

rsync 要先递归扫描建立文件清单，然后逐个比对（默认还要发文件签名做差分）。百万级小文件时，扫描和元数据比对的开销会远超实际数据传输，这也是 `--link-dest` 增量备份里"目录本身无法硬链接、每次都要新建"造成额外 inode 开销的根因。

对策方向：能排除的先排除（`--exclude`）、必要时 `-W` 关差分、或者用 `--files-from` 把范围缩小到真正关心的子集。

### 本章考点

- `--bwlimit` 单位是 KB/s；`-P` 是断点续传 + 进度。
- 三种差异口径的取舍：默认快、`--size-only` 抗时钟漂移、`-c` 最准最慢。
- `--inplace` 省空间但中断会留下坏文件；`--temp-dir` 解决目标盘满。
- `-R` 决定是否保留源路径层级，容易与"结尾斜杠"混淆，是两个独立机制。

---

## 第十章 属性、属主与权限的真实行为

这一章是 rsync 用得对不对的分水岭，也是"备份完权限全变 nobody"这类问题的答案所在。

### 10.1 非 root 执行时发生了什么

`-a` 里包含了 `-o`（属主）和 `-g`（属组），但**它们只对超级用户有效**。普通用户跑 `rsync -av` 时：

- `-o` `-g` **静默失效**（不报错，属主变成执行用户自己）
- `-p`（权限）仍然生效

所以想让备份保住原始属主属组，要么加 sudo，要么服务端在 daemon 配置里用 `uid`/`gid` 指定，要么开 `fake super = yes`。

### 10.2 符号链接家族

```bash
-l                      # 保留软链接本身（-a 已含）
-L, --copy-links        # 把链接指向的真实文件复制过去，目标端不再是链接
--safe-links            # 忽略指向源目录树之外的软链接（防逃逸，较严格）
--copy-unsafe-links     # 把指向树外的链接当普通文件复制
```

`-L` 有个常见用途和常见风险：用途是"备份时不要留一堆断链"；风险是指向 `/etc/passwd` 这类外部文件时，会被一并复制进备份，既占空间也可能造成敏感信息扩散。安全要求高的场景用 `--safe-links`。

### 10.3 硬链接 / ACL / 扩展属性

这三个必须显式加，`-a` 不带：

```bash
-H              # 保留硬链接关系（不是 -a 的一部分）
-A              # 保留 ACL
-X              # 保留扩展属性（SELinux 上下文、chattr 位等）
```

做系统级迁移、想接近"逐位一致"，就 `-aHAX` 一起上。

### 10.4 UID/GID 映射与强制改写

```bash
--numeric-ids            # 按数字 UID/GID 原样传输，不做用户名映射
                         # ★ 跨机器同步（两台机用户表不同）时强烈建议加，否则属主会张冠李戴
--chown=www:www          # 目标端统一改属主属组
--chown=:www             # 只改属组
--chmod=D0755,F0644      # 目录 0755、文件 0644
```

`--numeric-ids` 这个点很值得记：两台机器的同名用户 UID 不同（比如 A 机 www=1000，B 机 www=1001），不加 `--numeric-ids` 时 rsync 可能把 1000 落成 1001 之外的错误结果，跨环境备份极易踩到。

### 10.5 daemon 模式下属主归谁

- **客户端推送**：落盘文件属主属组由服务端 rsyncd.conf 的 `uid`/`gid` 决定（默认 `nobody`）。
- **客户端拉取**：拉回本地后的属主属组按客户端执行用户的权限视角处理（普通用户拉取时无法还原为原始属主）。
- 想让非 root 的 daemon 仍保留完整属性 → `fake super = yes`，把属性存在扩展属性里。

### 本章考点

- 非 root 时 `-o`/`-g` 静默失效、`-p` 仍生效：想保属主必须 sudo 或 uid/gid 或 fake super。
- `-H -A -X` 不在 `-a` 里，硬链接/ACL/xattr 需显式加。
- 跨机器同步建议 `--numeric-ids`，避免同名不同 UID 导致属主错乱。
- daemon 推送落盘属主 = 服务端 uid/gid，与虚拟用户名无关。

---

## 第十一章 备份方案设计：从"同步"到"可回滚"

前面十章解决的是"把数据搬到对的地方"。这一章解决"搬过去之后还能找回历史版本"。

### 11.1 覆盖前留一份旧版本

```bash
rsync -av --delete --backup --suffix='.bak' /data/src/ /data/dst/
# 目标端被覆盖的同名文件先重命名为 xxx.bak，再写入新内容

rsync -av --backup --backup-dir=/backup/history/$(date +%F) /data/src/ /data/dst/
# 所有被覆盖的旧版本集中归到当天目录，主目录保持整洁
```

`--backup` 的默认后缀是 `~`，用 `--suffix` 可自定义。**注意：`--backup` 处理的是"目标端被覆盖/删除的旧文件"**，它跟 `--delete` 配合时要先想清楚，否则历史目录会被自己塞满或落空。

### 11.2 硬链接快照：--link-dest（rsync 做备份最巧妙的地方）

原理：`--link-dest=DIR` 告诉 rsync —— 传输每个文件前，先去 DIR 里找有没有一模一样的文件（大小 + 时间戳都相同）。有则不重复传，直接在新的目标目录里建一个**指向 DIR 中该文件的硬链接**。

效果：**每个日期目录看起来都是一份完整备份，可以独立浏览；但物理磁盘只多占变化部分的空间。**

理解硬链接是关键：多个文件名指向同一个 inode，只有当所有指向它的硬链接都被删除，磁盘数据才真正释放。

```bash
# 第 1 天：全量
sudo rsync -a /data/src/ /backup/daily.0/

# 第 2 天：以上一天为参照，做增量
sudo rsync -a --link-dest=/backup/daily.0/ /data/src/ /backup/daily.1/
```

**两个硬约束：**

1. 硬链接**不能跨文件系统**，`--link-dest` 指向的目录必须和目标在同一文件系统（同分区）内。
2. 硬链接无法用于目录，所以每次备份的所有目录仍需真实创建，会有一点 inode/目录开销。

空间对比（社区常见估算）：10G 数据、每天变化 500MB，保留 7 份时，每天 `cp` 全量要 70G，而 `--link-dest` 方案约 13.5G（10G + 0.5G×7）。

### 11.3 一套可直接改写的每日轮转脚本

```bash
#!/bin/bash
# /usr/local/bin/rsync-daily-backup.sh
set -euo pipefail

SOURCE="/var/www/html/"                 # 源目录（注意斜杠语义）
BACKUP_ROOT="/backup/website"           # 备份根目录
KEEP_DAYS=14                            # 保留份数
LOG_FILE="/var/log/rsync-backup.log"
EXCLUDE_FILE="/etc/rsync_backup_exclude.list"
DATE_TAG="$(date +%Y-%m-%d_%H%M%S)"

LATEST_LINK="$BACKUP_ROOT/latest"
NEW_SNAP="$BACKUP_ROOT/$DATE_TAG"
PREV_SNAP="$(readlink -f "$LATEST_LINK" 2>/dev/null || true)"

mkdir -p "$NEW_SNAP"

RSYNC_OPTS=(-a --delete --stats -h --human-readable)
[[ -f "$EXCLUDE_FILE" ]] && RSYNC_OPTS+=(--exclude-from="$EXCLUDE_FILE")

if [[ -n "$PREV_SNAP" && -d "$PREV_SNAP" ]]; then
    # 有上一份快照：以它为 link-dest 参照
    rsync "${RSYNC_OPTS[@]}" --link-dest="$PREV_SNAP" "$SOURCE" "$NEW_SNAP/" >>"$LOG_FILE" 2>&1
else
    # 首次：全量
    rsync "${RSYNC_OPTS[@]}" "$SOURCE" "$NEW_SNAP/" >>"$LOG_FILE" 2>&1
fi

# 原子切换 latest 指向
ln -sfn "$NEW_SNAP" "$LATEST_LINK"

# 清理超过保留份数的旧快照（注意：只删目录，硬链接会自动释放）
cd "$BACKUP_ROOT"
ls -1dt */ 2>/dev/null | tail -n +$((KEEP_DAYS + 1)) | while read -r d; do rm -rf "$d"; done

echo "[$(date '+%F %T')] backup done: $NEW_SNAP" >>"$LOG_FILE"
```

使用时：

```bash
sudo chmod +x /usr/local/bin/rsync-daily-backup.sh
sudo /usr/local/bin/rsync-daily-backup.sh      # ★ 首次务必手动跑一遍验证权限
df -h /backup                                  # 观察空间增长是否符合预期
```

**这个脚本的护栏在哪：** 每个日期快照都是独立可浏览的完整视图，即使源目录被误删，`--delete` 影响的只是"新一轮快照"，历史快照里的数据仍在（除非你手动清理超过 KEEP_DAYS 的部分）。这正是第八节"防误删"推荐的方案。

### 11.4 搬运而非备份：--remove-source-files

```bash
rsync -av --remove-source-files /data/incoming/ user@archive:/archive/$(date +%F)/
```

传输成功后删除源端的**非目录文件**（目录本身不会被删）。适合"收集 + 清空"的管道，但请注意：它不是备份语义，源端删掉后如果目标端出问题，数据就双输。建议加 `-n` 先验，且只在有校验环节（如 `--checksum` 复核）的流水线里使用。

### 本章考点

- `--backup` + `--backup-dir` 是把被覆盖的旧版本归档，不是自动历史版本管理。
- `--link-dest` 用硬链接实现"逻辑全量、物理增量"，参照目录必须与目标同文件系统。
- 目录不能硬链接，所以每次快照仍有目录开销。
- `latest` 软链 + 日期目录 + 轮转清理，是 rsync 备份方案的标准三件套。
- `--remove-source-files` 删的是源端非目录文件，语义是搬运不是备份。

---

## 第十二章 自动化：定时与实时

### 12.1 crontab 定时同步

```bash
crontab -e

# 每天 02:00 增量同步到备份服务器（daemon 模式，免交互）
0 2 * * * /usr/bin/rsync -az --delete --max-delete=2000 --password-file=/etc/rsync.passwd /data/src/ rsync_backup@192.168.1.100::daily_backup >> /var/log/rsync-cron.log 2>&1

# 每天 03:30 执行本地快照脚本
30 3 * * * /usr/local/bin/rsync-daily-backup.sh
```

三个易错点：

1. **cron 环境里 PATH 很短**，rsync 要写绝对路径（`which rsync` 查）。
2. **crontab 里 `%` 是注释符**，`$(date +%F)` 这类写法中的 `%` 需转义为 `\%`，或干脆放进脚本里调用。
3. **免交互是前提**：SSH 模式必须先做免密，daemon 模式必须配 `--password-file`（权限 600）。

### 12.2 systemd service + timer（比 cron 更可观测）

```ini
# /etc/systemd/system/filesync.service
[Unit]
Description=rsync incremental sync

[Service]
Type=oneshot
ExecStart=/usr/bin/rsync -az --delete --max-delete=2000 /data/src/ deploy@192.168.1.100:/data/dst/
```

```ini
# /etc/systemd/system/filesync.timer
[Unit]
Description=run filesync daily

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now filesync.timer
systemctl list-timers --all
journalctl -f -u filesync.service
```

`Persistent=true` 会在机器错过时间点（比如关机期间）后补跑，这是 cron 不具备的能力。

### 12.3 inotify + rsync：近实时同步

定时任务有分钟级延迟，主备热备这类场景等不了。解法是"事件驱动"：用内核 inotify 机制监控目录变化，一旦有事件就触发一次 rsync。

**先调内核参数**（默认监控规模太小，大目录会丢事件）：

```bash
cat /proc/sys/fs/inotify/max_queued_events    # 默认 16384，事件队列长度
cat /proc/sys/fs/inotify/max_user_instances   # 默认 128，每用户实例数
cat /proc/sys/fs/inotify/max_user_watches     # 默认 8192，每实例监控文件数

sudo tee -a /etc/sysctl.conf >/dev/null <<'EOF'
fs.inotify.max_queued_events = 16384
fs.inotify.max_user_instances = 1024
fs.inotify.max_user_watches = 1048576
EOF
sudo sysctl -p
```

**装工具并测一次监控**：

```bash
sudo yum install inotify-tools -y      # 或 apt-get install inotify-tools

# -m 持续监控  -r 递归  -q 精简输出  -e 指定事件
inotifywait -mrq -e modify,create,attrib,move,delete /var/www/html/
# 另开终端 touch 一个文件，能看到 CREATE 事件即为成功
```

**触发式同步脚本**：

```bash
#!/bin/bash
# /usr/local/bin/rsync-realtime.sh
SRC="/var/www/html"
DEST="rsync_backup@192.168.1.200::daily_backup"
PASS="/etc/rsync.passwd"
LOG="/var/log/rsync-realtime.log"

# 首次先做一次全量对齐，之后进入事件驱动的增量阶段
rsync -az --delete --password-file="$PASS" "$SRC/" "$DEST" >>"$LOG" 2>&1

inotifywait -m -r -q --format '%T %w%f %e' --timefmt '%F %T' \
  -e modify,create,attrib,move,delete "$SRC" \
| while read -r line; do
    rsync -az --delete --password-file="$PASS" "$SRC/" "$DEST" >>"$LOG" 2>&1
    echo "[$line] synced" >> "$LOG"
  done
```

```bash
sudo chmod +x /usr/local/bin/rsync-realtime.sh
sudo nohup /usr/local/bin/rsync-realtime.sh >/dev/null 2>&1 &
tail -f /var/log/rsync-realtime.log
```

生产上更稳妥的做法是把这段循环包成 systemd 服务并配 `Restart=always`，进程崩溃能自动拉起。

### 12.4 实时同步的固有局限（必须知道）

1. **事件不等于事务边界**。inotify 只报告"发生了什么"，不保证文件此刻已经写完。稳妥策略是监听 `close_write` 而非 `modify`，或加延迟合并窗口，避免同步到半写文件。
2. **高频小变更要做去抖**。上面的脚本每次事件都跑一遍全量比对，写入很密时会反复触发；实际做法是合并短时间内的多次事件（如 sleep 2~5 秒再执行），或用 sersync 这类带并发与重试的成熟工具替代手写脚本。
3. **队列溢出会丢事件**。并发写入极大时 `max_queued_events` 可能溢出，丢事件就意味着永久不一致。所以实时同步必须**定期用 rsync 全量校验（配合 `-c`）兜底**。
4. **双向同步要防环**。A↔B 互相触发会导致无限循环或互相覆盖。要么设计"只由变更端发起"的规则，要么加 exclude 隔离各自的临时文件与备份目录，要么改用专为此设计的工具（如 Syncthing/DRBD）。

### 本章考点

- crontab 里 `%` 需转义、rsync 要写绝对路径、免交互是前提。
- systemd timer 的 `Persistent=true` 可补跑错过的任务，比 cron 可靠。
- inotify 三个内核参数默认值偏小，监控大目录前必须调。
- 事件通知不代表写入完成；实时同步必须搭配定期全量校验兜底，且双向同步需专门防环。

---

## 第十三章 实战案例集

### 案例一：本地发布代码到 Web 服务器

```bash
# 1. 干跑，看清将传什么、将删什么
rsync -avinz --delete --stats \
  --exclude='.git/' --exclude='node_modules/' --exclude='.env' --exclude='*.log' \
  ./dist/ deploy@192.168.1.100:/var/www/html/

# 2. 确认后正式执行（限速 + 保留权限/ACL/xattr）
rsync -avzHAX --delete --max-delete=200 --bwlimit=5000 --stats \
  --exclude='.git/' --exclude='node_modules/' --exclude='.env' --exclude='*.log' \
  ./dist/ deploy@192.168.1.100:/var/www/html/

# 3. 一致性复核（只看差异，不传输）
rsync -avinc --delete ./dist/ deploy@192.168.1.100:/var/www/html/
```

要点：结尾斜杠保证把 `dist` 的内容而不是 `dist` 目录本身推上去；`--delete` 清掉旧版本残留文件；`--max-delete` 防源码目录异常导致站点被清空。

### 案例二：把远端日志归集到中心机

```bash
# 拉取，按主机名分目录，不做 --delete（保留远端日志）
rsync -avz --copy-links \
  deploy@web01:/var/log/app/ /central/logs/$(hostname)/
```

日志里常有正在写入的文件，`--copy-links` 是为了把软链（如 `current -> 2026-10-01.log`）指向的真实文件带回来。这里不加 `--delete`，因为目标是"归集"而非"镜像"。

### 案例三：备份服务器 daemon 模式完整搭建

**服务端（192.168.1.100）**

```bash
sudo yum install rsync rsync-daemon -y
sudo useradd -M -s /sbin/nologin rsync
sudo mkdir -p /backup && sudo chown -R rsync:rsync /backup

sudo tee /etc/rsyncd.conf >/dev/null <<'EOF'
uid = rsync
gid = rsync
use chroot = no
max connections = 200
timeout = 300
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsyncd.lock
log file = /var/log/rsyncd.log
dont compress = *.gz *.tgz *.zip *.z *.Z *.rpm *.deb *.bz2

[daily_backup]
comment = daily backup
path = /backup
read only = no
list = no
ignore errors
hosts allow = 192.168.100.0/24
hosts deny = *
auth users = rsync_backup
secrets file = /etc/rsyncd.secrets
EOF

echo 'rsync_backup:<占位密码>' | sudo tee /etc/rsyncd.secrets
sudo chmod 600 /etc/rsyncd.secrets

sudo systemctl enable --now rsyncd
sudo firewall-cmd --permanent --add-port=873/tcp && sudo firewall-cmd --reload
sudo ss -tlnp | grep 873
```

**客户端（各业务机）**

```bash
sudo yum install rsync -y
echo '<占位密码>' | sudo tee /etc/rsync.passwd      # ★ 只写密码，不带用户名
sudo chmod 600 /etc/rsync.passwd

# 验证连通与模块可见
rsync --list-only rsync_backup@192.168.1.100:: 2>/dev/null || echo "list 已隐藏属正常"

# 推送（先干跑）
sudo rsync -avzn --delete --max-delete=1000 --password-file=/etc/rsync.passwd \
  /data/src/ rsync_backup@192.168.1.100::daily_backup/$(hostname)
```

### 案例四：基于 link-dest 的历史快照（可回滚）

接第十一章脚本，配到 systemd timer 或 cron 即可。恢复任意时点：

```bash
ls /backup/website/                     # 每个日期目录都是完整视图
cat /backup/website/latest/index.html   # 最新
sudo rsync -av /backup/website/2026-10-01_020000/ /var/www/html/   # 回滚到某天
```

想核对快照是否真省空间：

```bash
du -sh /backup/website/*/              # 每份看着都是全量
du -sh /backup/website                 # 总量只多占变化部分
```

### 案例五：迁移数据到 NAS

```bash
# 前提：ECS 已挂载 NAS 到 /mnt，且安全组放行 TCP 22
rsync -avP /data/DirToSync/ root@192.0.2.10:/mnt/DirToSync/
```

云厂商文档特别强调：**源路径结尾必须带斜杠**，否则同步后路径层级不匹配（与第二章的斜杠规则同源）。海量小文件想提效，可以按目录切分并发多路 rsync。

### 本章考点

- 发布走 SSH + `--delete --max-delete` + 三步走（干跑、执行、`-nc` 复核）。
- 归集日志不加 `--delete`；镜像才加——语义区分清楚再动手。
- daemon 搭建的四个必做项：建 uid 用户与目录属主、密码文件 600、`read only = no`、放行 873。
- 快照式备份（link-dest）的价值在于任意时点可独立浏览与回滚。

---

## 第十四章 常见故障排查

### 14.1 权限类

| 症状 | 根因 | 处置 |
| --- | --- | --- |
| `rsync: mkstemp "/.xxx.KkkKKK" failed: Permission denied (13)` | 目标目录缺 w/x 权限（rsync 先写临时文件再改名） | 检查并授予写权限；daemon 模式则改 `uid`/`gid` 或 `setfacl` |
| 推送成功但属主全是 `nobody` | daemon 的 `auth users` 是虚拟用户，落盘身份由未显式配置的 `uid`/`gid` 决定（默认 nobody） | 在 rsyncd.conf 指定 `uid`/`gid`，并把目录属主给它 |
| 普通用户跑 `-av` 后属主属组丢了 | `-o`/`-g` 仅 root 有效，非 root 会静默忽略 | 加 sudo，或 `--chown=`，或 `fake super = yes` |
| 权限/ACL/xattr 没保住 | `-a` 不含 `-A`/`-X`；ACL 需单独保留 | 用 `-aHAX` |
| 目标 SELinux 环境访问被拒 | 内核安全策略拦截，不是文件权限问题 | 查 `ausearch -m avc -ts recent`，按 SELinux 方式处理 |

### 14.2 连接类

```bash
# daemon 端口不通
sudo firewall-cmd --list-ports                 # 本地防火墙有没有放 873
sudo ss -tlnp | grep 873                       # 服务端有没有在听
telnet 192.168.1.100 873                       # 网络层可达性
# 云上还须检查安全组；参见上一份 firewalld 教程的入站放行做法

# SSH 模式报 command not found / 找不到 rsync
#   两端都要装 rsync；远端不在 PATH 时用 --rsync-path 指定

# @@@@@@ WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED @@@@@@
ssh-keygen -R 192.168.1.100                    # 主机指纹变更后需清 known_hosts

# 大量小文件时"卡住很久才开始传"
#   属正常：rsync 需要先递归扫描建立清单（增量递归）；先用 -n 评估范围
```

### 14.3 认证类（daemon 模式）

| 症状 | 根因 | 处置 |
| --- | --- | --- |
| `password must be available via the environment or from a file` | 没给密码且非交互环境（cron） | 加 `--password-file=/etc/rsync.passwd` |
| 密码文件正确但始终认证失败 | 客户端密码文件写成了 `user:password` | 客户端**只写密码**；服务端才写 `user:password` |
| `secrets file - read permissions should be restricted` / 认证被拒 | 密码文件权限不是 600，`strict modes` 拦截 | `chmod 600`；或改 `strict modes = no`（不建议） |
| `read error: Connection reset by peer` 推送时 | 模块 `read only = yes` | 改 `no` 并重启服务 |

### 14.4 数据类

**"为什么每次都要重传同样的文件？"** 逐条排查：

1. mtime 一直变（如远端时钟漂移、容器重建）→ 用 `--size-only` 或修时钟。
2. 权限/属主每次都不同 → 看是否 `-o`/`-g` 在两端映射不一致，试 `--numeric-ids`。
3. 目标被其他进程改动 → 排查并发写入。
4. 内容真的在变（追加型日志）→ 属正常，或用 `--append`。

**"改了内容 rsync 却没传"**：默认 quick check 只看大小与 mtime。用 `-c` 按校验和判断即可。

**"目标端多了不该有的文件"**：默认行为就是这样（rsync 不删东西）。要镜像就加 `--delete`，并务必先 `-n` 审清单。

**"备份目录一夜变空/被删了一大片"**：源路径写错层级（最常见是**忘了结尾斜杠**，把上层目录当成了镜像源）或源端异常清空。这就是第八章 `--max-delete` 与快照方案存在的意义。恢复思路：先看 link-dest 历史快照，再看 `--backup-dir` 归档，最后考虑文件系统级恢复（extundelete 等，成功率取决于是否已被覆盖写）。

### 14.5 排障顺序口诀

**一斜二权三连通，四验干跑五看日志**：

1. 斜杠（源路径层级对不对）
2. 权限（目录 w/x、属主、SELinux）
3. 连通（873/22 端口、防火墙、安全组、两端是否装 rsync）
4. 干跑（`-n` + `-i` 看清单）
5. 日志（`--log-file` / `/var/log/rsyncd.log` / `journalctl`）

---

## 第十五章 速记卡：背诵区

### 15.1 命令分组速查

**本地同步**

```bash
rsync -av /src/ /dst/                        # ★ 基本同步（源带斜杠=传内容）
rsync -avP big.iso /mnt/nas/                 # 大文件 + 进度 + 断点续传
rsync -avz --stats /src/ /dst/               # 看统计
rsync -avHAX --info=progress2 / /mnt/new/    # 整盘克隆（含 -x 时不跨文件系统）
rsync -avn --delete --stats /src/ /dst/      # ★ 干跑审删除
```

**远程（SSH 模式）**

```bash
rsync -avz /local/ user@host:/remote/                         # ★ 推送
rsync -avz user@host:/remote/ /local/                         # ★ 拉取
rsync -avz -e "ssh -p 2222 -i ~/.ssh/id_key" /local/ user@host:/remote/
rsync -avz --delete --exclude='*.log' ./dist/ deploy@host:/var/www/html/
```

**守护进程（daemon 模式）**

```bash
rsync -avz /src/ user@host::module                            # ★ 推送
rsync -avz user@host::module/ /dst/                           # ★ 拉取
rsync -avz --password-file=/etc/rsync.passwd /src/ user@host::module
rsync --list-only user@host::                                 # 列模块
rsync -avz rsync://user@host/module/ /dst/                    # URL 写法（等价）
```

**服务端运维**

```bash
sudo rsync --daemon                          # 手工起
sudo systemctl start rsyncd                  # systemd（需 rsync-daemon 包）
sudo ss -tlnp | grep 873                     # 确认监听
sudo kill $(cat /var/run/rsyncd.pid)         # 停（手工方式）
```

**过滤 / 限速 / 校验**

```bash
--exclude='*.log'  --exclude-from=list.txt  --files-from=清单.txt
--include='*.txt' --exclude='*'              # 白名单（顺序敏感）
--bwlimit=5000                               # 限速 KB/s
-c, --checksum                               # 按校验和判断差异
rsync -avnc /src/ /dst/                      # 干跑 + 校验和：只报告不一致
```

**备份方案**

```bash
rsync -a --link-dest=/backup/prev/ /src/ /backup/$(date +%F)/   # ★ 硬链接快照
rsync -av --backup --backup-dir=/backup/hist/$(date +%F) /src/ /dst/
ln -sfn /backup/2026-10-04 /backup/latest
```

### 15.2 十二个必会参数

| 参数 | 一句话 |
| --- | --- |
| `-a` | 归档模式 = `-rlptgoD`，递归并保留属性 |
| `-v` | 显示过程 |
| `-z` | 传输压缩（网络用，本地无意义） |
| `-n` | 干跑，只报不做（安全阀） |
| `-P` | `--partial --progress`，断点续传 + 进度 |
| `-H` | 保留硬链接（不在 -a 里） |
| `--delete` | 让目标成为源的镜像，删除多余文件 |
| `--max-delete` | 删除数量上限（保险丝） |
| `--exclude` | 排除规则 |
| `--bwlimit` | 限速，单位 KB/s |
| `-c` | 用校验和而非大小+时间判断差异 |
| `--link-dest` | 硬链接参照目录，实现"逻辑全量、物理增量" |

再加三个 daemon 专用：`-e`（指定远程 shell）、`--password-file`（免交互认证）、`--numeric-ids`（跨机保属主不乱）。

### 15.3 高频问答八条

**Q1：rsync 和 scp 的区别？**
A：scp 每次全量复制，不能跳过未变更文件、不能断点续传、不能限速、不支持删除同步；rsync 只传差异（默认按大小+修改时间判定，可选校验和），支持 `-P` 续传、`--bwlimit` 限速、`--delete` 镜像。两者都能走 SSH 加密。小文件一次性拷贝 scp 更简单，同步/备份场景 rsync 完胜。

**Q2：-a 等价于什么？**
A：`-rlptgoD`——递归、保留软链接、权限、时间、属组、属主、设备文件。它**不包含** `-H`（硬链接）、`-A`（ACL）、`-X`（扩展属性）。

**Q3：源路径加不加结尾斜杠有什么区别？**
A：加斜杠表示同步"目录里的内容"（目标端不出现该目录名），不加表示同步"目录本身"（目标端多一层同名目录）。这是 rsync 最高频的坑，用 `-n` 干跑即可提前发现。

**Q4：--delete 会不会删源端文件？**
A：不会，它删的是**目标端**那些源端已不存在的文件。真正的风险是源端被误删后，下一次带 `--delete` 的同步会把备份端也删掉，所以务必配 `--max-delete`，并用 link-dest 快照保留历史。

**Q5：--link-dest 的原理和限制？**
A：让新快照里未变化的文件以硬链接指向参照目录中的同一 inode，于是每个日期目录都能独立完整浏览，而磁盘只多占变化部分。限制：参照目录必须与目标在同一文件系统（硬链接不能跨文件系统），且目录本身无法硬链接、每次仍会新建。

**Q6：什么时候用 daemon 模式而不是 SSH 模式？**
A：需要模块级目录隔离、接入大量客户端、不想为每台客户端开系统账号（改用 `auth users` 虚拟用户）时用 daemon 模式（监听 873，走 rsyncd.conf + 密码文件）。只是两台机器少量互传就用 SSH 模式，配置最省。

**Q7：rsync 怎么判断文件需不需要传？不準时怎么办？**
A：默认 quick check 只比"大小 + 最后修改时间"。当时间戳被外部工具改动、或内容变了但大小和时间都没变时，会漏传，此时用 `-c`（`--checksum`）按校验和比对；反向的情形（两端时钟不一致导致反复重传）可用 `--size-only`。

**Q8：daemon 模式连不上，先查什么？**
A：按序查——服务端是否在听 873（`ss -tlnp`）、防火墙与安全组是否放行、密码文件权限是否 600、客户端密码文件是否误写成 `user:password`（应只写密码）、模块 `read only` 是否为 yes（推送需 no）、虚拟用户对 `path` 目录有无写权限（`mkstemp ... Permission denied`）。

---

## 结尾：接下来的学习路线

**把 rsync 用到够用是什么标准？** 能做到这三件事就过关了：一是写完任何同步命令前先 `-n` 干跑并看 `-i` 清单；二是能讲清 `-a` 的展开、斜杠语义、quick check 三件事；三是能独立搭起一台带 `--link-dest` 快照的备份机并接上定时任务。

再往上走：

1. **读两篇 man**：`man rsync`（重点看 quick check 那节和 ITEMIZE CHANGE STANZAS，`-i` 输出每个字母的权威定义都在那里）、`man rsyncd.conf`（daemon 全部参数）。忘语法时 `rsync --help` 按分组看比查文档快。
2. **理解"同步"与"备份"的分界**。rsync 只对齐两边状态，没有去重、加密、版本历史。真要长期留存数据，去看 rsnapshot（rsync + 硬链接的成熟封装）、rdiff-backup（反向差异）、BorgBackup / restic（去重 + 加密 + 保留策略）。
3. **补上相邻能力**。海量小文件的实时分发（sersync、Dragonfly）、跨地域大文件加速、块级同步（DRBD）、对象存储直传（各云厂商的迁移工具），这些是 rsync 之后自然要面对的边界。
4. **安全底线**：`--delete` 配 `--max-delete`；定时任务先手动跑一遍验权限；跨机同步加 `--numeric-ids`；不要为了省事把真实密码写进任何明文文档或聊天记录，凭据一律用 `chmod 600` 的文件并纳入密码管理流程。

---

## 参考文献

1. Linux Foundation Referenced Specifications. rsync（选项语义与 -a/-r/-R/-b/-u/-l/-L 等定义）. https://refspecs.linuxbase.org/LSB_1.2.0/gLSB/rsync.html
2. man7.org. rsync(1) Linux manual page（quick check、--itemize-changes、--link-dest）. https://linux.die.net/man/1/rsync
3. man7.org. rsyncd.conf(5) — configuration file for rsync in daemon mode. https://man7.org/linux/man-pages/man5/rsyncd.conf.5.html
4. Ubuntu Manpages. rsyncd.conf(5). https://manpages.ubuntu.com/manpages/trusty/man5/rsyncd.conf.5.html
5. Rocky Linux Documentation. rsync 简述 / rsync demo 01 / rsync 演示02 / rsync configuration file（zone 匹配、mkstemp 与 setfacl、uid/gid、rsync-daemon 包）. https://docs.rockylinux.org/zh/books/learning_rsync/01_rsync_overview/
6. Red Hat Documentation. 22.4 Configuration Examples — SELinux User's and Administrator's Guide（rsync 作为 daemon 的配置与安全策略）. https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-managing_confined_services-rsync-configuration_examples
7. firewalld.org. rsync 官方网站入口 https://rsync.samba.org（项目主页与 man 页）
8. Unix & Stack Exchange. Basic rsync command for bit-identical copies（-a 与 -H/-A/-X 关系、quick check 原文、增量递归与 --no-inc-recursive）. https://unix.stackexchange.com/questions/118883/basic-rsync-command-for-bit-identical-copies
9. 阿里云. 如何将非阿里云数据迁移 NAS 或将 NAS 数据迁移至线下（源路径必须带正斜杠、并发拷贝思路）. http://www.alibabacloud.com/help/zh/nas/user-guide/migrate-data-to-a-nas-file-system
10. linuxize.com. rsync Incremental Backups with --link-dest. https://linuxize.com/post/rsync-incremental-backups-with-link-dest/
11. Server Fault. Is Rsync --link-dest Saving Space（目录无法硬链接带来的额外开销成因）. https://serverfault.com/questions/605324/is-rsync-link-dest-saving-space
12. 阿里云开发者社区. 运维必看，Linux 远程数据同步工具详解（rsyncd.conf 参数逐项、xinetd 与独立运行方式）. https://developer.aliyun.com/article/1588494
13. 阿里云开发者社区. 配置 inotify + rsync 实现实时同步（三个内核参数默认值与调优、inotifywait 选项、触发脚本）. https://developer.aliyun.com/article/1105656
14. C语言中文网. Linux rsync 命令的用法（非常详细，附带示例）（斜杠语义、--exclude/--include、隐藏文件、大括号批量排除）. https://c.biancheng.net/view/c0ly7yc.html
15. 博客园. rsync 参数说明及使用参数笔记（rsyncd.conf 全参数注解、客户端密码文件只写密码）. https://www.cnblogs.com/koushuige/articles/9162895.html
16. 博客园. Rsync 远程同步指南 / rsync 进阶用法（完整备份与增量备份概念、模块认证语法）. https://www.cnblogs.com/dream-myself/articles/18893116.html
17. jhpce.jhu.edu. Itemize Output Key（-i 输出字段对照表）. https://jhpce.jhu.edu/files/rsync-itemize-table/

### 口径与差异说明（客观呈现）

- **9.1 节 `--delete-during` 是否为默认**：不同 rsync 版本行为有差异（较新版本在启用 `--delete` 时通常采用 during 策略）。文中未写死版本号，实际默认值请以 `rsync --help | grep delete` 与 `man rsync` 为准。
- **11.2 节的节省空间估算（10G/天变 500MB/7 份 ≈ 13.5G）**：来自社区实践文章的示例计算，属**示意性推算**而非实测数据，真实收益取决于文件数量与变更分布。
- **12.1 节 cron 的 `%` 转义、`--link-dest` 参照目录须与目标同文件系统**：属通用 Unix/cron 与硬链接机制约束，非 rsync 专有描述，已在正文以机制原理解释。
- **`--max-size` / `--min-size`**：版本可用性差异较大，文中未断言具体起始版本，改为给出"先自证再使用"的方法。

---

内容由 AI 生成
