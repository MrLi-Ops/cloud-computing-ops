# firewalld 防火墙从入门到使用

## 写在前面

如果你接手过一台 RHEL、CentOS、Rocky、AlmaLinux、Fedora、银河麒麟或 openEuler 的服务器，大概率绕不开一个命令：`firewall-cmd`。它是 firewalld 的命令行客户端，而 firewalld 从 CentOS 7 起取代 iptables service，成为 Red Hat 系发行版的默认防火墙管理工具。

这份文档的定位是**教学**，不是命令清单。它会带你按顺序走完五站：

1. 搞清楚 firewalld 到底是什么、和 iptables 谁管谁
2. 吃透四个核心概念：区域、服务、端口、富规则
3. 掌握日常八九成的操作命令
4. 学会 NAT、ICMP、自定义区域与服务这些进阶能力
5. 用真实案例串起来，并建立一套排障思路

每一节末尾有「本节考点」，全部学完后可以直接翻到第十一章的速记卡背诵。文中所有命令都需要 root 权限，建议前面加 `sudo`。

阅读约定：示例中出现的 IP（如 `192.168.1.100`）、网卡名（如 `ens33`）均为占位演示值，实际使用请替换为你环境的真实值；标注为「示意输出」的代码块，是命令的典型返回格式，不是你机器的真实结果。

---

## 第一章 入门认知：firewalld 到底是什么

### 1.1 它是防火墙管理工具，不是防火墙本体

新手最大的困惑是：我明明配了 firewalld，为什么 `iptables -L` 也能看到规则？要解开这个结，先记住一句话：

**firewalld 和 iptables 都不会过滤数据包，真正在内核里干活的是 netfilter。**

把它们理解成三层结构：

```
用户操作层   firewalld（守护进程 + firewall-cmd / firewall-config）
             iptables / nft 命令
                    │  这一层只负责"编写和维护规则"
                    ▼
内核执行层   netfilter / nftables  ← 真正做过滤、丢弃、转发的地方
```

firewalld 提供的是一个守护进程（`firewalld.service`）、一个命令行客户端（`firewall-cmd`）、一个图形配置工具（`firewall-config`）和一套 D-Bus 接口。它替代的是过去 `iptables service` 那一层"规则保存与加载"的活，而不是替代内核的包过滤能力。

### 1.2 后端是 iptables 还是 nftables

firewalld 自己不写内核规则，它把配置翻译给后端去落地。后端有两个：

| 场景 | 默认后端 |
| --- | --- |
| CentOS / RHEL 7 | iptables |
| CentOS / RHEL 8 及之后、Fedora 较新版本 | nftables（firewalld 0.6.0 起将 nftables 设为默认） |

查看与切换的位置在配置文件 `/etc/firewalld/firewalld.conf` 的 `FirewallBackend` 项，可取 `nftables`（默认）或 `iptables`。

这带来一个实用差异：在 CentOS 7 上，你用 `firewall-cmd` 配的规则基本都能通过 `iptables -L` 看到；但在 RHEL 8/9 上，部分规则用 `iptables` 命令看不全，要看真实生效结果应该用 `nft list ruleset`。

### 1.3 所谓"动态防火墙"体现在哪

对比的是老式 iptables service 的工作模式：

- **静态模式（iptables service）**：你改一条规则，它把整张规则表清空后重新加载一遍。远程操作时这个瞬间极危险——如果 INPUT 默认策略是 DROP，清空的一瞬间你的 SSH 连接就断了，人直接被锁在服务器外面。
- **动态模式（firewalld）**：只把变更的那部分增量更新到运行中的规则集，不重载全表，已建立的 SSH 连接、业务 TCP 连接不受影响。

这就是生产环境偏爱 firewalld 的首要原因：**改规则不断线**。

### 1.4 firewalld 只管入站

另一个必须提前建立的正确认知：**firewalld 默认管控的是入站（inbound）流量，本机主动向外发起的连接默认放行**。所以"我防火墙开了，为什么还能 curl 外网"不是配置出错，而是设计如此。

### 本节考点

- firewalld 自身不具备防火墙功能，底层靠内核 netfilter 实现，它是 iptables service 的替代者。
- firewalld 是动态防火墙：增量更新，不需要重载全部规则；iptables service 是静态的：改一条也要全表重读。
- 后端二选一：老系统 iptables，新系统 nftables；不要同时混用 firewalld 与手动 iptables 规则。
- firewalld 只管入站流量，出站默认放行。

---

## 第二章 四个核心概念：全文的地基

firewalld 的所有命令，本质上都在围绕这四个概念做增删改查。先把它们理顺，后面就不需要死记硬背。

### 2.1 区域 zone：一套预设的信任策略

zone 是 firewalld 最有特色的设计。**一个 zone 就是一套过滤规则集合**，可以把它想象成机场里不同的安检通道——有的严格、有的宽松、有的只查证件、有的开箱检查。

数据包要进出系统，必须先"选中一条通道"。你根据"对这个网络的信任程度"，把网卡或来源 IP 挂到对应的 zone 上，这部分流量就套用那套规则。

关键关系是**一对多**：一个连接（网卡/源地址）只能属于一个 zone，但一个 zone 可以承载多个连接。

### 2.2 服务 service：端口的语义化封装

`--add-service=http` 和 `--add-port=80/tcp` 效果是一样的。service 就是给"端口 + 协议"起了个人类可读的名字，封装在一个 XML 文件里。

```
服务名 ──封装──> 端口/协议组合
http   ───────> tcp 80
https  ───────> tcp 443
ssh    ───────> tcp 22
```

用服务名的好处：不用记端口号、不容易写错、语义清晰便于交接。此外部分服务还携带 netfilter helper 模块（如 FTP 的连接跟踪），这类服务直接开端口是配不出正确效果的。

### 2.3 端口 port：协议加端口号

当你的应用没有预定义服务（比如自研服务跑在 8080），就直接放行端口。格式固定为 `端口/协议`，协议可取 `tcp`、`udp`、`sctp`、`dccp`。支持范围写法 `2000-2100/tcp`。

这里最容易犯的错是**协议填错**。DNS 的 53 端口 TCP 和 UDP 都有，只开 `53/tcp` 时解析照样不通。

### 2.4 富规则 rich rule：带条件的复合规则

前三者只能表达"放行某个端口/服务"，而富规则能表达：

- **谁**（源地址、MAC、IP 集合）来的
- 访问**什么**（服务、端口、协议、ICMP 类型）
- 记录**日志**吗
- **限速**吗
- 执行什么**动作**（accept / reject / drop / mark）

一句话：**zone 管"从哪来"，port 和 service 管"到哪去"，rich rule 把这两端和日志、限速组合起来。**

