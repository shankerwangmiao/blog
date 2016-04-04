---
title: 如何禁用 OS X 的图形界面
date: 2016-04-05 02:19:33
tags: 
- 技术 
- 瞎搞
- OS X
---

之前和 [@happyaron](https://github.com/happyaron) 折腾 OS X Server，搞了一台 VBox 的虚拟机，上边安装原版的 OS X。后来突发奇想，因为 OS X Server 有远程客户端可以控制上边的服务，也可以用 ssh 来管理，没有必要启动图形界面。因此，类似于 Linux，应该有办法能让 OS X 不启动 GUI。

<!--more-->

在虚拟机上摸索一阵，发现其实方法很简单，也不需要关闭 SIP（系统完整性保护，又称 rootless），直接执行这几条指令就行：

``` bash
sudo launchctl enable system/com.apple.getty
sudo launchctl disable system/com.apple.WindowServer
sudo launchctl disable system/com.apple.loginwindow
sudo launchctl disable system/com.apple.watchdogd
```

执行完毕后重启，即可发现系统不会继续启动 GUI 了。

如果想恢复，就重新执行上述命令，把 `disable` 换成 `enable` ，再重启一次，就可以了。

一些常用的 OS X 下的命令行工具：

- `launchctl`

  类似于 `systemctl`，用于控制 `launchd`，而后者则是 OS X 上的 init
  
- `systemsetup`

  命令行版的“系统偏好设置”，用于配置系统相关的功能
  
- `networksetup`

  用于配置网络选项。注意，这个工具的效果等效于在“网络偏好设置”中的配置，重启后是会保存的
  
- `reboot` `halt`

  重启和关机
  
以上配置命令在 OS X 10.11.4 上测试通过。
