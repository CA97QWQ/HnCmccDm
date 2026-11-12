# 证据 02：移动卡门控与服务端编排

来源：`com/hihonor/regservice/DmUtils.java`、`cn/richinfo/dm/a/z.java`、`cn/richinfo/dm/DMConfig.java`。

## 1. 运营商类型判定（DmUtils.java 第 85-88/507-517 行）

```java
public static final int IMSI_TYPE_MOBILE = 2;
public static final int IMSI_TYPE_TELECOM = 3;
public static final int IMSI_TYPE_UNICOM = 1;
public static final int IMSI_TYPE_UNKNOWN = 4;

public static int getSimType(String str) {
    ...
    if (str.startsWith("46000") || str.startsWith("46002") || str.startsWith("46007") || str.startsWith("46008")) {
        // -> MOBILE (2)
    if (str.startsWith("46001") || str.startsWith("46006") || str.startsWith("46009")) {
        // -> UNICOM (1)
    return (str.startsWith("46003") || str.startsWith("46005") || str.startsWith("46011")) ? 3 : 4; // TELECOM=3
}
```

## 2. DM 路径守卫模式（DmUtils.java 第 459-462 行，另有 535/575/596/601/679 行同模式）

```java
if (isInvalidPhoneId(i, context) || getSimType(getSubscriberId(i, context)) != 2) {
    return -1;                       // 非移动卡 -> 直接跳过
}
if (getSimState(i, context) == 1) {
    return 0;                        // 无卡 -> 跳过
}
```

## 3. 非移动卡假 IMSI 占位（DmUtils.java 第 130/136/555-560 行）

```java
public static final String TELECOM_IMSI = "460222222222222";
public static final String UNICOM_IMSI = "460111111111111";
...
int simType = getSimType(subscriberId);
if (simType == 1) { return UNICOM_IMSI; }    // 联通卡 -> 固定假 IMSI
if (simType == 3) { return TELECOM_IMSI; }   // 电信卡 -> 固定假 IMSI
if (simType == 2) { return subscriberId; }   // 仅移动卡返回真实 IMSI
```

**含义**：上报报文中非移动卡身份以占位符替代；"不插移动卡，DM 主路径不执行"为代码级结论（混淆代码不能 100% 排除运行时旁路）。

## 4. 服务端编排配置键（cn/richinfo/dm/a/z.java 解析的服务端 JSON 键全集）

```
preInstalledApp / appInfo / appRate / regRate / regReportMax / ipRate / ipReqMax
baseUrl        / photoReg / wifi / bluetooth / batteryCharging / flowSwitch
internationalRoaming / logReport / logStartTime / logEndTime
autoRegTime / autoRegRetryCount / keepAlive / keepAliveRate / sleepSwitch / sleepTime
interval / networkInterval / appDataLifeTime / netDataLifeTime / wifiDataLiftTime
flowDataLifeTime / bluetoothDataLifeTime / batteryDataLifeTime / filter / type
validity / businessInfo / isSdkUpdate / startTime
```

关键点：

- **`baseUrl`**：上报目标地址可由服务端下发覆盖本地默认（`SPUtils` 第 134/138 行默认 `https://a.fxltsbl.com`）。
- **`photoReg`**：实名拍照上报开关（z.java 第 341-342 行解析后存入 SnapshotReportSwitch）。
- 各采集模块（应用/WiFi/蓝牙/电池/流量/日志）均有独立开关与数据保留期（*DataLifeTime）。
- 所有开关对终端用户不可见，当前取值只能经服务端下发后本地观测。

## 5. 重试策略（DMConfig.java）

```java
public static final long DAY = 86400000;
public static final String DEFAULT_REG_MAX_REPORT = "5#86400000";   // 最多 5 次 × 24h 间隔
```

## 6. 定时器实现（cn/richinfo/dm/util/AlarmManagerUtil.java / mthttp/HttpSDK.java）

```java
alarmManager.setExactAndAllowWhileIdle(2, SystemClock.elapsedRealtime() + j2, pendingIntent);
// 心跳：BROADCAST_ALARM_HEART；任务：BROADCAST_JOB_ALARM（MTAlarmJobService）
```

实机 `dumpsys alarm` 观测到两个 `ELAPSED_WAKEUP` 闹钟 origWhen≈604800855ms（7 天）。
