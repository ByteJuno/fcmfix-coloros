# 固件核对记录

给维护者看：ColorOS 的限制在固件中的位置、模块的处理，以及核对过但不用处理的项目。适配新固件时从这里开始。

## 已核对的固件

| 固件 | 系统 | 依据 |
| --- | --- | --- |
| 一加 15 `PLK110_17.0.0.102(CN01)` | Android 17 | 全量包的 `system`、`system_ext`：`services.jar`、`oplus-services.jar`、`oplus-framework.jar`、`oplus-service-jobscheduler.jar`、`Battery.apk`，维护者日常使用 |
| 一加 15 `PLK110_16.0.10.500(CN01)` | Android 16 | 全量包，与国际版 `CPH2747 16.0.10.500 EX01` 逐分区差分，实机验证 |
| 一加 13 `PJZ110_16.0.3.502(CN01)` | Android 16 | 反馈者上传的 `oplus-services.jar`（该机型在 `/system/framework`，不在 `system_ext`） |
| 真我 GT7 Pro `RMX5010_16.0.10.500(CN01)` | Android 16 | 只有反馈，完整固件和真机都未验证 |

方法：dexdump 导出全部方法和字段签名逐项比对，关键方法用 jadx 确认语义。

## 国行与国际版的差异

国行的 FCM 限制不是单一开关：主要代码跨区域共用，区域包通过 feature 和 XML 启用不同策略。
只把 `google_restric_info` 改为 `0`，仍可能在联网策略、应用分类、自启动名单、Hans/ELSA 或 Doze 阶段被拦截。

| 层 | 差异 | 影响 |
| --- | --- | --- |
| GMS 区域开关 | 国行独有 `oplus.software.disable_cn_gms` | 未启用谷歌服务时直接阻断 |
| Doze 白名单 | 国际版含 GMS、GSF、Play，国行只有 GSF | 国行可把 GMS 移出省电白名单 |
| Hans GMS 状态 | 国行启用 `oplus.software.hans_restriction`，监听 `google_restric_info` | 可向 Hans 各模块广播 GMS 受限 |
| ELSA/Hans 掩码 | GMS/GSF/Play 国行 `1111111110`，国际版 `0111111110` | 国行多出 close-socket/net 位 |
| 应用分类 | `isAppClassifyRestricted` 国际版直接返回 false，国行查限制表 | 可在广播、服务、Provider、Job 等入口提前拒绝 |
| GCM bind 自启动 | 国行 `bsgcm` 路径查过 `google_restric_info` 后仍调用 `isAllowAutoStartByList` | 只清 Google 限制开关，目标应用仍可能拉不起来 |
| LightOS 策略 | 国行 `sys_startup_policy_config.xml` 启用 Job、粘性广播等限制 | 增加后台启动和调度限制，并非都针对 FCM |

## 限制与处理

