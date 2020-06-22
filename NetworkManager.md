## NetworkManager中调用org.freedesktop.NetworkManager.Device.Wireless接口的RequestScan的问题，当调用失败时，将不会出发扫描时间，进而监听不到设备状态改变信号
### 目前发现的问题有两个，需要深入调研
1. 提示Error: scanning not allowed while unavailable or activating
通过分析源码可知，出现这种报错的可能原因有：
    1. 当设备状态Enable不为true
    2. 当前模块不为Wireless
    3. 当前设备状态为不可达
    4. 当前设备状态正处于被激活状态
NetworkManager中的源码为：
``` c
	if (   !priv->enabled
	    || !priv->sup_iface
	    || nm_device_get_state (device) < NM_DEVICE_STATE_DISCONNECTED
	    || nm_device_is_activating (device)) {
		g_dbus_method_invocation_return_error_literal (invocation,
		                                               NM_DEVICE_ERROR,
		                                               NM_DEVICE_ERROR_NOT_ALLOWED,
		                                               "Scanning not allowed while unavailable or activating");
		return;
	}
```
为了避免这种情况，在扫描时，可以设置一个定时器扫描，当条件不符合的时间，轮询查看，将扫描事件交给timer去完成，实现逻辑为：
``` go
    deviceNowState, err := dev.nmDev.State().Get(0)
	logger.Debugf("device state now is %v", deviceNowState)
    // device request scan when device is unavailable or in activating may cause scanning failed
	if err != nil || deviceNowState < nm.NM_DEVICE_STATE_DISCONNECTED || isDeviceStateInActivating(deviceNowState) {
		// reset timer when state is unavailable or is in activating, in case causing crash
		dev.requestScanTimer.Reset(2 * time.Second)
		return
	}
	// 蓝牙状态可扫描判断
	func isDeviceStateInActivating(state uint32) bool {
		if state >= nm.NM_DEVICE_STATE_PREPARE && state <= nm.NM_DEVICE_STATE_SECONDARIES {
			return true
		}
		return false
	}
```

2. 提示Error: scanning not allowed immediately following previous scan
通过源码分析可知，出现这种情况的可能性为：
    当两次扫描间隔小于10s时，会出现这种情况
``` c
    last_scan = nm_supplicant_interface_get_last_scan (priv->sup_iface);
	if (last_scan && (nm_utils_get_monotonic_timestamp_ms () - last_scan) < 10 * NM_UTILS_MSEC_PER_SECOND) {
		g_dbus_method_invocation_return_error_literal (invocation,
		                                               NM_DEVICE_ERROR,
		                                               NM_DEVICE_ERROR_NOT_ALLOWED,
		                                               "Scanning not allowed immediately following previous scan");
		return;
	}
``` 
需要注意的是其中的last_scan函数并不是org.freedesktop.NetworkManager.Device.Wireless接口中对外提供的LastScan属性，官方文档有以下说明：
``` c
The RequestScan() method
RequestScan (IN  a{sv} options);
// Request the device to scan. To know when the scan is finished, use the "PropertiesChanged" signal from "org.freedesktop.DBus.Properties" to listen to changes to the "LastScan" property.
// IN a{sv} options:
// Options of scan. Currently 'ssids' option with value of "aay" type is supported.
```
说明LastScan记录的是上次扫描完成的时间而不是开始的时间，而扫描的时间间隔是两次扫描之间的开始时间，所以需要自己定义额外的变量去记录扫描时间，
为了完成这个功能，实现逻辑为：
``` go
	// check if 15 seconds passed when last scan, if not, reset timer to scan later
	timeNow := time.Now().UTC()
	timePass := timeNow.Sub(dev.lastRequestScan)
	logger.Debugf("time pass is %v ", timePass)
	if timePass < NM_REQUESTSCAN_LIMIT_RATE*time.Second {
	    dev.requestScanTimer.Reset(NM_REQUESTSCAN_LIMIT_RATE*time.Second - timePass)
		return
	}
	// record request scan time
	dev.lastRequestScan = time.Now().UTC()
	logger.Debugf("begin to call request scan %v", dev.Path)
	err = dev.nmDev.RequestScan(0, nil)
    if err != nil {
		logger.Warningf("request scan failed, err: %v", err)
	}
	// reset timer when success or failed
	dev.requestScanTimer.Reset(NM_REQUESTSCAN_LIMIT_RATE * time.Second)
``` 
同时为了确保fi.epitest.hostap.WPASupplicant中的/fi/wi/wpa_supplicant/interface/x中的Scanning属性为true时，激活扫描而导致的Error:"Scanning not allowed while already scanning"
应该在每次扫描结束之后再去扫描，由于Scanning属性不易获取，可以根据扫描结束时间LastScan和扫描间隔时间10s去判断，查看LastScan说明为：
``` c
	LastScan  readable   x
	// The timestamp (in CLOCK_BOOTTIME milliseconds) for the last finished network scan. A value of -1 means the device never scanned for access points.
``` 
查看可知，LastScan为CLOCK_BOOTTIME时间，所以判断时间实现为：
``` go
	nowBootTime, err := getClockBootTime()
	bootPassTime := time.Duration(nowBootTime - lastScan*CONVERT_NANOSECONDS_AND_MILLISECONDS)
	if bootPassTime < NM_REQUESTSCAN_LIMIT_RATE {
		logger.Debugf("lastScan time is not enough, start scan in %v seconds", NM_REQUESTSCAN_LIMIT_RATE-bootPassTime)
		timeLeft = NM_REQUESTSCAN_LIMIT_RATE - bootPassTime
	}
```