### 2.5 第五个必须知道的概念：runtime 与 permanent

严格说这不是"规则类型"而是"配置状态"，但它是新手踩坑率最高的地方，值得单列。

| 状态 | 怎么写 | 生效时机 | 重启/重载后 |
| --- | --- | --- | --- |
| runtime（运行态） | 默认，不加参数 | 立即生效 | 丢失 |
| permanent（永久态） | 加 `--permanent` | **不生效**，只写入磁盘配置 | 永久保留，但需 reload 才加载进运行态 |

由此产生三个经典坑：

- 只加 `--permanent` 忘记 `--reload`：配置改了，但完全不生效。
- 只写运行态不加 `--permanent`：当场能通，机器一重启规则全没了。
- 生产上为了保险，推荐**同一条规则写两次**：一次 permanent 落盘，一次不带参数立即生效，这样连 reload 都不需要，业务零扰动。

```bash
firewall-cmd --permanent --add-service=http   # 写盘，重启不丢
firewall-cmd --add-service=http               # 立即生效，不用 reload
```

### 本节考点

- zone 与连接是一对多：一个连接只属于一个 zone，一个 zone 可绑定多个连接。
- service 本质是端口协议的命名封装，放行 http 等价于放行 tcp/80。
- 不加 `--zone` 时，所有命令操作的都是**默认 zone**。
- `--permanent` 只写文件不生效，必须 `--reload`；不加 `--permanent` 立即生效但重启丢。

---

## 第三章 上手第一步：启服务、看状态

### 3.1 确认与安装

```bash
# 查看是否已安装
rpm -q firewalld

# 未安装则安装（RHEL/CentOS 系）
sudo yum install firewalld -y      # 或 dnf install firewalld -y
```

### 3.2 服务生命周期管理

firewalld 本身是一个 systemd 服务，用 systemctl 管：

```bash
sudo systemctl status firewalld        # 查看服务状态
sudo systemctl start firewalld         # 启动
sudo systemctl stop firewalld          # 停止（生产环境慎用）
sudo systemctl restart firewalld       # 重启
sudo systemctl enable firewalld        # 开机自启
sudo systemctl disable firewalld       # 取消自启

# 一步到位：启动 + 开机自启
sudo systemctl enable --now firewalld
```

也可以用 firewalld 自己的方式看运行状态：

```bash
sudo firewall-cmd --state
```

输出有三种：`running`（正常）、`not running`（未启动）、`RUNNING_BUT_FAILED`（启动过程中出现过错误）。第三种容易被忽略，但它提示你规则集可能不完整，值得去看日志。

### 3.3 查看类命令族

这组命令是日常使用频率最高的，先背下来：

```bash
sudo firewall-cmd --get-zones          # 系统支持哪些 zone
sudo firewall-cmd --get-default-zone   # 默认 zone 是哪个
sudo firewall-cmd --get-active-zones   # 哪些 zone 正在被使用（绑定了网卡/源）
sudo firewall-cmd --list-all           # 默认 zone 的全部规则
sudo firewall-cmd --list-all --zone=home    # 指定 zone 的全部规则
sudo firewall-cmd --list-all-zones     # 所有 zone 的全部规则（很长）
```

`--get-zones` 的示意输出：

```
block dmz drop external home internal public trusted work
```

`--get-active-zones` 的示意输出：

```
public
  interfaces: ens33
```

这行的读法是：ens33 这块网卡挂在 public 上，所以从 ens33 进来的流量都走 public 的规则。

### 3.4 读懂 list-all 的输出

```bash
sudo firewall-cmd --list-all
```

示意输出与逐行解释：

```
public (active)                          # 当前 zone 名；(active) 表示已绑定接口或源
  target: default                        # 没匹配上任何规则时的兜底动作
  icmp-block-inversion: no               # ICMP 屏蔽列表是否反选
  interfaces: ens33                      # 绑定到本 zone 的网卡
  sources:                               # 绑定到本 zone 的源地址/网段
  services: ssh dhcpv6-client            # 本 zone 放行的服务
  ports:                                 # 本 zone 显式放行的端口
  protocols:                             # 本 zone 放行的协议
  forward-ports:                         # 端口转发规则
  source-ports:                          # 按源端口放行的规则
  icmp-blocks:                           # 被屏蔽的 ICMP 类型
  rich rules:                            # 富规则
  masquerade: no                         # 是否开启地址伪装
```

**看防火墙现状，认准 `--list-all` 就够了。** 它比 `--list-ports` 全面得多——因为通过服务放行的端口不会出现在 `--list-ports` 里，只看那个会误判"端口没开"。

### 本节考点

- `firewall-cmd --state` 与 `systemctl status firewalld` 都能看状态，前者看 firewalld 自述，后者看 systemd 视角。
- `--list-all` 是最常用的单条命令，输出含 target、interfaces、sources、services、ports、masquerade、forward-ports、icmp-blocks、rich rules。
- `--list-ports` 只列显式放行端口，不含服务隐含端口；查看整体规则应使用 `--list-all`。

---

## 第四章 区域详解：选对 zone 等于配好一半防火墙

### 4.1 九个内置区域

firewalld 自带 9 个 zone，按信任级别**从低到高**排列如下。这张表建议直接背下来，面试和工作都用得上。

| 区域 | 信任度 | 对入站流量的默认处理 | 典型场景 |
| --- | --- | --- | --- |
| drop | 最低 | 丢弃所有入站包，不返回任何响应，对方只能等到超时 | 想彻底隐身、不让扫描者判断主机存在 |
| block | 极低 | 拒绝所有入站连接，并返回 ICMP 禁止消息（IPv4 为 icmp-host-prohibited，IPv6 为 icmp6-adm-prohibited） | 明确告知对方"不让进" |
| public | 中低 | 只放行选定入站连接（默认 ssh、dhcpv6-client），**新接口的默认区域** | 公共网络、公网服务器 |
| external | 低 | 只放行 ssh，且默认开启 IPv4 地址伪装 | 路由器外网口 |
| dmz | 低 | 只放行 ssh，对外可达、有限访问内网 | 非军事区、对外服务器 |
| work | 中高 | 只放行选定入站连接（ssh、dhcpv6-client 等） | 办公网络 |
| home | 高 | 放行 ssh、dhcpv6-client、mdns、samba-client 等 | 家庭网络 |
| internal | 高 | 默认与 home 相同 | 企业内部网 |
| trusted | 最高 | **放行所有连接** | 完全信任的网络段 |

两个记忆要点：

