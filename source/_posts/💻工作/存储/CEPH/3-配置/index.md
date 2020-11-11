---
title: CEPH 集群操作入门--配置
toc: true
tags: CEPH
categories:
  - "\U0001F4BB 工作"
  - 存储
  - CEPH
  - 3-配置
date: 2020-05-26 12:27:56
---

# 概述

Ceph 存储集群是所有 Ceph 部署的基础。 基于 RADOS，Ceph 存储集群由两种类型的守护进程组成：Ceph OSD 守护进程（OSD）将数据作为对象存储在存储节点上; Ceph Monitor（MON）维护集群映射的主副本。 Ceph 存储集群可能包含数千个存储节点。 最小系统将至少有一个 Ceph Monitor 和两个 Ceph OSD 守护进程用于数据复制。

Ceph 文件系统，Ceph 对象存储和 Ceph 块设备从 Ceph 存储集群读取数据并将数据写入 Ceph 存储集群。

配置和部署

Ceph 存储集群有一些必需的设置，但大多数配置设置都有默认值。 典型部署使用部署工具来定义集群并引导监视器。 有关 ceph-deploy 的详细信息，请参阅部署。

# 配置

## 存储设备

### 概述

有两个 Ceph 守护进程将数据存储在磁盘上：

Ceph OSD（或 Object Storage Daemons）是 Ceph 中存储大部分数据的地方。 一般而言，每个 OSD 都由单个存储设备支持，如传统硬盘（HDD）或固态硬盘（SSD）。 OSD 还可以由多种设备组合支持，例如用于大多数数据的 HDD 和用于某些元数据的 SSD（或 SSD 的分区）。 集群中 OSD 的数量通常取决于将存储多少数据，每个存储设备的大小以及冗余的级别和类型（复制或纠删码）。

Ceph Monitor 守护程序管理集群状态，如集群成员和身份验证信息。 对于较小的集群，只需要几 GB，但对于较大的集群，监控数据库可以达到几十甚至几百 GB。

### 准备硬盘

Ceph 注重数据安全，就是说， Ceph 客户端收到数据已写入存储器的通知时，数据确实已写入硬盘。使用较老的内核（版本小于 2.6.33 ）时，如果日志在原始硬盘上，就要禁用写缓存；较新的内核没问题。

用 hdparm 禁用硬盘的写缓冲功能。

sudo hdparm -W 0 /dev/hda 0 

在生产环境，我们建议操作系统和 OSD 数据分别放到不同的硬盘。如果必须把数据和系统放在同一硬盘里，最好给数据分配一个单独的分区。

### OSD 后端

OSD 可以通过两种方式管理它们存储的数据。从 Luminous 12.2.z 版本开始，新的默认（和推荐）后端是 BlueStore。在 Luminous 之前，默认（也是唯一的选项）是 FileStore。

#### BLUESTORE

BlueStore 是一种专用存储后端，专为管理 Ceph OSD 工作负载的磁盘数据而设计。在过去十年中，使用 FileStore 支持和管理 OSD 的经验激发了它的动力。关键的 BlueStore 功能包括：

* 直接管理存储设备。 BlueStore 使用原始块设备或分区。这避免了可能限制性能或增加复杂性的任何中间抽象层（例如 XFS 等本地文件系统）。
* 使用 RocksDB 进行元数据管理。我们嵌入了 RocksDB 的键/值数据库，以便管理内部元数据，例如从对象名到磁盘上的块位置的映射。
* 完整数据和元数据校验和。默认情况下，写入 BlueStore 的所有数据和元数据都受一个或多个校验和的保护。在未经验证的情况下，不会从磁盘读取数据或元数据或将其返回给用户。
* 内联压缩。在写入磁盘之前，可以选择压缩写入的数据。
* 多设备元数据分层。 BlueStore 允许将其内部日志（预写日志）写入单独的高速设备（如 SSD，NVMe 或 NVDIMM）以提高性能。如果有大量更快的存储空间可用，内部元数据也可以存储在速度更快的设备上。
* 高效的写时复制。 RBD 和 CephFS 快照依赖于在 BlueStore 中高效实现的写时复制克隆机制。这能为常规快照和 erasure coded pools（依赖于克隆以实现有效的两阶段提交）实现更高效的 IO。

#### FILESTORE

FileStore 是在 Ceph 中存储对象的传统方法。它依赖于标准文件系统（通常为 XFS）与键/值数据库（传统上为 LevelDB，现为 RocksDB）的组合，用于某些元数据。FileStore 经过了充分测试并广泛用于生产，但由于其整体设计和依赖传统文件系统来存储对象数据而遭受许多性能缺陷。虽然 FileStore 通常能够在大多数 POSIX 兼容的文件系统（包括 btrfs 和 ext4）上运行，但我们只建议使用 XFS。 btrfs 和 ext4 都有已知的漏洞和缺陷，它们的使用可能会导致数据丢失。默认情况下，所有 Ceph 配置工具都将使用 XFS。

OSD 守护进程有赖于底层文件系统的扩展属性（ XATTR ）存储各种内部对象状态和元数据。底层文件系统必须能为 XATTR 提供足够容量， btrfs 没有限制随文件的 xattr 元数据总量； xfs 的限制相对大（ 64KB ），多数部署都不会有瓶颈； ext4 的则太小而不可用。使用 ext4 文件系统时，一定要把下面的配置放于 ceph.conf 配置文件的 \[osd\] 段下；用 btrfs 和 xfs 时可以选填。

filestore xattr use omap = true 

## 配置 Ceph

### 概述

你启动 Ceph 服务时，初始化进程会把一系列守护进程放到后台运行。 Ceph 存储集群运行两种守护进程：

* Ceph 监视器 （ ceph-mon ）
* Ceph OSD 守护进程 （ ceph-osd ）

要支持 Ceph 文件系统功能，它还需要运行至少一个 Ceph 元数据服务器（ ceph-mds ）；支持 Ceph 对象存储功能的集群还要运行网关守护进程（ radosgw ）。为方便起见，各类守护进程都有一系列默认值（很多由 ceph/src/common/config\_opts.h 配置），你可以用 Ceph 配置文件覆盖这些默认值。

### 选项名称

所有 Ceph 配置选项都有一个唯一的名称，由用小写字符组成的单词组成，并用下划线（\_）字符连接。

在命令行中指定选项名称时，可以使用下划线（\_）或短划线（ \- ）字符进行互换（例如， \- mon-host 相当于--mon\_host）。

当选项名称出现在配置文件中时，也可以使用空格代替下划线或短划线。

### 配置来源

每个 Ceph 守护程序，进程和库将从以下列出的几个源中提取其配置。 当两者都存在时，列表中稍后的源将覆盖列表中较早的源。

* 编译后的默认值
* 监视器集群的集中配置数据库
* 存储在本地主机上的配置文件
* 环境变量
* 命令行参数
* 由管理员设置的运行时覆盖

Ceph 进程在启动时所做的第一件事就是解析通过命令行，环境和本地配置文件提供的配置选项。 然后连接监视器集群，检测配置是否可用。 一旦完成检测，如果配置可用，守护程序或进程启动就会继续。

BOOTSTRAP 选项  
由于某些配置选项会影响进程联系监视器，认证和恢复集群存储配置，因此可能需要将它们存储在本地节点上并设置在本地配置文件中。这些选项包括：

* mon\_host，集群的监视器列表
* mon\_dns\_serv\_name（默认值：ceph-mon），用于通过 DNS 检查以识别集群监视器的 DNS SRV 记录的名称
* mon\_data，osd\_data，mds\_data，mgr\_data 以及类似的选项用来定义守护程序存储其数据的本地目录
* keying，ketfile，and/or key，可用于指定用于向监视器进行身份验证的身份验证凭据。请注意，在大多数情况下，默认密钥环位置位于上面指定的数据目录中。

在绝大多数情况下，这些的默认值是合适的，但 mon\_host 选项除外，它标识了集群监视器的地址。当 DNS 用于识别监视器时，可以完全避免本地 ceph 配置文件。任何进程可以通过选项--no-mon-config 以跳过从集群监视器检索配置的步骤。 这在配置完全通过配置文件管理或监视器集群当前已关闭但需要执行某些维护活动的情况下非常有用。

### 配置

任何给定的进程或守护程序对每个配置选项都有一个值。但是，选项的值可能会因不同的守护程序类型而异，甚至是相同类型的守护程序也会有不同的配置。Ceph 选项存储在监视器配置数据库或本地配置文件中被分组为多个部分，以指示它们应用于哪些守护程序或客户端。

这些部分包括：

* \[global\]

　　描述：全局下的设置会影响 Ceph 存储集群中的所有 daemon 和 client。  
　　示例：

log\_file = /var/log/ceph/\$cluster-\$type.\$id.log

* \[mon\]

　　描述：mon 下的设置会影响 Ceph 存储集群中的所有 ceph-mon 守护进程，并覆盖全局中的相同设置。  
　　示例：mon\_cluster\_log\_to\_syslog = true

* \[mgr\]

　　说明：mgr 部分中的设置会影响 Ceph 存储群集中的所有 ceph-mgr 守护程序，并覆盖全局中的相同设置。  
　　示例：mgr\_stats\_period = 10

* \[osd\]

　　描述：osd 下的设置会影响 Ceph 存储集群中的所有 ceph-osd 守护进程，并覆盖全局中的相同设置。  
　　示例：osd\_op\_queue = wpq