## 在看com.deepin.daemonNetwork模块代码时，发现GetSecrets并没有被充分利用
``` c
	GetSecrets (IN  a{sa{sv}} connection,
				IN  o         connection_path,
				IN  s         setting_name,
				IN  as        hints,
				IN  u         flags,
				OUT a{sa{sv}} secrets);
```
其中可以根据传的flag值，判断当前密码存储的方式和用户登陆情况
``` c
NM_SECRET_AGENT_GET_SECRETS_FLAG_NONE
NM_SECRET_AGENT_GET_SECRETS_FLAG_ALLOW_INTERACTION
NM_SECRET_AGENT_GET_SECRETS_FLAG_REQUEST_NEW
NM_SECRET_AGENT_GET_SECRETS_FLAG_USER_REQUESTED
NM_SECRET_AGENT_GET_SECRETS_FLAG_WPS_PBC_ACTIVE
NM_SECRET_AGENT_GET_SECRETS_FLAG_ONLY_SYSTEM
NM_SECRET_AGENT_GET_SECRETS_FLAG_NO_ERRORS

其中做常用的有NM_SECRET_AGENT_GET_SECRETS_FLAG_NO_ERRORS NM_SECRET_AGENT_GET_SECRETS_FLAG_ALLOW_INTERACTION NM_SECRET_AGENT_GET_SECRETS_FLAG_REQUEST_NEW NM_SECRET_AGENT_GET_SECRETS_FLAG_WPS_PBC_ACTIVE
NM_SECRET_AGENT_GET_SECRETS_FLAG_ALLOW_INTERACTION，值为0x1：当传了这个值，说明如果在NetworkManager存储中，未知道密码，则允许用户交互，通常为弹窗
NM_SECRET_AGENT_GET_SECRETS_FLAG_REQUEST_NEW，值为0x2：当传这个值，意味这存储或者输入的密码为错误的
NM_SECRET_AGENT_GET_SECRETS_FLAG_WPS_PBC_ACTIVE，值为0x8：用户更适合按路由器的PIN码按钮，而不是输入密码
```



参考资料:
https://developer.gnome.org/NetworkManager/stable/gdbus-org.freedesktop.NetworkManager.Device.Wireless.html
https://phabricator.kde.org/D23578d
https://github.com/NetworkManager/NetworkManager/blob/master/src/devices/nm-device.c
