---
title: AA-ALUA-AP
toc: true
tags: others
categories:
  - "\U0001F4BB 工作"
  - 存储
  - 双活
date: 2020-09-01 12:27:56
---
## A/A

对称双活（Symmetric Active/Acivie），对于特定的 LUN 来说，在它的路径中，两个存储控制器的目标端口均处于主动/优化（active/optimized）状态。两个控制器之间实现高速互联的通讯，一个 IO 发到控制器端，两个控制器可同时参与处理；当一个控制器繁忙，系统不需要主机端的负载均衡软件参与就可以自动实现负载均衡。

## ALUA
异步逻辑单元访问（Asymmetric Logical Unit Access），属于非对称双活（Asymmetric Active/Active）。对于特定的 LUN 来说，主机多路径会将对磁盘的物理路径划分优先级，一个控制器的目标端口处于主动/优化（active/optimized）状态，另一个控制器的目标端口处于主动/非优化（active/unoptimized）状态。在某一个时刻，某个 LUN 只是属于某一个控制器，要想实现两边的负载均衡，就是将任务 A 扔给控制器 A，将任务 B 扔给控制器 B。对于同一个任务来说，任何时候只有一个控制器在控制。

AO 路径：最佳 IO 访问路径，对应工作控制器上的路径。
AN 路径：次优 IO 访问路径，对应非工作控制器上的路径。

### 非双活 ALUA 工作原理及故障切换
当其中一条 AO 路径无法提供访问时，主机 I/O 将下发到另外的 AO 路径；

当工作控制器的所有 AO 路径断开无法提供访问时，主机 I/O 将从非工作控制器上的 AN 路径下发
![非对称双活下的ALUA](/images/ALUA-AS-AA.png)


### 双活 ALUA 工作原理及故障切换
- 当工作在负载均衡模式时
主机多路径将选择双活存储的所有阵列的工作控制器上的路径为 AO 路径，其他所有控制器上的路径为 AN 路径；两端阵列的 AO 路径提供访问，当其中一条 AO 路径无法提供访问时，主机 I/O 将下发到另外的 AO 路径；当其中一台阵列 AO 路径所在控制器故障时，另一个控制器将转换为工作控制器，继续维持负载均衡;
![负载均衡模式下的ALUA](/images/ALUA-load-balance.png)
- 当工作在本端优选模式时
主机多路径将选择本端阵列工作控制器上的路径为 AO 路径，保证 IO 只在本端阵列的工作控制器下发，减小链路消耗；当本端阵列工作控制器的所有 AO 路径断开无法提供访问时，主机 I/O 将从非工作控制器上的 AN 路径下发。当工作控制器故障时，另一个控制器将转换为工作控制器，继续维持本端优选。
![本端优选模式下的ALUA](/images/ALUA-node-first.png)

### ALUA 的优点
- ALUA 阵列和 Active/Passive 阵列的不同
在 Active/Passive 阵列中，IO 只能被发送到拥有该 LUN 的控制器。如果在 non-owning controller 端口上收到 IO 请求，则会向启动器返回类似 "ILLEGAL REQUEST" 的 SCSI 错误，这意味着 "No Entry" 或 "No, you can’t do that"

但是在启用 ALUA 的阵列中，即使在 non-owning controller 上收到 IO，它也不会抛出任何 SCSI 错误，并通过后端通道将 IO 请求重定向到 owning controller。这种 IO 重定向可提升主机 IO 或整体系统性能。

- 都启用了 ALUA 的非对称 Active/Active 阵列与对称 Active/Active 阵列的不同

在对称 Active/Active 阵列，两个控制器都可以接收和处理 IO，但这些阵列不会通知主机路径特性，如带宽、延迟、owning controller 声明的路径、non-owning controller 声明的路径等。
因此，主机在负载平衡和故障情况下平等地考虑每条路径。

在非对称 Active/Active 阵列中，两个控制器都可以接收 IO，但 IO 只能由 owning controller 发出。这样主机知道哪些路径属于 owning 和 non-owning controller 非常重要。
基于该信息，它可以在负载平衡和路径故障期间进行精确和智能的决策，以提升主机性能。Active Optimized/Active Non-Optimized 等路径信息由该阵列的 ALUA 功能提供。
![非对称ALUA重定向](/images/ASymmetricLogicalUnitAccess.png)
如上图，SP-B 为 owning controller。如果请求从 SP-A 进来，ALUA 试图将 IO 从 SP-A 重定向到 SP-B，由 SP-B 处理后，再原路返回给主机。这种方式对性能有影响，但是可以减少因为 trespass 带来的短暂中断问题。

因为如果重定向的次数到了一定阈值，ALUA 会认为原来的路径已经完全不可用，不期望恢复，于是主机端多路径软件触发 trespass 把 LUN 切换到 SP-A。

trespass: LUN 从一个控制器切换到另一个控制器

## A/P
Active/Passive，对于特定的 LUN 来说，在它的路径中，一个控制器的目标端口处于主动/优化（active/optimized）状态，另一个控制器的目标端口处于备用（standby）状态。其负载均衡及任务处理方式与 ALUA 类似。

在 A/A 阵列中，管理员无需指定每个 LUN 的默认所有者，当路径出现故障，将离线故障路径并重定向 IO 到其他路径，IO 重定向期间，存储控制器会充分考虑负载平衡等因素并选择最合适的路径。对于应用程序，路径切换过程是透明的的，几乎不会有延迟（延迟时间一般为几秒）。 

在 ALUA 或 A/P 阵列中，管理员需指定每个 LUN 的默认所有者，设置一些 LUN 的默认所有者为控制器 A，另外一些 LUN 的默认所有者为控制器 B, 人为在两个控制器之间进行负载均衡；如果路径发生故障，将重新分配 IO 流量到其他可用的路径，同时，停止故障路径上的 IO。对于应用程序，路径切换过程是透明的，然而，会有延迟（延迟时间一般为几十秒）。

## 参考链接
[关于 ALUA 详解_Mark-CSDN 博客](https://blog.csdn.net/xoopqy/article/details/19238463)
[控制器双活 负载均衡_松猪的专栏-CSDN 博客](https://blog.csdn.net/songzhulikesleep/article/details/79184125)
[ALUA 工作原理介绍 - 华为 SAN 存储在 AIX 系统下的主机连通性指南 - 华为](https://support.huawei.com/enterprise/zh/doc/EDOC1000158278/620be992)
[iSCSI: ALUA 概述 - Code & Life](https://rjerk.xyz/index.php/archives/161/)