- `trusted` 里再配 service/port 是毫无意义的，因为已经全放行了。
- `block` 和 `drop` 都是"全拒"，差别只在**回不回应**：block 立刻回拒绝包（对方秒知不可达），drop 静默丢包（对方干等到超时）。两者都不影响本机主动访问外网。

### 4.2 target：zone 的兜底动作

target 决定"数据包没匹配上任何规则时怎么办"。预定义 zone 的 target 基本都是 `default`。可取值：

| target | 含义 |
| --- | --- |
| default | 按该 zone 的规则处理，没匹配上就拒绝 |
| ACCEPT | 全部接受（除被其他规则显式拒绝的） |
| REJECT | 全部拒绝并通知源端 |
| DROP | 全部拒绝且不通知源端 |

在 `--list-all-zones` 输出里，你偶尔会看到 `target: %%REJECT%%` 这类写法，这是 firewalld 内部对 zone 默认拒绝动作的表示形式，不是配置错误，读作 REJECT 即可。

修改 target：

```bash
sudo firewall-cmd --permanent --zone=public --set-target=REJECT
sudo firewall-cmd --permanent --zone=public --get-target
sudo firewall-cmd --reload
```

### 4.3 数据包如何选中 zone（重点）

这是 firewalld 的路由逻辑，务必形成条件反射：

```
1. 先看源地址：数据包来源 IP 是否绑定到某个 zone 的 source  → 命中就用它
2. 再看入站网卡：从哪块网卡进来的，该网卡绑定了哪个 zone     → 命中就用它
3. 都没命中：交给默认 zone（public）
```

简记：**source 优先于 interface，最后是 default**。

另有一个高频细节：firewalld 默认把 **lo 回环接口映射到 trusted 区域**。所以本机 127.0.0.1 的服务访问不会被防火墙拦住——这也是为什么"本机 curl 自己通了，外部访问不通"完全正常。

### 4.4 查看与绑定

```bash
# 查网卡属于哪个 zone
sudo firewall-cmd --get-zone-of-interface=ens33

# 把网卡切到指定 zone（change 用于已绑定的，add 用于尚未绑定的）
sudo firewall-cmd --zone=home --change-interface=ens33
sudo firewall-cmd --permanent --zone=home --change-interface=ens33

# 查源地址属于哪个 zone
sudo firewall-cmd --get-zone-of-source=10.1.1.0/24

# 把某个网段的流量交给 home 处理
sudo firewall-cmd --add-source=10.1.1.0/24 --zone=home
sudo firewall-cmd --permanent --add-source=10.1.1.0/24 --zone=home
sudo firewall-cmd --reload

# 相关查询/移除
sudo firewall-cmd --list-sources --zone=home
sudo firewall-cmd --query-source=10.1.1.0/24 --zone=home
sudo firewall-cmd --remove-source=10.1.1.0/24 --zone=home
sudo firewall-cmd --remove-interface=ens33 --zone=home
```

关于永久绑定网卡要多留一句：`firewall-cmd` 配合 `--permanent` 操作接口时，通常会涉及更新 NetworkManager 的连接配置，以保证重启后绑定关系仍在。这也是 Red Hat 文档特别点出的 firewalld 与 NetworkManager 的联动机制。如果你的机器没用 NetworkManager，网卡绑定更稳妥的做法是写在网卡配置里。

改默认 zone（注意：这条命令会同时改运行态和永久配置，立即生效）：

```bash
sudo firewall-cmd --set-default-zone=public
```

### 4.5 自定义 zone

内置九个不够用时可以自己建。以新建一个专门给 Web 用的 `myweb` 为例：

```bash
# 1. 创建（自定义 zone 必须用 --permanent）
sudo firewall-cmd --permanent --new-zone=myweb

# 2. 重载，新 zone 才会出现在运行态
sudo firewall-cmd --reload

# 3. 配置它：设兜底动作、放行需要的服务
sudo firewall-cmd --permanent --zone=myweb --set-target=REJECT
sudo firewall-cmd --permanent --zone=myweb --add-service=http
sudo firewall-cmd --permanent --zone=myweb --add-service=https

# 4. 把网卡挂上去，再重载生效
sudo firewall-cmd --permanent --zone=myweb --add-interface=ens33
sudo firewall-cmd --reload

# 删除自定义 zone
sudo firewall-cmd --permanent --delete-zone=myweb
sudo firewall-cmd --reload
```

### 本节考点

- 九个 zone 从 drop 到 trusted，信任度递增；public 是新接口默认区域，trusted 全放行。
- zone 匹配顺序：source 绑定 > interface 绑定 > 默认 zone。
- lo 回环默认属于 trusted，因此本机互访不受防火墙限制。
- 自定义 zone 必须 `--permanent --new-zone` 后 `--reload` 才可见。

---

## 第五章 日常操作主力：服务与端口

命令呈现固定套路：**add 添加、remove 删除、query 查询是否存在、list 列出全部**。掌握这个规律，几十个参数就不用逐个背。

### 5.1 服务管理（推荐首选）

```bash
# 查看系统支持的所有预定义服务（很长，几十个）
sudo firewall-cmd --get-services

# 查看当前 zone 已放行的服务
sudo firewall-cmd --list-services

# 放行服务
sudo firewall-cmd --add-service=http                        # 运行态，立即生效
sudo firewall-cmd --permanent --add-service=http            # 永久态
sudo firewall-cmd --permanent --zone=public --add-service=https

# 查询某服务是否已放行（返回 yes/no）
sudo firewall-cmd --query-service=http

# 移除服务
sudo firewall-cmd --remove-service=http
sudo firewall-cmd --permanent --remove-service=http
sudo firewall-cmd --reload
```

一条命令可以连续加多个服务，用多个 `--add-service`：

```bash
sudo firewall-cmd --permanent --zone=public --add-service=http --add-service=https
```

### 5.2 端口管理

没有预定义服务时直接开端口：

```bash
# 单个端口
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=53/udp

# 端口范围
sudo firewall-cmd --permanent --add-port=2000-2100/tcp

# 指定 zone
sudo firewall-cmd --permanent --zone=work --add-port=3689/tcp --add-port=5353/udp

# 查看已放行端口
sudo firewall-cmd --list-ports

# 查询某端口是否放行（yes/no）
sudo firewall-cmd --query-port=8080/tcp

# 移除端口
sudo firewall-cmd --permanent --remove-port=8080/tcp
sudo firewall-cmd --reload
```

三个注意点：

1. **协议必须写对**。TCP 与 UDP 是两条独立规则，只开一个通常不够。
2. **移除时参数要和添加时完全一致**，`--remove-port=8080/tcp` 删不掉 `8080/udp`。
3. **端口范围用短横线**，写成 `2000:2100` 会报错。

### 5.3 让规则自动过期：timeout

