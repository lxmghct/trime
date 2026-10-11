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
- 已在 `/storage/emulated/0/rime/默认配置调整.trime.yaml` 中加入 `Control_L` 预设和新版中英文切换配置；修饰键调整前版本 MD5 为 `2a666a51ba6ab240f29f2cecfd70b3bb`，应用专属同步目录中的副本 MD5 相同。
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
- 修饰键调整前的最终配置已推送到手机公共目录和应用专属目录，两处 MD5 均为 `2a666a51ba6ab240f29f2cecfd70b3bb`。重启输入法后，真机键盘显示 `Ctrl`，模式键可在“中文”和“英文”之间切换。
- 原始配置与最终配置已分别保存为 [`默认配置调整.v3.3.0.yaml`](默认配置调整.v3.3.0.yaml) 和 [`默认配置调整.v3.3.13.yaml`](默认配置调整.v3.3.13.yaml)。
- GitHub fork：`https://github.com/lxmghct/trime.git`，目标分支为 `develop`；本次收尾文档提交后推送。

## dev-config.yaml 修饰键调整（2026-10-12）

- 已在修改前备份手机配置：`/storage/emulated/0/rime/默认配置调整.trime.yaml.codex-before-modifier-20261012-1`，备份 MD5 为 `2a666a51ba6ab240f29f2cecfd70b3bb`。
- 方案名称已改为 `dev-config.yaml`；最终配置已保存为 [`dev-config.yaml`](dev-config.yaml)，最终 MD5 为 `de9dbe4cf4b382f3b0a9618741bfb91f`。
- `Shift_L` 已由 `shift_lock: ascii_long` 改为 `shift_lock: long`：长按进入 CapsLock 式 Shift 锁定，再次长按关闭。真机已验证锁定状态、高亮显示和再次长按解除。
- W 键的 `-` 长按/上划动作改为直接提交文本，避免 Shift 状态下被 Android 按键映射成 `_`；真机在 Shift 锁定状态下验证结果为 `-`。
- 源码已支持 Alt：`KeyCode` 包含 `Alt_L/Alt_R`，`KeyModifier` 能生成 `META_ALT_ON`，输入法服务也会发送 Alt 按键事件。因此无需改动 APK 源码，只需在主题中增加 `Alt_L` 预设。
- Alt 已放到 Ctrl 键的长按和上划动作上：长按/上划 Ctrl 后按 A，真机日志分别确认了 `process key: Alt+a` 和 `KeyModifiers(modifiers=8)`。

## 符号直提交与“电脑”键盘（2026-10-12）

- Shift 状态下的符号问题并非只有 W 键：数字行的 `! @ # $ % ^ & * ( )`，以及 `_ - + = | \\ [ ] { } : ;`、反引号、`~ " ' < > ?` 等长按/上划符号，统一改为 `commit` 动作并保留原标签。这样动作提交的是目标字符，不会再次按当前 Shift 状态解释；真机验证 R 长按可得到 `=`，不再变成 `+`。
- 新增 `preset_keyboards.computer`，名称为“电脑”。布局包含 Esc、Tab、Caps、Ins/Del、Home/End、PgUp/PgDn、F1–F12、Ctrl/Alt/Meta/Shift、方向键、常用 ASCII 符号、空格、回车和退格；右下角“返回”回到 `.default`。
- 默认键盘 Ctrl 的长按和上划现在进入 `电脑`，不再直接触发 Alt。电脑键盘中提供真实的 `Alt_L` 点击键，因而 Alt 可以独立锁定并参与后续组合键。
- 键盘切换前后修饰键状态由 `KeyboardWindow` 共享并恢复到目标键盘；新增 `Keyboard.restoreModifier()` 和 `Key.restoreOn()`，避免缓存键盘各自持有状态导致 Alt/Ctrl 丢失。真机实测“电脑”中点击 Alt → 右下角返回 → 点击 A，日志收到 `process key: Alt+a`，`modifiers=8`。

## 最新构建与真机记录（2026-10-12）

- 使用 `BUILD_ABI=arm64-v8a`、NDK `28.0.13004108` 构建 debug APK；`./gradlew :app:compileDebugKotlin` 和 arm64 debug APK 构建通过。全 ABI 构建仍受到工程既有 Lua `fseeko/ftello` 与 Android NDK 的兼容错误影响，未把第三方临时修补带入提交。
- 真机配置修改前备份为 `/storage/emulated/0/rime/默认配置调整.trime.yaml.codex-before-computer-20261012-1`；标签微调前又保存了 `/storage/emulated/0/rime/默认配置调整.trime.yaml.codex-before-labels-20261012-1`。
- 最终配置已同步到公共目录和应用专属目录，最终 MD5 为 `36d24369582675452db2e02e541f97bb`；同步副本一致。最终副本为本文件对应的 [`dev-config.yaml`](dev-config.yaml)。
