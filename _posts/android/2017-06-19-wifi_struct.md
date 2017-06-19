---
layout: post
title: "android wifi struct"
description:
category: android
tags: [android, wifi]
mathjax: 
chart:
comments: false
---

###1 android中wifi相关代码路径

wifi driver		: vendor/qcom/opensource/wlan/prima/
wpa_supplicant	:external/wpa_supplicant_8/
				:hardware/qcom/wlan/qcwcn/wpa_supplicant_8_lib
wifi HAL		:hardware/libhardware_legacy/wifi/
netd			:system/netd
wifi service	:frameworks/opt/net/wifi/service/java/com/android/server/wifi

###2 framework层调用到wpa_supplicant全过程

1. 以scan举例

frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiStateMachine.java中

	private boolean startScanNative(int type, String freqs)
	{
		mWifiNative.scan(type, freqs)//private WifiNative mWifiNative;
	}

frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiNative.java中

	public boolean scan(int type, String freqList) {
       doBooleanCommand("SCAN");
	}

	private boolean doBooleanCommand(String command) {
		boolean result = doBooleanCommandNative(mInterfacePrefix + command);//private native boolean doBooleanCommandNative(String command);
	}//之后会调用jni中的函数

在frameworks/opt/net/wifi/service/jni/com_android_server_wifi_WifiNative.cpp中

	static jboolean doBooleanCommand(JNIEnv* env, jstring javaCommand) {
		doCommand(env, javaCommand, reply, sizeof(reply))
	}

	static bool doCommand(JNIEnv* env, jstring javaCommand, char* reply, size_t reply_len) {
		wifi_command(command.c_str(), reply, &reply_len) 
	}

在hardware/libhardware_legacy/wifi.c中

	int wifi_command(const char *command, char *reply, size_t *reply_len)
	{
	    return wifi_send_command(command, reply, reply_len);
	}

	int wifi_send_command(const char *cmd, char *reply, size_t *reply_len)
	{
		wpa_ctrl_request(ctrl_conn, cmd, strlen(cmd), reply, reply_len, NULL);
	}

在external/wpa_supplicant_8/src/common/wpa_ctrl.c中

	int wpa_ctrl_request(struct wpa_ctrl *ctrl, const char *cmd, size_t cmd_len,char *reply, size_t *reply_len,void (*msg_cb)(char *msg, size_t len))
	{
		if (send(ctrl->s, _cmd, _cmd_len, 0) < 0) 
	}

2. 直接下cmd，wpa_cli -p /data/misc/wifi/sockets -i wlan0 scan的调用栈

在wpa_cli.c中，main函数中

	wpa_request(ctrl_conn, argc - optind,&argv[optind]);//（1）
		match->handler(ctrl, argc - 1, &argv[1]);(2)
			static int wpa_cli_cmd_scan(struct wpa_ctrl *ctrl, int argc, char *argv[])(3)
				wpa_cli_cmd(ctrl, "SCAN", 0, argc, argv);
					wpa_ctrl_command(ctrl, buf);
						_wpa_ctrl_command(ctrl, cmd, 1);
							wpa_ctrl_request(ctrl, cmd, os_strlen(cmd), buf, &len,wpa_cli_msg_cb);(4)

+ (1)main函数中把argv传给wpa_request
+ (2)解析传来的参数，调用call back函数call到wpa_cli_cmd_scan
+ (3)是一个callback函数，放在wpa_cli_commands数组中
+ (4)最终调用到wpa_ctrl_request，去与wpa_supplicant沟通

NOTE：无论是android那一路还是wpa_cli直接调用，核心都在wpa_ctrl_request这一函数，通过这个函数向wpa_supplicant发出相应请求

###3 wpa_supplicant数据接收流程，这边接收client发送过来的信息，然后去传给driver去控制hw，用pno举例(代码路径external/wpa_supplicant_8/)

	static void wpa_supplicant_global_ctrl_iface_receive(int sock, void *eloop_ctx,void *sock_ctx)//（1）
		wpa_supplicant_global_ctrl_iface_process(global, buf, &reply_len);//（2）
			wpas_global_ctrl_iface_ifname(struct wpa_global *global,const char *ifname,char *cmd, size_t *resp_len)//（3）
				wpa_supplicant_ctrl_iface_process(struct wpa_supplicant *wpa_s,char *buf, size_t *resp_len)//（4）
					wpa_supplicant_ctrl_iface_set()//（5）
						wpas_ctrl_pno(wpa_s, value);//（6）
							wpas_start_pno(wpa_s);//（7）

+ （1）这一函数是socket的handle，在wpas_global_ctrl_iface_open_sock()这一函数中eloop_register_read_sock(priv->sock,wpa_supplicant_global_ctrl_iface_receive,global, priv);被调用，创建一个socket，注册进eloop，注册进eloop实质就是将这一socket加入到epoll(最后调到这一函数eloop_sock_table_add_sock())中，在wpa_supplicant初始化的时候就已经eloop_run了。
+ （2）,（3）这两个函数在ctrl_iface.c中
+ （4）函数在ctrl_iface.c中，解析传过来的命令，"SET"命令，因此会调用到（5）
+ （5）cmd为pno，会调用（6）
+ （7）在scan.c中

###4 app层到framework调用栈

在packages/apps/AsusSetting/.../WifiConfigManager.java中

	wifiManager.setWifiEnabled(true)；//（1）
		public boolean setWifiEnabled(boolean enabled){mService.setWifiEnabled(enabled);};//(2)wifimanager
			public synchronized boolean setWifiEnabled(boolean enable){mWifiController.sendMessage(CMD_WIFI_TOGGLED);}//(3)wifiservice
				ApStaDisabledState中processMessage的transitionTo(mDeviceActiveState);

###5 wcnss_service.c  		//hardware/qcom/wlan/wcnss-service

	main()
	{
		setup_wlan_config_file();//先去看/data/misc/wifi/WCNSS_qcom_cfg.ini这个文件是不是最新的（与/system/etc/wifi/WCNSS_qcom_cfg.ini相比），是最新的就不更新，不是最新的就更新。
			dynamic_nv_replace();//把/persist/WCNSS_qcom_wlan_nv.bin写到system/etc/wifi/nvbin/<target_board_platform,soc_id>文件中
			fd_dev = open(WCNSS_DEVICE, O_RDWR);
			rc = wcnss_write_cal_data(fd_dev);//把/data/misc/wifi/WCNSS_qcom_wlan_cal.bin写到/dev/wcnss_wlan中
			setup_wlan_driver_ath_prop();
			setup_wifi_version_driver_prop();//再设置一些prop
	}


	setup_wlan_config_file()
	{
		asus_copy_cfg();
		property_set("wlan.driver.config", WLAN_INI_FILE_DEST);
	}

	asus_copy_cfg()
	{
		get_mac_address();//从/factory/wifimac.txt中拿到wifi mac地址
		get_country_code();//从ro.config.versatility节点拿到country code
	}

总结：wcnss_service这一service只是开机跑一遍，为wifi的使用准备好相关的配置文件，即把相关文件copy到相应目录，包括做一些修改。我们在里面加了配置wifi mac地址的函数等。