排障或临时联调时，希望规则到期自动消失，避免留下长期敞口：

```bash
# 放行 30 分钟后自动撤销
sudo firewall-cmd --add-service=http --timeout=30m

# 也支持秒（纯数字）、分钟、小时、天
sudo firewall-cmd --add-port=3306/tcp --timeout=600
sudo firewall-cmd --add-port=3306/tcp --timeout=2h
sudo firewall-cmd --add-port=3306/tcp --timeout=1d
```

`--timeout` 只对运行态规则有效——本来运行态就会随重载丢失，timeout 只是加了个更早的闹钟。

### 5.4 该用 service 还是 port

| 情况 | 选择 |
| --- | --- |
| 有对应的预定义服务 | 用 service。语义清晰，且自动带上该服务需要的 helper 模块 |
| 自研应用、端口非标准 | 用 port |
| 需要改端口的标准服务（如 SSH 跑在 22022） | 用 port 开 22022/tcp，或自建同名 service |

最后一种情况补充说明：SSH 改了端口的话，光 `--add-service=ssh` 是没用的（它只放 22），要么 `--add-port=22022/tcp`，要么自定义一个 service XML 把端口写进去。

### 本节考点

- 命令四件套：add / remove / query / list，配合对象名（service、port、source、interface、icmp-block、rich-rule）即可组合出绝大多数操作。
- add 和 remove 的参数必须完全对称，否则删不掉。
- `--timeout` 给临时规则加自动过期时间，适合联调与应急放行。
- 优先 service 后 port；SSH 改端口后必须单独放行新端口。

---

## 第六章 富规则：真正体现 firewalld 威力的地方

### 6.1 语法骨架

富规则整条是一个字符串，结构如下（源自 `man firewalld.richlanguage`）：

```
rule [family="ipv4|ipv6"]
     [source [NOT] address="地址[/掩码]" | mac="MAC" | ipset="集合名"]
     [destination [NOT] address="地址[/掩码]"]
     <元素>                        # 二选一或多选，见下表
     [log [prefix="前缀"] [level="级别"] [limit value="速率/时长"]]
     [audit]
     <动作>                        # accept | reject | drop | mark
```

元素（element）的可选项：

| 元素 | 写法 | 说明 |
| --- | --- | --- |
| 服务 | `service name="ssh"` | 用服务名匹配 |
| 端口 | `port port="3306" protocol="tcp"` | 支持范围 `1000-2000` |
| 协议 | `protocol value="esp"` | 按 IP 协议号/名匹配，取值参考 /etc/protocols |
| ICMP 屏蔽 | `icmp-block name="echo-request"` | 内部隐含 reject，**不允许再写动作** |
| 地址伪装 | `masquerade` | 内部隐含，**不允许再写动作** |
| 端口转发 | `forward-port port="80" protocol="tcp" to-port="8080" to-addr="10.1.1.100"` | 内部隐含 accept，**不允许再写动作** |
| 源端口 | `source-port port="6800" protocol="udp"` | 按来源端口匹配 |

动作四选一：

| 动作 | 效果 |
| --- | --- |
| accept | 放行 |
| reject | 拒绝并回 ICMP 错误，可用 `type=` 指定错误类型 |
| drop | 静默丢弃，不回应 |
| mark `set="标记[/掩码]"` | 在 mangle 表 PREROUTING 链打标记，用于后续策略路由/QoS |

三条书写铁律：

1. **元素的选项必须紧跟在元素后面**，写错位置会被当成别的元素的参数或直接报错。
2. **用了 source/destination 地址就必须写 family**。
3. **做源黑白名单时（只有 source 没有元素），不能写 destination**。

限速写法：`limit value="速率/时长"`，时长取 `s`/`m`/`h`/`d`，最大值是 `2/d`（每天最多匹配两次）。log 级别取 `emerg`、`alert`、`crit`、`error`、`warning`、`notice`、`info`、`debug`，不写默认 `warning`。

### 6.2 六类高频场景

以下命令统一按"永久写入 + 重载"的标准写法给出。

**场景一：只允许指定 IP 访问指定端口（数据库白名单）**

```bash
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.100" port protocol="tcp" port="3306" accept'
sudo firewall-cmd --reload
```

**场景二：允许整个网段访问某服务**

```bash
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" service name="http" accept'
sudo firewall-cmd --reload
```

**场景三：拉黑单个 IP（全端口拒绝）**

```bash
# drop：静默丢弃，对方等到超时
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.200" drop'

# reject：立即回绝，对方马上看到 Connection refused
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.200" reject'
sudo firewall-cmd --reload
```

两者的现场差异很直观：同网段主机 SSH 上来，`reject` 立刻返回 `Connection refused`，`drop` 则是 `Connection timed out` 干等。

**场景四：放行所有人但拒绝某个网段（SSH 精细控制）**

```bash
# 先放行 ssh 服务
sudo firewall-cmd --permanent --zone=public --add-service=ssh

# 再对该网段拒绝
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="172.16.1.0/24" service name="ssh" reject'
sudo firewall-cmd --reload
```

**场景五：连接限速（缓解暴力破解）**

```bash
# 每分钟最多 3 次 SSH 新建连接
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule service name="ssh" limit value="3/m" accept'
sudo firewall-cmd --reload
```

**场景六：带日志的拒绝（便于取证）**

```bash
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" service name="ftp" log prefix="FTP-BLOCK: " level="warning" limit value="3/m" reject'
sudo firewall-cmd --reload

# 日志落在内核日志，可用下面方式查看
sudo journalctl -k | grep FTP-BLOCK
```

### 6.3 查看、验证与删除

```bash
# 列出富规则
sudo firewall-cmd --list-rich-rules
sudo firewall-cmd --zone=public --list-rich-rules

# 查询某条富规则是否存在（存在返回退出码 0，不存在返回 1，适合脚本判断）
sudo firewall-cmd --query-rich-rule='rule family="ipv4" source address="192.168.1.100" port protocol="tcp" port="3306" accept'

# 删除：把 add 换成 remove，其余字符串必须完全一致
sudo firewall-cmd --permanent --remove-rich-rule='rule family="ipv4" source address="192.168.1.100" port protocol="tcp" port="3306" accept'
sudo firewall-cmd --reload
```

**删除富规则是新手最容易翻车的操作**：引号内多一个空格、少一个 `family` 都删不掉，而且 firewalld 通常还回你个 `success`。稳妥做法是先 `--list-rich-rules` 把原规则完整复制出来，再把 `add` 改 `remove`。

### 6.4 区域内规则的匹配顺序

Red Hat 文档给出的 zone 内处理顺序是：

