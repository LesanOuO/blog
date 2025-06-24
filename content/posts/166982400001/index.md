---
title: "CentOS7系统安装"
date: "2022-12-01"
tags: ['Linux']
categories: ['动手实践']
---

本片文章记录个人安装CentOS7在台式主机上作为小型服务器

## 制作U盘启动盘

### 1. iso镜像下载

可以通过[阿里云镜像站](https://mirrors.aliyun.com/centos/7/isos/x86_64/)下载iso文件，本人下载的是`CentOS-7-x86_64-DVD-2009.iso`

### 2. 下载启动盘制作软件

可以通过软件[UltraISO](https://cn.ultraiso.net/xiazai.html)进行启动盘制作

- 打开软件 -> 点击左上角`文件` -> 选择`打开` -> 打开对应iso
- 点击菜单栏`启动` -> 选择`写入硬盘镜像`
- 选择对应U盘 -> 其余参数可默认

## 安装CentOS7

### 1. 插入U盘到主机

### 2. 配置 Bios 制作U盘启动

### 3. 选择对应U盘后即可开始安装

### 注意事项

- 通过UltraISO刻入后安装会报错，无法正常安装

1. 选择`Install CentOS 7`时按下TAB键，屏幕下方会出现一串文字
`vmlinuz initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rd.live.check quiet`

2. 将上述文字改为`vmlinuz initrd=initrd.img linux dd quiet`然后键入回车查看设备名称

3. 查看U盘启动盘的名称比如：sda，sdb，sdc。 **ps**：label一列会显示Centos7等字样的

4. 重启后到第一步界面按下TAB键

5. 将`vmlinuz initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rd.live.check quiet` 改为 `vmlinuz initrd=initrd.img inst.stage2=hd:/dev/sda4 quiet` **ps**：sda4就是你看到的启动盘名称

6. 开始正常CentOS 7安装配置