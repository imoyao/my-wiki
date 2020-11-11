---
title: 各种同步工具之间的差异| DRBD vs SCP vs rsync vs mirror
toc: true
tags:
  - 存储
  - 工具
  - Linux
  - SCP
categories:
  - "\U0001F4BB 工作"
  - 存储
date: 2020-08-07 12:27:56
---
## DRBD

> [DRBD](https://www.linbit.com/drbd/) operates on block device level. This makes it useful for synchronizing systems that are under heavy load. Lsyncd on the other hand does not require you to change block devices and/or mount points, allows you to change uid/gid of the transferred files, separates the receiver through the one-way nature of rsync. DRBD is likely the better option if you are syncing databases.
>
> [GlusterFS](http://www.gluster.org/) and [BindFS](http://bindfs.org/) use a FUSE-Filesystem to interject kernel/userspace filesystem events.
>
> [Mirror](https://github.com/stephenh/mirror) is an asynchronous synchronisation tool that takes use of the inotify notifications much like Lsyncd. The main differences are: it is developed specifically for master-master use, thus running on a daemon on both systems, uses its own transportation layer instead of rsync and is Java instead of Lsyncd’s C core with Lua scripting.

DRBD 用于块级别设备之间的同步，这使得它对于在高负载下的系统（如数据库）同步非常有用。
## SCP
是用于在 Linux 下本地或通过网络进行远程纯线性拷贝文件的命令，和它类似的命令有 cp，不过 cp 只是在本机进行拷贝不能**跨服务器**，而且 `scp` 是**加密传输**的。可能会稍微影响一下速度。当你服务器硬盘变为只读（read only ）系统时，用 `scp` 可以帮你把文件移出来。另外，`scp` **资源消耗非常小**，不会提高多少系统负荷，在这一点上，`rsync` 就远远不及它了。虽然 `rsync` 比 `scp` 会快一点，但当**小文件**众多的情况下，`rsync` 会导致硬盘 I/O 非常高，而 `scp` 基本不影响系统正常使用。
## rsync
`rsync` 是一个本地或通过网络数据同步工具，可通过 LAN/WAN 快速同步多台主机间的文件。`rsync` 使用所谓的“**rsync 算法**”来使本地和远程两个主机之间的文件达到同步，这个算法只传送两个文件的不同部分，而不是每次都整份传送，因此**速度相当快**。

我们通过下面的实例来分析：
```plain
rsync A host:B
```

*   `rsync`将检查文件大小和修改时间戳的 **A**和 **B**，如果它们匹配，则跳过任何进一步的处理。
*   如果目标文件**B**已存在，增量传输算法将确保仅通过网络发送**A**和**B**之间的差异。
*   `rsync`会将数据写入临时文件**T**，然后将目标文件**B**替换为**T**，以使更新对于可能正在使用**B**的进程而言显得“原子”（atomic）。
*   `rsync`在缓慢且不可靠的连接上运行可能很有用。 因此，如果您的下载在大文件中间中止，则`rsync`将能够**断点续传**。

它们之间的另一个区别涉及调用。 `rsync`具有许多命令行选项，允许用户微调其行为。 它支持复杂的过滤规则，可在批处理模式，守护程序等模式下运行。scp 只有几个开关。

总之，将`scp`用于日常任务。 您偶尔在交互式`shell`上键入的命令。 它使用起来更简单，在这种情况下，`rsync`优化不会有太大帮助。

对于诸如`cron`作业之类的重复任务，请使用`rsync`。 如前所述，在多次调用中，它将利用已传输的数据，执行速度非常快并节省资源。 这是使两个目录通过网络保持同步的出色工具。

另外，在处理大文件时，请将`rsync`与`-P`选项一起使用。 如果传输中断，则可以通过重新发出命令在停止的位置继续传输。参见[此处](https://stackoverflow.com/a/21476688)。

关于安全问题，可以使用`rsync --rsh=ssh`使其加密传输。

## 参考链接
[rsync 的核心算法 | | 酷 壳 - CoolShell](https://coolshell.cn/articles/7425.html)
[Lsyncd - Live Syncing (Mirror) Daemon](https://axkibe.github.io/lsyncd/)
[How does `scp` differ from `rsync`? - Stack Overflow](https://stackoverflow.com/questions/20244585/how-does-scp-differ-from-rsync)
[scp 命令，Linux scp 命令详解：加密的方式在本地主机和远程主机之间复制文件 - Linux 命令搜索引擎](https://wangchujiang.com/linux-command/c/scp.html)
[rsync 命令，Linux rsync 命令详解：远程数据同步工具 - Linux 命令搜索引擎](https://wangchujiang.com/linux-command/c/rsync.html)