1. 先匹配该 zone 设置的**端口转发与地址伪装**规则
2. 再匹配该 zone 的**允许类**规则
3. 最后匹配该 zone 的**拒绝类**规则
4. log 与 audit 可以和上面三类同时生效，不影响判定
5. **富规则的优先级高于区域内其他规则**
6. 全部不匹配时，按 zone 的 target 处理（通常拒绝；trusted 例外）

社区笔记里更常见的简化表述是"富规则 > 端口/服务/源规则 > zone 默认策略"，日常按这个理解判断结果基本够用。

由此得到一条书写习惯：**越具体的规则越先写，白名单和黑名单不要互相打架**。如果既要"放行所有人"又要"拒绝某网段"，用富规则做拒绝，而不是去 zone 里改 target。

### 本节考点

- 富规则整条是单个字符串，结构为：rule + family + source + destination + 元素 + log/audit + 动作。
- icmp-block、masquerade、forward-port 三类元素内部已隐含动作，写 action 会报错。
- 写 source/destination 地址必须指定 family；纯源黑白名单规则不能带 destination。
- limit 时长单位 s/m/h/d，上限 2/d；log 默认级别 warning。
- 富规则优先级高于区域内其他规则；删除需与添加时字符串完全一致。

---

## 第七章 NAT 与 ICMP：把服务器当网关用

### 7.1 地址伪装 masquerade（SNAT）

作用：让没有公网 IP 的内网机器，借用本机的公网出口上网。属于源地址转换，external 区域默认就带这个特性。

```bash
# 开启
sudo firewall-cmd --add-masquerade --zone=external
sudo firewall-cmd --permanent --add-masquerade --zone=external

# 查询是否开启
sudo firewall-cmd --query-masquerade --zone=external

# 关闭
sudo firewall-cmd --permanent --remove-masquerade --zone=external
sudo firewall-cmd --reload
```

### 7.2 端口转发 forward-port（DNAT）

把打到本机的端口，转到本机另一个端口或另一台机器的端口。

```bash
# 本机 8000 → 本机 80
sudo firewall-cmd --permanent --zone=public --add-forward-port=port=8000:proto=tcp:toport=80

# 本机 1022 → 后端机器 10.1.1.11 的 22
sudo firewall-cmd --permanent --zone=public --add-forward-port=port=1022:proto=tcp:toport=22:toaddr=10.1.1.11

# 转发到别的机器（不指定 toport 则端口不变）
sudo firewall-cmd --permanent --zone=public --add-forward-port=port=80:proto=tcp:toaddr=192.168.1.100:toport=8080

# 删除（把 add 改 remove，参数一致）
sudo firewall-cmd --permanent --zone=public --remove-forward-port=port=8000:proto=tcp:toport=80
sudo firewall-cmd --reload
```

**两个前置条件缺一不可**，这也是转发配好了不通的头号原因：

1. 当前 zone 必须先开启 `masquerade`
2. 转发到**其他机器**时，需要打开内核 IP 转发

```bash
# 开启内核 IP 转发并持久化
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

参数格式记法：`port=源端口:proto=协议:toport=目标端口:toaddr=目标地址`，四个字段用冒号分隔，`toport` 和 `toaddr` 按需省略。

### 7.3 ICMP 控制：管住 ping

```bash
# 支持哪些 ICMP 类型
sudo firewall-cmd --get-icmptypes

# 屏蔽 ping（别人 ping 不通你）
sudo firewall-cmd --permanent --add-icmp-block=echo-request
sudo firewall-cmd --reload

# 放通 ping（撤销屏蔽）
sudo firewall-cmd --permanent --remove-icmp-block=echo-request
sudo firewall-cmd --reload

# 查看当前屏蔽了哪些
sudo firewall-cmd --list-icmp-blocks

# 反选开关：开启后，屏蔽列表变成"只放行列表"
sudo firewall-cmd --permanent --add-icmp-block-inversion
```

`icmp-block-inversion` 是个容易看走眼的字段。它设为 yes 时，语义反转成"列表之外的 ICMP 全部屏蔽"，此时看到 `icmp-blocks: echo-request` 加上 inversion 为 yes，实际效果是**只放通 ping**。排查 ping 不通时记得同时看这一行。

另外要理解：`icmp-block` 内部用的是 reject 动作，所以被屏蔽的 ping 是"对方收到拒绝响应"，而不是超时。

### 本节考点

- masquerade 是 SNAT 地址伪装，forward-port 是 DNAT 端口转发。
- forward-port 要求本 zone 已开 masquerade；跨机转发还需内核 `net.ipv4.ip_forward=1`。
- forward-port 的参数用冒号分隔：`port=:proto=:toport=:toaddr=`。
- icmp-block-inversion 会把黑名单语义翻成白名单，排查 ping 问题时必看。

---

## 第八章 配置文件、自定义服务与备份

### 8.1 两个配置目录的优先级

```
/usr/lib/firewalld/     # 软件包自带默认配置。不要改，包更新会被覆盖
/etc/firewalld/         # 管理员自定义配置，优先级高于上面
```

子目录结构两边对称：`zones/`（区域定义）、`services/`（服务定义）、`icmptypes/`（ICMP 类型）、`ipsets/`（IP 集合）、`helpers/`（连接跟踪辅助）。主配置 `/etc/firewalld/firewalld.conf` 存放后端选择、默认 zone、恐慌模式行为等全局开关。

```bash
ls /etc/firewalld/zones/            # 你改过的 zone 会在这里生成 xml
ls /usr/lib/firewalld/services/ | head
cat /usr/lib/firewalld/zones/trusted.xml
```

`trusted.xml` 的原文结构，很能说明 zone 的本质：

```xml
<?xml version="1.0" encoding="utf-8"?>
<zone target="ACCEPT">
  <short>Trusted</short>
  <description>All network connections are accepted.</description>
</zone>
```

`http.xml` 说明 service 的本质就是端口封装：

```xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>WWW (HTTP)</short>
  <description>HTTP Service</description>
  <port protocol="tcp" port="80"/>
</service>
```

### 8.2 自定义 service（推荐给非标端口用）

假设自研应用跑在 9000/tcp，与其在文档里记"开了 9000 端口"，不如注册成服务：

```bash
sudo tee /etc/firewalld/services/myapp.xml > /dev/null <<'EOF'
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>MyApp</short>
  <description>My custom application</description>
  <port protocol="tcp" port="9000"/>
</service>
EOF

