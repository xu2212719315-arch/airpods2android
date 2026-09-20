# AirPods Guard for Huawei — 最高开发规范与执行基线

> **文档状态**：MASTER / Source of Truth  
> **版本**：v1.0  
> **日期**：2026-09-20  
> **目标设备**：HUAWEI Pocket 2，HarmonyOS 4.3  
> **目标耳机**：AirPods 5，优先兼容 AirPods 5 with Wireless Charging Case  
> **开发方式**：ChatGPT 负责总设计、研究、规格、验收与复审；CodeBuddy/WorkBuddy + DeepSeek V4.1 Flash 负责本地执行  
> **当前优先级**：先证明“真实可用的防丢能力”，再做功能扩展和外观  
> **默认原则**：不 Root、不劫持 Apple 账号、不伪造 Find My、不承诺系统实际做不到的能力

---

# 0. 本文档的权威级别

本文件是项目的最高开发依据。

当以下内容与本文件冲突时，以本文件为准：

1. Agent 临时生成的实现方案
2. 代码注释
3. README 中未同步的旧说明
4. 旧 Prompt
5. Agent 对系统能力的猜测
6. 第三方项目的默认实现
7. 为了“让测试通过”而临时改变的业务逻辑

只有以下两种情况可以修改本文件：

- 用户明确要求修改项目目标或标准
- ChatGPT 在完成新的技术验证后，提出修改，并得到用户确认

**执行 Agent 不得自行修改本文件。**

项目中建议固定保存为：

```text
/docs/MASTER_SPEC.md
```

---

# 1. 项目一句话定义

开发一款以 **HUAWEI Pocket 2 + AirPods 5** 为首要目标的 AirPods 伴侣与防丢 App。

它首先解决一个非常具体的真实场景：

> 用户经常只戴一只 AirPod 出门，而另一只耳机和充电盒可能留在别处。App 必须尽可能保护“当前实际佩戴的那一只耳机”，同时不能因为另一只耳机和盒子被故意留在家中而不停误报。

项目不是简单做一个“AirPods 电量弹窗”。

它的核心目标是：

**降低单只 AirPod 在佩戴、摘下、掉落、断连、离开蓝牙范围等情况下真正丢失的概率，并在丢失后提供尽可能有效的找回手段。**

---

# 2. 当前已确认的硬件与用户场景

## 2.1 目标手机

首要测试机：

```text
HUAWEI Pocket 2
HarmonyOS 4.3
```

华为官网当前 Pocket 2 规格页明确列出 HarmonyOS 4.3。

本项目必须以这台设备的真实表现为验收依据，而不是只在 Android 模拟器或其他品牌 Android 手机上运行成功就算通过。

---

## 2.2 目标耳机

当前目标：

```text
AirPods 5
```

优先适配：

```text
AirPods 5 with Wireless Charging Case
```

Apple 2026 年官方规格显示 AirPods 5 使用 H2 芯片和 Bluetooth 5.3。Wireless Charging Case 版本的充电盒带有用于 Find My 的扬声器。

Apple 当前 Find My 文档还确认，对于 AirPods 5 with Wireless Charging Case，Apple 的 Find My 可以把：

```text
左耳
右耳
充电盒
```

分别作为查找对象。

这说明“左耳、右耳、盒子分别管理”是合理的产品模型。

但是，这并不等于第三方 Huawei App 可以调用 Apple Find My API。后文会明确区分。

---

# 3. 最高产品目标

产品按优先级只有四个目标。

## P0 — 防止正在佩戴的单耳丢失

这是整个项目最重要的目标。

典型场景：

```text
左耳戴着
右耳 + 盒子留在宿舍

用户出门
```

正确行为：

```text
不因为右耳和盒子远离而报警
重点保护左耳
```

如果左耳：

- 突然从耳中取出
- 意外掉落
- 蓝牙断开
- 信号快速变弱
- 长时间没有重新出现

App 应该尽快进行提醒，并保存最后可信位置。

---

## P1 — 丢失后的近距离寻找

当耳机可能掉在：

- 房间
- 教室
- 商场
- 路边
- 沙发附近
- 包内

App 尽可能通过 BLE 信号和连接状态帮助用户判断：

```text
没有信号
很弱
较弱
中等
较强
很强
```

不得在只有 RSSI 的情况下伪装成“距离 1.3 米”这种虚假精度。

---

## P2 — 记录“最后一次可信出现位置”

当目标耳机由可见变为不可见时，记录：

- 时间
- 手机位置
- 当时连接状态
- 信号强度
- 当时耳机状态
- 哪一个组件最后被观察到
- 证据来源

后续在地图中展示。

---

## P3 — 与 Apple Find My 形成合法的辅助关系

Apple 官方目前允许通过：

```text
icloud.com/find
```

在网页上定位 AirPods。

因此本项目可以提供：

```text
打开 Apple Find My
```

并由系统浏览器进入 Apple 官方页面。

**本项目禁止：**

- 在 App 内保存 Apple ID 密码
- 抓取 Apple Find My 私有接口
- 模拟 Apple 登录
- 绕过双重验证
- 通过非官方接口读取 Find My 位置
- 声称已经把 Huawei 定位网络接入 Apple Find My 网络

当前可做的是：

```text
华为本地防丢能力
+
Apple 官方网页 Find My 入口
```

而不是伪造一个不存在的统一网络 API。

---

# 4. 项目的真实能力边界

这一节必须被所有 Agent 阅读。

## 4.1 已确定可做

### A. 标准蓝牙连接状态

可以获取 AirPods 与手机整体的：

- 已连接
- 正在连接
- 已断开
- 蓝牙状态变化

### B. Android/HarmonyOS 设备本身的位置

可以在用户授权后获取手机当前位置。

因此可以实现：

```text
AirPods 最后一次可见
+
此刻手机位置
=
Last Seen
```

### C. BLE 扫描

Android 标准 BLE API 支持扫描附近设备。

Android 12 及以后需要：

```text
BLUETOOTH_SCAN
BLUETOOTH_CONNECT
```

如果扫描结果被用于推断位置，还需要相应的位置权限。

### D. 后台提醒

可以通过：

- Companion Device 能力
- Foreground Service
- 系统通知
- 华为后台保活设置

组合实现。

但 Huawei 的实际后台策略必须真机测试。

### E. Apple Find My 网页入口

Apple 官方确认：

```text
iCloud.com/find
```

可以定位 AirPods。

AirPods 5 with Wireless Charging Case 还可以在 Apple Find My 中分别看到左耳、右耳和充电盒的位置。

---

## 4.2 有可能做，但必须先验证

以下功能绝不能先写进“已完成能力”。

