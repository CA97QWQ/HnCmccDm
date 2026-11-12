# HnCmccDm.apk（com.hihonor.regservice）反编译分析

- **分析日期**：2026-11-12；工具：jadx 1.5.6；声明：本文件为技术分析结论，不含反编译产物本身（版权考虑），复现只需自己拉取 APK + jadx
- **样本**（荣耀 Magic6 Pro BVL-AN16 / MagicOS 11.0.0 / Android 17 实机拉取）：

| APK | 路径 | 大小 | SHA-256 |
|---|---|---|---|
| HnCmccDm.apk | /system/app/HnCmccDm/ | 1,643,415 B | `954a5680436c345ba60ef4adc218852486eb6ff2dbd22018c4349eb183eae4d5` |
| ChnTelecomAutoReg.apk | /version/cust/app/ChnTelecomAutoReg/ | 183,783 B | `bf495bd709d90b0bfad6acd1b27df4f5b31a1f31089a564dddf87e2538177f0d` |
| UEInfoCheck.apk | /version/cust/app/UEInfoCheck/ | 151,948 B | `27d1c3b82a2e9f4cafe5aa26a142dc1ae3df4177006b4c9fccaa7327abf1cb0f` |

---

## 一、身份坐实：中国移动 TSdk（Richinfo 终端 SDK），与小米 dmregservice 同源

- 代码命名空间：`cn.richinfo.dm.*`（DM 主框架）、`cn.richinfo.mt.*` / `com.richinfo.mthttp.*`（MT 传输模块）、`com.dmyk.android.telephony.*`（DM 云卡电话封装）——**Richinfo（深圳国技/亿阳信通系供应商）命名空间与 acxun 对小米 `com.miui.dmregservice` 的分析完全同构**。
- 端点完全一致：`a/b/m.fxltsbl.com`、`auc.fxltsbl.com`（实名拍照）、CoAP `:5683`——连 acxun 在小米版记录的 `EncryptJsonRequest` 类名拼写错误（"encyptrequest" 包名）都在荣耀构建里原样保留。
- **结论**：荣耀 HnCmccDm 与小米 dmregservice 是同一套中国移动 TSdk 的不同 OEM 集成构建【A，本分析一手代码】。

## 二、网络端点全表（cn.richinfo.dm.net.UrlConstant + PublicConstance + mthttp.Constant）

| 常量 | 值 | 含义 |
|---|---|---|
| BASE_URL_RELEASE | `https://a.fxltsbl.com` | 生产上报基址（TLS） |
| BASE_URL_DEBUG | `https://b.fxltsbl.com` | 调试基址 |
| DEFAULT_LWM2M_URL | `coap://m.fxltsbl.com:5683` | CoAP/LwM2M 通道（MT 模块） |
| URL_IP_GET | `https://a.fxltsbl.com/getIpAddr` | 公网 IP 获取 |
| URL_SNAPSHOT_MSG_REPORT_RELEASE | `https://auc.fxltsbl.com/AuthCenter/datareport?func=tsdk:photoReg` | **实名拍照（photoReg）** |
| URL_MSG_PRE_INSTALL | `/preInstalledApp?` | **预装应用清单上报** |
| URL_APP_MSG_REPORT | `/sdkService?func=tsdk:postapplylog` | 应用消息上报 |
| URL_WIFI_LIST_REPORT | `/wifiInfo?` | **WiFi 列表上报** |
| URL_TIMEREALATEDTRAFFICE_MSG_REPORT | `/flowCollectService?func=tsdk:flowcollect` | 流量采集 |
| URL_BLUETOOTH_MSG_REPORT | `/bluetooth?func=tsdk:postBluetoothInfo` | 蓝牙信息 |
| URL_BATTERY_MSG_REPORT | `/batteryCharging?` | 电池充电状态 |
| URL_NETWORK_MSG_REPORT | `/network?` / URL_ROAMING_MSG_REPORT `/internationalRoaming?` | 网络/漫游 |
| URL_REGISTER_MSG_REPORT | `/sdkService?func=tsdk:postreglog` | 注册日志 |
| URL_HEARTBEAT_MSG_REPORT | `/sdkService?func=tsdk:posthblog` | 心跳 |
| URL_LOG_FILE_REPORT / URL_LOG_DIAGNOSIS_REPORT | `/logReport?` / `/errorDiagnosis?` | **日志文件/诊断上报** |
| URL_CONFIG | `/TerminalConfigService/terminalConf?func=tsdk:termimalfullconfig` | 拉取服务端配置 |

## 三、卡门控（代码级，回应"不插移动卡是否激活"）

`com.hihonor.regservice.DmUtils`：