sudo firewall-cmd --reload
sudo firewall-cmd --get-services | tr ' ' '\n' | grep myapp
sudo firewall-cmd --permanent --add-service=myapp
sudo firewall-cmd --reload
```

注意：**必须放在 `/etc/firewalld/services/`，不能放 `/usr/lib/firewalld/services/`**。

### 8.3 手改 xml 的正确流程

直接编辑配置文件是合法操作，但要清楚它的边界：

- 改 `/etc/firewalld/zones/public.xml` 这类永久文件，**改完必须 `firewall-cmd --reload`**，否则运行态毫无变化。
- XML 语法写错（少个尖括号、标签不闭合）会导致 firewalld 启动失败，远程管理时可能直接失联。
- 因此生产环境的建议顺序永远是：**先 firewall-cmd，改不动了再考虑编辑 xml**。图形化环境另有 `firewall-config` 可用。

### 8.4 备份与恢复

```bash
# 备份一：导出当前规则文本（可读，便于比对）
sudo firewall-cmd --zone=public --list-all > fw_backup_$(date +%F).txt

# 备份二：整目录打包（可恢复，含所有 zone/service）
sudo cp -r /etc/firewalld /etc/firewalld.bak_$(date +%F)

# 恢复
sudo rm -rf /etc/firewalld && sudo cp -r /etc/firewalld.bak_2026-10-04 /etc/firewalld
sudo systemctl restart firewalld
```

单文件级备份也常用：

```bash
sudo cp /etc/firewalld/zones/public.xml /etc/firewalld/zones/public.xml.bak
```

### 本节考点

- `/etc/firewalld` 覆盖 `/usr/lib/firewalld`；后者是包自带，改了会被更新覆盖。
- service XML 的本质是"端口协议的命名封装"。
- 手改 XML 后必须 reload；语法错误可导致 firewalld 起不来。
- 恢复配置首选整目录备份 + 重启服务。

---

## 第九章 实战案例集

这一章把前面的碎片能力串成可直接照抄的完整流程。所有案例假设为公网 Linux 服务器、默认 zone 为 public。

### 案例一：新服务器基线加固

需求：保住在手 SSH，开放 Web，并把 SSH 收紧到只允许办公 IP。

```bash
# 第 0 步（最重要）：先确认当前 SSH 用的什么端口，别把自己锁外面
ss -tlnp | grep sshd

# 第 1 步：确保防火墙起来且自启，且 SSH 已在放行列表
sudo systemctl enable --now firewalld
sudo firewall-cmd --list-services        # 确认里面有没有 ssh

# 第 2 步：开放 Web
sudo firewall-cmd --permanent --zone=public --add-service=http
sudo firewall-cmd --permanent --zone=public --add-service=https

# 第 3 步：SSH 只对管理 IP 开放
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="203.0.113.10" service name="ssh" accept'
sudo firewall-cmd --permanent --zone=public --remove-service=ssh

# 第 4 步：重载并验证
sudo firewall-cmd --reload
sudo firewall-cmd --list-all

# 第 5 步：保持当前会话不断开，另开一个新终端测试能否登录
```

**关键顺序**：先加白名单规则，再删 ssh 服务。反过来做，中间那次 reload 就会切断你的连接。而且验证必须在旧会话还活着的时候用新终端测，这是铁律。

### 案例二：MySQL 只放行应用网段

```bash
# 只允许应用服务器网段访问 3306
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="10.0.0.0/24" port port="3306" protocol="tcp" accept'

# 显式拒绝其他所有来源访问 3306
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" port port="3306" protocol="tcp" reject'
sudo firewall-cmd --reload
sudo firewall-cmd --list-rich-rules
```

这里要提醒一层：**防火墙不是数据库权限**。3306 不对外开放只是减少暴露面，MySQL 自身的账号 host 限制、TLS 等仍需配置。

### 案例三：内网机器通过网关共享上网

拓扑：网关机两张网卡，ens33 接外网（public），ens34 接内网 192.168.100.0/24（external）；内网机器只配网关为默认路由。

```bash
# 1. 开内核转发
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# 2. 把内网网卡放进 external 区域
sudo firewall-cmd --permanent --zone=external --add-interface=ens34

# 3. 在外网区域开地址伪装（SNAT）
sudo firewall-cmd --permanent --zone=public --add-masquerade

# 4. 需要把外网 1022 转到内网 Web 机 443
sudo firewall-cmd --permanent --zone=public --add-forward-port=port=1022:proto=tcp:toport=443:toaddr=192.168.100.20

# 5. 生效
sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --query-masquerade
```

第 4 步能成立的前提，正是第 3 步开了 masquerade 且第 1 步开了转发——这就是第七章那两个前置条件的由来。

### 案例四：应急封禁攻击 IP

```bash
# 单个 IP 立刻拉黑（运行态，不等 reload）
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="198.51.100.23" drop'

# 确认有效后再落盘
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="198.51.100.23" drop'

# 整个网段在攻击 → 交给 drop 区域处理
sudo firewall-cmd --permanent --zone=drop --add-source=198.51.100.0/24
sudo firewall-cmd --reload
```

如果攻击面已经失控，需要"一枪毙掉所有入站"，用恐慌模式：

```bash
sudo firewall-cmd --panic-on      # 丢弃所有流量，已建立连接也会断
sudo firewall-cmd --query-panic   # yes/no
sudo firewall-cmd --panic-off
```

**⚠️ 这是自断后路级别的操作**：panic-on 会把包括 SSH 在内的所有入站全部切断，远程管理时一执行立刻失联。它设计给的是机房带外/控制台可及的场景。

### 案例五：调试期规则固化

一种更省心的工作流：全程只操作运行态快速试错，试对了再一次性固化。

```bash
# 调试阶段：不加 --permanent，改了就生效，反正 reload 会清空
sudo firewall-cmd --add-port=8080/tcp
sudo firewall-cmd --add-service=http

# 测到业务正常后，一次性把运行态全部写入永久配置
sudo firewall-cmd --runtime-to-permanent

# 校验永久配置
sudo firewall-cmd --permanent --zone=public --list-all
```

`--runtime-to-permanent` 的语义是：保存当前运行配置，并用它**覆盖**永久配置。所以用之前要确认运行态是你想要的完整状态，别把不该留的临时规则一起固化了。

### 本节考点

- 收紧 SSH 访问的铁律：先加白名单再删服务，保持旧会话验证新会话。
- 应急封禁先用运行态规则验证，确认后 `--permanent` 落盘。
- `--runtime-to-permanent` 是运行态覆盖永久配置，不是追加。
- `--panic-on` 会切断包括 SSH 在内的所有入站，仅限可控接入场景。

---

## 第十章 常见故障排查

### 10.1 端口开了但外部仍不通：四层排查链路

这是面试和工作中出现频率最高的问题。按顺序查，不要跳：

```bash
# 第 1 层：服务真的在监听吗？监听地址对不对？
sudo ss -tlnp | grep :8080
#   监听 127.0.0.1:8080 → 外部永远进不来，应用要改成 0.0.0.0
#   完全没有监听 → 应用没起来，防火墙无罪

