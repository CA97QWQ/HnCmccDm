# HnCmccDm — 荣耀 MagicOS 中国移动 DM 组件反编译分析

**包名** `com.hihonor.regservice` ｜ **APK** `/system/app/HnCmccDm/HnCmccDm.apk` ｜ **分析日期** 2026-11-12

> **Abstract (EN)** — Reverse-engineering notes on `com.hihonor.regservice` (HnCmccDm.apk), the pre-installed China Mobile device-management (DM) client found on an Honor Magic6 Pro (MagicOS 11.0.0 / Android 17). The component is a build of China Mobile's TSdk by Richinfo — the same SDK family as Xiaomi's `com.miui.dmregservice` — reporting to `*.fxltsbl.com` over HTTPS/CoAP, collecting installed-app lists and full-history usage statistics behind 40+ server-controlled switches, and gated at code level to activate only with a China Mobile SIM. **No decompiled code or APK is distributed in this repository**; only an analysis document and short quoted excerpts as research evidence.

> **免责声明**：本仓库仅供安全研究与教育目的。**不包含、也不分发任何反编译代码或 APK 文件**；文中少量代码片段仅作为研究评论证据引用，完整版权归原权利人所有。本分析为非官方独立研究，与荣耀、中国移动、Richinfo 及任何厂商无关联。结论按证据等级标注（A=可复现一手分析 / B=可靠转引 / C=待复核）。

---

## 核心发现（TL;DR）

1. **身份坐实**：HnCmccDm 是**中国移动 TSdk（Richinfo 终端 SDK）**的荣耀 OEM 构建——`cn.richinfo.dm.*` 命名空间、`a/b/m.fxltsbl.com` 端点、`EncryptJsonRequest` 类名（含拼写错误"encyptrequest"）均与 acxun 对小米 `com.miui.dmregservice` 的分析**逐项一致**，两者为同一 SDK 的不同 OEM 集成。【A】
2. **采集面**：预装应用清单（包名/版本/安装时间/**签名 MD5**）、`UsageStatsManager.queryAndAggregateUsageStats(0,…)` **全历史使用统计聚合**、WiFi 列表、流量、蓝牙、电池充电、漫游、设备日志文件、**实名拍照（photoReg）**、公网 IP。【A】
3. **移动卡硬门控**：DM 各路径代码级要求"有卡且 SimType==移动"，非移动卡在报文中以固定假 IMSI 占位（`460111111111111` / `460222222222222`）——不插移动卡则主路径跳过。【A】
4. **服务端编排**：40+ 配置键可由服务器远程下发——各采集模块开关、频率、数据保留期、上报上限，**包括 `baseUrl`（上报目标地址本身）**；应用清单为一次性采集缓存，是否启用取决于服务端开关（客户端无法判断当前状态）。
5. **传输**：HTTPS（`a.fxltsbl.com:443`）+ CoAP/LwM2M（`m.fxltsbl.com:5683`）+ 应用层 AES（**ECB/PKCS5Padding**，密钥在 native 库 `libaes-jni.so`）+ GZIP。传输有 TLS，但应用层选择了与联通 SystemHelper 同款的弱模式 ECB。
6. **同设备电信组件**：`ChnTelecomAutoReg.apk`（com.hihonor.android.AutoRegSms）为**纯短信通道**——数据短信发至 `10659401`（目标号码可被配置覆盖），支持卫星短信（HnSatelliteSmsManager），无任何 HTTP 端点。【A】
7. **同设备 UEInfoCheck**：重度混淆、Manifest 解码不全、无可见端点/采集调用——**结论保持"未知"**，待 baksmali 复核。

## 与联通 SystemHelper 对比（同生态两种实现）

| 维度 | 联通 SystemHelper | 荣耀 HnCmccDm（移动 TSdk） |
|---|---|---|
| 采集范围 | 标识+应用清单+使用+粗位置 | 标识+应用清单+全历史使用统计+WiFi/蓝牙/电池/流量+日志+实名拍照——**不小于前者** |
| 传输 | **明文 HTTP**:18080 | HTTPS + CoAP + AES/GZIP |
| 触发 | 任意大陆卡/WiFi，30 分钟级 | 移动卡硬门控，7 天周期 + 心跳 + 服务端调度 |
| 用户开关 | 无 | 无（但模块级服务端远程开关，40+ 键） |
| 导出组件防护 | RegisterHelperService 无权限守护（有篡改 PoC） | 关键接收器带 `com.dm.permission.THIRD_SEND` 权限保护 |

结论：荣耀版在**工程规范**上明显优于联通版（TLS、卡门控、权限守护），但**采集面设计上限更高、服务端编排更彻底**，且 40+ 开关全部对用户不可见——"规范"不等于"用户可感知/可关闭"。

## 样本与复现

| 文件 | SHA-256 |
|---|---|
| HnCmccDm.apk (1,643,415 B) | `954a5680436c345ba60ef4adc218852486eb6ff2dbd22018c4349eb183eae4d5` |
| ChnTelecomAutoReg.apk (183,783 B) | `bf495bd709d90b0bfad6acd1b27df4f5b31a1f31089a564dddf87e2538177f0d` |
| UEInfoCheck.apk (151,948 B) | `27d1c3b82a2e9f4cafe5aa26a142dc1ae3df4177006b4c9fccaa7327abf1cb0f` |

```bash
# 自有设备拉取（需开启 USB 调试；只读操作）
adb shell pm path com.hihonor.regservice
adb pull /system/app/HnCmccDm/HnCmccDm.apk
# 反编译（jadx >= 1.5）
jadx-gui HnCmccDm.apk
# 定位关键点：全局搜索 "DeviceInfoReader" / "fxltsbl" / "preInstall" / "photoReg"
```

## 仓库结构

```
docs/HnCmccDm-decompile-analysis.zh.md   完整分析报告（中文）
evidence/01-endpoints-and-collection.md  端点全表 + 应用清单/使用统计/加密摘录
evidence/02-gating-and-server-config.md  移动卡门控 + 假IMSI占位 + 服务端40+配置键
evidence/03-components-and-autoregsms.md Manifest组件/权限 + 电信短信通道 + UEInfoCheck
```

## 局限与待验证（如实声明）

- 静态分析不含运行时流量：设备上是否实际发出过报文、服务端开关当前状态，需 PCAPdroid 长周期抓包验证。
- 混淆代码不能 100% 排除未发现的运行时旁路。
- UEInfoCheck 结论为"未知"（非"无风险"也非"有风险"）。

## 相关研究与参考

- SystemHelper（联通版）2023 学术实证：arXiv:2302.01890 *Android OS Privacy Under the Loupe — A Tale from the East*
- 小米版移动 DM 取证：https://github.com/acxun/hyperos-dm-report （§4 com.miui.dmregservice）
- 联通版取证：https://github.com/DreamingCats/com.mobiletools.systemhelper_analysis
- 联通版反编译（注：该仓库声明不得再分发）：https://github.com/t04dJ14n9/xiaomi-unicom-systemhelper-decompiled
- 英文社区汇总：https://xdaforums.com/t/analysis-systemhelper-undocumented-telemetry-covert-screen-capturing-built-in-on-multiple-cn-android-roms.4808230/
- 荣耀包清单佐证：https://github.com/bruceCzK/honor-debloat

## License

文档以 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 发布；引用代码片段版权归原权利人，仅作研究评论之用（见 [LICENSE](LICENSE)）。
