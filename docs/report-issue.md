# 如何提交问题与日志

先更新到最新版，再按下面三步收集信息，然后[提交 Issue](https://github.com/ByteJuno/fcmfix-coloros/issues/new/choose)。
FCMFix 的日志在系统 logcat 中（标签 `fcmfix`），不在 LSPosed 管理器的“模块日志”里。

## 1. 先看“状态”页

“状态”页逐项检查模块、作用域、Hook、配置同步和 FCM 连接：

- “未加载”：通常是作用域没有勾全，或改动后没有重启；
- “失配”：某个 Hook 与当前固件不匹配，点开可以看到是哪一项；
- FCM 连接异常：按下文[判断是网络问题还是系统限制](#fcm-断开网络问题还是系统限制)。

再到“应用”页确认目标应用已放行，并看它最近一次收到推送的时间。收到了推送却没有通知，问题通常在应用这一侧：
点开它查看通知状态，或进入系统设置检查。一直没有收到，再排查 GMS 与网络。

## 2. 导出报告（需要 Root）

复现问题后，在“应用”页点开收不到通知的应用，点“导出报告”；或在“状态”页底部点“导出完整报告”，选择应用。授予 Root 后分享或保存。

打不开 FCMFix 时，可以手动运行 [`collect-report.sh`](../scripts/collect-report.sh)：保存到手机的 `Download` 目录，在 Termux 或 `adb shell` 中执行两次。

```sh
# 重启后 3–5 分钟内执行，开机时的 Hook 结果很快会被 logcat 冲掉
su -c sh /sdcard/Download/collect-report.sh fcmfix-report-boot.txt
# 复现问题后执行，把包名换成收不到推送的应用
su -c sh /sdcard/Download/collect-report.sh fcmfix-report-issue.txt org.telegram.messenger
```

报告保存在 `Download` 目录。脚本只读取系统状态和日志，不修改任何设置。

## 3. 提交时附上

- 报告，**以附件上传**：把 `.txt` 文件拖进 Issue 输入框。报告通常有几万字，直接粘贴可能被截断；
- “状态”页底部“复制结果”的内容；
- FCM Diagnostics 截图和 LSPosed 中的作用域截图；
- 网络环境（Wi‑Fi 或移动数据，是否使用代理）、测试应用、发送测试消息的大致时间。

报告中包含已安装应用的包名，公开前可以打码，但请保留 `fcmfix`、`GoogleController`、`OplusGoogle` 相关行。

## 常见问题

**放行的应用被“强行停止”后也会被唤醒吗？** 会。放行即允许可信的 FCM 唤醒它，包括被手动强行停止的情况。

**需要把应用加入系统“流量管理”或自启动白名单吗？** 不需要。模块只在每次推送到达后的短时间内放开限制，之后恢复系统原有的省电策略。

**还需要 `coloros_gms_extreme_fix` 这类 GMS Magisk 模块吗？** 不需要，实测只启用本模块时 GMS 长连接和联网策略都正常。OTA 后请重新确认。

**微信没有走 FCM？** 微信等受系统保护的 IM 应用平时走自己的长连接，FCM 日志中没有记录属正常，强行停止后再测试能否被 FCM 唤醒。
电池设置为“优化”时可能被冻结而漏消息，建议设为“允许完全后台行为”。

## FCM 断开：网络问题还是系统限制

FCMFix 修复的是 GMS 收到消息之后被 ColorOS 拦截的问题，不能让 GMS 连上 FCM 服务器。
打开 FCM Diagnostics（“状态”页，或拨号 `*#*#426#*#*`），事件都带时间戳，判断的关键是断开的时机：

| 现象 | 更可能的原因 |
| --- | --- |
| 断开后几秒到几十秒内重新连上 | 正常，运营商或路由器清理了空闲连接，GMS 自动重连 |
| 亮屏使用中也连不上或反复断开 | 网络 |
| 只在某个网络下出问题 | 网络 |
| 断开前刚切换 Wi‑Fi / 移动数据 | 网络切换，属正常 |
| 断开发生在 `client entering doze` 之后，直到亮屏才恢复 | 系统限制 |
| 开机后一直未连接，`last close code` 为 `ERR_CLOSE_BY_USER_UNLOCKED` | 解锁后被系统断网 |

`client entering doze` 本身是正常记录，要看它之后连接是否保持。

`ERR_CLOSE_BY_USER_UNLOCKED` 也是正常记录，但之后一直未连接说明解锁后的重连失败了：电池服务在开机完成约 5 秒后探测 Google，
此时代理通常还没启动，探测失败就会把 GMS 设为禁止联网。请确认两个作用域都已勾选。

### 测试网络连通性

在出问题的网络下，用 Termux 执行（先 `pkg install curl dnsutils`）：

```sh
nslookup mtalk.google.com
curl -v --max-time 5 telnet://mtalk.google.com:5228
```

- `nslookup` 返回的不是 Google 地址（常见为 `142.250.x.x`、`74.125.x.x`、`172.217.x.x`、`108.177.x.x`）：DNS 被污染，修改 hosts 或换 DNS 通常就能恢复；
- `curl` 输出 `Connected to ...`：端口可连通；超时或 `Connection refused`：端口被封或 IP 不对，只能走代理。

代理能通、直连不行，基本可以确定是网络问题：ColorOS 的限制按 GMS 的 UID 施加，GMS 被禁网时开代理也没有网。

代理应用在后台被杀或被冻结时，FCM 会随之断开，看起来像 FCM 的问题。请给它开启自启动并关闭电池优化。

### 排除系统限制

在报告中确认：

- “Google 核心包联网策略”中 GMS 为 `policy=0`，待机分组不是 `40`；
- 日志中有 `Oplus Battery Google restrict broadcast hooks active` 和 `Oplus Google alarm restriction hook active`，没有 `hook error`。

这些都正常、连通性测试不通，是网络问题。连通性正常、断开却集中在 Doze 期间，可能是尚未覆盖的系统限制，请附上报告和断开时间提交 Issue。

最直接的验证：连一个确定可用的网络，熄屏放置半小时。在该网络下不断开，而在原网络下断开，就是原网络的问题。

## 流量管理里没有 Google 入口

ColorOS 17 国行的“流量管理”会**隐藏** Google Play 服务、Google 服务框架和 Play 商店的联网开关，也拒绝用户修改。
这三个包由电池组件自动管理：探测 Google 失败时设为“全部禁止”，成功时解除。这是系统设计，不是故障。
电池作用域会把这次“全部禁止”改为“不限制”，勾选并重启后通常会自动恢复。

这些策略保存在 ColorOS 自己的联网控制服务中，**不在** `dumpsys netpolicy` 里。查看报告的“Google 核心包联网策略”一栏，或用 Root 执行：

```sh
su -c 'service call networking_control 2 i32 $(pm list packages -U com.google.android.gms | grep "^package:com.google.android.gms " | grep -o "uid:[0-9]*" | cut -d: -f2)'
```

输出 `Parcel(00000000 00000000 ...)` 表示不限制，`Parcel(00000000 00000004 ...)` 表示全部禁止。

系统框架作用域丢弃禁网规则后，这里记录的值可能仍是 4，但实际不会断网，以 FCM Diagnostics 是否连接为准。

如果一直是 4 且连不上：系统记录了“用户手动改过”这些包时（报告中 `oplus_user_change_gms_network_control` 不为 `0`），电池组件会一直跳过它们。
模块通常会自动处理，日志中有 `Oplus Battery ignores stale GMS user-change flag`。未生效时手动清除后重启：

```sh
su -c 'settings put global oplus_user_change_gms_network_control 0'
```

### 防火墙链

ColorOS 17 的按应用断网挂在 `fw_INPUT` / `fw_OUTPUT` 及其子链上，被限制的 UID 记录在 BPF 表中，规则本身不出现 UID。
报告的“防火墙链”一栏列出这些规则及命中次数（`pkts`）。FCM 连不上时，复现前后各导出一次，`pkts` 明显增加的规则就是拦截来源：

| BPF 程序名包含 | 系统功能 |
| --- | --- |
| `reject_wlan_uid`、`drop_cell_uid`、`reject_qcom_uid` | 流量管理联网策略（含电池服务探测失败后的 Google 禁网） |
| `netdisable` | 深度睡眠按应用断网 |
| `hans` | 应用冻结时断网 |

“ColorOS Google 拦截规则清理”一类模块会删除这些规则，排查时请先停用并重启。

## 报告怎么看

| 报告内容 | 含义 |
| --- | --- |
| GMS 为 `policy=4`（或 1、2） | GMS 被禁止全部（或部分）联网，见[流量管理里没有 Google 入口](#流量管理里没有-google-入口) |
| 待机分组为 `40`，或 `google_restric_info` 为 `1` 但日志中没有 `restrict broadcast cleared` | ColorOS 17 的 Google 限制未被解除 |
| 日志中有 `hook error` 或 `Unsupported` | 某个 Hook 与当前固件不匹配，请在 Issue 中贴出这些行 |
| “Doze 白名单”显示 GMS/GSF 不在白名单 | Doze 白名单 Hook 未生效，日志中应有 `Oplus Doze whitelist hook active` |
| 深度睡眠记录与 FCM 断开吻合，且 `disNetType = 4` | 按应用断网，模块会放行 GMS，日志中应有 `Oplus night network whitelist: added GMS` |
| 深度睡眠记录中 `disNetType` 为 1、2 或 3 | 整机断网，模块无法阻止，可在设置中关闭“睡眠待机优化” |
| 以上都正常，但 FCM Diagnostics 一直 disconnected | 网络问题（DNS 污染或端口被封） |
| 日志中有 `FCM trusted sender`（GMS 已发出），但没有通知 | 看“目标应用”一栏：`POST_NOTIFICATIONS` 为 `granted=false`、`importance` 为 `0` 或 `NONE`、渠道 `mImportance=0` 表示通知被关闭；`stopped=true` 且没有 `Start proc` 表示应用未被拉起，请提交 Issue |
| 有 `Successful broadcast`、进程也被拉起，但通知仍延迟 | 看有没有 `Oplus FCM delivery window`、`Oplus FCM Hans-freeze bypass`、`Oplus FCM socket-close bypass`。二次冻结或断网只是可能原因之一，还要结合应用内部同步、消息优先级和网络判断 |