### A. 读取左耳、右耳、盒子的独立电量

历史 AirPods BLE 广播协议中存在：

```text
Left battery
Right battery
Case battery
```

第三方研究项目也证明这些字段在部分 AirPods 和固件中能够读取。

但是：

- 新型号协议可能变化
- 新固件可能加密部分字段
- AirPods 5 是 2026 新型号
- Huawei 蓝牙栈可能表现不同

因此必须通过实机原始广播包证明。

### B. 判断左耳 / 右耳是否在耳中

历史 Apple Proximity / AAP 协议包含相关状态。

但是必须测试 AirPods 5。

### C. 判断左耳 / 右耳是否在盒中

同上。

### D. 单独扫描某一只 AirPod 的 RSSI

这对本项目价值很高。

但 AirPods 对手机呈现的蓝牙身份、地址轮换和广播行为可能让“左耳和右耳分别追踪”变得困难。

在完成 Spike 前，不承诺能够独立得到：

```text
left RSSI
right RSSI
case RSSI
```

### E. AAP/AACP 私有协议连接

LibrePods 等项目公开记录了 AirPods 的 Apple Accessory Protocol。

例如研究资料中出现：

```text
L2CAP PSM 0x1001
```

并能获得：

- battery
- ear detection
- noise mode
- conversational awareness

但是 LibrePods 自己也说明，很多 Android 版本上完整功能需要 Root / Xposed 或系统层能力。

因此：

**HUAWEI Pocket 2 不 Root 是本项目默认条件。**

AAP 只能作为实验分支。

如果公共 Android API 无法建立所需 L2CAP 通道，就放弃这条路径，不要求用户 Root。

---

## 4.3 当前不能作为本项目能力承诺

### A. 直接调用 Apple Find My 网络

Apple 公共开发者文档提供的是：

- Find My accessory program
- MFi 体系
- 让第三方硬件加入 Apple Find My 网络

这不是给第三方 Android App 查询用户 AirPods 位置的公共 API。

因此 V1 不存在：

```text
Huawei App
直接请求 Apple Find My Server
返回 AirPods 坐标
```

### B. 在 Huawei 上复刻 Apple 精确查找

Apple Nearby Interaction 是 Apple 平台框架，并面向兼容 Apple 设备或经过配套协议的第三方硬件。

本项目不能假设 Huawei 手机能调用 AirPods 的 Apple UWB 精确查找能力。

### C. 建立一个真正等价于全球 Find My 的“自建网络”

如果只有一个用户安装 App，自建众包网络没有实际覆盖价值。

因此 V1 不做云端众包定位。

---

# 5. 不 Root 原则

除非用户未来明确改变要求，否则：

```text
NO ROOT
NO XPOSED
NO MAGISK
```

原因：

1. 目标手机是日常主力手机
2. Root 可能影响系统安全和支付应用
3. 会显著提高维护难度
4. 不利于将来推广到其他 Android/Huawei 设备
5. 防丢 App 不应该以系统完整性为代价

第三方项目如果依赖 Root，只允许作为研究参考。

---

# 6. 产品模式设计

App 必须支持三种保护模式。

## 6.1 Wear-One 模式

默认重点模式。

场景：

```text
一只耳机戴在耳中
另一只 + 盒子在别处
```

行为：

```text
正在佩戴的耳机 = Active Protected Component

另一只耳机 = Expected Separated
充电盒 = Expected Separated
```

此时不得因为：

```text
右耳不在附近
盒子不在附近
```

产生高优先级报警。

---

## 6.2 Full-Kit 模式

适合：

```text
两只耳机 + 盒子一起携带
```

此时任何一个组件被确认遗留，都可以触发提醒。

---

## 6.3 Manual Watch 模式

用户可以手动指定：

```text
保护左耳
保护右耳
保护盒子
```

主要用于调试和极端场景。

---

# 7. 最重要的功能 — Ear Drop Guard

如果我们能够可靠获取“耳机在耳 / 出耳”状态，这将成为本项目最有价值的功能。

## 7.1 理想状态机

```text
IN_EAR
  |
  | detected out of ear
  v
OUT_OF_EAR_GRACE
  |
  | 3~8 秒内未恢复
  v
DROP_SUSPECTED
  |
  | 同时信号下降 / 断连
  v
LOSS_RISK_HIGH
```

如果在 Grace Period 内：

```text
重新 IN_EAR
```

或：

```text
IN_CASE
```

则取消警告。

---

## 7.2 为什么它比普通断连提醒更重要

耳机掉到地面时：

```text
耳机可能仍然保持蓝牙连接
```

如果只等待 Bluetooth Disconnected：

```text
用户可能已经走出几十米
```

因此：

```text
IN_EAR -> OUT_OF_EAR
```

是更早的风险信号。

---

## 7.3 如果 AirPods 5 的单耳状态无法无 Root 获取

则 V1 降级为：

```text
Bluetooth disconnect
+
BLE disappearance
+
Last Seen
+
快速提醒
```

不允许为了“功能完整”而使用假的 ear state。

---

# 8. 组件状态模型

内部必须从第一天就把 AirPods 建模为三个组件。

```kotlin
enum class AirPodsComponent {
    LEFT,
    RIGHT,
    CASE
}
```

每个组件状态示例：

```kotlin
enum class ComponentPresence {
    UNKNOWN,
    IN_EAR,
    OUT_OF_EAR_NEARBY,
    IN_CASE,
    CONNECTED,
    SEEN_BY_BLE,
    MISSING_SUSPECTED,
    MISSING_CONFIRMED
}
```

注意：

这些不是简单互斥的物理事实。

实际代码应该使用：

```text
Observation
+
Reducer
+
Derived State
```

而不是让多个 Service 随意修改一个全局布尔值。

---

# 9. Observation 事件体系

任何状态变化都必须来自可追溯 Observation。

建议统一模型：

```kotlin
data class DeviceObservation(
    val timestamp: Instant,
    val component: AirPodsComponent?,
    val source: ObservationSource,
    val type: ObservationType,
    val rssi: Int?,
    val batteryPercent: Int?,
    val rawPayloadHash: String?,
    val confidence: Confidence,
    val metadata: Map<String, String>
)
```

---

## 9.1 ObservationSource

至少包含：

```text
BT_CLASSIC_CONNECTION
BLE_ADVERTISEMENT
AAP_NOTIFICATION
SYSTEM_AUDIO_ROUTE
USER_ACTION
LOCATION
APPLE_WEB_HANDOFF
```

---

## 9.2 Confidence

只允许：

```text
HIGH
MEDIUM
LOW
UNKNOWN
```

不得用虚假的：

```text
93.7% confidence
```

