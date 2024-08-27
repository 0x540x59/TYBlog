+++
title = '重置大法'
date = 2024-08-16T16:55:33+08:00
draft = false
tags = ['逆向']
+++

从头刷机就不说了，看别人的博客，已经整理的非常详尽了。

[Pixel2XL解锁BL-刷入Twrp-获取Root权限 - 刘清政 - 博客园 (cnblogs.com)](https://www.cnblogs.com/liuqingzheng/p/17462146.html)



更多可能的场景是：

1. 设备不正常，比如抓不到包了，实在找不到原因就祭出重置大法。

2. 设备被封了，需要重置之后重新生成设备ID。



恢复出厂设置之后，如何快速初始化到工作状态：

1. 开启usb debugging
2. 重新刷入root

这个patched包是刷机的时候通过magisk生成的，保留好，这个时候就能直接复用，不用再来一遍。

**注意: 和机器是对应的，不同机型是不通用的，不能乱刷**。

```shell
adb reboot bootloader
fastboot devices
fastboot boot magisk_patched-26100_FafCW.img

```

3. 安装magisk/socksdroid
4. 安装movecert/charles证书

