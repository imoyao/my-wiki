---
title: 推荐几本学习 MySQL 的好书
tags:
  - MySQL
  - 数据库
categories:
  - "\U0001F4BB 工作"
  - 数据库
  - MySQL
date: 2019-08-21 12:27:56
---

我这里推荐几本 MySQL 的好书，应该能够有效避免学习 MySQL 的弯路，并且达到一个不错的水平。 我这里推荐的书或材料分为两个部分，分别是 MySQL 的使用和 MySQL 的源码学习。在介绍的过程中，我会穿插简单的评语或感想。

## MySQL 的使用

### MySQL 技术内幕:InnoDB 存储引擎

学习 MySQL 的使用，首推姜承尧的《MySQL 技术内幕:InnoDB 存储引擎》，当然不是因为姜 sir 是我的经理才推荐这本书。这本书确实做到了由渐入深、深入浅出，是中国人写的最赞的 MySQL 技术书籍，符合国人的思维方式和阅读习惯，而且，这本书简直就是面试宝典，对于近期有求职 MySQL 相关岗位的朋友，可以认真阅读，对找工作有很大的帮助。当然，也有人说这本书入门难度较大，这个就自己取舍了，个人建议就以这本书入门即可，有不懂的地方可以求助官方手册和 google。

![MySQL技术内幕](http://mingxinglai.com/cn/image/mysql-book-1.jpg)

### MySQL 的官方手册

我刚开始学习 MySQL 的时候误区就是，没有好好阅读 MySQL 的官方手册。例如，我刚开始很难理解 InnoDB 的锁，尤其是各个情况下如何加锁，这个问题在我师弟进入百度做 DBA 时，也困扰了他一阵子，我们两还讨论来讨论去，其实，MySQL 官方手册已经写得清清楚楚，什么样的 SQL 语句加什么样的锁，当然，MySQL 的官方手册非常庞大，一时半会很难看完，建议先看 InnoDB 相关的部分。

[http://dev.mysql.com/doc/refman/5.7/en/innodb-storage-engine.html](http://dev.mysql.com/doc/refman/5.7/en/innodb-storage-engine.html)

### MySQL 排错指南

《[MySQL 排错指南](http://book.douban.com/subject/26591051/)》是 2015 年夏天引入中国的书籍，这本书可以说是 DBA 速成指南，介绍的内容其实比较简单，但是也非常实用，对于 DBA 这个讲究经验的工种，这本书就是传授经验的，可能对有较多工作经验的 DBA 来说，这本书基本没有什么用，但是，对于刚入职场的新人，或学校里的学生，这本书会有较大的帮助，非常推荐。

![MySQL排错指南](http://mingxinglai.com/cn/image/mysql-book-2.jpg)

### 高性能 MySQL

《[高性能 MySQL](http://book.douban.com/subject/23008813/)》是 MySQL 领域的经典之作，拥有广泛的影响力，学习 MySQL 的朋友都应该有所耳闻，所以我就不作过多介绍，唯一的建议就是仔细看、认真看、多看几遍，我每次看都会有不小的收获。这就是一本虽然书很厚，但是需要一页一页、一行一行都认真看的书。

![高性能MySQL](http://mingxinglai.com/cn/image/mysql-book-3.jpg)

### 数据库索引设计与优化

如果认真学习完前面几本书，基本上都已经对 MySQL 掌握得不错了，但是，如果不了解如何设计一个好的索引，仍然不能成为牛逼的 DBA，牛逼的 DBA 和不牛逼的 DBA，一半就是看对索引的掌握情况，《[数据库索引设计与优化](http://book.douban.com/subject/26419771/)》就是从普通 DBA 走向牛逼 DBA 的捷径，这本书在淘宝内部非常推崇，但是在中国名气却不是很大，很多人不了解。这本书也是今年夏天刚有中文版本的，非常值得入手以后跟着练习，虽然知道的人不多，豆瓣上也几乎没有什么评价，但是，强烈推荐、吐血推荐！

![数据库索引设计与优化](http://mingxinglai.com/cn/image/mysql-book-4.jpg)

### Effective MySQL 系列

《[Effective MySQL 系列](http://book.douban.com/subject/11653424/)》是指:

*   Effective MySQL Replication Techniques in Depth
*   Effective MySQL 之 SQL 语句最优化
*   Effective MySQL 之备份与恢复

![effective](http://mingxinglai.com/cn/image/mysql-book-5.jpg)

这一系列并不如前面推荐的好，其中，我只看了前两本，这几本书只能算是小册子，如果有时间可以看看，对某一个”模块”进入深入了解。

## MySQL 的源码

关于 MySQL 源码的书非常少，还好现在市面上有两本不错的书，而且刚好一本讲 server 层，一本讲 innodb 存储引擎层，对于学习 MySQL 源码会很有帮助，至少能够更加快速地了解 MySQL 的原理和宏观结构，然后再深入细节。此外，还有一些博客或 PPT 将得也很不错，这里推荐最好的几份材料。

### InnoDB - A journey to the core

《[InnoDB - A journey to the core](https://www.percona.com/live/mysql-conference-2013/sites/default/files/slides/InnoDB%20-%20A%20journey%20to%20the%20core%20-%20PLMCE%202013.pdf)》 是 MySQL 大牛 Jeremy Cole 写的 PPT，介绍 InnoDB 的存储模块，即表空间、区、段、页的格式、记录的格式、槽等等。是学习 Innodb 存储的最好的材料。感谢 Jeremy Cole!

### 深入 MySQL 源码

登博的分享《[深入 MySQL 源码](http://hotpu-meeting.b0.upaiyun.com/2014dtcc/post_pdf/hedengcheng.pdf)》，相信很多想了解 MySQL 源码的朋友已经知道这份 PPT，就不过多介绍，不过，要多说一句，登博的参考资料里列出的几个博客，都要关注一下，干货满满，是学习 MySQL 必须关注的博客。

### 深入理解 MySQL 核心技术

《[深入理解 MySQL 核心技术](http://book.douban.com/subject/4022870/)》是第一本关于 MySQL 源码的书，着重介绍了 MySQL 的 Server 层，重点介绍了宏观架构，对于刚开始学习 MySQL 源码的人，相信会有很大的帮助，我在学习 MySQL 源码的过程中，反复的翻阅了几遍，这本书刚开始看的时候会很痛苦，但是，对于研究 MySQL 源码，非常有帮助，就看你是否需要，如果没有研究 MySQL 源码的决心，这本书应该会被唾弃。

![深入理解MySQL核心技术](http://mingxinglai.com/cn/image/mysql-book-6.jpg)

### MySQL 内核：InnoDB 存储引擎

我们组的同事写的《[MySQL 内核：InnoDB 存储引擎](http://img4.douban.com/lpic/s27266366.jpg)》，可能宇宙范围内这本书就数我学得最认真了，虽然书中有很多编辑错误，但是，平心而论，还是写得非常好的，相对于《深入理解 MySQL 核心技术》，可读性更强一些，建议研究 Innodb 存储引擎的朋友，可以了解一下，先对 Innodb 有一个宏观的概念，对大致原理有一个整体的了解，然后再深入细节，肯定会比自己从头开始研究会快很多，这本书可以帮助你事半功倍。

![MySQL内核](http://mingxinglai.com/cn/image/mysql-book-7.jpg)

### MySQL Internals Manual

《[MySQL Internals Manual](http://dev.mysql.com/doc/internals/en/)》相对于 MySQL Manual 来说，写的太粗糙，谁让人家是官方文档呢，研究 MySQL 源码的时候可以简单地参考一下，但是，还是不要指望文档能够回答你的问题，还需要看代码才行。

[http://dev.mysql.com/doc/internals/en/](http://dev.mysql.com/doc/internals/en/)

### MariaDB 原理与实现

评论里提到的《[MariaDB 原理与实现](https://book.douban.com/subject/26340413/)》我也买了一本，还不错，MariaDB 讲的并不多，重点讲了 Group Commit、线程池和复制的实现，都是 MySQL Server 层的知识，对 MySQL Server 层感兴趣的可以参考一下。

![MariaDB](http://mingxinglai.com/cn/image/mysql-book-8.jpg)

## 后记

希望这里推荐的材料对学习 MySQL 的同学、朋友有所帮助，也欢迎推荐靠谱的学习材料，大家共同进步。

## 参考链接
[推荐几本学习 MySQL 的好书 | 赖明星](http://mingxinglai.com/cn/2015/12/material-of-mysql/)