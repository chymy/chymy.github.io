---
layout:     post
title:      systemd 启动服务报Got notification...问题
subtitle:   systemd启动服务报错
date:       2020-03-10
author:     chymy
header-img: img/post-bg-article.jpg
catalog: true
tags:
    - systemd
    - Linux
---

有时候通过systemd启动服务的时候报：

Got notification message from PID XXXX, but reception only permitted for main PID YYYY

错误。

**启动报上面的错主要是 在ExecStart选项中通过bash启动的服务进程。**



可见服务已经成功完成启动，但是systemd接收到的消息并不是来自主进程从而丢弃消息，并**kill了服务**。

主要是:

当服务单元文件设置了Type=notify标志时，服务将会在启动完成之后通过sd_notify之类的接口发送一个通知消息。systemd 将会在激活服务之前，首先确保该进程已经成功的发送了这个消息。

如果设置Type=notify,那么NotifyAccess=只能设为非 none 值，如果没有明确制定，默认为main. 意味着，systemd只接收来自主进程发送的消息。如果设置为all,表示接受该服务cgroup内所有进程的消息.



所以若**在ExecStart选项中通过bash启动的服务进程，且Type=notify需要设置NotifyAccess=all。**

**或者只设置Type=simple**
