---
title: 日常指令
toc: true
tags: wiki
categories:
  - "\U0001F4BB 工作"
  - Linux
date: 2020-05-23 12:27:56
---
## dos2unix
在进行日常开发的时候，很多人可能用的是 Windows 系统，而在代码运行的开发环境一般是 Linux 环境，直接上传可能文件格式不同，所以需要进行转换。

### 单个文件转换
1. 查看文件格式
使用 vim 打开`set ff`可以看到 dos 或 unix 的字样；
2. 设置文件类型
`set ff=unix`
把它强制设为 unix 格式, 然后存盘退出

### 整个目录转换
下面列出怎么对整个目录中的文件做`dos2unix`操作  
```bash
find . -type f -exec dos2unix {} \;
```

其中具体命令的解释如下：
```plain
find .
= find files in the current directory		查找当前目录
-type f
= of type f		文件类型为f的（文件file）
-exec dos2unix {} \;
= and execute dos2unix on each file found	执行文件转换
```
## 磁盘点灯
改脚本未验证。
```shell
HDDLEDcheck()
{
    sasnum=`sas3flash -list|grep -i "Board Tracer Number"|wc -l`
    if [[ $sasnum -eq 1 ]];then
        enclosu=`sas3ircu 0 display|grep "Enclosure #"|awk NR==1|awk -F ":" '{print $2}'`
        for a in {0..12};
        do
            checksasledstatus ${a} $enclosu
        done
    fi

    radnum=`storcli64 /call show|grep -i "Status = Success"|wc -l`
    if [[ $radnum -eq 1 ]];then
        enclosu=`storcli64 /c0 show|grep -A 3  "EID:Slt DID State DG"|awk NR==3|awk -F ":" '{print $1}'`
        for b in {0..12};
        do
            checkradledstatus ${b} $enclosu
        done
    fi
}
HDDLEDcheck;
```
不管服务器配置什么类型的卡，我们通过上面的脚本可以直接将磁盘定位灯点亮，如果某个磁盘出现故障将会备点亮成红色定位灯，如果正常，就会显示蓝色定位灯。通过点亮定位灯的颜色我们就可以直接找到故障磁盘。