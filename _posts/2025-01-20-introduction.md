---
title: Introduction
author: Jianhan Liu
date: 2025-01-20
category: index
layout: post
---
# 实验环境

## 实验环境设备

* 自制OS的CPU：Intel 80386
* 模拟80386平台的虚拟机： QEMU&#x20;
* 交叉编译的编译器： GCC 调试工具：GDB&#x20;
* QEMU，GCC，GDB的运行平台： Linux&#x20;
* 编程语言： C，x86 Assembly，Makefile

## 实验环境搭建

在物理机或VirtualBox上安装发行版Linux系统，例如Debian，Ubuntu，Arch，Fedora（x86，arm 架构皆可），在安装完成的Linux上安装QEMU，Vim，GCC，GDB，Binutils，Make，Perl，Git等工 具，以Ubuntu为例

> 建议使用Ubuntu 18.04，这个发行版我们助教测试过没问题！（你要是已经有别的发行版本，例如Ubuntu 20.04或者Ubuntu 22.04等，那不妨先用这个环境）
> 
> * 至于其他版本，我们不清楚生成的image能否在QEMU上运行。遇到问题，你问我，我大概率也回答不上来。
> * 众所周知，近年来越来越多的计算机采用Arm架构，而本实验使用x86指令集架构。理论上只要是符合Unix规范的系统，能够运行QEMU。至于怎么编译出QEMU能够运行的x86代码，那就不是助教能够回答的了。
{: .block-tip}

```bash
sudo apt-get update
sudo apt-get install qemu-system-x86
sudo apt-get install vim
sudo apt-get install gcc
sudo apt-get install gdb
sudo apt-get install binutils
sudo apt-get install make
sudo apt-get install perl
sudo apt-get install git
```

如果有同学使用的是amd64架构，且在代码中使用了标准库，gcc使用`-m32`编译选项时需要进行额外 配置

第一步：确认 64 位架构的内核

```bash
$ dpkg --print-architecture
amd64
```

第二步：确认打开了多架构支持功能

```bash
$ dpkg --print-foreign-architectures
i386
```

说明已打开，否则需要手动打开

```bash
sudo dpkg --add-architecture i386
sudo apt-get update
sudo apt-get dist-upgrade
```

这样就拥有了 64 位系统对 32 位的支持

最后安装gcc multilib

```bash
sudo apt install gcc-multilib g++-multilib
```

> 我们推荐使用Git进行项目版本管理，在整个OS实验过程中，你可能会尝试多个想法实现课程需求，也可能因为一条路走不通选择另外的思路。
> 
> 这时候Git就为你提供了一个在不同版本之间穿梭的机器，也可能成为你调试不通时的**后悔药**。
{: .block-note}

安装好git之后，你需要先进行配置工作

```bash
git config --global user.name "Zhang San" # your name
git config --global user.email "zhangsan@foo.com" # your email
```

现在你可以使用git了，你需要切换到项目目录，然后输入

```bash
git init
...
git add file.c
...
git commit
...
```

这里只是简单的给出一些示例，具体的git使用方法需要同学们自行学习

## 代码运行与调试

利用QEMU模拟 80386 平台，运行自制的操作系统镜像`os.img`

```bash
qemu-system-i386 os.img
```

利用QEMU模拟 80386 平台，Debug自制的操作系统镜像`os.img`，选项`-s`在TCP的 1234 端口运行一个 `gdbserver`，选项`-S`使得QEMU启动时不运行 80386 的CPU

```bash
qemu-system-i386 -s -S os.img
```

**另开一个shell** ，启动GDB，连接上述`gdbserver`，在程序计数器`0x7c00`处添加断点，运行 80386 的 CPU，显示寄存器信息，单步执行下一条指令

```bash
gdb
(gdb) target remote localhost:
...
(gdb) b *0x7c
...
(gdb) continue
...
(gdb) info registers
...
(gdb) si
...
```

实际上在后续的实验中，你可能很难通过程序计数器添加断点，而我们生成的可执行文件也可能没有符号表信息，这时候就需要使用

```bash
(gdb) $ file example # 可执行文件
```

这样就支持使用行号、函数名等方式添加断点，具体内容请参考gdb手册或自行查阅资料

## 简单上手

在这一小节，我们会DIY一个主引导扇区，然后用qemu启动它，并在屏幕上输出"Hello, World!"