除非以后有真实统计模型支持。

---

# 10. “最后出现位置”标准

Last Seen 不能简单理解为：

```text
最后一次 BLE callback 的 GPS
```

需要记录“证据快照”。

建议：

```kotlin
data class LastSeenSnapshot(
    val component: AirPodsComponent?,
    val timestamp: Instant,
    val latitude: Double?,
    val longitude: Double?,
    val horizontalAccuracyMeters: Float?,
    val rssi: Int?,
    val connectionState: String?,
    val presenceState: String?,
    val evidenceSource: ObservationSource,
    val confidence: Confidence
)
```

---

## 10.1 Last Seen 写入条件

只在满足以下任一条件时更新：

- 高可信连接仍存在
- BLE 广播明确来自目标 AirPods
- AAP 明确报告某组件存在
- 用户主动确认“耳机就在这里”

不要让：

```text
随机 Apple 设备 BLE 广播
```

覆盖用户自己的 AirPods Last Seen。

---

# 11. 防丢状态机

V1 不采用复杂机器学习。

使用可解释的状态机。

推荐状态：

```text
SAFE
SUSPECTED
SEARCHING
ALERTED
RECOVERED
LOST
```

---

## 11.1 SAFE

满足任一：

- 被保护组件稳定连接
- 被保护组件可靠可见
- 被保护组件明确在盒中且当前策略允许

---

## 11.2 SUSPECTED

触发条件示例：

- 刚刚断连
- RSSI 连续降低
- 被保护耳机刚刚出耳
- BLE 广播突然消失

此阶段：

```text
启动短时间高强度扫描
更新 Last Seen
```

先不一定立刻大声报警。

---

## 11.3 ALERTED

确认风险较高后：

- 强通知
- 震动
- 可选声音
- 锁屏可见
- 显示最后位置
- 显示“立即寻找”

---

## 11.4 RECOVERED

重新检测到目标后：

- 自动结束风险状态
- 保存一次 recovery event
- 提示已找回

---

# 12. 防误报是核心指标

这个项目如果：

```text
每天误报 20 次
```

用户很快会关闭保护功能。

因此“召回率”和“误报率”必须一起考虑。

必须特别避免：

### 正常动作误报

- 用户主动摘下一只耳机
- 把耳机放进盒子
- 留一只耳机和盒子在家
- 短暂切换蓝牙
- 手机重启
- 飞行模式
- 耳机电量耗尽
- App 被系统杀死

---

# 13. 近距离寻找 Finder

Finder 是 V1 第二核心功能。

## 13.1 禁止假距离

BLE RSSI 受以下因素影响非常大：

- 人体遮挡
- 墙体
- 手机方向
- 耳机方向
- 反射
- 干扰
- 口袋
- 包

因此默认 UI：

```text
未发现
极弱
弱
中
强
很强
```

而不是：

```text
距离 1.7m
```

---

## 13.2 RSSI 平滑

允许实现：

```text
Median window
+
EMA
```

例如：

```text
最近 5 个样本取中值
再做指数平滑
```

具体参数必须通过真机实验确定。

参数必须放在：

```text
config/FinderTuning.kt
```

或等价配置中。

禁止散落 magic numbers。

---

## 13.3 Finder UI

最低要求：

```text
目标组件
当前是否检测到
信号等级
近 10 秒信号趋势
最后检测时间
开始/停止扫描
```

不要第一版做炫酷雷达动画。

---

# 14. Apple Find My 辅助入口

这是本项目非常重要但必须诚实实现的功能。

## 14.1 入口

按钮：

```text
使用 Apple Find My 查找
```

行为：

```text
打开系统默认浏览器
进入 https://icloud.com/find
```

---

## 14.2 安全规则

App：

```text
不接收 Apple 密码
不存储 Apple Cookie
不注入脚本
不解析 iCloud 页面
```

登录完全在 Apple 官方页面完成。

---

## 14.3 当前 Apple 官方能力

Apple 官方 AirPods 使用手册说明：

AirPods 5 with Wireless Charging Case 可以在 Find My 中分别显示：

```text
左耳
右耳
盒子
```

并且可以在 iCloud.com/find 上找到 AirPods。

但是 Apple 官方同时注明：

```text
iCloud.com 上不能对 AirPods 5 Wireless Charging Case 的盒子播放声音
```

所以 App UI 不允许写：

```text
“网页可让盒子响铃”
```

---

# 15. 首版技术路线

## 15.1 平台策略

**Phase 1 先做 Android APK。**

原因：

- HUAWEI Pocket 2 当前为 HarmonyOS 4.3
- 优先验证 Android 蓝牙 API 在目标机上的实际表现
- 开发速度快
- 将来容易扩展到其他 Android 设备
- 不依赖 Google Play Services

但是：

**第一阶段必须先验证 APK、Bluetooth API、后台行为在 Pocket 2 上实际可用。**

如果 HarmonyOS 4.3 对关键 API 的兼容性阻塞项目，再决定是否做 ArkTS 原生 HarmonyOS 分支。

不要一开始同时维护：

```text
Android
+
HarmonyOS Native
```

---

# 16. Android 技术栈

建议：

```text
Language: Kotlin
UI: Jetpack Compose
Architecture: MVVM + unidirectional state
Async: Kotlin Coroutines + Flow
DI: 手工 DI，首版不引入 Hilt
Persistence: Room
Settings: DataStore
Bluetooth: Android Bluetooth APIs
Location: Android LocationManager 首选
Map: 后期接 Huawei Map Kit 或系统地图 Intent
Logging: structured local logs
Testing: JUnit + instrumentation + 真机场景测试
Build: Gradle Kotlin DSL
```

---

# 17. 为什么首版不使用 Google Play Services

目标机是 Huawei。

因此核心能力禁止依赖：

```text
Google Play Services
Google FusedLocationProvider
Firebase 作为核心依赖
```

位置优先使用：

```text
Android LocationManager
```

如果后续需要更好的 Huawei 体验，可以增加：

```text
Huawei Location Kit
Huawei Map Kit
```

但必须放在 adapter 层。

---

# 18. 推荐模块架构

```text
app/
core-model/
core-domain/
bluetooth/
location/
protection/
finder/
storage/
ui/
diagnostics/
```

如果首版不希望 Gradle 模块太多，也允许先做单 module，但 package 必须按上面的边界分层。

---

## 18.1 建议包结构

```text
com.example.airpodsguard

├── app
├── bluetooth
│   ├── classic
│   ├── ble
│   ├── parser
│   └── aap
├── domain
│   ├── model
│   ├── reducer
│   └── usecase
├── protection
│   ├── engine
│   ├── policy
│   └── alert
├── finder
├── location
├── storage
├── diagnostics
└── ui
```

