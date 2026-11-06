# FCMFix ColorOS

修复**国行 ColorOS 16 / 17** 拦截 Google FCM 推送的 LSPosed 模块：应用被划掉或被系统停止后，仍能及时收到推送。
需要 **Root + LSPosed**（libxposed API 100 及以上）。

基于 [kooritea/fcmfix](https://github.com/kooritea/fcmfix) 53 版。已在一加 15（PLK110）上验证，其他 ColorOS 16 / 17 机型可能可用，
但未经同等级验证。不是 ColorOS 国行请用上游版本。

## 安装

1. 从 [Release](https://github.com/ByteJuno/fcmfix-coloros/releases/latest) 或 LSPosed 模块仓库安装。
2. 在 LSPosed 中启用模块，作用域**同时勾选**“系统框架”和“电池（`com.oplus.battery`）”，然后重启手机。
3. 打开 FCMFix，放行需要推送的应用，或开启“自动放行包含 FCM 的应用”，并确认这些应用的通知权限已开启。

不要同时启用两个 FCMFix。从旧包名 `com.fcmfix.coloros` 或 `com.kooritea.fcmfix.op15` 迁移时不能覆盖安装，请先停用并卸载旧版。

更新通知：Telegram 频道 [@ByteJuno](https://t.me/ByteJuno)，新版本发布、构建结果都会推送，欢迎关注。

## 能解决什么

- 划掉后台、被系统清理或处于“已停止”状态的应用，FCM 仍能唤醒它并显示通知；
- 推送到达后，应用不会马上被再次冻结或断网，能及时拉取消息内容；
- GMS、Play 商店等 Google 核心服务不再被禁止联网，熄屏、Doze 和“睡眠待机优化”期间保持连接。

## 不能解决什么

- GMS 连不上 FCM 服务器：常见于 DNS 污染或 5228–5230 端口被封，需要自行解决网络（[判断方法](docs/report-issue.md#fcm-断开网络问题还是系统限制)）；
- 应用不使用 FCM、服务器没有发送消息，或 Google Play 服务未登录、被禁用；
- 系统整机断网，例如深度睡眠直接关闭 Wi‑Fi / 移动数据；
- 工作资料、应用分身和多用户下的推送（未承诺支持）。

本模块不安装或替代 GMS，也不是推送服务器、代理或保活工具。

## 风险与隐私

- 这是 system_server 级别的 Hook，安装前请保留进入安全模式或禁用 LSPosed 模块的恢复手段。
  OTA 后 Hook 可能与固件不匹配，此时保持系统原行为，并在“状态”页显示失配；
- APK 不申请 `INTERNET` 权限，不经过任何服务器中转；导出的报告只保存在本机，由你决定是否分享。

## 文档

- [遇到问题](docs/report-issue.md)：自查、导出报告、常见问题
- [工作原理](docs/how-it-works.md)：ColorOS 的限制位置、模块改了什么、与上游的区别
- [固件核对记录](docs/firmware-notes.md)：各固件上的 Hook 点与实测
- [开发说明](docs/development.md)：信任边界、构建与发布

## 许可证

本项目自己的代码以 [MIT](LICENSE) 授权。上游 fcmfix 没有发布许可证，来自上游的代码不在授权范围内，详见 [NOTICE](NOTICE)。
