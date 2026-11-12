# 证据 03：Manifest 组件与电信短信通道（AutoRegSms）

来源：`HnCmccDm.apk` AndroidManifest.xml、`jadx_AutoRegSms/sources/com/hihonor/android/AutoRegSms/h.java`、`jadx_UEInfoCheck`。

## 1. HnCmccDm 声明权限（AndroidManifest.xml）

```
ACCESS_COARSE_LOCATION / ACCESS_FINE_LOCATION / ACCESS_NETWORK_STATE / ACCESS_WIFI_STATE
BLUETOOTH / FOREGROUND_SERVICE / INTERNET / PACKAGE_USAGE_STATS / QUERY_ALL_PACKAGES
READ_PHONE_STATE / READ_PRIVILEGED_PHONE_STATE / RECEIVE_BOOT_COMPLETED
SCHEDULE_EXACT_ALARM / USE_CREDENTIALS / WAKE_LOCK / BIND_JOB_SERVICE
```

注：dumpsys 显示的 `ACCESS_UDID / BIND_CARRIER_SERVICES / LOCATION_BYPASS / GET_INSTALLED_APPS` 等为平台/特权允许清单授予，非本 APK Manifest 声明。Manifest 含 `SEND_SMS` 声明，但源码未发现任何 `sendTextMessage/sendDataMessage` 调用（疑为 SDK 模板遗留；实机 UID 1001 共享组内真正发送短信的是电信组件 AutoRegSms）。

## 2. 导出组件（节选）

```xml
<receiver
    android:name="com.hihonor.regservice.SimStateChangeReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.SIM_STATE_CHANGED"/>
        <action android:name="android.intent.action.BOOT_COMPLETED"/>
    </intent-filter>
</receiver>
<receiver
    android:name="cn.richinfo.dm.receiver.DMBroadCastReceiver"
    android:permission="com.dm.permission.THIRD_SEND"
    android:exported="true">
    <intent-filter>
        <action android:name="com.dmyk.android.telephony.action.SIM_STATE_CHANGED"/>
        <action android:name="android.dmyk.net.conn.CONNECTIVITY_CHANGE"/>
        <action android:name="android.dmyk.action.BATTERY_CHARGE_STATE_CHANGE"/>
    </intent-filter>
</receiver>
```

`SimStateChangeReceiver` exported 无权限守护，但其监听的均为系统 protected broadcast，第三方伪造受限；`DMBroadCastReceiver` 带自定义签名类权限保护（优于联通 SystemHelper 无守护的导出服务）。其余组件：DMService、MTService、MTHeartJobService、MTAlarmJobService、VolteStateReceiver。

## 3. 电信短信通道（ChnTelecomAutoReg.apk / com.hihonor.android.AutoRegSms）

声明权限：`SEND_SMS / RECEIVE_SMS / READ_SMS / MODIFY_PHONE_STATE / RECEIVE_BOOT_COMPLETED / SATELLITE_COMMUNICATION`。

发送逻辑（h.java 第 320-347 行，节选）——目标号码可被配置覆盖，默认 **10659401**：

```java
String strX = e.x(this.f134a);
if (strX == null) { strX = "10659401"; }
byte[] bArrA = this.f135b.a();
...
if (MSimTelephonyManager.getDefault().isMultiSimEnabled()) {
    MSimSmsManager.getDefault().sendDataMessage(str, (String) null, (short) 0, bArrA, broadcast, (PendingIntent) null, e.z(e.f125d));
} else {
    SmsManager.getDefault().sendDataMessage(str, null, (short) 0, bArrA, broadcast, null);
}
```

卫星通道（j/g.java 第 168-173 行）：目标同样为 10659401，经 `HnSatelliteSmsManager.sendDataMessage` 发送。

**全源码 grep 无任何 HTTP/HTTPS 端点**——荣耀版电信链路为纯短信形态，未实现 MTK 规范中的数据域自注册（小米平台同功能组件 com.xiaomi.registration 含 zzhc.vnet.cn / typdm.tydevice.com 数据域路径，见 acxun 报告 §8）。

## 4. UEInfoCheck（com.hihonor.android.UEInfoCheck）——结论"未知"

- 42 个源码文件，重度混淆（`a/a.java`～`l.java`、`com/hihonor/android/UEInfoCheck/a`～`h.java`）。
- jadx 提取的 Manifest 未解析出组件声明；Java 源码未见 HTTP 端点、getInstalledPackages、UsageStatsManager、Location 调用。
- 实机 dumpsys 显示其持有 ACCESS_FINE_LOCATION / PACKAGE_USAGE_STATS 授权，与提取 Manifest 存在出入。
- **结论：证据不足，保持"未知"，需 baksmali 复核后才能更新本文件。**
