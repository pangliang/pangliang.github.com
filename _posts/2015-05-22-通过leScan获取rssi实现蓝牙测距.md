---
layout : post
categories: [其他]
tags : [小米手环, 蓝牙测距, BLE]
keywords : 
excerpt: 
---
{% include JB/setup %}

领导送了个小米的手环, 看到如果是小米手机, 支持手环免密码解锁手机. 但是现在用的是mx3, android4.4的, 不支持. 于是就想知道, 它怎么实现的

google之, 蓝牙4.0有个叫低功耗`bluetooth.le`的协议, android api里有个`leScan()`方法, 可以对周边的低功耗蓝牙设备进行扫描, 扫描时, 就会返回信号强度`RSSI`的值; 如下面的代码, 就可以每3-5秒钟返回一次结果

```java

private BluetoothAdapter.LeScanCallback	mLeScanCallback = new BluetoothAdapter.LeScanCallback() {
@Override
		public void onLeScan(final BluetoothDevice device, final int rssi,final byte[] scanRecord)
		{
			Log.i(TAG, "name:"+device.getName()
						+",add:"+device.getAddress()
						+",type:"+device.getType()
						+",bondState:"+device.getBondState()
						+",rssi:"+rssi);
		}
};

BluetoothAdapter adapter = BluetoothAdapter.getDefaultAdapter();
adapter.startLeScan(mLeScanCallback);

```

```bash
05-22 12:41:42.450: I/==[shouhuantest]==(6174): name:MI,add:88:0F:10:80:A2:4A,type:2,bondState:12,rssi:-59
05-22 12:41:45.220: I/==[shouhuantest]==(6174): name:MI,add:88:0F:10:80:A2:4A,type:2,bondState:12,rssi:-60
05-22 12:41:46.880: I/==[shouhuantest]==(6174): name:MI,add:88:0F:10:80:A2:4A,type:2,bondState:12,rssi:-61
05-22 12:41:47.985: I/==[shouhuantest]==(6174): name:MI,add:88:0F:10:80:A2:4A,type:2,bondState:12,rssi:-58
05-22 12:41:50.185: I/==[shouhuantest]==(6174): name:MI,add:88:0F:10:80:A2:4A,type:2,bondState:12,rssi:-60
05-22 12:41:51.835: I/==[shouhuantest]==(6174): name:MI,add:88:0F:10:80:A2:4A,type:2,bondState:12,rssi:-68
05-22 12:41:53.495: I/==[shouhuantest]==(6174): name:MI,add:88:0F:10:80:A2:4A,type:2,bondState:12,rssi:-80
05-22 12:41:57.335: I/==[shouhuantest]==(6174): name:MI,add:88:0F:10:80:A2:4A,type:2,bondState:12,rssi:-81
05-22 12:41:58.995: I/==[shouhuantest]==(6174): name:MI,add:88:0F:10:80:A2:4A,type:2,bondState:12,rssi:-75
05-22 12:41:59.775: I/==[shouhuantest]==(6174): name:MI,add:88:0F:10:80:A2:4A,type:2,bondState:12,rssi:-73
05-22 12:42:00.885: I/==[shouhuantest]==(6174): name:MI,add:88:0F:10:80:A2:4A,type:2,bondState:12,rssi:-79
05-22 12:42:02.550: I/==[shouhuantest]==(6174): name:MI,add:88:0F:10:80:A2:4A,type:2,bondState:12,rssi:-77
05-22 12:42:05.310: I/==[shouhuantest]==(6174): name:MI,add:88:0F:10:80:A2:4A,type:2,bondState:12,rssi:-78
05-22 12:42:07.510: I/==[shouhuantest]==(6174): name:MI,add:88:0F:10:80:A2:4A,type:2,bondState:12,rssi:-76
```

把手环贴在手机上, 基本rssi值在-60左右, 离开一米, 差不多-80左右

但是这个结果频率太低了, 3-5秒一次, 你还得取个3-5次做个平均值吧; 那这个延迟用来做解锁太坑了;

android API level 21 也就是android 5.0 还多了一个 `android.bluetooth.le`包, 里面有个`BluetoothLeScanner.startScan(List<ScanFilter> filters, ScanSettings settings, ScanCallback callback)`, 支持在扫描的时候设置过滤器, 和扫描设置什么的;

看到`ScanSettings`还有`getReportDelayMillis()`, 看来还可以设置扫描的频率; 手头没有android5.0, 没法测试了...

## 5月27日更新

之前也不是很清楚蓝牙的api, 其实这个不应该这么来用

`BluetoothGatt`里有个`readRemoteRssi()'函数, 每次读取都会获取新的RSSI值, 所以要监控的话起个线程不停读这个函数就好了

简单使用方法:

```java
//实现个gattcallback

BluetoothGattCallback gattCallback = new BluetoothGattCallback()
{
	@Override
	public void onConnectionStateChange(BluetoothGatt gatt, int status, int newState)
	{
		//设备连接状态改变会回调这个函数
		super.onConnectionStateChange(gatt, status, newState);
		if (newState == BluetoothProfile.STATE_CONNECTED)
		{
			//连接成功, 可以把这个gatt 保存起来, 需要读rssi的时候就
			gatt.readRemoteRssi();
		}
	}
	
	@Override
	public void onReadRemoteRssi(BluetoothGatt gatt, int rssi, int status)
	{
		super.onReadRemoteRssi(gatt, rssi, status);
		if (BluetoothGatt.GATT_SUCCESS == status)
		{
			//读取成功, rssi就是新的值
		} 
	}
}


//开启扫描
BluetoothAdapter adapter = BluetoothAdapter.getDefaultAdapter();
		adapter.startLeScan(new LeScanCallback() {
			@Override
			public void onLeScan(final BluetoothDevice device, final int rssi,
					final byte[] scanRecord)
			{
				//扫到设备, 停止扫描
				BluetoothAdapter adapter = BluetoothAdapter.getDefaultAdapter();
				adapter.stopLeScan(this);
				
				//设备连接上gatt
				device.connectGatt(context, false, gattCallback);
			}
		});
```

