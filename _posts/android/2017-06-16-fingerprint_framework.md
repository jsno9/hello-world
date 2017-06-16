---
layout: post
title: "android fingerprint 代码调用栈"
description:
category: android
tags: [android, fingerprint]
mathjax: 
chart:
comments: false
---

###1 framerwork层到底层调用逻辑，注册指纹为例

framerworks/base/core/java/android/hardware/fingerprint/fingerprintmanger.java:  
public void enroll(byte [] token, CancellationSignal cancel, int flags,EnrollmentCallback callback)
中间会调用mService.enroll，调用的是framerworks/base/service/core/java/com/android/server/fingerprint/fingerprintservice.java: 
	public void enroll(final IBinder token, final byte[] cryptoToken, final int groupId,final IFingerprintServiceReceiver receiver, final int flags)
	{
		FingerprintService.this.getEnrolledFingerprints(userId).size();//先去get已经注册的指纹数量，如果超过限制，则停止注册返回
		public void run() {
                    startEnrollment(token, cryptoClone, effectiveGroupId, receiver, flags, restricted);
                }//如果没有超限，则开始注册

	}

	void startEnrollment(IBinder token, byte[] cryptoToken, int groupId,IFingerprintServiceReceiver receiver, int flags, boolean restricted)
	{
		IFingerprintDaemon daemon = getFingerprintDaemon();
		final int result = daemon.enroll(cryptoToken, groupId, timeout);//会调用到hal层fingerprintd中的注册函数，往下看。
	}

system/core/fingerprintd/中FingerprintDaemonProxy.cpp中
	int32_t FingerprintDaemonProxy::enroll(const uint8_t* token, ssize_t tokenSize, int32_t groupId,int32_t timeout)
	{
		mDevice->enroll(mDevice, authToken, groupId, timeout);//调用到各家厂商的hal层代码，goodix就是gx_fpd中的代码。
	}
