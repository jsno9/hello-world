---
layout: post
title: "android指纹简介"
description:
category: android
tags: [android]
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

![fingerprint view](/images/linux/fingerprintview.png)