- `getSimType()`：IMSI 前缀 46000/46002/46007/46008→**移动(2)**；46001/46006/46009→联通(1)；46003/46005/46011→电信(3)；其他→未知(4)。
- DM 各路径的统一守卫模式：`getSimState(slot)==1（无卡）|| getSimType(getSubscriberId(slot))!=2` → 直接 return——即 **"有卡且该卡必须是中国移动"** 才执行（第 459/535/575/596/601/679 行多处）。
- **非移动卡占位替换**：对联通卡返回固定假 IMSI `460111111111111`，电信卡 `460222222222222`——上报报文中非移动卡身份以占位符替代。
- **结论**：不插移动卡时，DM 注册/上报主路径被代码跳过（用户推断在代码层证实）；但组件仍响应 BOOT/SIM 变化并保持 7 天闹钟挂载（见 dumpsys 实测），"唤醒但不行动"。**注意**：此门控是静态代码结论，运行时是否存在 WiFi 分支绕过未见——本样本 grep 未发现绕过 SimType 检查的上报路径，但混淆代码不能 100% 排除。

## 四、采集面明细

| 模块 | 代码证据 | 字段/内容 | 触发特征 |
|---|---|---|---|
| **预装应用清单** | `cn.richinfo.dm.a.r` → `getInstalledPackages(0)` → `/preInstalledApp?` | 应用名、包名、版本名、安装时间、**签名 MD5**、appFlags（按服务端开关逐项决定是否采集） | **一次性采集后本地缓存**（"预装应用信息已采集"），受 `DMSDK.isCollectPreinstall()` 服务端开关门控 |
| **应用使用统计** | `cn.richinfo.dm.a.C0020b` → `UsageStatsManager.queryAndAggregateUsageStats(0L, …)` | **从时间 0 起的全历史聚合使用统计**（LOG_TAG_APP） | 同类服务端开关 |
| WiFi 列表 | UrlConstant `/wifiInfo?` | — | 服务端开关 `wifi` |
| 流量/蓝牙/电池/漫游/网络 | UrlConstant 各端点 | — | 各自开关（flowSwitch/bluetooth/batteryCharging/internationalRoaming） |
| 实名拍照 | `photoReg` | snapshot 上报 | **服务端开关** `photoReg`（z.java 解析后存储） |
| 公网 IP | `/getIpAddr` | IP | `ipRate`/`ipReqMax` 限频 |
| 日志文件/诊断 | `/logReport?`、`/errorDiagnosis?`、`configlog` | 设备日志 | `logReport` + logStartTime/EndTime 窗口 |
| 设备标识 | DMDeviceInfoCollection / DmUtils | IMSI（非移动卡占位替换）、双卡 ICCID 变更检测（iccid1/iccid2）、RegType（T/A/R/B/F/S）分类 | — |

## 五、服务端编排（cn.richinfo.dm.a.z 配置键全集，40+ 项）

`preInstalledApp / appInfo / appRate / regRate / regReportMax / ipRate / ipReqMax / baseUrl / photoReg / wifi / bluetooth / batteryCharging / flowSwitch / internationalRoaming / logReport / logStartTime / logEndTime / autoRegTime / autoRegRetryCount / keepAlive / keepAliveRate / sleepSwitch / sleepTime / interval / networkInterval / appDataLifeTime / netDataLifeTime / wifiDataLiftTime / flowDataLifeTime / bluetoothDataLifeTime / batteryDataLifeTime / filter / type / validity / businessInfo / isSdkUpdate / startTime / keepAlive …`

**含义**：各采集模块的开关、频率、数据保留期、上报上限、**甚至 `baseUrl`（上报目标地址）都由服务端配置下发并可覆盖本地默认**——这是与 SystemHelper 同款的"服务器可改上报地址"模式（但本 SDK 的导出面防护更好，见七）。注册重试默认 `DEFAULT_REG_MAX_REPORT = "5#86400000"`（5 次 × 24h 间隔）。

## 六、传输与加密

- 传输层：HTTPS（a.fxltsbl.com:443，acxun 解析 IP 117.134.71.39）+ **CoAP/LwM2M**（m.fxltsbl.com:5683，物联网协议通道）。
- 应用层：`AesUtil`（dm）、`AesCoder`（mt）、`GZIPUtil`——AES + GZIP 管道；请求类 `EncryptJsonRequest / EncryptStringRequest / GZIPJsonRequest`。**未发现 SystemHelper 式明文 HTTP**。
- 密钥管理细节未深挖（可从 AesUtil 常量继续分析）。

## 七、组件与攻击面（AndroidManifest）