# 第 2 层：规则加对 zone 了吗？流量走的真是这个 zone 吗？
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
#   规则加到 public，但网卡在 home → 白加

# 第 3 层：永久规则 reload 了吗？
sudo firewall-cmd --permanent --zone=public --list-all   # 永久态里有
sudo firewall-cmd --zone=public --list-all               # 运行态里没有 → 忘了 reload
sudo firewall-cmd --reload

# 第 4 层：云上/上游设备放行吗？
#   云安全组、硬件防火墙、上游 ACL 都得一起放行，主机防火墙只是其中一环
```

再加两个横向检查：

- **SELinux 端口上下文**：把 Web 服务跑在非标准端口（如 8080）时，SELinux 可能拦截，这属于防火墙之外的独立机制，别把它误判成防火墙问题。
- **从本机验证**：`curl 127.0.0.1:8080` 通而外网通不了，说明应用正常、问题在网络路径或防火墙；两个都不通，先查应用。

### 10.2 规则重启后失效

根因几乎只有一个：只写了运行态。验证方法：

```bash
sudo firewall-cmd --zone=public --list-all                        # 运行态
sudo firewall-cmd --permanent --zone=public --list-all            # 永久态
```

两份输出不一致，差出来的那些就是重启后会丢的。补救：`sudo firewall-cmd --runtime-to-permanent`。

另外要知道 `--reload` 本身也会**丢弃运行态中未保存的临时修改**（永久配置成为新的运行配置）。所以"加了临时规则 → 执行了 reload → 规则消失"是正常行为，不是 bug。

### 10.3 把自己踢下线了怎么办

典型场景：远程 SSH 时执行了 `--complete-reload`、`--panic-on`，或者误删了 ssh 服务并 reload。

自救路径（按可及性排序）：

1. 云控制台 VNC / 带外管理口登录
2. 机房物理终端
3. 有另一台能进的机器，通过它跳板修改规则

预防措施永远比自救重要：

```bash
# 动 SSH 相关规则前，先挂一条带自动过期的保底放行
sudo firewall-cmd --add-service=ssh --timeout=30m

# 变更前备份
sudo cp -r /etc/firewalld /etc/firewalld.bak_$(date +%F)

# 保持当前会话存活，新开终端测试通过后再退出旧会话
```

### 10.4 与 iptables / ufw 混用的症状

firewalld 和 iptables service 都往内核写规则，同时启用会互相覆盖，表现通常是："我明明用 iptables 加了规则，`iptables -L` 也看得到，但流量行为跟配置不符"或者"重启后手工规则全没了"。

排查与规范：

```bash
# 看谁在跑
systemctl is-active firewalld
systemctl is-active iptables
systemctl is-active nftables
```

规矩只有一个：**一台机器只启用一种防火墙管理方式**。如果确实要用 iptables 接管，先 `systemctl disable --now firewalld`。

补充一个中间态：firewalld 运行期间直接用 `iptables` 命令临时插规则，当场是生效的；但 firewalld 重启或 reload 时，它的永久配置会覆盖掉你手工写入的内容。这种临时手法只适合抓包排障，绝不能当长期配置。

### 10.5 reload 到底断不断连接

| 命令 | 作用 | 对已建立连接 |
| --- | --- | --- |
| `firewall-cmd --reload` | 用永久配置重建运行配置，丢弃未保存的临时修改 | **保留状态，不断连**，日常推荐 |
| `firewall-cmd --complete-reload` | 完全重载，连 netfilter 内核模块一起重载 | **状态丢失，连接大概率中断**，仅在规则状态异常时急救 |
| `systemctl restart firewalld` | 重启守护进程 | 不推荐在远程会话中使用 |

### 本节考点（排障口诀）

**一听二看三重载，四查安全组**：听（ss 看监听地址）→ 看（zone 选对没）→ 重载（permanent 生效没）→ 查（云安全组/上游 ACL/SELinux）。

---

## 第十一章 速记卡：背诵区

### 11.1 命令速查表

**状态与查看**

```bash
firewall-cmd --state                       # running / not running
systemctl status firewalld
firewall-cmd --get-zones                   # 所有可用 zone
firewall-cmd --get-default-zone            # 默认 zone
firewall-cmd --set-default-zone=public     # 改默认 zone（立即生效）
firewall-cmd --get-active-zones            # 活动 zone 及其绑定
firewall-cmd --list-all                    # ★ 最常用
firewall-cmd --list-all --zone=home
firewall-cmd --list-all-zones
firewall-cmd --get-zone-of-interface=ens33
```

**服务与端口**

```bash
firewall-cmd --get-services                        # 支持的预定义服务
firewall-cmd --list-services
firewall-cmd --permanent --add-service=http        # ★
firewall-cmd --permanent --remove-service=http
firewall-cmd --query-service=http

firewall-cmd --list-ports
firewall-cmd --permanent --add-port=8080/tcp        # ★
firewall-cmd --permanent --add-port=2000-2100/tcp
firewall-cmd --permanent --remove-port=8080/tcp
firewall-cmd --query-port=8080/tcp
```

**区域绑定**

```bash
firewall-cmd --permanent --zone=home --change-interface=ens33
firewall-cmd --permanent --zone=home --add-source=10.1.1.0/24
firewall-cmd --permanent --remove-source=10.1.1.0/24 --zone=home
firewall-cmd --permanent --new-zone=myweb
firewall-cmd --permanent --delete-zone=myweb
firewall-cmd --permanent --zone=myweb --set-target=REJECT
```

**富规则**

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.100" port port="3306" protocol="tcp" accept'   # ★
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="1.2.3.4" drop'                                            # 拉黑
firewall-cmd --permanent --remove-rich-rule='<原规则原文>'
firewall-cmd --list-rich-rules
```

**NAT 与 ICMP**

```bash
firewall-cmd --permanent --add-masquerade
firewall-cmd --query-masquerade
firewall-cmd --permanent --add-forward-port=port=1022:proto=tcp:toport=22:toaddr=10.1.1.11
firewall-cmd --permanent --add-icmp-block=echo-request
firewall-cmd --get-icmptypes
```

**生效与应急**

```bash
firewall-cmd --reload                        # ★ 不中断已有连接
firewall-cmd --complete-reload               # 中断连接，慎用
firewall-cmd --runtime-to-permanent          # 运行态固化
firewall-cmd --add-service=http --timeout=30m  # 到期自动撤销
firewall-cmd --panic-on / --panic-off / --query-panic
```