* \[mds\]

　　描述：mds 部分中的设置会影响 Ceph 存储集群中的所有 ceph-mds 守护程序，并覆盖全局中的相同设置。  
　　示例：mds\_cache\_size = 10G

* \[client\]

　　描述：客户端下的设置会影响所有 Ceph 客户端（例如，挂载的 Ceph 文件系统，挂载的 Ceph 块设备等）以及 Rados Gateway（RGW）守护程序。  
　　示例：objecter\_inflight\_ops = 512

  
还可以指定单个守护程序或客户端名称。例如，mon.foo，osd.123 和 client.smith 都是有效的节名。

任何给定的守护程序都将从全局部分，守护程序或客户端类型部分以及指定名称的部分中提取其设置。指定名称部分中的设置优先，例如，如果在同一源（即，在同一配置文件中）的 global，mon 和 mon.foo 中指定了相同的选项，则将使用 mon.foo 值。

请注意，本地配置文件中的值始终优先于监视器配置数据库中的值，而不管它们出现在哪个部分。

### 元变量

元变量可以显着简化 Ceph 存储集群配置。 当在配置值中设置元变量时，Ceph 会在使用配置值时将元变量扩展为具体值。 Ceph 元变量类似于 Bash shell 中的变量扩展。Ceph 支持以下元变量：

* \$cluster

　　描述：扩展为 Ceph 存储群集名称。 在同一硬件上运行多个 Ceph 存储集群时很有用。  
　　例如：

/etc/ceph/\$cluster.keyring

　　默认值：CEPH

* \$type

　　描述：扩展为守护进程或进程类型（例如，mds，osd 或 mon）  
　　例如：

/var/lib/ceph/\$type

* \$id

　　描述：扩展为守护程序或客户端标识符。 对于 osd.0，这将是 0; 对于 mds.a，它将是 a。  
　　例如：

/var/lib/ceph/\$type/\$cluster-\$id

* \$host

　　描述：扩展为运行进程的主机名。

* \$name

　　描述：扩展为

\$type.\$id

　　例如：

/var/run/ceph/\$cluster-\$name.asok

* \$ pid

　　描述：扩展为守护进程 pid。  
　　例如：

/var/run/ceph/\$cluster-\$name-\$pid.asok

### 配置文件

默认的 Ceph 配置文件位置相继排列如下：

1.  \$CEPH\_CONF（CEPH\_CONF 环境变量所指示的路径）;
2.  \-c path / path（-c 命令行参数）;
3.  /etc/ceph/ceph.conf
4.  〜/.ceph/配置
5.  ./ceph.conf（就是当前所在的工作路径。
6.  仅限 FreeBSD 系统， /usr/local/etc/ceph/\$cluster.conf

Ceph 配置文件使用 ini 样式语法。 您可以通过前面的注释添加注释，使用'＃'或';'。 

### 监控配置数据库

监视器集群管理整个集群可以使用的配置选项数据库，从而为整个系统提供简化的中央配置管理。绝大多数配置选项可以并且应该存储在此处，以便于管理和透明。少数设置可能仍需要存储在本地配置文件中，因为它们会影响连接到监视器，验证和获取配置信息的能力。在大多数情况下，这仅限于 mon\_host 选项，尽管通过使用 DNS SRV 记录也可以避免这种情况。

监视器存储的配置选项有全局部分，守护程序类型部分或特定守护程序部分中，就像配置文件中的选项一样。此外，选项还可以具有与其关联的掩码，以进一步限制该选项适用于哪些守护进程或客户端。掩码有两种形式：

* 类型:location：其中 type 是 CRUSH 属性，如 rack 或 host，location 是该属性的值。例如，host：foo 将选项限制为在特定主机上运行的守护程序或客户端。
* class:device-class ：其中 device-class 是 CRUSH 设备类的名称（例如，hdd 或 ssd）。例如，class：ssd 会将选项仅限制为 SSD 支持的 OSD。 （此掩码对非 OSD 守护进程或客户端没有影响。）

设置配置选项时，可以是由斜杠（/）字符分隔的服务名称，掩码或两者的组合。例如，osd / rack：foo 意味着 foo 机架中的所有 OSD 守护进程。查看配置选项时，服务名称和掩码通常分为单独的字段或列，以便于阅读。

命令

以下 CLI 命令用于配置集群：

* ceph config dump

　　将转储集群的整个配置数据库。

* ceph config get \<who>

　　将转储特定守护程序或客户端（例如，mds.a）的配置，存储在监视器的配置数据库中。

* ceph config set \<who> \<option> \<value>

　　将在监视器的配置数据库中设置配置选项。

* ceph config show \<who>

　　将显示正在运行的守护程序的报告运行配置。如果还有正在使用的本地配置文件或者在命令行或运行时覆盖了选项，则这些设置可能与监视器存储的设置不同。选项值的来源将作为输出的一部分进行报告。

* ceph config assimilate-conf -i \<input file> -o \<output file>

　　将从输入文件中提取配置文件，并将任何有效选项移动到监视器的配置数据库中。任何无法识别，无效或无法由 monitor 控制的设置都将以存储在输出文件中的缩写配置文件的形式返回。此命令对于从旧配置文件转换为基于监视器的集中式配置非常有用。

### help

您可以通过以下方式获得特定选项的帮助：

ceph config help \<option>

请注意，这将使用已经编译到正在运行的监视器中的配置架构。 如果您有一个混合版本的集群（例如，在升级期间），您可能还想从特定的运行守护程序中查询选项模式：

ceph daemon \<name> config help \[option\]

例如

\[root\@node1 \~\]# ceph config help log\_file
log\_file \- path to log file \(std::string, basic\)
  Default \(non\-daemon\): 
  Default \(daemon\): /var/log/ceph/\$cluster-\$name.log
  Can update at runtime: false See also: \[log\_to\_stderr,err\_to\_stderr,log\_to\_syslog,err\_to\_syslog\]

或

\[root\@node1 \~\]# ceph config help log\_file -f json-pretty
\{ "name": "log\_file", "type": "std::string", "level": "basic", "desc": "path to log file", "long\_desc": "", "default": "", "daemon\_default": "/var/log/ceph/\$cluster-\$name.log", "tags": \[\], "services": \[\], "see\_also": \[ "log\_to\_stderr", "err\_to\_stderr", "log\_to\_syslog", "err\_to\_syslog" \], "enum\_values": \[\], "min": "", "max": "", "can\_update\_at\_runtime": false \}

 level 属性可以是 basic，advanced 或 dev 中的任何一个。 开发选项供开发人员使用，通常用于测试目的，不建议操作员使用。

### 运行时修改

在大多数情况下，Ceph 允许您在运行时更改守护程序的配置。 此功能对于增加/减少日志记录输出，启用/禁用调试设置，甚至用于运行时优化非常有用。

一般来说，配置选项可以通过 ceph config set 命令以通常的方式更新。 例如，在特定 OSD 上启用调试日志级别：

ceph config set osd.123 debug\_ms 20

请注意，如果在本地配置文件中也自定义了相同的选项，则将忽略监视器设置（其优先级低于本地配置文件）。

覆盖参数值

您还可以使用 Ceph CLI 上的 tell 或 daemon 接口临时设置选项。 这些覆盖值是短暂的，因为它们只影响正在运行的进程，并且在守护进程或进程重新启动时被丢弃/丢失。

覆盖值可以通过两种方式设置：

1.从任何主机，我们都可以通过网络向守护进程发送消息：

ceph tell osd.123 config set debug\_osd 20

tell 命令还可以接受守护程序标识符的通配符。 例如，要调整所有 OSD 守护进程的调试级别：

ceph tell osd.\* config set debug\_osd 20

2.从进程运行的主机开始，我们可以通过/ var / run / ceph 中的套接字直接连接到进程：

ceph daemon \<name> config set \<option> \<value>

例如

ceph daemon osd.4 config set debug\_osd 20

请注意，在 ceph config show 命令输出中，这些临时值将显示为覆盖源。

### 查看运行参数值

您可以使用 ceph config show 命令查看为正在运行的守护程序设置的参数。 例如：

ceph config show osd.0

将显示该守护进程的（非默认）选项。 您还可以通过以下方式查看特定选项：

ceph config show osd.0 debug\_osd

或查看所有选项（即使是那些具有默认值的选项）：

ceph config show-with-defaults osd.0

您还可以通过管理套接字从本地主机连接到该守护程序来观察正在运行的守护程序的设置。 例如：

ceph daemon osd.0 config show

将转储所有当前设置：

ceph daemon osd.0 config diff

将仅显示非默认设置（以及值来自的位置：配置文件，监视器，覆盖等），以及：

ceph daemon osd.0 config get debug\_osd

将报告单个选项的值。

## 常用设置

### 概述

单个 Ceph 节点可以运行多个守护进程。 例如，具有多个驱动器的单个节点可以为每个驱动器运行一个 ceph-osd。 理想情况下，某些节点只运行特定的 daemon。 例如，一些节点可以运行 ceph-osd 守护进程，其他节点可以运行 ceph-mds 守护进程，还有其他节点可以运行 ceph-mon 守护进程。

每个节点都有一个由主机设置标识的名称。 监视器还指定由 addr 设置标识的网络地址和端口（即域名或 IP 地址）。 基本配置文件通常仅为每个监视器守护程序实例指定最小设置。 例如：

