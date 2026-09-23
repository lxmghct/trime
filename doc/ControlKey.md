<!--
SPDX-FileCopyrightText: 2026 Rime community

SPDX-License-Identifier: GPL-3.0-or-later
-->

# 独立 Ctrl 键

## 结论

Trime 的运行时已经具备独立修饰键所需的处理链路：`KeyCode` 能把 `Control_L` / `Control_R` 映射为 Android 的 Ctrl key code，`KeyAction` 会识别修饰键，`Keyboard` 保存锁定状态，后续普通按键会继承 Ctrl 状态。若 Rime 没有消费组合键，`TrimeInputMethodService` 会把它作为带 `META_CTRL_ON` 的 Android `KeyEvent` 发给当前应用。

此前默认主题只有 `Control+a`、`Control+x` 等预设组合键，没有可点击的 `Control_L` 入口，因此无法从默认键盘发出 `Ctrl+B`。问题在主题入口，不在 Android key event 底层。

## 源码改动

- 在内置 `trime.yaml` 增加 `Control_L`、`Control_R` 预设，显示为 `Ctrl`，采用 `shift_lock: click`。
- 在默认键盘底行加入 `Control_L`；空格宽度由 30 调整为 20，给 Ctrl 腾出空间并保持该行不换行。
- 修正同文风主题中已有但缺少锁定语义的 `Control_L` 定义。
- 增加单元测试，验证预设键数量、`Control_L` 映射和锁定行为。
- 详细源码提交：`4153d2d9 feat: add standalone ctrl key`。

自定义主题可以复用：

```yaml
preset_keys:
  Control_L: {label: Ctrl, send: Control_L, shift_lock: click}

preset_keyboards:
  default:
    keys:
      - {click: Control_L}
```

如果主题继承内置 `preset_keys`，通常只需加入键盘项；如果覆盖了 `preset_keys`，需要同时保留上面的预设定义。

## 构建验证

- 基线：`develop` 分支，提交 `f3ca316f`。
- 单元测试：`./gradlew.bat :app:testDebugUnitTest --no-daemon`，320 个测试通过。
- 真机调试 APK：`app/build/outputs/apk/debug/com.osfans.trime-v3.3.12-49-g4153d2d9-arm64-v8a-debug.apk`（安装后包内版本名为 `3.3.13`）。
- 构建时因 Android NDK 的 Lua 代码使用了 Linux 的 `fseeko/ftello`，临时使用 Android 条件分支改为 `fseek/ftell`；构建完成后已恢复第三方源码，未作为功能改动提交。

## 真机验证记录（2026-09-23）

- 设备：OPPO PJX110，Android 15，ADB 序列号 `fa9d8eac`。
- 安装包：`com.osfans.trime.debug`，版本名 `3.3.13`；测试完成后已卸载旧版 `com.osfans.trime` v3.3.0，手机仅保留新版。
- 新版本首次运行需要选择“内部存储/rime”目录。选择后应用使用 `EXTERNAL_SYNC`，并导入了原有 Rime 配置。
- 修改前已备份：`/storage/emulated/0/rime/默认配置调整.trime.yaml.codex-backup-20260923-1`。
- 备份文件与原配置的 MD5 均为 `4bfd00444dc6b00ecc9c37b808612d00`。
- 已在 `/storage/emulated/0/rime/默认配置调整.trime.yaml` 中加入 `Control_L` 预设和新版中英文切换配置；当前修改版 MD5 为 `2a666a51ba6ab240f29f2cecfd70b3bb`，应用专属同步目录中的副本 MD5 相同。
- 中英文切换修复前的配置另备份为 `/storage/emulated/0/rime/默认配置调整.trime.yaml.codex-before-switch-fix-20260923-1`，MD5 为 `7538a1458d98a4aa8aa73df72e3d879c`。
- 真机部署并重新选择“默认配置调整”主题后，键盘正常显示 `Ctrl`，点击后会高亮锁定；点击 `B` 后可继续操作，之后再次点击 `Ctrl` 可解除锁定。
- 自定义主题第一次修改时曾因 YAML 底行缩进多一个空格而被应用拒绝；已修正并重新部署，未触及备份文件。

tmux 是否切换窗口，最终还取决于终端或 SSH 客户端是否把 Android 的 Ctrl 组合事件传递给远端终端；Trime 端的 Ctrl 修饰键和 `Ctrl+B` 输入链路已在真机键盘上验证。

## 中英文切换兼容性复核（2026-09-23）

- 真机点击自定义主题的“方案/中文”键后，日志显示处理成 `VoidSymbol`，Rime 状态仍为 `isAsciiMode=false`，因此没有发生切换。
- 自定义文件第 938 行仍使用旧写法：`Mode_switch: {toggle: ascii_mode, send: Mode_switch, ...}`。
- 当前版本内置 `trime.yaml` 使用 `send: SWITCH_CHARSET`。切换到内置主题后，真机可以正常在“中文”和“英文”之间切换，证明运行时功能正常。
- 因此这次失效是旧自定义主题配置与新版内置按键定义不一致造成的，不是 APK 的中英文切换实现整体失效。已将该配置中所有旧的 `send: Mode_switch` 替换为新版的 `send: SWITCH_CHARSET`，包括 `Mode_switch`、`Zenkaku_Hankaku`、`Henkan`、`Charset_switch`、`Punct_switch` 以及候选/注释/助记开关。
- 修复后真机点击切换键可得到 `ascii_mode=true`（键面显示“西文”），再次点击可得到 `ascii_mode=false`（恢复“中文”）。

## 配置清理与最终真机验证（2026-09-23）

- 模式键状态文字已由“中文/西文”改为“中文/英文”；并将相关状态键的旧 `send: Mode_switch` 统一改为新版 `send: SWITCH_CHARSET`。
- 清理了新版运行时不再使用或会被忽略的旧配置项，包括旧 `layout`/`window` 样式字段、预览/本地化兼容字段、`android_keys` 参考块等；以 `comment_position: TOP` 保留注释位置设置。
- 同步更新新版预设命令：`SEARCH`、`FUNCTION`、`HELP`、`MENU`，并将应用入口改为 `com.osfans.trime.debug`；`Deploy` 和 `Sync` 改用新版 `FUNCTION` 调用方式。
- 最终配置已推送到手机公共目录和应用专属目录，两处 MD5 均为 `2a666a51ba6ab240f29f2cecfd70b3bb`。重启输入法后，真机键盘显示 `Ctrl`，模式键可在“中文”和“英文”之间切换。
- 原始配置与最终配置已分别保存为 [`默认配置调整.v3.3.0.yaml`](默认配置调整.v3.3.0.yaml) 和 [`默认配置调整.v3.3.13.yaml`](默认配置调整.v3.3.13.yaml)。
- GitHub fork：`https://github.com/lxmghct/trime.git`，目标分支为 `develop`；本次收尾文档提交后推送。
