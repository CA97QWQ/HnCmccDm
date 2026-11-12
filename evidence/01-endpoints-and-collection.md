# 证据 01：端点全表与采集面摘录

来源：`jadx 反编译 cn/richinfo/dm/net/UrlConstant.java`、`cn/richinfo/dm/constance/PublicConstance.java`、`com/richinfo/mthttp/Constant.java`、`cn/richinfo/dm/a/r.java`、`cn/richinfo/dm/a/C0020b.java`、`cn/richinfo/dm/util/AesUtil.java`。片段仅作研究评论证据引用。

## 1. 上报基址（PublicConstance.java 第 6-7 行 / Constant.java 第 12-13 行）

```java
public static final String BASE_URL_DEBUG = "https://b.fxltsbl.com";
public static final String BASE_URL_RELEASE = "https://a.fxltsbl.com";
// mthttp 模块：
public static final String DEFAULT_HTTP_URL = "https://a.fxltsbl.com";
public static final String DEFAULT_LWM2M_URL = "coap://m.fxltsbl.com:5683";
```

另：`MtUtils.java` 第 50 行 `MT_HTTP_URL = "https://a.fxltsbl.com/"`；调试 CoAP 为 `coap://shipei.fxltsbl.com:5683`。

## 2. 接口常量全表（UrlConstant.java）

```java
public static final String URL_ACCEPT = "/accept";
public static final String URL_APP_MSG_REPORT = "/sdkService?func=tsdk:postapplylog";
public static final String URL_BATTERY_MSG_REPORT = "/batteryCharging?";
public static final String URL_BLUETOOTH_MSG_REPORT = "/bluetooth?func=tsdk:postBluetoothInfo";
public static final String URL_CONFIG = "/TerminalConfigService/terminalConf?func=tsdk:termimalfullconfig";
public static final String URL_CONFIGLOG_MSG_REPORT = "/sdkService?func=tsdk:configlog";
public static final String URL_HEARTBEAT_MSG_REPORT = "/sdkService?func=tsdk:posthblog";
public static final String URL_IP_GET = "https://a.fxltsbl.com/getIpAddr";
public static final String URL_LOG_DIAGNOSIS_REPORT = "/errorDiagnosis?";
public static final String URL_LOG_FILE_REPORT = "/logReport?";
public static final String URL_MSG_PRE_INSTALL = "/preInstalledApp?";
public static final String URL_NETWORK_MSG_REPORT = "/network?";
public static final String URL_QUERY_SDK_UPDATE = "TerminalConfigService/terminalConf?func=tsdk:checkupdate";
public static final String URL_QUERY_SNAPSHOT_CONFIG = "TerminalConfigService/auth?tsdk:sdkAuth";
public static final String URL_REGISTER_MSG_REPORT = "/sdkService?func=tsdk:postreglog";
public static final String URL_ROAMING_MSG_REPORT = "/internationalRoaming?";
public static final String URL_SNAPSHOT_MSG_REPORT_DEBUG = "sdkphoto/accept?func=tsdk:photoReg";
public static final String URL_SNAPSHOT_MSG_REPORT_RELEASE = "https://auc.fxltsbl.com/AuthCenter/datareport?func=tsdk:photoReg";
public static final String URL_TIMEREALATEDTRAFFICE_MSG_REPORT = "/flowCollectService?func=tsdk:flowcollect";
public static final String URL_WIFI_LIST_REPORT = "/wifiInfo?";
```

## 3. 预装应用清单采集（cn/richinfo/dm/a/r.java，节选）

采集字段按服务端开关逐项决定；一次性采集后写入本地缓存（"预装应用信息已采集"）：

```java
List<PackageInfo> installedPackages = packageManager.getInstalledPackages(0);
boolean zIfCollect  = BusinessUtils.ifCollect(jSONObjectConfigCheck, "appversion");
boolean zIfCollect2 = BusinessUtils.ifCollect(jSONObjectConfigCheck, "installtime");
boolean zIfCollect3 = BusinessUtils.ifCollect(jSONObjectConfigCheck, "signatureinfo");
boolean zIfCollect4 = BusinessUtils.ifCollect(jSONObjectConfigCheck, "appFlag");
// 逐应用：label / packageName / versionName / firstInstallTime / SignUtil.getSignMD5(...) / flags
// 门控：
if (DMSDK.isCollectPreinstall()) { return true; }
LogUtils.collectW(PublicConstance.LOG_TAG_PI,
    BusinessUtils.toggleCloseMsg(PublicConstance.LOG_TAG_PI, "setCollectPreinstall"));
```

## 4. 全历史使用统计聚合（cn/richinfo/dm/a/C0020b.java，第 77/293 行）

```java
Map<String, UsageStats> mapQueryAndAggregateUsageStats =
    ((UsageStatsManager) f24a.getSystemService("usagestats"))
        .queryAndAggregateUsageStats(0L, j);   // 起始时间 0 = 设备以来全部历史
```

同文件第 143/267 行另有 `getInstalledPackages(8197)`（含 GET_META_DATA 等 flags）。

## 5. 应用层加密（cn/richinfo/dm/util/AesUtil.java，节选）

```java
System.loadLibrary("aes-jni");                 // 密钥相关逻辑在 native 库
SecretKeySpec secretKeySpec = new SecretKeySpec(bArr2, "AES");
Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
```

传输层另有 HTTPS（a.fxltsbl.com:443）与 CoAP/LwM2M（m.fxltsbl.com:5683）。请求封装类：`EncryptJsonRequest / EncryptStringRequest / GZIPJsonRequest`（包名 `cn.richinfo.dm.net.encyptrequest`——"encypt" 拼写错误与 acxun 对小米 com.miui.dmregservice 的记录一致，为同源证据之一）。