---

# 19. 蓝牙层设计

蓝牙层必须分成四层。

## 19.1 Classic Connection Monitor

只做：

- AirPods 是否已配对
- 连接 / 断连
- audio profile 状态
- 设备名称和 address 等标准元数据

---

## 19.2 BLE Scanner

只做：

- 扫描
- 过滤
- RSSI
- raw manufacturer data
- timestamp

**BLE Scanner 不负责业务报警。**

---

## 19.3 Apple Advertisement Parser

输入：

```text
raw manufacturer data
```

输出：

```text
ParsedAirPodsAdvertisement?
```

不得让 scanner 直接包含大量 Apple 协议解析逻辑。

---

## 19.4 AAP Experimental Adapter

单独放：

```text
bluetooth/aap/experimental
```

默认 feature flag 关闭。

如果不 Root 无法工作：

```text
保留实验代码
不进入 production path
```

---

# 20. BLE 识别策略

Apple company identifier：

```text
0x004C
```

但：

```text
0x004C
```

代表 Apple，不代表一定是 AirPods。

因此不能只根据 manufacturer id 判断目标设备。

必须结合：

- paired device
- advertisement payload pattern
- model bytes
- observed address
- historical identity
- 用户确认

建立 TargetIdentity。

---

# 21. TargetIdentity

建议模型：

```kotlin
data class TargetIdentity(
    val bluetoothAddress: String?,
    val bluetoothName: String?,
    val modelHint: String?,
    val knownManufacturerPayloadSignature: ByteArray?,
    val firstConfirmedAt: Instant,
    val userConfirmed: Boolean
)
```

如果 AirPods 使用随机地址：

```text
MAC address 不得作为唯一身份依据
```

---

# 22. 后台运行策略

这个部分必须保守设计。

## 22.1 不做 24 小时无限高频 BLE 扫描

原因：

- 耗电
- 系统节流
- HarmonyOS 后台限制
- 误报增加

使用分级扫描。

---

## 22.2 NORMAL

平时：

- 监听标准蓝牙连接事件
- 低频或系统管理的 presence
- 不持续高功耗扫描

---

## 22.3 WATCH

当正在佩戴 AirPods 时：

- Protection Service 激活
- 保持必要监听
- 可使用 connectedDevice 类型 Foreground Service

---

## 22.4 SUSPECTED

一旦检测：

```text
断连
出耳异常
BLE 消失
```

短时间提升扫描强度。

例如初始建议：

```text
15~30 秒
```

最终值通过测试决定。

---

# 23. Android Companion Device Manager

Android 官方 CompanionDeviceManager 可以：

- 关联 companion device
- 监听设备 presence
- 在某些情况下帮助 App 在后台被系统唤起

必须做 capability probe：

```text
PackageManager.FEATURE_COMPANION_DEVICE_SETUP
```

如果 Pocket 2 支持：

优先评估它是否能降低我们长期后台 Service 的需求。

如果不支持：

降级到常规 Bluetooth + Foreground Service。

---

# 24. Foreground Service 标准

需要持续保护时，可使用：

```text
foregroundServiceType="connectedDevice"
```

如果持续获取位置：

```text
foregroundServiceType="location"
```

但必须遵守：

- 用户主动开启保护
- 显示持续通知
- 不偷偷后台启动
- 只在需要时请求权限

---

# 25. Huawei 后台保活测试

必须测试：

1. 屏幕亮
2. 屏幕灭
3. 锁屏 5 分钟
4. 锁屏 30 分钟
5. 锁屏 2 小时
6. App 从最近任务划走
7. 省电模式
8. 超级省电模式
9. 手机重启
10. 蓝牙关闭再开启

每项记录：

```text
是否继续监听
是否收到断连
是否触发通知
是否保存 Last Seen
是否恢复
```

---

# 26. 权限最小化

## 26.1 Bluetooth

Android 12+：

```text
BLUETOOTH_SCAN
BLUETOOTH_CONNECT
```

如果确实需要 advertise 才申请：

```text
BLUETOOTH_ADVERTISE
```

V1 大概率不需要。

---

## 26.2 Location

只有启用：

```text
Last Seen
```

时才请求。

位置权限要解释：

> 用于在耳机失联时记录“手机当时所在位置”，帮助找回耳机。

---

## 26.3 Background Location

**默认不立即请求。**

先证明：

```text
foreground service
```

能满足保护场景。

只有真实场景必须后台持续记录定位时，再评估 Background Location。

---

# 27. 数据隐私

位置是敏感数据。

V1 默认：

```text
Local First
No account
No analytics SDK
No ad SDK
No cloud upload
```

数据库只保留：

- Last Seen
- protection events
- diagnosis logs

---

## 27.1 数据保留

默认：

```text
Last Seen：保留最近 30 天
raw BLE debug：开发版最多 7 天
普通运行日志：最多 7 天
```

用户可以：

```text
清除历史位置
清除日志
清除已配对设备资料
```

---

# 28. 日志不能包含

禁止日志记录：

- Apple ID
- Apple 密码
- OAuth token
- Cookie
- 其他敏感认证信息

Bluetooth address 如需导出用于 Issue，应提供脱敏选项。

---

# 29. UI 信息架构

首版只需要五个页面。

## 29.1 Home

显示：

```text
AirPods 5
连接状态

左耳
右耳
盒子

保护状态
最后出现
```

如果状态无法确定：

必须显示：

```text
未知
```

禁止猜测。

---

## 29.2 Protection

显示：

```text
Wear-One
Full-Kit
Manual Watch
```

当前 protected component 要非常明显。

---

## 29.3 Find Nearby

显示：

- 当前目标
- Signal level
- 最近一次检测
- 信号趋势
- Start / Stop

---

## 29.4 Last Seen

显示：

- 时间
- 地点
- 精度
- 状态
- 导航
- 打开 Apple Find My

---

## 29.5 Diagnostics

这是开发期间必须有的页面。

显示：

```text
Bluetooth adapter
Paired device
Classic connection
BLE permissions
Location permission
Background status
Battery optimization
Raw Apple advertisements
Parser result
Component state
App service status
```

并支持：

```text
Export diagnostics
```

---

# 30. 不要先追求 Apple 风格动画

开发优先级：

```text
真实性
>
稳定性
>
防误报
>
找回价值
>
性能
>
UI
>
动画
```

直到核心指标通过前，不投入大量时间做：

- 复杂波纹动画
- 动态岛效果
- 3D AirPods
- 花哨雷达
- Apple 风格转场

---

# 31. 第一阶段必须完成的技术 Spike

