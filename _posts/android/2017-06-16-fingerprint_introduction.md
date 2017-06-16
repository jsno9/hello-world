---
layout: post
title: "android指纹简介"
description:
category: android
tags: [android, fingerprint]
mathjax: 
chart:
comments: false
---

###1.指纹识别简介
指纹识别是在Android6.0之后新增的功能，使用指纹的主要有两种场景：

+ 本地使用。即用户本地使用指纹，不需要与远端服务器交互。比如指纹解锁手机。
+ 远端交互。用户在本地完成指纹识别后，需要将指纹信息传给远端服务器判定。比如支付宝微信指纹支付。

由于使用指纹识别功能需要一个加密对象，该对象一般是由对称加密或者非对称加密获得。本地使用指纹识别功能，只需要对称加密。远端交互使用指纹识别功能需要非对称加密，将私钥用于本地指纹识别，识别成功后加密信息传给远端服务器，服务器使用公钥解密，获取用户信息。

###2.指纹使用的软件流程简介

先来看图， 以下是fingerprint流程图：

![fingerprint view](https://raw.githubusercontent.com/jsno9/jsno9.github.io/master/images/android/fingerprintview.png)

+ FingerprintManager(app)可以说是应用层的代理模块。每一个需要用到fingerprint的app都是通过调用FingerprintManager的接口实现相关功能的。它向上提供了preEnroll、enroll、postEnroll、authenticate、getAuthenticatorId等接口，向下注册了回调函数。底层将注册结果，认证结果等回调通知到FingerprintManager，后续有专门文章分析代码流程。
+ FingerprintService(framework)管理整个注册，识别，删除指纹，检查权限等流程的逻辑。
+ Fingerprintd(hal)初始化hal层的fingerprint模块，为FingerprintService提供操作指纹的接口；向hal层注册消息回调函数；向KeystoreService添加认证成功后获取到的auth_token。
+ gx_fpd(hal)负责与driver通信以及与ta的通信，hal层与上层通信使用的是binder，与下层通信使用的是ioctl和netlink，gx_fpd又叫ta的ca。
+ ta(tz)是在trust zone中的app，trust zone是一块独立于cpu的soc，常见的是conrtex-m3的芯片，是负责系统安全的模块，像指纹信息这样的安全度要求很高的信息会被存储在tz中。
+ hw，有汇顶，新思的等公司的模块。

流程图介绍

####2.1.每层代码结构

介绍一下每层代码结构，framwork层详细的代码流程及接口会在后续文章中介绍

1. framework

2. hal

3. kernel