- 导出组件：`SimStateChangeReceiver`（exported=true，**无权限保护**，监听 SIM_STATE_CHANGED + BOOT_COMPLETED；两者均为系统 protected broadcast，第三方伪造受限）；`DMBroadCastReceiver`（exported=true，**带 `com.dm.permission.THIRD_SEND` 签名类权限保护**）。
- 其余：DMService、MTService、MTHeartJobService、MTAlarmJobService、VolteStateReceiver、DMApplication。
- **对比 SystemHelper**：SystemHelper 的 `RegisterHelperService` exported 且无权限守护（可被第三方改服务器地址，有 PoC）；HnCmccDm 的关键接收器带权限保护、上报地址覆盖需服务端配置通道——**工程化程度与防护明显更好**。
- Manifest 声明权限含 SEND_SMS，但**源码未发现 sendTextMessage/sendDataMessage 调用**——疑为 SDK 模板遗留；dumpsys 观察到的 SEND_SMS appops 痕迹属于 UID 1001 共享组（同组的电信 AutoRegSms 才是发短信者）。

## 八、ChnTelecomAutoReg.apk（电信短信自注册）分诊

- 133 个源码文件，功能纯粹：**开机校验（IMSI/ESN 一致性）→ `SmsManager.sendDataMessage` 发送数据短信至端口号码 "10659401"**（`com.hihonor.android.AutoRegSms.h` 第 323/345-347 行；`j/g.java` 第 168 行）。
- **支持卫星通道**：`HnSatelliteSmsManager.sendDataMessage`（荣耀卫星通信特性下经卫星发注册短信）。
- **无任何 HTTP 端点**（全源码 grep 无 http/域名）——荣耀版电信链路是纯短信形态，未实现 MTK 规范中的 zzhc.vnet.cn 数据域自注册。
- 权限：SEND/RECEIVE/READ_SMS、MODIFY_PHONE_STATE、BOOT、SATELLITE_COMMUNICATION。

## 九、UEInfoCheck.apk 分诊（不确定性最大，如实声明）

- 42 个源码文件，**重度混淆**（顶层 `a/a.java`～`l.java` + `com.hihonor.android.UEInfoCheck.a`～`h.java`）。
- jadx 提取的 Manifest 中**未解析出任何组件声明**（与 dumpsys 显示其持有 FINE_LOCATION/PACKAGE_USAGE_STATS 授权存在出入）——疑 manifest 解码不完整，需 baksmali 级复核后才能下结论。
- Java 源码中**未发现 HTTP 端点、getInstalledPackages、UsageStatsManager、Location 调用**——若解码无重大遗漏，其行为更接近"本地 UE 信息校验"，但**不能据此断定其无网络行为**。此包结论保持"未知"，不做推断。

## 十、与联通 SystemHelper 对比（同一运营商 DM 生态的两种实现）

| 维度 | 联通 SystemHelper（2026 样本） | 荣耀 HnCmccDm（移动 TSdk，本分析） |
|---|---|---|
| 采集范围 | 标识+应用清单+使用+粗位置+WiFi | 标识+应用清单+全历史使用统计+WiFi+蓝牙+电池+流量+漫游+日志+实名拍照——**不小于前者** |
| 传输 | 明文 HTTP:18080 | **HTTPS + CoAP/LwM2M** + AES/GZIP |
| 触发 | 任意大陆卡/WiFi，30 分钟级 | **移动卡硬门控**（代码级），7 天周期 + 心跳 + 服务端调度 |
| 开关 | 无用户开关、无同意检查 | 无用户开关；但模块级**服务端远程开关**（采集项逐项可配） |
| 上报地址可篡改性 | 导出服务无守护，本地可改（PoC） | 服务端配置通道可下发 baseUrl；导出接收器带权限保护 |
| 应用清单上报 | 持续上报（AppInfo 循环） | 一次性采集缓存 + 服务端开关门控 |

## 十一、结论与待验证

1. **定性**：荣耀的"移动 DM"是中国移动 TSdk 的 OEM 构建，采集面设计上不小于联通 SystemHelper，但传输（TLS）、门控（移动卡硬性）、工程防护（权限守护、服务端编排）更规范。**"规范"不等于"用户可感知/可关闭"**——40+ 个服务端开关全部对用户不可见。
2. **对你的设备（无移动卡）**：DM 主路径代码级被跳过；但 7 天闹钟挂载、SIM/WiFi 监听常驻。要完全坐实"零移动卡流量"，仍建议 PCAPdroid 长周期观察 a.fxltsbl.com / m.fxltsbl.com:5683。
3. **待深挖**：AesUtil 密钥与算法参数；UEInfoCheck 需 baksmali 复核；`/preInstalledApp` 与 UsageStats 模块在服务端开关默认值（出厂时开关是开是关）无法从客户端代码判断——这正是"服务端编排"的透明度问题。