| 限制 | 位置 | 处理 |
| --- | --- | --- |
| Google 探测失败后禁网 | 电池 `GoogleRestrictionController`：`setUidPolicy(uid, 4)`，经 `networking_control` 到 `OplusNetworkManagementService.setFirewallUidRuleForNetworkType`；开机完成 5 秒后也探测一次 | 电池进程把写入改为 `POLICY_NONE`；系统框架丢弃 GMS、GSF、Play、ConfigUpdater 的拒绝规则（按 appId，仅当电池组件声明 `IgnoreGmsUserSet=true`） |
| Google 限制广播 | 探测失败时广播 `oplus.intent.action.google_restrict_change`：`OplusGoogleAlarmRestrict` 把 GMS 闹钟降级为不唤醒，`AppStandbyControllerExtImpl` 放入待机分组 40 | 电池进程和 `broadcastIntentWithFeature` 入口都把 `restrict_enable=true` 改为 `false`；`isGoogleRestrct()` 返回 `false` 兜底 |
| GMS 受限状态 | `OplusBgSceneManager.isGmsRestricted()`：丢弃 GMS/GSF 唤醒锁、OGuard 耗电管控、限制前台应用唤醒 GMS | getter 恒为 `false`；`updateGmsRestrict`、`registerGmsRestrictObserver` 空实现，getter 被内联也不受影响 |
| 弱信号防火墙 | `OplusHansConnectivityManager.setWeakSignalUidFireWallChain`、`OplusHansRestriction.isBlockedAlarmPolicy` | `isWeakSignalNetWhiteList` 对 GMS/GSF/Play 返回 `true` |
| 应用分类、GCM bind | `OplusStartupStrategy.isAppClassifyRestricted`、`OplusAppStartupManager.isAllowStartFromBindService`（`bsgcm` / `system[gcm]`） | 允许列表内的真实 FCM 放行 |
| 冷启动拦截 | `OplusAppStartupManager.validStartProcessFromBroadcast` | 按 `BroadcastRecord` 核验可信 FCM 后放行 |
| 云控恶意应用名单 | `MaliciousRestrictPolicy.shouldPreventBroadcastByMaliciousCheck` / `shouldPreventServiceByMaliciousCheck` | 广播按 `BroadcastRecord` 核验；服务只放行真实 GMS 或投递窗口内目标自身 |
| 关联启动限制 | `OplusLinkStartManager.handleProcessBroadcastStartLocked` | 可信 FCM 放行 |
| 广播代理 | `BroadcastProxyAction.enqueueProxyBroadcastLocked`：Osense 在游戏、相机等场景暂存冷进程广播，action 列表由云控下发 | 真实 GMS 发给允许列表目标的 RECEIVE 不代理 |
| 深度睡眠按应用断网 | 电池 `ControllerCenter` 的白名单含 `com.heytap.mcs`、VoWiFi 等，不含 GMS，最终到 `OAppNetControlService.networkDisableWhiteList` | 系统框架在开始断网时补入 GMS UID，恢复 3 秒后发 `GCM_RECONNECT`；电池进程在 `OAppNetControlManager.networkDisableWhiteList(List,int)` 也补入 |
| 深度睡眠恢复后强停 GMS | `ControllerCenter.onScreenOn` → Osense `CommonExternalClean` strategy 2，reason `DeepSleepLogicDisNetRestore` | 只跳过这一次强停，其他清理不受影响 |
| 深度睡眠闹钟延后 | `OplusDeepSleepHelper.filterDeepSleepAlarm` / `ruleMatchDeepSleepAlarm` | GMS/GSF 闹钟不延后 |
| Hans 作业限制 | `OplusHansManager.checkJobIfRestricted` | 投递窗口内的 UID 不拦截 |
| Doze 白名单 | `OplusDeviceIdleHelper.getNewWhiteList`（私有短方法，可能被内联） | 在原列表后追加 GMS/GSF/Play，`whiteListChangedHandle` 调用前再补一次 |

## 版本差异

ColorOS 17 相对 16（未列出的 Hook 点签名不变）：

- `BroadcastController.broadcastIntentLocked` → `broadcastIntentLockedTraced`：包装层加实际实现，两者都 Hook，调用链去优化；
- `isAppClassifyRestricted(int,String,String,Long)` 新增，没有 Intent，跳过；
- `HansCGroup.FastFreezeEnter(int)` 移除，改为 `fastFreezeEnter(int,String)`，已纳入匹配；
- `OplusDeviceIdleHelper` 从 `oplus-services.jar` 移到 `oplus-service-jobscheduler.jar`，同一 ClassLoader，不用改；
- `IOplusGoogleDozeRestrict` 新接口，运行时是默认空实现，不用处理。

ColorOS 16 的入口不同：

- 强停入口是 `OsenseResManagerService$3.requestSceneAction(Bundle): void`，17 是 `requestSceneActionSync(Bundle): boolean`，两者都匹配。
  `PLK110_16.0.10.500` 的电池组件中没有找到发起这条清理的代码，“Hook 已安装”不代表存在触发行为；
- 没有 `ControllerCenter.addPkgWhiteArray` / `addUidWhiteArray`。名单由 `determineNetWorkWhiteUidList` 等构造，
  经 `networkDisableWhiteListInner` 反射调用 `OAppNetControlManager.networkDisableWhiteList(List,int)`。
  Hook 这个最终接口也避开了 17 上白名单方法被内联的问题；恢复调用（enable=1）不改写。

### 什么时候记为“不适用”

只有识别出下面已核对的结构才记“不适用”；类缺失、未知结构、方法更名或重载变化、Hook 安装失败都报告失配。

- **弱信号（`PLK110_16.0.10.500`）**：`OplusHansConnectivityManager` 只有蓝牙管控。SDK 36、已知蓝牙方法和字段类型一致、父类为 Object、
  没有 weak/firewall 成员时成立；