这是项目最重要的一部分。

在没有完成这些 Spike 前，不进入完整 App 开发。

---

## SPIKE-00：APK 基础兼容

### 目的

确认我们的 Android App 在 Pocket 2 HarmonyOS 4.3 上基本可行。

### 执行

创建最小 Kotlin App。

测试：

- 安装
- 启动
- 通知
- runtime permission
- BluetoothAdapter
- LocationManager

### PASS

所有核心 API 可调用。

### FAIL

如果 Bluetooth 核心 API 被兼容层限制，重新评估 HarmonyOS Native。

---

## SPIKE-01：AirPods 5 标准连接

### 测试

记录：

```text
device name
address
bond state
classic connection
A2DP state
headset state
```

场景：

1. 两耳佩戴
2. 只戴左耳
3. 只戴右耳
4. 一耳 + 盒子在远处
5. 盒盖打开
6. 盒盖关闭
7. AirPods 断开
8. Bluetooth off/on

### PASS

App 能稳定识别目标 AirPods 的整体连接变化。

---

## SPIKE-02：Raw BLE Capture

### 核心目标

不要一开始写 parser。

先收原始数据。

每个 scan result 保存：

```text
timestamp
device address
device name
RSSI
manufacturer id
manufacturer payload HEX
service UUID
```

---

### 必须采集的状态矩阵

#### State A

```text
两耳 + 盒子全部一起
```

#### State B

```text
左耳佩戴
右耳在盒中
```

#### State C

```text
右耳佩戴
左耳在盒中
```

#### State D

```text
左耳佩戴
右耳 + 盒子远离
```

#### State E

```text
右耳佩戴
左耳 + 盒子远离
```

#### State F

```text
左耳单独放桌面
```

#### State G

```text
右耳单独放桌面
```

#### State H

```text
盒子单独
```

#### State I

```text
盒盖打开
```

#### State J

```text
盒盖关闭
```

每个状态至少采集：

```text
30~60 秒
```

---

## SPIKE-03：AirPods 5 Advertisement Reverse Mapping

根据 SPIKE-02 的数据，寻找：

- 哪些 bytes 不变
- 哪些 bytes 随左耳变化
- 哪些 bytes 随右耳变化
- 哪些 bytes 随盒子变化
- 是否存在 battery fields
- 是否存在 in-ear / in-case bits
- payload 是否明显加密
- 单耳是否独立广播
- 是否可以可靠区分 L/R

必须做差分对比。

不允许仅仅照抄 AirPods 4 / Pro 2 协议并假设正确。

---

## SPIKE-04：单耳掉落模拟

这是最关键测试。

### 场景

左耳戴着。

然后：

```text
左耳取出
放桌上
人继续走
```

记录：

- Bluetooth connection
- BLE advertisement
- RSSI
- audio route
- possible ear state
- time until disconnect

再做右耳。

### 目标

回答：

> 我们能否在真正 Bluetooth disconnect 之前识别“刚才佩戴的耳机已经掉出耳朵”？

---

## SPIKE-05：单耳与盒子分别远离

目的：

证明 Wear-One 模式可行。

场景：

```text
左耳戴着
右耳 + 盒子放家里

人带手机 + 左耳离开
```

App 必须：

```text
不报“右耳丢失”
不报“盒子丢失”
左耳持续受保护
```

---

## SPIKE-06：背景存活

按第 25 节测试。

目标：

确认锁屏后保护仍然有效。

---

## SPIKE-07：AAP 无 Root 可行性

仅实验。

目标：

尝试公共 API 能否访问所需协议。

如果需要：

```text
hidden API
root
xposed
system app
```

则标记：

```text
NOT AVAILABLE FOR PRODUCTION
```

并结束这条路线。

---

# 32. Spike 的判定原则

任何 Spike 最后必须输出：

```text
PASS
PARTIAL
FAIL
```

并附：

```text
真实日志
测试设备
系统版本
耳机固件
复现步骤
结论
```

Agent 不得只写：

```text
“理论上应该可行”
```

---

# 33. MVP 功能定义

只有完成关键 Spike 后才冻结 MVP。

预计 MVP：

## MVP-1

目标 AirPods 绑定。

## MVP-2

稳定连接状态。

## MVP-3

Wear-One 模式。

## MVP-4

断连保护。

## MVP-5

Last Seen。

## MVP-6

Nearby Finder。

## MVP-7

Apple Find My 官方网页入口。

## MVP-8

Diagnostics。

如果 ear status 验证成功：

加入：

## MVP-9

Ear Drop Guard。

---

# 34. MVP 验收指标

## 34.1 连接检测

测试 30 次连接 / 断连。

目标：

```text
无崩溃
事件不丢失或极少丢失
```

---

## 34.2 断连提醒延迟

从真实 Bluetooth disconnect 到用户看到高优先级提醒：

目标首版：

```text
< 5 秒
```

Pocket 2 真机计时。

---

## 34.3 Last Seen

发生断连时：

```text
能够保存最后位置
```

并显示：

- timestamp
- location accuracy

不得把 500m 精度的位置显示成“精确位置”。

---

## 34.4 Finder

在同一房间内：

```text
20 次启动
```

目标目标设备可检测率：

```text
>= 95%
```

前提是 AirPods 当时确实持续发送可扫描广播。

如果 AirPods 固件本身不广播，这项必须标记为“硬件限制”，不得伪造成功。

---

## 34.5 Wear-One 误报

模拟：

```text
另一只耳机 + 盒子留在家
佩戴一只离开
```

连续 20 次。

高优先级误报目标：

```text
0
```

这是首版最重要的业务验收指标之一。

---

# 35. 性能目标

保护模式下：

- 不出现明显持续发热
- 不做永久 SCAN_MODE_LOW_LATENCY
- 电池额外消耗需要实测
- 日志不得无限增长

首版测试：

```text
2 小时
8 小时
```

记录：

- App battery usage
- background uptime
- event loss
- crashes

---

# 36. Repo 结构

建议：

```text
airpods-guard/
│
├── README.md
├── LICENSE
├── .gitignore
├── docs/
│   ├── MASTER_SPEC.md
│   ├── ARCHITECTURE.md
│   ├── DECISIONS.md
│   ├── TASK_CURRENT.md
│   ├── TEST_MATRIX.md
│   └── reports/
│
├── app/
├── gradle/
├── scripts/
├── testdata/
│   └── ble/
└── tools/
```

---

# 37. testdata/ble 是重要资产

Raw BLE capture 不得只存在 Logcat。

保存格式建议 JSONL：