\[global\]
mon\_initial\_members \= ceph1
mon\_host \= 10.0.0.1

主机设置是节点的短名称（即，不是 fqdn）。 它也不是 IP 地址。 在命令行中输入 hostname \-s 以检索节点的名称。 除非您手动部署 Ceph，否则请勿将主机设置用于初始监视器以外的任何其他设置。 在使用 chef 或 ceph-deploy 等部署工具时，您不能在各个守护程序下指定主机，这些工具将在集群映射中为您输入适当的值。

### 网络

有关配置与 Ceph 一起使用的网络的详细讨论，请参阅网络配置参考。

### 监控

Ceph 生产集群通常使用至少 3 个 Ceph Monitor 守护程序进行部署，以确保监视器实例崩溃时的高可用性。 至少三（3）个监视器确保 Paxos 算法可以确定哪个版本的 Ceph Cluster Map 是法定人数中大多数 Ceph 监视器的最新版本。

注意您可以使用单个监视器部署 Ceph，但如果实例失败，则缺少其他监视器可能会中断数据服务可用性。

Ceph 监视器通常侦听端口 6789.例如：

\[mon.a\]
host \= hostName
mon addr \= 150.140.130.120:6789

默认情况下，Ceph 希望您将以下列路径存储监视器的数据：

/var/lib/ceph/mon/\$cluster-\$id

您或部署工具（例如，ceph-deploy）必须创建相应的目录。 在完全表达元变量和名为“ceph”的集群的情况下，上述目录将评估为：

/var/lib/ceph/mon/ceph-a

有关其他详细信息，请参阅“监视器配置参考”。

### 认证

Bobtail 版本的新功能：0.56

对于 Bobtail（v 0.56）及更高版本，您应该在 Ceph 配置文件的\[global\]部分中明确启用或禁用身份验证。

auth cluster required = cephx
auth service required \= cephx
auth client required \= cephx

此外，您应该启用 message signing（消息签名）。升级时，我们建议先明确禁用身份验证，然后再执行升级。 升级完成后，重新启用身份验证。 有关详细信息，请参阅 Cephx 配置参考。

### OSD

Ceph 生产集群通常部署 Ceph OSD 守护进程，一般一个 OSD 守护进程运行在一个存储驱动器上。 典型部署指定 jorunal 大小。 例如：

\[osd\]
osd journal size \= 10000 \[osd.0\]
host \= \{hostname\} #manual deployments only.

默认情况下，Ceph 希望您使用以下路径存储 Ceph OSD 守护程序的数据：

/var/lib/ceph/osd/\$cluster-\$id

您或部署工具（例如，ceph-deploy）必须创建相应的目录。 在完全表达元变量和名为“ceph”的集群的情况下，上述目录将评估为：

/var/lib/ceph/osd/ceph-0

您可以使用 osd 数据设置覆盖此路径。 我们不建议更改默认位置。 在 OSD 主机上创建默认目录。

ssh \{osd-host\} sudo mkdir /var/lib/ceph/osd/ceph-\{osd-number\}

osd 数据路径理想地导致具有硬盘的安装点，该硬盘与存储和运行操作系统和守护进程的硬盘分开。 如果 OSD 用于操作系统磁盘以外的磁盘，请准备与 Ceph 一起使用，并将其安装到刚刚创建的目录中：

ssh \{new-osd-host\} sudo mkfs -t \{fstype\} /dev/\{disk\} sudo mount -o user\_xattr /dev/\{hdd\} /var/lib/ceph/osd/ceph-\{osd-number\}

我们建议在运行 mkfs 时使用 xfs 文件系统。（不建议使用 btrfs 和 ext4，不再测试。）有关其他配置详细信息，请参阅 OSD 配置参考。

### 心跳

在运行时操作期间，Ceph OSD 守护进程检查其他 Ceph OSD 守护进程并将其发现报告给 Ceph Monitor。 您不必提供任何设置。 但是，如果您遇到网络延迟问题，则可能希望修改设置。

有关其他详细信息，请参阅配置 Monitor / OSD Interaction。

### 日志/调试

有时您可能会遇到 Ceph 需要修改日志记录输出和使用 Ceph 调试的问题。 有关日志轮换的详细信息，请参阅调试和日志记录。

### ceph.conf 示例

\[global\]
fsid \= \{cluster-id\}
mon initial members \= \{hostname\}\[, \{hostname\}\]
mon host \= \{ip-address\}\[, \{ip-address\}\]

#All clusters have a front\-side public network.
#If you have two NICs, you can configure a back side cluster 
#network for OSD object replication, heart beats, backfilling,
#recovery, etc.
public network \= \{network\}\[, \{network\}\]
#cluster network \= \{network\}\[, \{network\}\] 

#Clusters require authentication by default.
auth cluster required \= cephx
auth service required \= cephx
auth client required \= cephx

#Choose reasonable numbers for your journals, number of replicas
#and placement groups.
osd journal size \= \{n\}
osd pool default size \= \{n\}  # Write an object n times.
osd pool default min size \= \{n\} # Allow writing n copy in a degraded state.
osd pool default pg num \= \{n\}
osd pool default pgp num \= \{n\}

#Choose a reasonable crush leaf type.
#0 for a 1\-node cluster.
#1 for a multi node cluster in a single rack
#2 for a multi node, multi chassis cluster with multiple hosts in a chassis
#3 for a multi node cluster with hosts across racks, etc.
osd crush chooseleaf type \= \{n\}

### 运行多集群

使用 Ceph，您可以在同一硬件上运行多个 Ceph 存储集群。 与在具有不同 CRUSH 规则的同一群集上使用不同池相比，运行多个群集可提供更高级别的隔离。 单独的群集将具有单独的监视器，OSD 和元数据服务器进程。 使用默认设置运行 Ceph 时，默认群集名称为 ceph，这意味着您将在/ etc / ceph 默认目录中保存 Ceph 配置文件，文件名为 ceph.conf。

有关详细信息，请参阅 ceph-deploy new。 .. \_ceph-deploy new：../ ceph-deploy-new

运行多个群集时，必须为群集命名并使用群集名称保存 Ceph 配置文件。 例如，名为 openstack 的集群将在/ etc / ceph 缺省目录中具有文件名为 openstack.conf 的 Ceph 配置文件。

群集名称必须仅包含字母 a-z 和数字 0-9。

单独的集群意味着单独的数据磁盘和日志，它们不在集群之间共享。 cluster 元变量代表集群名称（即前面例子中的 openstack）。 各种设置使用 cluster 元变量，包括：

keyring  
admin socket  
log file  
pid file  
mon data  
mon cluster log file  
osd data  
osd journal  
mds data  
rgw data

创建默认目录或文件时，应在路径中的适当位置使用群集名称。 例如：

sudo mkdir /var/lib/ceph/osd/openstack-0
sudo mkdir /var/lib/ceph/mon/openstack-a

在同一主机上运行监视器时，应使用不同的端口。 默认情况下，监视器使用端口 6789.如果已使用端口 6789 使用监视器，请为其他群集使用不同的端口。

ceph -c \{cluster-name\}.conf health
ceph \-c openstack.conf health

## 网络设置

### 概述

网络配置对构建高性能 Ceph 存储来说相当重要。 Ceph 存储集群不会代表 Ceph 客户端执行请求路由或调度，相反， Ceph 客户端（如块设备、 CephFS 、 REST 网关）直接向 OSD 请求，然后 OSD 为客户端执行数据复制，也就是说复制和其它因素会额外增加集群网的负载。

我们的快速入门配置提供了一个简陋的 Ceph 配置文件，其中只设置了监视器 IP 地址和守护进程所在的主机名。如果没有配置集群网，那么 Ceph 假设你只有一个“公共网”。只用一个网可以运行 Ceph ，但是在大型集群里用单独的“集群”网可显著地提升性能。

我们建议用两个网络运营 Ceph 存储集群：一个公共网（前端）和一个集群网（后端）。为此，各节点得配备多个网卡