### 11.2 九个 zone 一句话记忆

按信任度从低到高串成一句：**丢（drop）盘（block）众（public）外（external）隔（dmz）班（work）家（home）内（internal）信（trusted）**。

- drop：丢包不回应
- block：拒绝并回 ICMP
- public：默认，白名单
- external：只放 ssh，自带伪装
- dmz：对外区，只放 ssh
- work：办公，白名单
- home：家庭，多放 samba/mdns
- internal：等同 home
- trusted：全放行

### 11.3 高频问答八条

**Q1：firewalld 和 iptables 什么区别？**
A：iptables 是静态防火墙管理工具，改规则需全量重载，可能中断现有连接，用表/链模型；firewalld 是动态防火墙，增量更新规则不中断连接，用 zone 模型，是 RHEL/CentOS 7+ 的默认工具。两者自身都不过滤数据包，真正干活的是内核 netfilter。

**Q2：zone 的作用是什么？**
A：对不同网络环境预定义一套策略。把网卡或源地址绑定到 zone，即可对该部分流量做分类控制；一个连接只能属于一个 zone，一个 zone 可绑多个连接。

**Q3：`--permanent` 为什么加了不生效？**
A：`--permanent` 只把规则写入磁盘配置，不影响运行态，必须 `firewall-cmd --reload` 才会加载进运行配置。最佳实践是 permanent 加一次、不带参数再加一次。

**Q4：`--reload` 和 `--complete-reload` 区别？**
A：reload 用永久配置重建运行配置，保留连接状态信息，不断线；complete-reload 连 netfilter 内核模块一起重载，状态信息丢失、现有连接会断，只在防火墙状态异常时急救使用。

**Q5：drop 和 reject 有什么区别？**
A：reject 主动拒绝并回 ICMP 错误，对方立刻看到 Connection refused；drop 静默丢弃不回应，对方只能等到 Connection timed out。drop 更隐蔽（扫描者难判断主机存在），reject 反馈更明确。

**Q6：masquerade 是什么？**
A：地址伪装，即 SNAT。把私网地址映射并隐藏在公网 IP 之后，实现内网主机共享一个公网出口上网，external 区域默认启用。

**Q7：富规则能做什么、优先级如何？**
A：富规则把源/目的地址、MAC、IP 集合、端口、协议、日志、限速和动作组合成复合条件，可实现 IP 白名单、黑名单、限速、带日志拒绝。区域内富规则优先级高于普通服务/端口规则。

**Q8：`/etc/firewalld` 和 `/usr/lib/firewalld` 有什么区别？**
A：前者是管理员自定义配置，优先级高；后者是软件包自带的默认模板，会被更新覆盖，不要直接改。自定义 zone/service 一律落在 `/etc/firewalld` 下。

---

## 结尾：接下来的学习路线

**firewalld 学到什么程度算够用？** 能独立完成第九章那五个案例、能在 10.1 的四层链路上讲清楚排查逻辑，日常运维就够用了。

再往上走，三个方向：

1. **读 man page**。这是最省钱的进阶方式，四篇按优先级看：`man firewall-cmd`、`man firewalld.richlanguage`（富规则权威定义）、`man firewalld.zones`、`man firewalld.conf`。忘了语法时先 `firewall-cmd --help` 看选项分组，再用 `--list-all` 核对当前状态。
2. **补 nftables**。理解 firewalld 生成的规则在内核里长什么样，`nft list ruleset` 是照妖镜。当 firewalld 的抽象表达不了你的需求时（复杂连接跟踪、动态地址集高性能匹配），需要直接写 nft。
3. **补同层安全组件**。主机安全不止防火墙：SELinux 管的是"进程能访问什么"，firewalld 管的是"包能不能进来"，fail2ban 负责"根据日志动态拉黑"。三者常配套出现，也是面试里连问的三个点。

最后一句实操忠告：**每次改防火墙之前，先备份 `/etc/firewalld`，永远在旧会话存活时用新会话验证。** 这两条习惯能帮你消掉 90% 的失联事故。

---

## 参考文献

1. Oracle Help Center. Configuring firewalld Zones. Oracle Linux 8 Firewall Guide. https://docs.oracle.com/en/operating-systems/oracle-linux/8/firewall/firewall-ConfiguringfirewalldZones.html
2. Red Hat Documentation. 1.7 使用 firewalld 区 / 配置防火墙和数据包过滤器. Red Hat Enterprise Linux 9. https://docs.redhat.com/zh-cn/documentation/red_hat_enterprise_linux/9/html/configuring_firewalls_and_packet_filters/working-with-firewalld-zones_using-and-configuring-firewalld
3. Ubuntu Manpage. firewalld.zones - firewalld zones (man5). https://manpages.ubuntu.com/manpages/noble/man5/firewalld.zones.5.html
4. Ubuntu Manpage. firewalld.richlanguage - Rich Language Documentation (man5). https://manpages.ubuntu.com/manpages/xenial/en/man5/firewalld.richlanguage.5.html
5. Ubuntu Manpage. firewall-cmd - firewalld command line client (man1). https://manpages.ubuntu.com/manpages/resolute/en/man1/firewall-cmd.1.html
6. Arch Linux Man Pages. firewalld.conf(5) FirewallBackend. https://man.archlinux.org/man/firewalld.conf.5.en
7. firewalld.org. nftables backend. https://firewalld.org/2018/07/nftables-backend
8. Fedora Docs. Security — firewalld now uses nftables as its default backend. https://docs.fedoraproject.org/sq/fedora/f32/release-notes/sysadmin/Security/
9. Red Hat Documentation. 5.15 Configuring Complex Firewall Rules with the Rich Language Syntax. RHEL 7 Security Guide. https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/security_guide/configuring_complex_firewall_rules_with_the_rich-language_syntax
10. firewalld.org. Documentation — Manual Pages — firewalld.richlanguage. https://firewalld.org/documentation/man-pages/firewalld.richlanguage.html
11. ArchWiki. Firewalld — Rich rules. https://wiki.archlinux.org/title/Firewalld
12. 阿里云开发者社区. firewalld 详细介绍配置. https://developer.aliyun.com/article/1581098
13. Linux Command Library. firewall-cmd man. https://linuxcommandlibrary.com/man/firewall-cmd
14. FreeBuf. 新手勇闯网络安全 Linux 安全基础（三）. https://www.freebuf.com/news/500790.html

说明：第六章 6.4 中"富规则优先级高于区域内其他规则"、第十章中规则匹配顺序表述，参考自 Red Hat 系列文档的社区整理稿，与官方 manpage 逐字表述存在差异，已按通行口径归纳；实际匹配行为建议以实验验证为准。

---

内容由 AI 生成