- **弱信号（`PJZ110_16.0.3.502`）**：没有这个类，`OplusHansManager` 中也没有名字含 weak 的成员；
- **可疑组件（`PJZ110_16.0.3.502`）**：没有 `MaliciousRestrictPolicy`，名单只在
  `OplusAppStartupManager.preventByMaliciousCompStartegy(int,int,String,String): boolean` 中检查，唯一调用方是 `shouldPreventStartActivity`，
  只拦 Activity，不影响 FCM。名字含 malicious 的方法只有它（`dumpMaliciousConfig` 除外）且签名一致时成立。

## 实测记录

- **强停后唤醒**（`PLK110_16.0.10.500`，`53-oneplus15-cos16-3`）：`am force-stop` Nekogram 后包状态 `stopped=true`、无进程；测试消息到达时 GMS 发出
  `RECEIVE`，BroadcastFix、OplusProxyFix、AutoStartFix 都在调用栈中；Nekogram 被拉起，约 0.57 秒后发出通知。
  `coloros_gms_extreme_fix` 保持禁用，GMS 的 5228 长连接保持 `ESTABLISHED`，四个 Google 包的联网策略都是 `POLICY_NONE`。
- **夜间断线 9 小时**（`53-coloros-11`）：`deepsleepRcd.txt` 显示下发的白名单不含 GMS，电池 APK 中的短方法很可能被内联，Hook 没有触发。
  GMS 心跳无回应后连接被重置，8 次重连失败后不再重试，手动发 `GCM_RECONNECT` 立即恢复。因此改到系统框架处理，并在恢复后主动重连。
- **解锁后一直未连接**（`last close code` 为 `ERR_CLOSE_BY_USER_UNLOCKED`）：首次解锁时代理通常还没启动，开机探测失败后 Google 核心包被禁网。
  系统框架丢弃拒绝规则后，策略文件中的值不变，`service call networking_control 2` 仍可能读到 4，以 FCM 连接状态和日志
  `Oplus Google network reject dropped` 为准。

每次 OTA 后都要重做强停回归：确认 Hook 全部生效，`am force-stop` 后发 FCM，再测熄屏、Doze 和长时间待机。

## 核查后不用处理

- `OplusPartialWakeLockCheck.FORCE_RELEASE_LIST` 含 GMS，但只处理熄屏后持有超过约 300 秒的唤醒锁，FCM 的唤醒锁只有数秒；
- `OplusAppStartTracker` 启动配额只对 Oplus 自研应用生效；
- Hans 服务代理 `isProxyService`、广播延后 `DeferProxyPolicy` 只作用于被冻结的 UID，投递窗口内的目标不会被冻结；
- `OplusResourcePreloadManager.preloadServiceBlock` 只针对系统预加载的进程；
- `OplusAppStartupConfig.isInSysLongDelayList` 在本固件没有调用方；
- `OplusPermissionInterceptPolicy`、`OplusEapManager`、Hans GMS 定位代理与推送无关；
- 防火墙：按 UID 拒绝联网的来源只有 `networking_control` 和弱信号防火墙；
- 熄屏闹钟对齐 `OplusAlarmAdjustment`：GMS 心跳是非精确闹钟，取整偏移在一个间隔内，不会让心跳失效；
- 临时白名单 `shouldIgnoreTempWhitelistChange` 只跳过重复通知，不阻止 FCM 高优先级消息的临时放行；
- `OSysNetControlService` 增强夜间断网和电池深度睡眠的关网方式都是整机断网，白名单不适用，恢复时 GMS 自行重连；
- `CNGmsControlService`：国行“Google 移动服务”开关，关闭属用户选择。出境自动开启后回国会自动关闭，少见，未处理。

## 未采纳的方案

- 上游 PR #262 的 Doze 修复：清空已合并的白名单后用 `mDefaultWhitelist` 重建，而该列表不含 GMS，还会丢失区域和 RUS 条目，所以只追加缺失的包；
- UnifiedPush action：与“只信任真实 GMS 发送者”的模型冲突；
- `shouldPreventStartProcess` 放行：17 上该方法只拦伪造的进程记录，放行反而让伪造记录继续启动；
- `isAllowStartService`、`skipScheduleReceiver*` 整体放行：范围过宽，已由带来源核验的放行覆盖；
- 投递窗口 60 秒：保留 20 秒；
- Ice Box 同步激活：与不阻塞广播线程的设计相反。

## 尚未核实

- ColorOS 16 的电池组件是否声明 `IgnoreGmsUserSet`，流量管理是否显示 Google 联网开关；
- ELSA `whitePkg` / `hansKeepAlive` 在 ColorOS 17 中的作用；目前没有发现 Hans 冻结 GMS 本身的路径；
- 真我 GT7 Pro 的装机与深度睡眠测试。
