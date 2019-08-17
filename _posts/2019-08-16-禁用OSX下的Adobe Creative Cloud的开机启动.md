---
layout:     post
title:      OSX下禁用Adobe Creative Cloud的开机启动
subtitle:   
date:       2019-08-16
author:     JC
header-img: img/adobe.jpg
catalog: false
tags:
    - Adobe
---

电脑里不装个PS还真不行，Adobe挺讲究，免费赠送安装了Adobe Creative Cloud等程序，启动系统时会跟上一堆Adobe的各种服务，就差全家捅了。看着碍眼，不用时也占系统资源。网上看了一些去除Adobe自启动的方法，有那么费劲吗？算了来干货：

	KingCHdeMacBook-Pro:~ JonathanChoo$ cd /Library/LaunchAgents/
	
	KingCHdeMacBook-Pro:LaunchAgents JonathanChoo$ ls
	
	com.adobe.AAM.Updater-1.0.plist		com.oracle.java.Java-Updater.plist
	com.adobe.AdobeCreativeCloud.plist	com.sogou.SogouServices.plist
	com.adobe.GC.AGM.plist			com.tencent.LemonMonitor.plist
	com.adobe.GC.Invoker-1.0.plist


先来收拾第一个

	sudo vim com.adobe.AAM.Updater-1.0.plist

按i进入vim的编辑模式

把框选的位置改成false

![](http://bbs.pcbeta.com/data/attachment/forum/201811/11/134504zzf3dbl7uflvg7rg.png)

做完之后按ESC键，在英文状态下输入冒号，在键入wq，回车保存。

其余关于adobe相关的plist文件如法炮制。

现在，开机再看不见Adobe Creative Cloud了，进程中也没有一堆堆的Adobe程序了。