在配置好实验环境之后，先建立一个操作系统实验文件夹存放本实验代码

```bash
mkdir os2025
```

进入创建好的文件夹，创建一个`mbr.s`文件

```bash
cd os2025
mkdir index
cd index
touch mbr.s
```

然后将以下内容保存到`mbr.s`中

```armasm
.code16
.global start
start:
    movw %cs, %ax
    movw %ax, %ds
    movw %ax, %es
    movw %ax, %ss
    movw $0x7d00, %ax
    movw %ax, %sp # setting stack pointer to 0x7d00
    pushw $13 # pushing the size to print into stack
    pushw $message # pushing the address of message into stack
    callw displayStr # calling the display function
loop:
    jmp loop

message:
    .string "Hello, World!\n\0"

displayStr:
    pushw %bp
    movw 4(%esp), %ax
    movw %ax, %bp
    movw 6(%esp), %cx
    movw $0x1301, %ax
    movw $0x000c, %bx
    movw $0x0000, %dx
    int $0x10
    popw %bp
    ret
```

接下来使用gcc编译得到的`mbr.s`文件

```bash
gcc -c -m32 mbr.s -o mbr.o
```

文件夹下会多一个`mbr.o`的文件，接下来使用ld进行链接

```bash
ld -m elf_i386 -e start -Ttext 0x7c00 mbr.o -o mbr.elf
```

我们会得到mbr.elf文件，查看一下属性

```bash
...
-rwxrwxr-x 1 ljh ljh 3588 1月  10 17:31 mbr.elf
-rw-rw-r-- 1 ljh ljh  656 1月  10 17:31 mbr.o
-rw-rw-r-- 1 ljh ljh  508 1月  10 17:31 mbr.s
...
```

我们发现mbr.elf的大小有3588byte，这个大小超过了一个扇区，不符合我们的要求

不管是i386还是i386之前的芯片，在加电后的第一条指令都是跳转到BIOS固件进行开机自检，然后将磁盘的主引导扇区（Master Boot Record, MBR ； 0 号柱面， 0 号磁头， 0 号扇区对应的扇区，512 字节，末尾两字节为魔数`0x55`和`0xaa`）加载到`0x7c00`
{: .block-note}

所以我们使用objcopy命令尽量减少mbr程序的大小

```bash
objcopy -S -j .text -O binary mbr.elf mbr.bin
```

再查看，发现mbr.bin的大小小于一个扇区

```bash
$ ls -al mbr.bin
-rwxrwxr-x 1 ljh ljh 65 1月  10 17:31 mbr.bin
```

然后我们需要将这个mbr.bin真正做成一个MBR，新建一个genboot.pl文件

```bash
touch genboot.pl
```

将以下内容保存到文件中

```perl
#!/usr/bin/perl

open(SIG, $ARGV[0]) || die "open $ARGV[0]: $!";

$n = sysread(SIG, $buf, 1000);

if($n > 510){
    print STDERR "ERROR: boot block too large: $n bytes (max 510)\n";
    exit 1;
}

print STDERR "OK: boot block is $n bytes (max 510)\n";

$buf .= "\0" x (510-$n);
$buf .= "\x55\xAA";

open(SIG, ">$ARGV[0]") || die "open >$ARGV[0]: $!";
print SIG $buf;
close SIG;

```

### 给文件可执行权限

```bash
chmod +x genboot.pl
```

然后利用genboot.pl生成一个MBR，再次查看mbr.bin，发现其大小已经为 512 字节了

```bash
$ ./genboot.pl mbr.bin
OK: boot block is 65 bytes (max 510)
$ ls -al mbr.bin
-rwxrwxr-x 1 ljh ljh 512 1月  10 17:31 mbr.bin
```

一个MBR已经制作完成了，接下来就是查看我们的成果

```bash
qemu-system-i386 mbr.bin
```

弹出这样一个窗口

![img1]({{site.baseurl}}/assets/intro-1.png)

## 可能的坑

* 在操作系统实验过程中， 谨慎打开gcc编译优化 。
* 在没有得到允许的情况下， 不能在实验中使用任何C标准库。

# 相关资料

## 相关网站