```json
{
  "timestamp": "...",
  "scenario": "LEFT_IN_EAR_RIGHT_IN_CASE",
  "rssi": -54,
  "manufacturerId": 76,
  "payloadHex": "...",
  "deviceAddressMasked": "...",
  "phone": "HUAWEI Pocket 2",
  "os": "HarmonyOS 4.3",
  "airpodsModel": "AirPods 5",
  "firmware": "..."
}
```

这些数据以后用于：

- parser 回归测试
- 协议研究
- 避免每次都重新采集

---

# 38. Parser 必须 Test-Driven

先保存真实 capture。

再编写：

```text
AirPodsAdvertisementParser
```

每一个已知状态都做 fixture。

例如：

```text
left_in_ear_right_case.json
left_out_right_case.json
...
```

如果 parser 改动破坏旧数据：

测试必须失败。

---

# 39. 代码标准

## 39.1 禁止巨大类

目标：

```text
BluetoothManager.kt 2000 行
```

属于失败设计。

必须拆分职责。

---

## 39.2 禁止业务层直接调用 Android API

例如：

```text
ProtectionEngine
```

不应该直接：

```kotlin
BluetoothAdapter.getDefaultAdapter()
```

应该依赖 interface。

---

## 39.3 所有不确定协议必须 fail safe

解析失败：

```text
UNKNOWN
```

而不是猜：

```text
LEFT_IN_EAR
```

---

## 39.4 Magic number

协议 byte value 可以存在，但必须集中定义并写来源。

例如：

```kotlin
const val APPLE_COMPANY_ID = 0x004C
```

---

# 40. Feature Flags

实验功能必须通过 Feature Flag。

例如：

```text
enableAapExperimental
enableIndividualPodBleTracking
enableEarDropGuard
enableBackgroundAggressiveScan
```

默认值必须保守。

---

# 41. Debug Build 与 Release Build

Debug：

- Raw BLE logging
- Diagnostics
- protocol hex
- detailed logs

Release：

- 默认关闭 raw payload 长期存储
- 只保留必要诊断
- 不显示开发者测试按钮

---

# 42. Git 标准

默认分支：

```text
main
```

每个任务：

```text
feat/spike-ble-capture
feat/protection-engine
fix/background-reconnect
```

---

## 42.1 Agent 禁止直接在 main 大规模修改

流程：

```text
branch
-> implement
-> test
-> diff
-> report
-> review
-> merge
```

---

## 42.2 Commit

推荐：

```text
feat: add raw BLE capture
test: add AirPods 5 advertisement fixtures
fix: debounce disconnect alert
docs: record Pocket 2 background test
```

---

# 43. 不允许 Agent 做的事情

以下是永久规则。

```text
Never modify MASTER_SPEC.md.

Never fabricate device test results.

Never claim a Bluetooth packet was observed unless it exists in a real capture.

Never replace failing hardware behavior with hard-coded demo values.

Never change acceptance criteria just to make a task pass.

Never request root without explicit approval.

Never install Magisk/Xposed.

Never store Apple credentials.

Never scrape iCloud Find My.

Never add cloud services without approval.

Never add Google Play Services as a core dependency.

Never silently broaden permissions.

Never disable tests to get green.

Never delete raw test captures.

Never invent AirPods protocol fields.

When uncertain, stop and report.
```

---

# 44. Agent 的职责边界

## ChatGPT

负责：

- 总体方案
- 技术研究
- 任务拆解
- 风险判断
- API 能力判断
- 验收标准
- Review
- Debug 思路
- 是否进入下一阶段

---

## CodeBuddy / WorkBuddy + DeepSeek V4.1 Flash

负责：

- 创建项目
- 读代码
- 修改文件
- 执行 Gradle
- 执行测试
- 运行 adb
- 收集 logcat
- 生成测试报告
- Git diff
- Commit

---

## 用户

负责：

- 手机物理操作
- 耳机物理操作
- 授权权限
- 真实场景测试
- 最终确认

---

# 45. DeepSeek / CodeBuddy 配置原则

DeepSeek 当前官方 API 模型名：

```text
deepseek-flash
```

对应 V4.1 Flash。

DeepSeek 官方文档已经提供 WorkBuddy/CodeBuddy 自定义模型接入方式。

API Key 必须：

```text
使用环境变量
```

禁止提交：

```text
API key
```

到 Git。

---

# 46. Agent 每次执行前必须阅读

```text
docs/MASTER_SPEC.md
docs/DECISIONS.md
docs/TASK_CURRENT.md
```

如果 TASK_CURRENT 和 MASTER_SPEC 冲突：

```text
停止
报告冲突
```

不得自行选择。

---

# 47. TASK_CURRENT 标准模板

```markdown
# TASK

## ID

SPIKE-02

## Objective

Capture raw BLE advertisements from the target AirPods 5 on HUAWEI Pocket 2.

## Read First

- docs/MASTER_SPEC.md
- docs/DECISIONS.md

## Allowed Files

...

## Frozen Files

- docs/MASTER_SPEC.md
- existing raw capture fixtures

## Required Work

...

## Commands

...

## Manual Steps Required From User

...

## Acceptance Criteria

...

## Stop Conditions

...

## Final Report Format

...
```

---

# 48. Agent 最终报告必须包含

每一次执行结束必须返回：

```text
STATUS
PASS / PARTIAL / BLOCKED / FAIL

TASK ID

FILES CHANGED

FILES CREATED

COMMANDS EXECUTED

BUILD RESULT

TEST RESULT

DEVICE TEST RESULT

REAL OBSERVATIONS

UNVERIFIED ASSUMPTIONS

KNOWN LIMITATIONS

GIT DIFF SUMMARY

NEXT RECOMMENDED STEP
```

---

# 49. 第一张 TASK_CURRENT — 可以直接执行

下面就是项目真正的第一步。

---

## TASK ID

```text
BOOTSTRAP-001
```

## Objective

建立 AirPods Guard Android 项目骨架。

暂时不实现任何“Find My”、复杂 AirPods parser 或漂亮 UI。

---

## Required Work

创建 Kotlin Android 项目。

至少包含：

```text
Home
Diagnostics
```

Diagnostics 显示：

```text
Manufacturer
Model
OS version
Bluetooth supported
Bluetooth enabled
BLE supported
Location provider available
Notification permission status
Bluetooth Scan permission
Bluetooth Connect permission
battery optimization state
```

加入日志基础设施。

创建：

```text
docs/
testdata/ble/
```

把本文件保存：

```text
docs/MASTER_SPEC.md
```

---

## Constraints

```text
No Google Play Services
No Firebase
No Root
No Apple private API
No LibrePods source copy
No cloud backend
```

---

## Acceptance

在 HUAWEI Pocket 2：

