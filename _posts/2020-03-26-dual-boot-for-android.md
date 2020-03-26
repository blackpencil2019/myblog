---
layout: post
title: Android双系统思路
categories: [Android]
description: 通过分区法实现Android手机双系统
keywords: 双系统, 分区
---

### 话不多说

给 Android 手机实现双系统能带来很多便利，特别是在手机储存容量越来越大的今天！

#### 基本原理

分区法（类似于电脑装在不同磁盘的双系统）。

#### 工具/条件

* 第三方 Rec
* sgdisk 命令（或其他分区命令，例如 parted 等）
* 解锁 BL

#### 提醒

* 分区操作会删除 userdata 分区所有数据，需要提前备份个人数据；
* 手机需要解BL锁，因此会带来一定安全隐患；
* 操作错误会有变砖风险，确定具备（9008）线刷能力。

### 如何分区

更新 Android 系统时主要是更新 system、vendor 分区的文件，我们通过缩小现有 userdata 分区的空间，来建立安装系统2所需的分区。

系统2一般需要独立的 system、vendor、boot 这三个分区（较早的机型可能没有 vendor 分区；部分机型还需要 cust 分区）

以下命令可以考虑在 ADB Shell 或 Rec 的终端中执行

#### 查看分区

```sh
cat /proc/partitions
```

```shell
ls -l /dev/block/bootdevice/by-name
```

例如 userdata -> /dev/block/sda30

#### 打印分区表

用法：

```sh
sgdisk --print [块设备]
```

例如 userdata 在 sda，我们打印 sda：

```sh
sgdisk --print dev/block/sda
```

#### 删除 userdata 分区

```shell
sgdisk --delete=30 /dev/block/sda
```

用法：

```shell
sgdisk --delete=[分区号] 块设备
```

注：不同闪存芯片块设备路径有所不同，原理同

####  新建分区

```shell
sgdisk --new=0:0:+1gib /dev/block/sda #vendor2
sgdisk --new=0:0:+3gib /dev/block/sda #system2
sgdisk --new=0:0:+128mib /dev/block/sda #boot2
sgdisk --new=0:0:+4gib /dev/block/sda #userdata2
sgdisk --new=0:0:0 /dev/block/sda #userdata
```

用法：

```shell
sgdisk --new=[分区号]:[起始位置]:[结束位置] 块设备
```

注：分区号/地址为0表示第一个可用分区号/地址，起始地址和终止地址可以用偏移量表示，其中偏移量可能不允许小数。系统2的分区大小建议按原系统的大小设置，userdata 分区大小根据实际需要自行设置。

要使新的分区生效，需要重启一次（重启至Rec）

#### 格式化新建的分区

```shell
mke2fs -t ext4 /dev/block/sda31
```

```sh
mkfs.ext4 /dev/block/sda31
```

#### 重命名分区（重要）

Android 系统根据分区名称来引导系统，因此将新建的分区分别重命名为 system、vendor、boot 即可对系统2进行操作（安装系统或启动）。

由于分区名称必须唯一，在将系统2的一系列分区更名前必须先对当前的启动系统——系统1的分区重命名。

用法：

```shell
sgdisk --change-name=[分区号]:[分区名称] 块设备
```

例子：

```shell
sgdisk --change-name=39:boot1 /dev/block/sde
```

#### 切换系统

通过一系列重命名分区操作来实现。可以写成脚本。

### 可能遇到的问题

#### 循环进 Fastboot

可能由于AVB2.0校验导致，需要在Rec中或命令方式关闭AVB2.0校验