![如图](https://img2018.cnblogs.com/blog/1191664/201811/1191664-20181126103555230-182309917.png)

运营两个独立网络的考量主要有：

1.  性能： OSD 为客户端处理数据复制，复制多份时 OSD 间的网络负载势必会影响到客户端和 Ceph 集群的通讯，包括延时增加、产生性能问题；恢复和重均衡也会显著增加公共网延时。[参见](http://docs.ceph.org.cn/architecture#scalability-and-high-availability)
2.  安全： 大多数人都是良民，很少的一撮人喜欢折腾拒绝服务攻击（ DoS ）。当 OSD 间的流量失控时，归置组再也不能达到 active \+ clean 状态，这样用户就不能读写数据了。挫败此类攻击的一种好方法是维护一个完全独立的集群网，使之不能直连互联网；另外，请考虑用消息签名防止欺骗攻击。

### 防火墙

守护进程默认会绑定到 6800：7300 间的端口，你可以更改此范围。更改防火墙配置前先检查下 iptables 配置。  

sudo iptables -L

某些 Linux 发行版规矩拒绝除 SSH 之外所有网络接口的的所有入站请求。 例如：

REJECT all -- anywhere anywhere reject-with icmp-host-prohibited

你得先删掉公共网和集群网对应的这些规则，然后再增加安全保护规则。

监视器默认监听 6789 端口，而且监视器总是运行在公共网。按下例增加规则时，要把\{iface\}替换为公共网接口（如 eth0，eth1 等等），\{ip-address\}替换为 公共网 IP，\{netmask\}替换为公共网掩码。

sudo iptables -A INPUT -i \{iface\} -p tcp -s \{ip-address\}/\{netmask\} --dport 6789 -j ACCEPT

MDS 服务器会监听公共网 6800 以上的第一个可用端口。需要注意的是，这种行为是不确定的，所以如果你在同一主机上运行多个 OSD 或 MDS，或者在很短的时间内 重启了多个守护进程，它们会绑定更高的端口号;所以说你应该预先打开整个 6800-7300 端口区间。按下例增加规则时，要把\{iface\}替换为公共网接口（如 eth0 ，eth1 等等），\{ip-address\}替换为公共网 IP，\{netmask\}替换为公共网掩码。

sudo iptables -A INPUT -i \{iface\} -m multiport -p tcp -s \{ip-address\}/\{netmask\} --dports 6800:7300 -j ACCEPT

OSD 守护进程默认绑定从 6800 起的第一个可用端口，需要注意的是，这种行为是不确定的，所以如果你在同一主机上运行多个 OSD 或 MDS，或者在很短的时间内 重启了多个守护进程，它们会绑定更高的端口号。一主机上的各个 OSD 最多会用到 4 个端口：  
　　1.一个用于和客户端，监视器通讯;  
　　2.一个用于发送数据到其他 OSD;  
　　3.两个用于各个网卡上的心跳;

![如图](https://img2018.cnblogs.com/blog/1191664/201811/1191664-20181126104456037-902739347.png)

当某个守护进程失败并重启时没释放端口，重启后的进程就会监听新端口。你应该打开整个 6800-7300 端口区间，以应对这种可能性。

如果你分开了公共网和集群网，必须分别为之设置防火墙，因为客户端会通过公共网连接，而其他 OSD 会通过集群网连接。按下例增加规则时，要把\{iface\}替换为网 口（如 eth0，eth1 等等），\{ip-address\}替换为公共网或集群网 IP，\{netmask\}替换为公共网或集群网掩码。例如：

sudo iptables -A INPUT -i \{iface\}  -m multiport -p tcp -s \{ip-address\}/\{netmask\} --dports 6800:7300 -j ACCEPT

如果你的元数据服务器和 OSD 在同一节点上，可以合并公共网配置。

### Ceph 网络

Ceph 的网络配置要放到 \[global\] 段下。前述的 5 分钟快速入门提供了一个简陋的 Ceph 配置文件，它假设服务器和客户端都位于同一网段， Ceph 可以很好地适应这种情形。然而 Ceph 允许配置更精细的公共网，包括多 IP 和多掩码；也能用单独的集群网处理 OSD 心跳、对象复制、和恢复流量。不要混淆你配置的 IP 地址和客户端用来访问存储服务的公共网地址。典型的内网常常是 192.168.0.0 或 10.0.0.0 。

如果你给公共网或集群网配置了多个 IP 地址及子网掩码，这些子网必须能互通。另外要确保在防火墙上为各 IP 和子网开放了必要的端口。Ceph 用 CIDR 法表示子网，如 10.0.0.0/24。

配置完几个网络后，可以重启集群或挨个重启守护进程.Ceph 守护进程动态地绑定端口，所以更改网络配置后无需重启整个集群。

公共网络  
要配置公共网络，请将以下选项添加到 Ceph 配置文件的\[global\]部分。
```plain
\[global\]
        # ... elided configuration
        public network \= \{public-network/netmask\}
```
集群网络  
如果声明群集网络，OSD 将通过群集网络路由心跳，对象复制和恢复流量。 与使用单个网络相比，这可以提高性能。 要配置群集网络，请将以下选项添加到 Ceph 配置文件的\[global\]部分。
```plain
\[global\]
        # ... elided configuration
        cluster network \= \{cluster-network/netmask\}
```
我们希望无法从公共网络或 Internet 访问群集网络以增强安全性。

### Ceph Daemon

有一个网络配置是所有守护进程都要配的：各个守护进程都必须指定主机，Ceph 也要求指定监视器 IP 地址及端口。一些部署工具（如 ceph-deploy，Chef）会给你创建配置文件，如果它能胜任那就别设置这些值。主机选项是主机的短名，不是全资域名 FQDN，也不是 IP 地址。在命令行下输入主机名-s 获取主机名。
```plain
\[mon.a\]
        host \= \{hostname\}

        mon addr \= \{ip-address\}:6789 \[osd.0\]
        host \= \{hostname\}
```
您不必为守护程序设置主机 IP 地址。 如果您具有静态 IP 配置并且公共和集群网络都在运行，则 Ceph 配置文件可以为每个守护程序指定主机的 IP 地址。 要为守护程序设置静态 IP 地址，以下选项应出现在 ceph.conf 文件的守护程序实例部分中。
```plain
\[osd.0\]
        public addr \= \{host-public-ip-address\}
        cluster addr \= \{host-cluster-ip-address\}
```
单网卡 OSD，双网络集群

一般来说，我们不建议用单网卡 OSD 主机部署两个网络。然而这事可以实现，把公共地址选择配在\[osd.n\]段下即可强制 OSD 主机运行在公共网，其中 n 是其 OSD 号。另外，公共网和集群网必须互通，考虑到安全因素我们不建议这样做。

### 网络配置选项

网络配置选项不是必需的，Ceph 假设所有主机都运行于公共网，除非你特意配置了一个集群网。

  
公共网

公共网配置用于明确地为公共网定义 IP 地址和子网。你可以分配静态 IP 或用 public addr 覆盖 public network 选项。

public network
描述:公共网（前端）的 IP 地址和掩码（如 192.168.0.0/24 ），置于 \[global\] 下。多个子网用逗号分隔。 
类型:\{ip\-address\}/\{netmask\} \[, \{ip-address\}/\{netmask\}\] 
是否必需:No 
默认值:N/A 

public addr
描述:用于公共网（前端）的 IP 地址。适用于各守护进程。 
类型:IP 地址 
是否必需:No 
默认值:N/A 

集群网

集群网配置用来声明一个集群网，并明确地定义其 IP 地址和子网。你可以配置静态 IP 或为某 OSD 守护进程配置 cluster addr 以覆盖 cluster network 选项。

cluster network
描述:集群网（后端）的 IP 地址及掩码（如 10.0.0.0/24 ），置于 \[global\] 下。多个子网用逗号分隔。 
类型:\{ip\-address\}/\{netmask\} \[, \{ip-address\}/\{netmask\}\] 
是否必需:No 
默认值:N/A

cluster addr
描述:集群网（后端） IP 地址。置于各守护进程下。 
类型:Address 
是否必需:No 
默认值:N/A 

绑定

绑定选项用于设置 OSD 和 MDS 默认使用的端口范围，默认范围是 6800:7300 。确保防火墙开放了对应端口范围。

你也可以让 Ceph 守护进程绑定到 IPv6 地址。

ms bind port min
描述:OSD 或 MDS 可绑定的最小端口号。 
类型:32\-bit Integer 
默认值:6800 是否必需:No 

ms bind port max
描述:OSD 或 MDS 可绑定的最大端口号。 
类型:32\-bit Integer 
默认值:7300 是否必需:No. 

ms bind ipv6
描述:允许 Ceph 守护进程绑定 IPv6 地址。 
类型:Boolean 
默认值:false 是否必需:No 

主机

Ceph 配置文件里至少要写一个监视器、且每个监视器下都要配置 mon addr 选项；每个监视器、元数据服务器和 OSD 下都要配 host 选项。

mon addr
描述:\{hostname\}:\{port\} 条目列表，用以让客户端连接 Ceph 监视器。如果未设置， Ceph 查找 \[mon.\*\] 段。 
类型:String 
是否必需:No
默认值:N/A 

host
描述:主机名。此选项用于特定守护进程，如 \[osd.0\] 。 
类型:String 
是否必需:Yes, for daemon instances. 
默认值:localhost 

不要用 localhost 。在命令行下执行 hostname \-s 获取主机名（到第一个点，不是全资域名），并用于配置文件。用第三方部署工具时不要指定 host 选项，它会自行获取。

TCP

Ceph 默认禁用 TCP 缓冲。

ms tcp nodelay
描述:Ceph 用 ms tcp nodelay 使系统尽快（不缓冲）发送每个请求。禁用 Nagle 算法可增加吞吐量，但会引进延时。如果你遇到大量小包，可以禁用 ms tcp nodelay 试试。 
类型:Boolean 
是否必需:No 
默认值:true ms tcp rcvbuf
描述:网络套接字接收缓冲尺寸，默认禁用。 
类型:32\-bit Integer 
是否必需:No 
默认值:0 ms tcp read timeout
描述:如果一客户端或守护进程发送请求到另一个 Ceph 守护进程，且没有断开不再使用的连接，在 ms tcp read timeout 指定的秒数之后它将被标记为空闲。 
类型:Unsigned 64\-bit Integer 
是否必需:No 
默认值:900 15 minutes. 

## 认证设置

### 概述

cephx 协议已经默认开启。加密认证要耗费一定计算资源，但通常很低。如果您的客户端和服务器网络环境相当安全，而且认证的负面效应更大，你可以关闭它，通常不推荐您这么做。如果禁用了认证，就会有篡改客户端/服务器消息这样的中间人攻击风险，这会导致灾难性后果。

### 部署情景

部署 Ceph 集群有两种主要方案，这会影响您最初配置 Cephx 的方式。 Ceph 用户大多数第一次使用 ceph-deploy 来创建集群（最简单）。 对于使用其他部署工具（例如，Chef，Juju，Puppet 等）的集群，您将需要使用手动过程或配置部署工具来引导您的监视器。

* CEPH-DEPLOY

使用 ceph-deploy 部署集群时，您不必手动引导监视器或创建 client.admin 用户或密钥环。 您在 Storage Cluster 快速入门中执行的步骤将调用 ceph-deploy 为您执行此操作。

当您执行 ceph-deploy new \{initial-monitor（s）\}时，Ceph 将为您创建一个监视密钥环（仅用于引导监视器），它将为您生成一个初始 Ceph 配置文件，其中包含以下身份验证设置 ，表明 Ceph 默认启用身份验证：

auth\_cluster\_required = cephx
auth\_service\_required \= cephx
auth\_client\_required \= cephx

当您执行 ceph-deploy mon create-initial 时，Ceph 将引导初始监视器，检索包含 client.admin 用户密钥的 ceph.client.admin.keyring 文件。 此外，它还将检索密钥环，使 ceph-deploy 和 ceph-volume 实用程序能够准备和激活 OSD 和元数据服务器。

当您执行 ceph-deploy admin \{node-name\}时（注意：首先必须安装 Ceph），您将 Ceph 配置文件和 ceph.client.admin.keyring 推送到节点的/ etc / ceph 目录。 您将能够在该节点的命令行上以 root 身份执行 Ceph 管理功能。

* 手动部署

手动部署集群时，必须手动引导监视器并创建 client.admin 用户和密钥环。 要引导监视器，请按照 [Monitor Bootstrapping](http://docs.ceph.com/docs/master/install/manual-deployment#monitor-bootstrapping)中的步骤操作。 监视器引导的步骤是使用 Chef，Puppet，Juju 等第三方部署工具时必须执行的逻辑步骤。

### 启用/禁用 CEPHX

为监视器，OSD 和元数据服务器部署密钥时需启用 Cephx。 如果你只是打开/关闭 Cephx，你不必重复 bootstrapping 程序。

启用 Cephx

启用 cephx 后，Ceph 将在默认搜索路径（包括/etc/ceph/ceph.\$name.keyring）里查找密钥环。你可以在 Ceph 配置文件的\[global\]段里添加 keyring 选项来修改，但不推荐。

在禁用了 cephx 的集群上执行下面的步骤来启用它，如果你（或者部署工具）已经生成了密钥，你可以跳过相关的步骤。

1.创建 client.admin 密钥，并为客户端保存此密钥的副本：

ceph auth get-or-create client.admin mon 'allow \*' mds 'allow \*' osd 'allow \*' -o /etc/ceph/ceph.client.admin.keyring

警告：此命令会覆盖任何存在的/etc/ceph/client.admin.keyring 文件，如果部署工具已经完成此步骤，千万别再执行此命令。多加小心！

2.创建监视器集群所需的密钥环，并给它们生成密钥。

ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow \*'

3.把监视器密钥环复制到 ceph.mon.keyring 文件，再把此文件复制到各监视器的 mon 数据目录下。比如要把它复制给名为 ceph 集群的 mon.a，用此命令：

cp /tmp/ceph.mon.keyring /var/lib/ceph/mon/ceph-a/keyring

4.为每个 MGR 生成密钥，\{\$ id\}是 OSD 编号：

ceph auth get-or-create mgr.\{\$id\} mon 'allow profile mgr' mds 'allow \*' osd 'allow \*' -o /var/lib/ceph/mgr/ceph-\{\$id\}/keyring

5.为每个 OSD 生成密钥， \{\$id\} 是 MDS 的标识字母：

ceph auth get-or-create osd.\{\$id\} mon 'allow rwx' osd 'allow \*' -o /var/lib/ceph/osd/ceph-\{\$id\}/keyring

6.为每个 MDS 生成密钥， \{\$id\} 是 MDS 的标识字母：

ceph auth get-or-create mds.\{\$id\} mon 'allow rwx' osd 'allow \*' mds 'allow \*' mgr 'allow profile mds' -o /var/lib/ceph/mds/ceph-\{\$id\}/keyring

7.把以下配置加入 Ceph 配置文件的 \[global\] 段下以启用 cephx 认证：

auth cluster required = cephx
auth service required \= cephx
auth client required \= cephx

8.启动或重启 Ceph 集群

禁用 Cephx

下述步骤描述了如何禁用 Cephx。如果你的集群环境相对安全，你可以减免认证耗费的计算资源，然而我们不推荐。但是临时禁用认证会使安装，和/或排障更简单。

1.把下列配置加入 Ceph 配置文件的\[global\]段下即可禁用 cephx 认证：

auth cluster required = none
auth service required \= none
auth client required \= none

2.启动或重启 Ceph 集群

### 配置选项

 启动

auth cluster required
描述:如果启用了，集群守护进程（如 ceph\-mon 、 ceph-osd 和 ceph-mds ）间必须相互认证。可用选项有 cephx 或 none 。 
类型:String 
是否必需:No 
默认值:cephx. 

auth service required
描述:如果启用，客户端要访问 Ceph 服务的话，集群守护进程会要求它和集群认证。可用选项为 cephx 或 none 。 
类型:String 
是否必需:No 
默认值:cephx. 

auth client required
描述:如果启用，客户端会要求 Ceph 集群和它认证。可用选项为 cephx 或 none 。 
类型:String 
是否必需:No 
默认值:cephx. 

密钥

如果你的集群启用了认证， ceph 管理命令和客户端得有密钥才能访问集群。

给 ceph 管理命令和客户端提供密钥的最常用方法就是把密钥环放到 /etc/ceph ，通过 ceph-deploy 部署的 Cuttlefish 及更高版本，其文件名通常是 ceph.client.admin.keyring （或 \$cluster.client.admin.keyring ）。如果你的密钥环位于 /etc/ceph 下，就不需要在 Ceph 配置文件里指定 keyring 选项了。

我们建议把集群的密钥环复制到你执行管理命令的节点，它包含 client.admin 密钥。

你可以用 ceph-deploy admin 命令做此事，手动复制可执行此命令：

sudo scp \{user\}\@\{ceph-cluster-host\}:/etc/ceph/ceph.client.admin.keyring /etc/ceph/ceph.client.admin.keyring

确保给客户端上的 ceph.keyring 设置合理的权限位（如 chmod 644）。

你可以用 key 选项把密钥写在配置文件里（不推荐），或者用 keyfile 选项指定个路径。

keyring
描述:密钥环文件的路径。 
类型:String 
是否必需:No 
默认值:/etc/ceph/\$cluster.\$name.keyring,/etc/ceph/\$cluster.keyring,/etc/ceph/keyring,/etc/ceph/keyring.bin 

keyfile
描述:到密钥文件的路径，如一个只包含密钥的文件。 
类型:String 
是否必需:No 
默认值:None 

key
描述:密钥（密钥文本），最好别这样做。 
类型:String 
是否必需:No 
默认值:None 

DAEMON KEYRINGS  
管理用户或部署工具（例如，ceph-deploy）可以以与生成用户密钥环相同的方式生成守护进程密钥环。 默认情况下，Ceph 将守护进程密钥环存储在其数据目录中。 默认密钥环位置以及守护程序运行所需的功能如下所示。

ceph-mon
Location:          \$mon\_data/keyring
Capabilities:      mon 'allow \*' ceph\-osd
Location:          \$osd\_data/keyring
Capabilities:      mgr 'allow profile osd' mon 'allow profile osd' osd 'allow \*' ceph\-mds
Location:          \$mds\_data/keyring
Capabilities:      mds 'allow' mgr 'allow profile mds' mon 'allow profile mds' osd 'allow rwx' ceph\-mgr
Location:          \$mgr\_data/keyring
Capabilities:      mon 'allow profile mgr' mds 'allow \*' osd 'allow \*' radosgw
Location:          \$rgw\_data/keyring
Capabilities:      mon 'allow rwx' osd 'allow rwx' 

监视器密钥环（即 mon. ）包含一个密钥，但没有能力，且不是集群 auth 数据库的一部分。

守护进程数据目录位置默认格式如下：

/var/lib/ceph/\$type/\$cluster-\$id

例如， osd.12 的目录会是：

/var/lib/ceph/osd/ceph-12

你可以覆盖这些位置，但不推荐。

签名

在 Bobtail 及后续版本中， Ceph 会用开始认证时生成的会话密钥认证所有在线实体。然而 Argonaut 及之前版本不知道如何认证在线消息，为保持向后兼容性（如在同一个集群里运行 Bobtail 和 Argonaut ），消息签名默认是关闭的。如果你只运行 Bobtail 和后续版本，可以让 Ceph 要求签名。

像 Ceph 认证的其他部分一样，客户端和集群间的消息签名也能做到细粒度控制；而且能启用或禁用 Ceph 守护进程间的签名。

cephx require signatures
描述:若设置为 true ， Ceph 集群会要求客户端签名所有消息，包括集群内其他守护进程间的。 
类型:Boolean 
是否必需:No 
默认值:false cephx cluster require signatures
描述:若设置为 true ， Ceph 要求集群内所有守护进程签名相互之间的消息。 
类型:Boolean 
是否必需:No 
默认值:false cephx service require signatures
描述:若设置为 true ， Ceph 要求签名所有客户端和集群间的消息。 
类型:Boolean 
是否必需:No 
默认值:false cephx sign messages
描述:如果 Ceph 版本支持消息签名， Ceph 会签名所有消息以防欺骗。 
类型:Boolean 
默认值:true 

生存期

auth service ticket ttl  
描述:Ceph 存储集群发给客户端一个用于认证的票据时分配给这个票据的生存期。 
类型:Double 
默认值:60\*60 

## Monitor 设置

### 概述

理解如何配置 Ceph 监视器是构建可靠的 Ceph 存储集群的重要方面，任何 Ceph 集群都需要至少一个监视器。一个监视器通常相当一致，但是你可以增加，删除，或替换集群中的监视器

### 背景

监视器们维护着集群运行图的“主副本”，就是说客户端连到一个监视器并获取当前运行图就能确定所有监视器、 OSD 和元数据服务器的位置。 Ceph 客户端读写 OSD 或元数据服务器前，必须先连到一个监视器，靠当前集群运行图的副本和 CRUSH 算法，客户端能计算出任何对象的位置，故此客户端有能力直接连到 OSD ，这对 Ceph 的高伸缩性、高性能来说非常重要。监视器的主要角色是维护集群运行图的主副本，它也提供认证和日志记录服务。 Ceph 监视器们把监视器服务的所有更改写入一个单独的 Paxos 例程，然后 Paxos 以键/值方式存储所有变更以实现高度一致性。同步期间， Ceph 监视器能查询集群运行图的近期版本，它们通过操作键/值存储快照和迭代器（用 leveldb ）来进行存储级同步

![如图](https://img2018.cnblogs.com/blog/1191664/201811/1191664-20181127090427115-627338682.png)

自 0.58 版以来已弃用  
在 0.58 及更早版本中，Ceph 监视器每个服务用一个 Paxos 例程，并把运行图存储为文件。

### CLUSTER MAPS

集群运行图是多个图的组合，包括监视器图、 OSD 图、归置组图和元数据服务器图。集群运行图追踪几个重要事件：哪些进程在集群里（ in ）；哪些进程在集群里（ in ）是 up 且在运行、或 down ；归置组状态是 active 或 inactive 、 clean 或其他状态；和其他反映当前集群状态的信息，像总存储容量、和使用量。

当集群状态有明显变更时，如一 OSD 挂了、一归置组降级了等等，集群运行图会被更新以反映集群当前状态。另外，监视器也维护着集群的主要状态历史。监视器图、 OSD 图、归置组图和元数据服务器图各自维护着它们的运行图版本。我们把各图的版本称为一个 epoch 。

运营集群时，跟踪这些状态是系统管理任务的重要部分。

### MONITOR QUORUM

本文入门部分提供了一个简陋的 Ceph 配置文件，它提供了一个监视器用于测试。只用一个监视器集群可以良好地运行，然而单监视器是一个单故障点，生产集群要实现高可用性的话得配置多个监视器，这样单个监视器的失效才不会影响整个集群。

集群用多个监视器实现高可用性时，多个监视器用 Paxos 算法对主集群运行图达成一致，这里的一致要求大多数监视器都在运行且够成法定人数（如 1 个、 3 之 2 在运行、 5 之 3 、 6 之 4 等等）。

mon force quorum join 描述：强制监视器加入仲裁，即使它先前已从 MAP 中删除
类型：Boolean
默认值：false

### 一致性

你把监视器加进 Ceph 配置文件时，得注意一些架构问题， Ceph 发现集群内的其他监视器时对其有着严格的一致性要求。尽管如此， Ceph 客户端和其他 Ceph 守护进程用配置文件发现监视器，监视器却用监视器图（ monmap ）相互发现而非配置文件。

一个监视器发现集群内的其他监视器时总是参考 monmap 的本地副本，用 monmap 而非 Ceph 配置文件避免了可能损坏集群的错误（如 ceph.conf 中指定地址或端口的拼写错误）。正因为监视器把 monmap 用于发现、并共享于客户端和其他 Ceph 守护进程间， monmap 可严格地保证监视器的一致性是可靠的。

严格的一致性也适用于 monmap 的更新，因为关于监视器的任何更新、关于 monmap 的变更都是通过称为 Paxos 的分布式一致性算法传递的。监视器们必须就 monmap 的每次更新达成一致，以确保法定人数里的每个监视器 monmap 版本相同，如增加、删除一个监视器。 monmap 的更新是增量的，所以监视器们都有最新的一致版本，以及一系列之前版本。历史版本的存在允许一个落后的监视器跟上集群当前状态。

如果监视器通过配置文件而非 monmap 相互发现，这会引进其他风险，因为 Ceph 配置文件不是自动更新并分发的，监视器有可能不小心用了较老的配置文件，以致于不认识某监视器、放弃法定人数、或者产生一种 Paxos 不能确定当前系统状态的情形

### 初始化监视器

在大多数配置和部署案例中，部署 Ceph 的工具可以帮你生成一个监视器图来初始化监视器（如 ceph-deploy 等），一个监视器需要的选项：

* 文件系统标识符： fsid 是对象存储的唯一标识符。因为你可以在一套硬件上运行多个集群，所以在初始化监视器时必须指定对象存储的唯一标识符。部署工具通常可替你完成（如 ceph-deploy 会调用类似 uuidgen 的程序），但是你也可以手动指定 fsid 。
* 监视器标识符： 监视器标识符是分配给集群内各监视器的唯一 ID ，它是一个字母数字组合，为方便起见，标识符通常以字母顺序结尾（如 a 、 b 等等），可以设置于 Ceph 配置文件（如 \[mon.a\] 、 \[mon.b\] 等等）、部署工具、或 ceph 命令行工具。
* 密钥： 监视器必须有密钥。像 ceph-deploy 这样的部署工具通常会自动生成，也可以手动完成。

### 配置

要把配置应用到整个集群，把它们放到 \[global\] 下；要用于所有监视器，置于 \[mon\] 下；要用于某监视器，指定监视器例程，如 \[mon.a\] ）。按惯例，监视器例程用字母命名。

\[global\]

\[mon\]

\[mon.a\]

\[mon.b\]

\[mon.c\]

最小配置

Ceph 监视器的最简配置必须包括一主机名及其监视器地址，这些配置可置于 \[mon\] 下或某个监视器下。

\[mon\]
        mon host \= hostname1,hostname2,hostname3
        mon addr \= 10.0.0.10:6789,10.0.0.11:6789,10.0.0.12:6789

\[mon.a\]
        host \= hostname1
        mon addr \= 10.0.0.10:6789

这里的监视器最简配置假设部署工具会自动给你生成 fsid 和 mon. 密钥。一旦部署了 Ceph 集群，监视器 IP 地址不应该更改。然而，如果你决意要改，必须严格按照更改监视器 IP 地址来改。

客户端也可以使用 DNS SRV 记录找到监视器

集群 ID

每个 Ceph 存储集群都有一个唯一标识符（ fsid ）。如果指定了，它应该出现在配置文件的 \[global\] 段下。部署工具通常会生成 fsid 并存于监视器图，所以不一定会写入配置文件， fsid 使得在一套硬件上运行多个集群成为可能。

fsid
描述:集群 ID ，一集群一个。 
类型:UUID 
是否必需:Yes. 
默认值:无。若未指定，部署工具会生成。 

如果你用部署工具就不能设置

初始成员

我们建议在生产环境下最少部署 3 个监视器，以确保高可用性。运行多个监视器时，你可以指定为形成法定人数成员所需的初始监视器，这能减小集群上线时间。

\[mon\]
        mon initial members \= a,b,c

mon initial members
描述:集群启动时初始监视器的 ID ，若指定， Ceph 需要奇数个监视器来确定最初法定人数（如 3 ）。 
类型:String 
默认值:None 

数据

Ceph 提供 Ceph 监视器存储数据的默认路径。为了在生产 Ceph 存储集群中获得最佳性能，我们建议在 Ceph OSD 守护进程和 Ceph Monitor 在不同主机和驱动器上运行。由于 leveldb 使用 mmap（）来编写数据，Ceph Monitors 经常将内存中的数据刷新到磁盘，如果数据存储与 OSD 守护进程共存，则可能会干扰 Ceph OSD 守护进程工作负载。

在 Ceph 版本 0.58 及更早版本中，Ceph Monitors 将其数据存储在文件中。这种方法允许用户使用 ls 和 cat 等常用工具检查监控数据。但是，它没有提供强大的一致性。

在 Ceph 0.59 及后续版本中，监视器以键/值对存储数据。监视器需要 ACID 事务，数据存储的使用可防止监视器用损坏的版本进行恢复，除此之外，它允许在一个原子批量操作中进行多个修改操作。

一般来说我们不建议更改默认数据位置，如果要改，我们建议所有监视器统一配置，加到配置文件的 \[mon\] 下。

mon data
描述：监视器的数据位置。
类型：String
默认值：/var/lib/ceph/mon/\$cluster-\$id mon data size warn
描述：当监视器的数据存储超过 15GB 时，在群集日志中发出 HEALTH\_WARN。
类型：整型
默认值：15 \* 1024 \* 1024 \* 1024 \* mon data avail warn
描述：当监视器数据存储的可用磁盘空间小于或等于此百分比时，在群集日志中发出 HEALTH\_WARN。
类型：整型
默认值：30 mon data avail crit
描述：当监视器数据存储的可用磁盘空间小于或等于此百分比时，在群集日志中发出 HEALTH\_ERR。
类型：整型
默认值：5 mon warn on cache pools without hit sets
描述：如果缓存池未配置 hit\_set\_type 值，则在集群日志中发出 HEALTH\_WARN。有关详细信息，请参阅 hit\_set\_type。
类型：Boolean
默认值：true mon warn on crush straw calc version zero
描述：如果 CRUSH 的 straw\_calc\_version 为零，则在集群日志中发出 HEALTH\_WARN。有关详细信息，请参阅 CRUSH map tunables。
类型：Boolean
默认值：true mon warn on legacy crush tunables
描述：如果 CRUSH 可调参数太旧（早于 mon\_min\_crush\_required\_version），则在集群日志中发出 HEALTH\_WARN
类型：Boolean
默认值：true mon crush min required version
描述：群集所需的最小可调参数版本。有关详细信息，请参阅 CRUSH map tunables。
类型：String
默认值：firefly

mon warn on osd down out interval zero
描述：if mon osd down out interval is zero，则在集群日志中发出 HEALTH\_WARN。在 leader 上将此选项设置为零的行为与 noout 标志非常相似。在没有设置 noout 标志的情况下很难弄清楚集群出了什么问题但是现象差不多，所以我们在这种情况下报告一个警告。
类型：Boolean
默认值：true mon cache target full warn ratio
描述：cache\_target\_full 和 target\_max\_object 之间，我们开始警告
类型：float 默认值：0.66 mon health data update interval
描述：仲裁中的监视器与其对等方共享其健康状态的频率（以秒为单位）。 （负数禁用它）
类型：float 默认值：60 mon health to clog
描述：定期启用向群集日志发送运行状况摘要。
类型：Boolean
默认值：true mon health to clog tick interval
描述：监视器将健康摘要发送到群集日志的频率（以秒为单位）（非正数禁用它）。 如果当前运行状况摘要为空或与上次相同，则监视器不会将其发送到群集日志。
类型：整型
默认值：3600 mon health to clog interval
描述：监视器将健康摘要发送到群集日志的频率（以秒为单位）（非正数禁用它）。 无论摘要是否更改，Monitor 都将始终将摘要发送到群集日志。
类型：整型
默认值：60

存储容量

Ceph 存储集群利用率接近最大容量时（即 mon osd full ratio ），作为防止数据丢失的安全措施，它会阻止你读写 OSD 。因此，让生产集群用满可不是好事，因为牺牲了高可用性。 full ratio 默认值是 .95 或容量的 95\% 。对小型测试集群来说这是非常激进的设置。

Tip:监控集群时，要警惕和 nearfull 相关的警告。这意味着一些 OSD 的失败会导致临时服务中断，应该增加一些 OSD 来扩展存储容量。

在测试集群时，一个常见场景是：系统管理员从集群删除一个 OSD 、接着观察重均衡；然后继续删除其他 OSD ，直到集群达到占满率并锁死。我们建议，即使在测试集群里也要规划一点空闲容量用于保证高可用性。理想情况下，要做好这样的预案：一系列 OSD 失败后，短时间内不更换它们仍能恢复到 active + clean 状态。你也可以在 active + degraded 状态运行集群，但对正常使用来说并不好。

下图描述了一个简化的 Ceph 集群，它包含 33 个节点、每主机一个 OSD 、每 OSD 3TB 容量，所以这个小白鼠集群有 99TB 的实际容量，其 mon osd full ratio 为 .95 。如果它只剩余 5TB 容量，集群就不允许客户端再读写数据，所以它的运行容量是 95TB ，而非 99TB 。

![如图](https://img2018.cnblogs.com/blog/1191664/201811/1191664-20181127095500225-1199215238.png)

在这样的集群里，坏一或两个 OSD 很平常；一种罕见但可能发生的情形是一个机架的路由器或电源挂了，这会导致多个 OSD 同时离线（如 OSD 7-12 ），在这种情况下，你仍要力争保持集群可运行并达到 active + clean 状态，即使这意味着你得在短期内额外增加一些 OSD 及主机。如果集群利用率太高，在解决故障域期间也许不会丢数据，但很可能牺牲数据可用性，因为利用率超过了 full ratio 。故此，我们建议至少要粗略地规划下容量。

确定群集的两个数字：OSD 的数量和集群的总容量

如果将群集的总容量除以群集中的 OSD 数，则可以找到群集中 OSD 的平均平均容量。 考虑将该数字乘以您期望在正常操作期间同时失败的 OSD 数量（相对较小的数量）。 最后将群集的容量乘以全部比率，以达到最大运行容量; 然后，减去那些预期会故障的 OSD 从而计算出合理的 full ratio。 用更多数量的 OSD 故障（例如，一组 OSD）重复上述过程，以得到合理的 near full ratio。

以下设置仅适用于群集创建，然后存储在 OSDMap 中。
```plain
\[global\]


        mon osd full ratio \= .80 mon osd backfillfull ratio \= .75 mon osd nearfull ratio \= .70

mon osd full ratio
```
描述：OSD 被视为已满之前使用的磁盘空间百分比。
类型：float 默认值：0.95 mon osd backfillfull ratio
说明：在 OSD 被认为太满而无法 backfill 之前使用的磁盘空间百分比。
类型：float 默认值：0.90 mon osd nearfull ratio
描述：在 OSD 被认为接近满之前使用的磁盘空间百分比。
类型：float 默认值：0.85

如果一些 OSD 快满了，但其他的仍有足够空间，你可能配错 CRUSH 权重了。

这些设置仅适用于群集创建期间。 之后需要使用 ceph osd set-almostfull-ratio 和 ceph osd set-full-ratio 在 OSDMap 中进行更改

心跳

Ceph 监视器要求各 OSD 向它报告、并接收 OSD 们的邻居状态报告，以此来掌握集群。 Ceph 提供了监视器与 OSD 交互的合理默认值，然而你可以按需修改

监视器存储同步

当你用多个监视器支撑一个生产集群时，各监视器都要检查邻居是否有集群运行图的最新版本（如，邻居监视器的图有一或多个 epoch 版本高于当前监视器的最高版 epoch ），过一段时间，集群里的某个监视器可能落后于其它监视器太多而不得不离开法定人数，然后同步到集群当前状态，并重回法定人数。为了同步，监视器可能承担三种中的一种角色：  
　　1.Leader: Leader 是实现最新 Paxos 版本的第一个监视器。  
　　2.Provider: Provider 有最新集群运行图的监视器，但不是第一个实现最新版。  
　　3.Requester: Requester 落后于 leader ，重回法定人数前，必须同步以获取关于集群的最新信息。

有了这些角色区分， leader 就 可以给 provider 委派同步任务，这会避免同步请求压垮 leader 、影响性能。在下面的图示中， requester 已经知道它落后于其它监视器，然后向 leader 请求同步， leader 让它去和 provider 同步。

![如图](https://img2018.cnblogs.com/blog/1191664/201811/1191664-20181127101603877-310693997.png)

新监视器加入集群时有必要进行同步。在运行中，监视器会不定时收到集群运行图的更新，这就意味着 leader 和 provider 角色可能在监视器间变幻。如果这事发生在同步期间（如 provider 落后于 leader ）， provider 能终结和 requester 间的同步。

一旦同步完成， Ceph 需要修复整个集群，使归置组回到 active + clean 状态。

mon sync trim timeout
描述:    
类型:    Double
默认： 30.0  
  
mon sync heartbeat timeout
描述:    
类型:    Double
默认： 30.0  
 mon sync heartbeat interval
描述:    
类型:    Double
默认： 5.0  
 mon sync backoff timeout
描述:    
类型:    Double
默认： 30.0  
 mon sync timeout
描述:    Number of seconds the monitor will wait for the next update message from its sync provider before it gives up and bootstrap again.
类型:    Double
默认： 60.0  
 mon sync max retries
描述:    
类型:    Integer
默认： 5  
 mon sync max payload size
描述:    The maximum size for a sync payload \(in bytes\).
类型: 32\-bit Integer
默认： 1045676  
 paxos max join drift
描述:    在我们必须首先同步监控数据存储之前的最大 Paxos 迭代。 当监视器发现其对等体远远超过它时，它将首先与数据存储同步，然后再继续。 类型:    Integer
默认： 10  
 paxos stash full interval
描述:    多久（在提交中）存储 PaxosService 状态的完整副本。 当前此设置仅影响 mds，mon，auth 和 mgr PaxosServices。 类型:    Integer
默认： 25  
 paxos propose interval
描述:    Gather updates for this time interval before proposing a map update.
类型:    Double
默认： 1.0  
 paxos min
描述:    The minimum number of paxos states to keep around
类型:    Integer
默认： 500  
 paxos min wait 描述:    The minimum amount of time to gather updates after a period of inactivity.
类型:    Double
默认： 0.05  
 paxos trim min
描述:    Number of extra proposals tolerated before trimming
类型:    Integer
默认： 250  
 paxos trim max
描述:    The maximum number of extra proposals to trim at a time 类型:    Integer
默认： 500  
 paxos service trim min
描述:    The minimum amount of versions to trigger a trim \(0 disables it\)
类型:    Integer
默认： 250  
 paxos service trim max
描述:    The maximum amount of versions to trim during a single proposal \(0 disables it\)
类型:    Integer
默认： 500  
 mon max log epochs
描述:    The maximum amount of log epochs to trim during a single proposal
类型:    Integer
默认： 500  
 mon max pgmap epochs
描述:    The maximum amount of pgmap epochs to trim during a single proposal
类型:    Integer
默认： 500  
 mon mds force trim to
描述:    Force monitor to trim mdsmaps to this point \(0 disables it. dangerous, use with care\)
类型:    Integer
默认： 0  
 mon osd force trim to
描述:    Force monitor to trim osdmaps to this point, even if there is PGs not clean at the specified epoch \(0 disables it. dangerous, use with care\)
类型:    Integer
默认： 0  
 mon osd cache size
描述:    The size of osdmaps cache, not to rely on underlying store’s cache
类型:    Integer
默认： 10  
 mon election timeout
描述:    On election proposer, maximum waiting time for all ACKs in seconds.
类型:    Float
默认： 5  
 mon lease
描述:    The length \(in seconds\) of the lease on the monitor’s versions.
类型:    Float
默认： 5  
 mon lease renew interval factor
描述:    mon lease \* mon lease renew interval factor will be the interval for the Leader to renew the other monitor’s leases. The factor should be less than 1.0.
类型:    Float
默认： 0.6  
 mon lease ack timeout factor
描述:    The Leader will wait mon lease \* mon lease ack timeout factor for the Providers to acknowledge the lease extension.
类型:    Float
默认： 2.0  
 mon accept timeout factor
描述:    The Leader will wait mon lease \* mon accept timeout factor for the Requester\(s\) to accept a Paxos update.   
　　　　　　　　　　It is also used during the Paxos recovery phase for similar purposes.
类型:    Float
默认： 2.0  
 mon min osdmap epochs
描述:    Minimum number of OSD map epochs to keep at all times.
类型: 32\-bit Integer
默认： 500  
 mon max pgmap epochs
描述:    Maximum number of PG map epochs the monitor should keep.
类型: 32\-bit Integer
默认： 500  
  
mon max log epochs
描述:    Maximum number of Log epochs the monitor should keep.
类型: 32\-bit Integer
默认： 500

 时钟

Ceph 守护进程将关键消息传递给彼此，必须在守护进程达到超时阈值之前处理这些消息。 如果 Ceph 监视器中的时钟不同步，则可能导致许多异常。 例如：

1.守护进程忽略收到的消息（例如，时间戳过时）  
2.当没有及时收到消息时，超时/延迟触发超时。

你应该在所有监视器主机上安装 NTP 以确保监视器集群的时钟同步。

时钟漂移即使尚未造成损坏也能被 NTP 感知， Ceph 的时钟漂移或时钟偏差警告即使在 NTP 同步水平合理时也会被触发。提高时钟漂移值有时候尚可容忍， 然而很多因素（像载荷、网络延时、覆盖默认超时值和监控器存储同步选项）都能在不降低 Paxos 保证级别的情况下影响可接受的时钟漂移水平。

Ceph 提供了下列这些可调选项，让你自己琢磨可接受的值。

clock offset
描述:    时钟可以漂移多少，详情见 Clock.cc 。
类型:    Double
默认： 0 自 0.58 版以来已弃用。

mon tick interval
描述:    监视器的心跳间隔，单位为秒。
类型: 32\-bit Integer
默认： 5 mon clock drift allowed
描述:    监视器间允许的时钟漂移量
类型:    Float
默认：    .050 mon clock drift warn backoff
描述:    时钟偏移警告的退避指数。
类型:    Float
默认： 5 mon timecheck interval
描述:    和 leader 的时间偏移检查（时钟漂移检查）。单位为秒。
类型:    Float
默认： 300.0 mon timecheck skew interval
描述:    当 leader 存在 skew 时间时，以秒为单位的时间检查间隔（时钟漂移检查）。
类型:    Float
默认： 30.0

客户端

mon client hunt interval
描述:    客户端每 N 秒尝试一个新监视器，直到它建立连接 类型:    Double
默认： 3.0 mon client ping interval
描述:    客户端每 N 秒 ping 一次监视器。 类型:    Double
默认： 10.0 mon client max log entries per message
描述:    某监视器为每客户端生成的最大日志条数。
类型:    Integer
默认： 1000 mon client bytes
描述:    内存中允许存留的客户端消息数量（字节数）。 类型: 64\-bit Integer Unsigned
默认： 100ul \<\< 20

pool 设置

从版本 v0.94 开始，支持池标志，允许或禁止对池进行更改。

如果以这种方式配置，监视器也可以禁止删除池。

mon allow pool delete
描述:    If the monitors should allow pools to be removed. Regardless of what the pool flags say.
类型:    Boolean
默认： false osd pool default flag hashpspool
描述:    在新池上设置 hashpspool 标志
类型:    Boolean
默认： true osd pool default flag nodelete
描述:    在新池上设置 nodelete 标志。 防止以任何方式删除使用此标志的池。 类型:    Boolean
默认： false osd pool default flag nopgchange
描述:    在新池上设置 nopgchange 标志。 不允许为池更改 PG 的数量。 类型:    Boolean
默认： false osd pool default flag nosizechange
描述:    在新池上设置 nosizechange 标志。 不允许更改池的大小。
类型:    Boolean
默认： false

杂项

mon max osd
描述:    集群允许的最大 OSD 数量。 类型: 32\-bit Integer
默认： 10000 mon globalid prealloc
描述:    为集群和客户端预分配的全局 ID 数量。 类型: 32\-bit Integer
默认： 100 mon subscribe interval
描述:    同步的刷新间隔（秒），同步机制允许获取集群运行图和日志信息。 类型:    Double
默认： 86400 mon stat smooth intervals
描述:    Ceph 将平滑最后 N 个 PG 地图的统计数据。 类型:    Integer
默认： 2 mon probe timeout
描述:    监视器在 bootstrapping 之前等待查找 peers 对等体的秒数。 类型:    Double
默认： 2.0 mon daemon bytes
描述:   给 mds 和 OSD 的消息使用的内存空间（字节）。 类型: 64\-bit Integer Unsigned
默认： 400ul \<\< 20 mon max log entries per event
描述:    每个事件允许的最大日志条数。
类型:    Integer
默认： 4096 mon osd prime pg temp
描述:    Enables or disable priming the PGMap with the previous OSDs when an out OSD comes back into the cluster.   
With the true setting the clients will continue to use the previous OSDs until the newly in OSDs as that PG peered.
类型:    Boolean
默认： true mon osd prime pg temp max time 描述:    当 OSD 返回集群时，监视器应花费多少时间尝试填充 PGMap。 类型:    Float
默认： 0.5 mon osd prime pg temp max time estimate
描述:    Maximum estimate of time spent on each PG before we prime all PGs in parallel.
类型:    Float
默认： 0.25 mon osd allow primary affinity
描述:    允许在 osdmap 中设置 primary\_affinity。 类型:    Boolean
默认：    False

mon osd pool ec fast read
描述:    是否打开池上的快速读取。如果在创建时未指定 fast\_read，它将用作新创建的 erasure pool 的默认设置。 类型:    Boolean
默认：    False

mon mds skip sanity
描述:    Skip safety assertions on FSMap \(in case of bugs where we want to continue anyway\).   
Monitor terminates if the FSMap sanity check fails, but we can disable it by enabling this option.
类型:    Boolean
默认：    False

mon max mdsmap epochs
描述:    The maximum amount of mdsmap epochs to trim during a single proposal.
类型:    Integer
默认： 500 mon config key max entry size
描述:    config-key 条目的最大大小（以字节为单位）类型:    Integer
默认： 4096 mon scrub interval
描述:    How often \(in seconds\) the monitor scrub its store by comparing the stored checksums with the computed ones of all the stored keys.
类型:    Integer
默认： 3600\*24 mon scrub max keys
描述:    The maximum number of keys to scrub each time.
类型:    Integer
默认： 100 mon compact on start
描述:    Compact the database used as Ceph Monitor store on ceph\-mon start.   
A manual compaction helps to shrink the monitor database and improve the performance of it if the regular compaction fails to work.
类型:    Boolean
默认：    False

mon compact on bootstrap
描述:    Compact the database used as Ceph Monitor store on on bootstrap.   
Monitor starts probing each other for creating a quorum after bootstrap. If it times out before joining the quorum, it will start over and bootstrap itself again.
类型:    Boolean
默认：    False

mon compact on trim
描述:    Compact a certain prefix \(including paxos\) when we trim its old states.
类型:    Boolean
默认：    True

mon cpu threads
描述:    Number of threads for performing CPU intensive work on monitor.
类型:    Boolean
默认：    True

mon osd mapping pgs per chunk
描述:    We calculate the mapping from placement group to OSDs in chunks. This option specifies the number of placement groups per chunk.
类型:    Integer
默认： 4096 mon osd max split count
描述:    Largest number of PGs per “involved” OSD to let split create.   
When we increase the pg\_num of a pool, the placement groups will be split on all OSDs serving that pool. We want to avoid extreme multipliers on PG splits.
类型:    Integer
默认： 300 mon session timeout
描述:    Monitor will terminate inactive sessions stay idle over this time limit.
类型:    Integer
默认： 300

## 通过 DNS 查看操作监视器

 从版本 11.0.0 开始，RADOS 支持通过 DNS 查找监视器。

 这样，守护进程和客户端在其 ceph.conf 配置文件中不需要 mon 主