```text
安装成功
启动成功
Diagnostics 能显示系统能力
无 crash
```

---

## Stop

如果 Android APK 无法正常安装或 Bluetooth API 与预期明显不兼容：

```text
停止后续实现
提交兼容性报告
```

---

# 50. 第二张任务预告

只有 BOOTSTRAP-001 PASS 后进入：

```text
SPIKE-01
```

目标：

```text
识别已配对 AirPods 5
记录标准 Bluetooth connection state
```

---

# 51. 第三张任务预告

```text
SPIKE-02
```

目标：

```text
Raw BLE Capture
```

这一阶段是整个项目未来能做到多少 AirPods 特有功能的基础。

---

# 52. 明确禁止的开发顺序

不要：

```text
先画 UI
↓
写完整 App
↓
最后才测试 AirPods 协议
```

正确顺序：

```text
设备能力
↓
真实数据
↓
协议验证
↓
防丢逻辑
↓
后台稳定性
↓
UI
```

---

# 53. 项目 Kill Criteria

某些能力如果被证明做不到，要及时停止。

## Kill A — 单耳独立本地定位

如果实测证明 AirPods 5 在非 Root Huawei 上：

- 不暴露可区分 L/R 的 BLE identity
- 私有协议也无法访问

则：

```text
放弃“本地独立左/右 RSSI”
```

但继续：

```text
整体 Nearby Finder
+
Last Seen
+
Apple Find My 网页
```

---

## Kill B — Ear Drop Guard

如果：

```text
无法可靠得到 in-ear state
```

则不要模拟。

降级为断连保护。

---

## Kill C — 盒子本地独立寻找

如果盒子：

```text
没有可用独立 BLE signal
```

则本地 App 不宣称能单独寻找盒子。

仍保留 Apple Find My 官方入口。

---

# 54. Success Criteria — 这个项目什么时候算真正有价值

项目不是以“写了多少代码”判断。

满足以下条件，才算取得第一阶段成功：

### 1

用户只戴一只耳机出门时，另一只和盒子不产生错误警报。

### 2

正在佩戴的那一只真正断连时，手机能很快报警。

### 3

断连点能保存可信 Last Seen。

### 4

在耳机仍有 BLE 广播时，可以提供实际有帮助的近距离信号寻找。

### 5

找不到时，可以快速进入 Apple 官方 Find My。

### 6

整个系统不要求 Root。

---

# 55. 第二阶段可扩展功能

只有核心防丢稳定后，再考虑：

- battery popup
- left/right/case battery
- listening mode control
- ear detection media control
- conversational awareness
- head gesture
- firmware metadata
- widgets
- lockscreen quick action
- Huawei outer screen adaptation
- Wearable integration

这些都低于防丢优先级。

---

# 56. Pocket 2 外屏能力

Pocket 2 有外屏。

后期可以探索：

```text
快速显示 AirPods 状态
丢失提醒
Find Nearby 快捷入口
```

但 V1 不做。

先确保主屏功能可靠。

---

# 57. 未来 Android 通用化原则

代码从第一天禁止写死：

```text
if (model == "Pocket 2")
```

Huawei 特有逻辑放：

```text
HuaweiCompatibilityAdapter
```

核心：

```text
ProtectionEngine
BLE parser
Finder
```

必须尽可能平台无关。

未来目标：

```text
Huawei HarmonyOS 4.x Android compatibility
↓
其他 Android
↓
必要时 HarmonyOS NEXT Native
```

---

# 58. HarmonyOS NEXT 原生版什么时候做

只有满足以下条件之一：

1. 目标手机升级到不再运行 APK 的纯 HarmonyOS 环境
2. Android 兼容层明显限制核心 BLE 能力
3. 项目决定正式面向 HarmonyOS NEXT 用户

才启动 ArkTS 分支。

不要现在双线开发。

---

# 59. 第三方代码与许可证标准

特别注意 LibrePods。

LibrePods 是 GPL-3.0 项目。

因此：

**不要直接复制其源代码进入一个我们未来想使用其他许可证的 App。**

允许：

- 阅读协议研究资料
- 理解公开观察到的 packet structure
- 独立重新实现 parser
- 在文档中标注研究来源

如果未来决定直接使用 GPL 代码：

必须先明确整个项目的许可证策略。

---

# 60. 安全底线

项目不得：

- 攻击附近 Apple 设备
- 批量扫描并保存陌生人的设备身份
- 建立人员跟踪功能
- 绕过 Apple Account 安全机制
- 上传陌生设备 MAC 数据到云端
- 将防丢功能改造成隐蔽跟踪工具

目标始终是：

```text
定位和保护用户自己的 AirPods
```

---

# 61. 调试数据采集标准

每次重要测试记录：

```text
Date
Phone model
HarmonyOS version
App commit
AirPods model
AirPods firmware
Scenario
Expected
Observed
PASS/PARTIAL/FAIL
Log file
Capture file
```

建议文件名：

```text
2026-09-20_SPIKE-02_left-in-ear.jsonl
```

---

# 62. DECISIONS.md 标准

每一个重大决定记录：

```markdown
## ADR-001

Decision:
Use Android APK first.

Reason:
Target device is Pocket 2 HarmonyOS 4.3 and Android path is the fastest way to validate Bluetooth capabilities.

Alternatives:
HarmonyOS Native.

Revisit When:
APK compatibility blocks a core feature.
```

这样避免 Agent 反复推翻旧决定。

---

# 63. 当前冻结决定

截至 v1.0：

```text
D001
Primary phone = HUAWEI Pocket 2, HarmonyOS 4.3

D002
Primary earbuds = AirPods 5

D003
No Root

D004
Android APK first

D005
No Google Play Services as core dependency

D006
Local-first, no custom cloud in V1

D007
Apple Find My only through official Apple user-facing route

D008
Wear-One is a first-class product mode

D009
No fake distance

D010
Hardware feasibility before UI polish

D011
Agent may not modify MASTER_SPEC

D012
Real device logs outrank assumptions
```

---

# 64. 目前最重要的未知问题

按优先级：

## U1

Pocket 2 HarmonyOS 4.3 上 Android BLE scan 的真实表现如何？

## U2

AirPods 5 的 Apple manufacturer advertisement 是否仍能稳定识别？

## U3

是否可以无 Root 区分左耳、右耳和盒子？

## U4

是否能无 Root 获得可靠 in-ear / in-case 状态？

## U5

一只耳机掉出耳朵时，最早能观察到什么信号？

## U6

Pocket 2 锁屏后后台扫描与连接广播是否可靠？

## U7

Huawei 电池优化对保护服务影响多大？

答案必须来自实测。

---

