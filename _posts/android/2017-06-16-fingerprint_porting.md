---
layout: post
title: "android指纹porting and debug"
description:
category: android
tags: [android, fingerprint]
mathjax: 
chart:
comments: false
---

###1. ta的移植(以ifaata为例，都是在此目录下TZ.BF.4.0.5/trustzone_images)

在8953.sh中添加./build.sh CHIPSET=msm8953 ifaata

1. cp -a apps/bsp/trustzone/qsapps/qpay apps/bsp/trustzone/qsapps/ifaata，把其中sconscript中的qpay改成ifaata
2. 在apps/securemsm/trustzone/qsapps/新建ifaata文件夹，在此ifaata下一般包含build,src,inc三个文件夹，build下有sconscript和xx.ld，src下放source code，inc下放lib。（如果是移植的话，厂商给的代码就是放在此目录下）
3. cp -a core/securemsm/trustzone/qsapps/libs/applib/proxy/build/qpay core/securemsm/trustzone/qsapps/libs/applib/proxy/build/ifaata
cp -a core/securemsm/trustzone/qsapps/libs/applib/common_applib/build/qpay core/securemsm/trustzone/qsapps/libs/applib/common_applib/build/ifaata
cp -a core/securemsm/trustzone/qsapps/libs/applib/qsee/build/qpay core/securemsm/trustzone/qsapps/libs/applib/qsee/build/ifaata
cp -a core/securemsm/trustzone/qsee/mink/libstd/build/qpay core/securemsm/trustzone/qsee/mink/libstd/build/ifaata
cp -a core/kernel/libstd/build/qpay core/kernel/libstd/build/ifaata
在core/securemsm/trustzone/qsapps/libs/applib/proxy/build/sconscript中添加ifaata相关
在core/securemsm/trustzone/qsapps/libs/applib/common_applib/build/sconscript中添加ifaata相关
在core/securemsm/trustzone/qsapps/libs/applib/qsee/build/sconscript中添加ifaata相关
在core/securemsm/trustzone/qsee/mink/libstd/build/sconscript中添加ifaata相关
在core/kernel/libstd/build/sconscript中添加ifaata相关
4. 在apps/bsp/trustzone/qsapps/build/secimage.xml添加ifaata相关
5. ./8953.sh即可


###2. driver移植

driver的移植，把厂商给的driver放入kernel/driver/fingerprint/中，添加相关dts。

###3. debug

1. debug初期，init.qcom.rc中加入oneshot，方便调试
service fingerprintd /system/bin/fingerprintd
class late_start
user system
group system
oneshot
service gx_fpd /system/bin/gx_fpd
class late_start
user system
group system
oneshot
2. fingerpint的log，qseelog以及logcat。需要给机台手动烧录persist.img。
3. debug初期，先关闭selinux，流程跑通后，再打开selinux去配置te文件。
