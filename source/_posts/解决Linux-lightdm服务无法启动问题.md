---
title: 解决Linux lightdm服务无法启动问题
date: 2023-04-28 17:25:21
tags: [Linux,"Arch Linux","lightdm"]
categories: Linux
---

 博主在玩*archlinux*的时候选择了用*lightdm*做我的登录窗口管理器

```
pacman -S lightdm-webkit2-greeter

systemctl enable lightdm.service
```

安装并开启自启，但是重启启动机子直接卡住，连login界面都没有。于是选择用笔记本ssh上看看，发现可以成功连接上去，于是自然想到是**lightdm服务**出问题了。

于是查看它的状态:

```
systemctl status lightdm.service
```

果然寄了。报错如下：

```
lightdm.service - Light Display Manager
Loaded: loaded (/usr/lib/systemd/system/lightdm.service; enabled; preset: disabled)
Active: failed (Result: exit-code) since Fri 2023-04-28 16:58:15 CST; 17min ago
Docs: man:lightdm(1)
Process: 787 ExecStart=/usr/bin/lightdm (code=exited, status=1/FAILURE)
Main PID: 787 (code=exited, status=1/FAILURE)
CPU: 7ms

4月 28 16:58:15 ArchLinux systemd[1]: lightdm.service: Scheduled restart job, restart counter is at 5.
4月 28 16:58:15 ArchLinux systemd[1]: Stopped Light Display Manager.
4月 28 16:58:15 ArchLinux systemd[1]: lightdm.service: Start request repeated too quickly.
4月 28 16:58:15 ArchLinux systemd[1]: lightdm.service: Failed with result 'exit-code'.
4月 28 16:58:15 ArchLinux systemd[1]: Failed to start Light Display Manager.
```



报错很多，但是没啥有用信息。不过信号我在lightdm的配置中开启了**debug模式**，应该能看到它的具体启动日志。查看`/var/log/lightdm/lightdm.log`文件，果然发现问题：

```
DEBUG: [LightDM] contains unknown option greeter-session
```

大概就是**conf文件被我改错了**，所以它以默认`lightdm-gtk-greeter`的形式启动，但是我又没装这个所以就启动不了了，于是修改为：

```
greeter-session=lightdm-webkit2-greeter
```

重启，成功解决问题