# 65. 第一轮开发完成后的交付物

必须至少产生：

```text
APK
Git repository
MASTER_SPEC.md
DECISIONS.md
TEST_MATRIX.md
Diagnostics screenshot
Pocket 2 capability report
AirPods Bluetooth connection logs
Raw BLE capture
Spike report
```

---

# 66. ChatGPT Review Gate

Agent 每完成一个 Spike，把：

- Agent report
- git diff
- relevant logs
- raw capture
- test result

交回 ChatGPT。

ChatGPT Review 后只有三种结果：

```text
APPROVE
REWORK
STOP / CHANGE DIRECTION
```

没有 Review，不进入下一重大阶段。

---

# 67. 给 CodeBuddy / DeepSeek 的顶层 System Prompt

可以在执行项目时使用以下内容作为长期规则：

```text
You are the implementation engineer for the AirPods Guard project.

The authoritative specification is docs/MASTER_SPEC.md.
You must read it before making any change.

Your job is to implement and verify tasks, not to redefine product requirements.

Rules:
1. Never modify docs/MASTER_SPEC.md.
2. Never fabricate hardware observations, logs, or test results.
3. Never claim a feature works until the required real-device test has passed.
4. Do not require root, Xposed, Magisk, or system-app privileges.
5. Do not store or request Apple account credentials.
6. Do not scrape or reverse engineer Apple Find My web services.
7. Do not add Google Play Services as a core dependency.
8. Do not add cloud infrastructure unless the current task explicitly requires it.
9. Do not copy GPL source code into the project unless the task explicitly approves the licensing consequence.
10. Do not invent AirPods protocol fields.
11. Treat unknown protocol values as unknown.
12. Do not alter tests or acceptance criteria merely to make implementation pass.
13. Run all verification commands stated in TASK_CURRENT.md.
14. Keep changes scoped to the current task.
15. If blocked by device capability or an unclear requirement, stop and report the blocker rather than guessing.

At the end of each task return:
STATUS,
FILES CHANGED,
COMMANDS EXECUTED,
TEST RESULTS,
REAL DEVICE RESULTS,
UNVERIFIED ASSUMPTIONS,
KNOWN LIMITATIONS,
GIT DIFF SUMMARY,
NEXT STEP.
```

---

# 68. 当前立即执行顺序

严格按这个顺序：

```text
BOOTSTRAP-001
↓
SPIKE-00 APK Compatibility
↓
SPIKE-01 Standard Bluetooth
↓
SPIKE-02 Raw BLE Capture
↓
SPIKE-03 Advertisement Mapping
↓
SPIKE-04 Single-Ear Drop Simulation
↓
SPIKE-05 Wear-One Separation
↓
SPIKE-06 Background Survival
↓
SPIKE-07 AAP Feasibility
↓
Freeze MVP
↓
Protection Engine
↓
Last Seen
↓
Nearby Finder
↓
Apple Find My handoff
↓
UI Polish
```

任何 Agent 不得跳过关键 Spike，直接宣布“完整 AirPods 定位 App 已完成”。

---

# 69. 外部技术依据

以下是本规范当前最重要的公开依据。外部资料只用于事实验证，不能覆盖真机结果。

## Apple

1. **AirPods 5 Technical Specifications**  
   Apple  
   https://www.apple.com/airpods-5/specs/

2. **Find your lost AirPods with Find My**  
   Apple Support  
   https://support.apple.com/en-gb/109020

3. **Find your AirPods / AirPods User Guide**  
   Apple Support  
   https://support.apple.com/guide/airpods/

4. **Find Devices on iCloud.com**  
   Apple Support  
   https://support.apple.com/guide/icloud/

5. **Find My network — Apple Developer**  
   https://developer.apple.com/find-my/

6. **Nearby Interaction — Apple Developer**  
   https://developer.apple.com/documentation/nearbyinteraction

---

## Huawei

7. **HUAWEI Pocket 2 specifications**  
   Huawei  
   https://consumer.huawei.com/cn/phones/pocket-2-youxiangban/specs/

8. **Huawei Location Kit**  
   Huawei Developer  
   https://developer.huawei.com/consumer/cn/hms/huawei-locationkit/

---

## Android

9. **Bluetooth permissions**  
   Android Developers  
   https://developer.android.com/develop/connectivity/bluetooth/bt-permissions

10. **BluetoothLeScanner**  
    Android Developers  
    https://developer.android.com/reference/android/bluetooth/le/BluetoothLeScanner

11. **CompanionDeviceManager**  
    Android Developers  
    https://developer.android.com/reference/android/companion/CompanionDeviceManager

12. **Foreground service types**  
    Android Developers  
    https://developer.android.com/develop/background-work/services/fgs/service-types

13. **Location permissions**  
    Android Developers  
    https://developer.android.com/develop/sensors-and-location/location/permissions

---

## AirPods interoperability research

14. **LibrePods**  
    https://github.com/librepods-org/librepods

15. **LibrePods AAP Definitions**  
    https://github.com/librepods-org/librepods/blob/main/docs/AAP%20Definitions.md

16. **airpods-notify proximity protocol notes**  
    https://github.com/fischejo/airpods-notify/blob/master/doc/proximity_protocol.md

这些是 reverse-engineering / community sources。

协议字段在 AirPods 5 上必须重新实测。

---

## Agent Execution

17. **DeepSeek V4.1 Flash**  
    https://www.deepseek.com/en/news/deepseek-v4-1-flash/

18. **DeepSeek WorkBuddy/CodeBuddy integration**  
    https://api-docs.deepseek.com/zh-cn/quick_start/agent_integrations/workbuddy/

19. **CodeBuddy models configuration**  
    https://www.codebuddy.cn/docs/cli/models

---

# 70. 最终原则

整个项目永远遵循下面这一条：

> **我们不是为了证明 AI 能写一个 AirPods App，而是为了在 Huawei Pocket 2 上，用真实、可验证、长期可用的方法，尽可能减少一只 AirPod 真正丢失的概率。**

因此：

```text
真实 > 演示
可验证 > 想象
不误报 > 功能数量
防丢 > 花哨
实机日志 > AI 推测
```

---

# 71. 当前正式起点

**下一任务：BOOTSTRAP-001。**

在执行它之前，Agent 需要：

1. 创建或打开项目目录
2. 把本文件放入 `docs/MASTER_SPEC.md`
3. 创建 `docs/TASK_CURRENT.md`
4. 只执行 BOOTSTRAP-001
5. 不开始 AirPods 私有协议实现
6. 完成后提交执行报告
7. 将报告和 Git diff 交回 ChatGPT Review

**MASTER_SPEC v1.0 END**
