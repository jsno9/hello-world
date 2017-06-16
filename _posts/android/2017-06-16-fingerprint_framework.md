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

###2 fingerprint 上报逻辑，以acquired举例

在fingerprintdaemonproxy.cpp中，这里接收到gx_fpd报上来的acquired

	hal_notify_callback()
	{
		switch (msg->type) {
		case FINGERPRINT_ACQUIRED:
            	ALOGD("onAcquired(%d)", msg->data.acquired.acquired_info);
            	callback->onAcquired(device, msg->data.acquired.acquired_info);//通过binder通信发送给fingerprintservice。
            	break;
		}
	}

在IFingerprintDaemonCallback.cpp中，上面函数调用的callback函数如下，transact就是binder的client发送给server端的函数

	virtual status_t onAcquired(int64_t devId, int32_t acquiredInfo) {
        Parcel data, reply;
        data.writeInterfaceToken(IFingerprintDaemonCallback::getInterfaceDescriptor());
        data.writeInt64(devId);
        data.writeInt32(acquiredInfo);
        return remote()->transact(ON_ACQUIRED, data, &reply, IBinder::FLAG_ONEWAY);
    }

在fingerprintservice.java中，收到hal报上来的信息。

	public void onAcquired(final long deviceId, final int acquiredInfo) {
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    handleAcquired(deviceId, acquiredInfo);
                }
            });
        }

	protected void handleAcquired(long deviceId, int acquiredInfo) {
        ClientMonitor client = mCurrentClient;
        if (client != null && client.onAcquired(acquiredInfo)) {//这里会调用到ClientMonitor.java中onAcquired
            removeClient(client);
        }
        if (client != null && !(client instanceof EnrollClient)) {
            if (!inLockoutMode()) {
                handleAcquiredError(acquiredInfo, client);
            }
        }
        if (mPerformanceStats != null && !inLockoutMode()
                && client instanceof AuthenticationClient) {
            // ignore enrollment acquisitions or acquisitions when we're locked out
            mPerformanceStats.acquire++;
        }
    }

在ClientMonitor.java中

	public boolean onAcquired(int acquiredInfo) {
        if (mReceiver == null)
            return true; // client not connected
        try {
            mReceiver.onAcquired(getHalDeviceId(), acquiredInfo);//这里通过binder通信发送给fingerprintmanager
            return false; // acquisition continues...
        } catch (RemoteException e) {
            Slog.w(TAG, "Failed to invoke sendAcquired:", e);
            return true; // client failed
        } finally {
            // Good scans will keep the device awake
            if (acquiredInfo == FingerprintManager.FINGERPRINT_ACQUIRED_GOOD) {
                notifyUserActivity();
            }
        }
    }

在fingerprintmanager中，先通过binder收到发来的数据，再做处理

	public void onAcquired(long deviceId, int acquireInfo) {
            mHandler.obtainMessage(MSG_ACQUIRED, acquireInfo, 0, deviceId).sendToTarget();
        }

	public void handleMessage(android.os.Message msg) {
            switch(msg.what) {
                case MSG_ENROLL_RESULT:
                    sendEnrollResult((Fingerprint) msg.obj, msg.arg1 /* remaining */);
                    break;
                case MSG_ACQUIRED:
                    sendAcquiredResult((Long) msg.obj /* deviceId */, msg.arg1 /* acquire info */);
                    break;
                case MSG_AUTHENTICATION_SUCCEEDED:
                    sendAuthenticatedSucceeded((Fingerprint) msg.obj, msg.arg1 /* userId */);
                    break;
                case MSG_AUTHENTICATION_FAILED:
                    sendAuthenticatedFailed();
                    break;
                case MSG_ERROR:
                    sendErrorResult((Long) msg.obj /* deviceId */, msg.arg1 /* errMsgId */);
                    break;
                case MSG_REMOVED:
                    sendRemovedResult((Long) msg.obj /* deviceId */, msg.arg1 /* fingerId */,
                            msg.arg2 /* groupId */);
            }
        }

	private void sendAcquiredResult(long deviceId, int acquireInfo) {
            if (mAuthenticationCallback != null) {
                mAuthenticationCallback.onAuthenticationAcquired(acquireInfo);//这里发出底层发上来的acquired，其他的process调用到fingerprintmanager的接收到acquired后做相应的处理。
            }
            final String msg = getAcquiredString(acquireInfo);
            if (msg == null) {
                return;
            }
            if (mEnrollmentCallback != null) {
                mEnrollmentCallback.onEnrollmentHelp(acquireInfo, msg);
            } else if (mAuthenticationCallback != null) {
                mAuthenticationCallback.onAuthenticationHelp(acquireInfo, msg);
            }
        }

在keyguardupdatemonitor.java中

	private FingerprintManager.AuthenticationCallback mAuthenticationCallback
            = new AuthenticationCallback()
	{

	public void onAuthenticationAcquired(int acquireInfo) {
            handleFingerprintAcquired(acquireInfo);
        }
	}

	private void handleFingerprintAcquired(int acquireInfo) {
        if (acquireInfo != FingerprintManager.FINGERPRINT_ACQUIRED_GOOD) {
            return;
        }
        for (int i = 0; i < mCallbacks.size(); i++) {
            KeyguardUpdateMonitorCallback cb = mCallbacks.get(i).get();
            if (cb != null) {
                cb.onFingerprintAcquired();//这里的callback函数是在systemui中
            }
        }
    }

在fingerprintunlockcontroller中这里处理就是跟各个apk各自功能有关了

	public void onFingerprintAcquired() {
        Trace.beginSection("FingerprintUnlockController#onFingerprintAcquired");
        releaseFingerprintWakeLock();
        if (!mUpdateMonitor.isDeviceInteractive()) {
            mWakeLock = mPowerManager.newWakeLock(
                    PowerManager.PARTIAL_WAKE_LOCK, FINGERPRINT_WAKE_LOCK_NAME);
            Trace.beginSection("acquiring wake-and-unlock");
            mWakeLock.acquire();
            Trace.endSection();
            if (DEBUG_FP_WAKELOCK) {
                Log.i(TAG, "fingerprint acquired, grabbing fp wakelock");
            }
            mHandler.postDelayed(mReleaseFingerprintWakeLockRunnable,
                    FINGERPRINT_WAKELOCK_TIMEOUT_MS);
            if (mDozeScrimController.isPulsing()) {

                // If we are waking the device up while we are pulsing the clock and the
                // notifications would light up first, creating an unpleasant animation.
                // Defer changing the screen brightness by forcing doze brightness on our window
                // until the clock and the notifications are faded out.
                mStatusBarWindowManager.setForceDozeBrightness(true);
            }
        }
        Trace.endSection();
    }


