---
layout:     post                    # 使用的布局（不需要改）
title:      React-Native ios 真机远程调试              # 标题 
subtitle:    #副标题
date:       2017-11-07              # 时间
author:     Eric                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - react-native
---


# 前言
在日常RN开发中免不了要调用原生组件，比如相机、通讯录等。那我们需要每次写好代码后，用手机连接Xcode安装app调试吗？至少我有某段时间就是这么干的😂，那时候整天吐槽Xcode编译效率极低。直到在网上看到了如此黑科技。
# 开始我们的表演
* 用Xcode打开你RN项目下ios的项目。
* 找到**AppDelegate.m**文件。

> 	NSURL *jsCodeLocation;
> 	
> 	#ifdef DEBUG
> 
> 	    jsCodeLocation = [NSURL URLWithString:@"http://X.X.X.X:8081/index.ios.bundle?platform=ios&dev=true"];
> 	  //jsCodeLocation = [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index.ios" fallbackResource:nil];
> 	#else
> 	  jsCodeLocation = [[NSBundle mainBundle] URLForResource:@"main" withExtension:@"jsbundle"];
> 	#endif

* 将ip地址改成Packager Server所在电脑的ip地址（就是你要调试的电脑ip）
* 当然我们还是需要通过Xcode安装app到手机上。
* 打开安装好的app，打开Debug JS Remotely就可以在电脑Chrome上看到调试信息了。
### 新技能Get！