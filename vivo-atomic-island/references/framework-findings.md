# 已验证固件：供无 vivo 真机的个人开发者参考

来源为设备所有者授权提取的系统文件及本机实验，整理于 2026-10-11。设备：iQOO 15 / `V2505A`，Android 16 / API 36，OriginOS 6，构建 `PD2505B_A_16.0.30.14.W10`，未 root。以下不是所有 vivo 固件的通用契约；来源指纹和证据定位见 [framework-evidence.json](framework-evidence.json)。

## 接口与权限

调用链：`NotificationManager` → `INotificationManager` Binder → `VivoNotificationManagerServiceImpl` → `VivoSuperXNotificationManager`。

| 接口 | 本固件行为 |
| --- | --- |
| `setSuperXInfosSceneList(List<String>, List<String>, List<String>, List<String>)` | Binder 事务 209；四个 StringArrayList，分别为场景、场景开关、包名组、包开关组 |
| `getSceneStatus(String pkg, String scene, int switchType)` | Binder 事务 210；`0` 原子通知、`1` 助手、`2` 两者同时开启；两参数重载默认 `0`。第三参数不是用户 ID |

写入检查 `isCallerSystemOrSystemUi()`，其判断包含 `STATUS_BAR_SERVICE` 权限。本机 `com.android.shell` 已持有该权限，用户授权的调试会话以 UID 2000 调用成功，无需 root。普通应用无权写入时直接返回；Binder 未抛异常不代表注册成功。查询允许有权系统调用者或查询自己包名的应用。

每个列表元素对应一个场景，布尔值是字符串 `"true"/"false"`；同一场景的包名及对应开关以 `-` 分隔。写入会替换所提交场景的整组配置，其他场景保留，随后重建允许映射并延迟约 5 秒保存。适配层应先读取并保留本场景已有映射，避免覆盖别人的包；事务号与这种编码只能用于已核实固件。

配置关联 `com.vivo.assistant.switchstateprovider` 与 `/data/bbkcore/vivo_superx_info_list.xml`。Provider 可见或权限为 normal，不代表业务写入有效；已检查的 `insert/update` 返回空或 0，未用 Provider 修改配置。正式分发仍走官方申请。

## 模板与通知生命周期

- 完整模板仍检查包名与 scene 的允许映射。配置前正确模板会降级普通通知；本机注册自有 `FOCUS_TIMER` 场景后才出现原生岛。`getSceneStatus=true` 需结合系统分类及实际画面判断。
- `baseInfos.icon` 使用 `Icon` 对象。按官方模板补齐 `infos`、`shortInfos`、`capsule`、`island` 等所需结构；仅有标题和计时文字不足以证明 payload 有效。
- 历史 `islandNotify=true` 路径仍检查 `com.vivo.notification.ISLAND_NOTI_SUPPORT`，本机定义为 signature。普通应用声明该权限或改成 demo 包名都没有获得豁免。
- 本机 END 分支前仍检查路由和模板；完整模板结束时保留 `scene` 与正确 `template`。通知 tag 会改写为 `VIVO_SUPERX_TAG`，清理自有通知时兼顾原 null tag 和该 tag，避免残留。

## 已完成的真机验证

自有场景启用、撤销及重新启用均核对回读和其他映射；注册后 `SUPERX_STATE/SUPERX_VALUE=true`，并观察到真实原子岛。18 个公开版式（主卡 5、右岛 6、辅助 7）已逐项显示；原应用前台服务退到后台后继续更新计时。用户完成 APK 内本机无线调试配对及注册回读，USB 断开后电脑 ADB 无设备。

无线调试用于启用或撤销，日常 CREATE/UPDATE/END 由应用前台服务发送。重启保持、云配置覆盖及 OriginOS 7 尚未验证。

## 无设备时怎么使用这些结论

可先用官方 demo 和存档编写通知构造器，将场景读写隔离为适配接口；通过模拟返回验证静默写入失败、读取不一致、场景映射保留，以及通知创建/更新/结束。无线调试客户端、引导和应用后台逻辑可独立开发。

将这版机型、构建号、事务号与编码放进明确的适配表。没有目标固件证据时返回“尚未适配”，不尝试猜测 Binder 编号。纯 AOSP 模拟器不能据此验收 vivo 的授权或渲染；最后由持有目标手机的测试者核对回读、系统分类、画面及拔线操作。

## IntentsUI 的补充探测

在六个已提取文件（framework、services、vivo-framework、vivo-services、SystemUI、助手）的 DEX 类定义和字符串中，找到标准 `android.app.appfunctions`，未找到文档涉及的 `vivo.app.intentsui` / `com.vivo.intentsui` 类或引用。检查范围不是完整 ROM；不能推断 OriginOS 7、系统后续更新或独立 SDK 不支持。

[IntentsUI 规范](https://dev.vivo.com.cn/documentCenter/doc/1107)描述系统触点中的单次交互及结果状态岛，未给出持续计时刷新、SDK 坐标或当前固件准入保证。与已验证的原子通知路径分别适配。