[维基网站](http://wiki.osdev.org)&#x20;

[问答网站 ](http://stackoverflow.com)

## 相关手册

* Intel 80386 Programmer's Reference Manual
* GCC 4.4.7 Manual
* GDB User Manual
* GNU Make Manual
* System V ABI
* [Linux Manual Page](https://www.man7.org/linux/man-pages/index.html)

# 实验列表

## Lab1-系统引导

实现一个简单的引导程序

## Lab2-系统调用

实现一个简单的系统调用，分化出内核、库和应用

## Lab3-进程管理

此实验始有进程概念和对应的数据结构以及相应的管理与调度，包括idle进程的构造、进程创建、进程内存镜像替换/应用加载、父子进程关系维护等等

未完待续。。

# 作业规范与提交

> ##### 作业规范
>* **学术诚信** : 如果你确实无法完成实验, 你可以选择不提交, 作为学术诚信的奖励, 你将会获得10%的分数; 但若发现抄袭现象, 抄袭双方(或团体)在本次实验中得 0 分.
>* 实验源码提交前需清除编译生成的临时文件（`make clean`），虚拟机镜像等无关文件
>* 请你在实验截止前务必确认你**提交的内容符合要求**(格式, 相关内容等), 你可以下载你提交的内容进行确认. 如果由于你的原因给我们造成了不必要的麻烦, 视情况而定, 在本次实验中你将会被扣除该次实验得分的部分分数, 最高可达50%
>* 实验不接受迟交，一旦迟交按 **学术诚信** 给分
>* 本实验给分最终解释权归助教所有
{: .block-danger}

## 提交格式

### 每次实验的框架代码结构如下

```
labX-STUID             # 修改文件夹名称
├── labX
│   ├── Makefile
│   └── ...
└── report
    └── 231220000.pdf      # 替换为自己的实验报告
```


> ##### 注意！
> 
> * labX中的`X`代表实验序号，如lab1，`labX/`目录存放最终版本的源代码、编译脚本
> * `report/`目录存放实验报告，要求为pdf格式
> * 在提交作业之前先将STUID更改为 自己的学号 ，例如`mv lab1-STUID lab1-231220000`
> * 然后用`zip -r lab1-231220000.zip lab1-231220100` 将`lab1-231220100`文件夹压缩成一个zip 包
> * 压缩包以学号命名，例如`lab1-231220000.zip` 是符合格式要求的压缩包名称
> * 为了防止出现编码问题, 压缩包中的所有文件名都不要包含中文
> * 我们只接受pdf格式, 命名只含学号的实验报告, 不符合格式的实验报告将视为没有提交报告. 例如 `231220000.pdf` 是符合格式要求的实验报告, 但`231220000.docx` 和`231220000张三实验报告.pdf` 不符合要求
> * 作业提交网站 [http://cslabcms.nju.edu.cn](http://cslabcms.nju.edu.cn)
{: .block-danger}

## 实验报告内容


> ##### 你必须在实验报告中描述以下内容
>
> * 姓名、学号、邮箱 等信息, 方便我们及时给你一些反馈信息.
> * 实验进度：简单描述即可，例如"我完成了所有内容"、"我只完成了xxx"。缺少实验进度的描述，或者描述与实际情况不符，将被视为没有完成本次实验.
> * 实验结果：图或说明都可，不需要太复杂，确保不要用其他同学的结果；否则以抄袭处理.
> * 实验修改的代码位置，简单描述为完成本次实验，修改或添加了哪些代码。不需要贴图或一行行解释，大致的文件和函数定位就行.
{: .block-danger }

你可以 **自由选择** 报告的其它内容。你不必详细地描述实验过程，但我们鼓励你在报告中描述如下内容：

* 你遇到的问题和对这些问题的思考
* 对讲义或框架代码中某些思考题的看法
* 或者你的其它想法, 例如实验心得, 对提供帮助的同学的感谢等

> 认真描述实验心得和想法的报告将会获得分数的奖励；思考题选做，完成了也不会得到分数的奖励，但它们是经过精心准备的，可以加深你对某些知识的理解和认识。如果你实在没有想法，你可以提交一份不包含任何想法的报告，我们不会强求。但请**不要**
>
> * 大量粘贴讲义内容&#x20;
> * 大量粘贴代码和贴图, 却没有相应的详细解释(让我们明显看出来是凑字数的)
> 来让你的报告看起来十分丰富，编写和阅读这样的报告毫无任何意义，你也**不会因此获得更多的分数**， 同时还可能带来**扣分**的可能。
{: .block-warning}