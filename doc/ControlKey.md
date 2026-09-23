<!--
SPDX-FileCopyrightText: 2026 Rime community

SPDX-License-Identifier: GPL-3.0-or-later
-->

# 独立 Ctrl 键

## 结论

Trime 的运行时已经具备独立修饰键所需的处理链路：

1. `KeyCode` 可以把 Rime 的 `Control_L` / `Control_R` 映射为 Android 的 Ctrl key code。
2. `KeyAction` 将修饰键识别为 modifier key，`Keyboard` 保存 Ctrl 的锁定状态。
3. 下一个普通按键会继承键盘的 Ctrl 状态。
4. 如果 Rime 没有消费这个组合键，`TrimeInputMethodService` 会将它作为带 `META_CTRL_ON` 的 Android `KeyEvent` 发给当前应用。

之前默认 `trime.yaml` 只有 `Control+a`、`Control+x` 等预设组合键，没有可点击的 `Control_L` 按键，因此普通用户无法从默认键盘发出 `Ctrl+B`。这不是 Rime key table 或 Android key event 底层不支持，而是默认主题缺少入口；上游已有的 `tongwenfeng.trime.yaml` 也曾定义 `Control_L`，但其预设没有锁定语义。

## 本次改动

- 在内置 `trime.yaml` 中加入 `Control_L` 和 `Control_R` 预设，显示为 `Ctrl`，点击锁定/解除锁定。
- 在默认键盘底部加入 `Control_L`；空格宽度由 30 调整为 20，保持该行总宽度不变。
- 修正同文风主题中已有的 `Control_L` 预设，使其显示为 `Ctrl` 并具有点击锁定语义。
- 增加 `KeyActionTest`，验证独立 Ctrl 键的 key code、锁定语义和 modifier mask。

点击 `Ctrl` 使其高亮，再点击 `B`，发送给终端的目标事件应等价于 `Ctrl+B`；再次点击 `Ctrl` 可解除锁定。由于 tmux 前缀通常是控制字符 `0x02`，最终是否切换窗口仍取决于终端/SSH 客户端是否把 Android key event 正确传递到远端终端。

## 自定义主题

自定义键盘可以复用相同的动作名：

```yaml
preset_keys:
  Control_L: {label: Ctrl, send: Control_L, shift_lock: click}

# 在目标 keyboard 的 keys 列表中加入
- {click: Control_L}
```

如果主题已经继承内置 `preset_keys`，通常只需加入键盘项；如果主题覆盖了 `preset_keys`，则同时保留上面的预设定义。

## 验证记录

- 基线：`git clone https://github.com/osfans/trime.git`，`develop` 分支提交 `f3ca316f`。
- 环境：JDK 21.0.5、Gradle 9.7.1、Android SDK `D:\dev\Android\SDK`、NDK 28.0.13004108。
- 基线单元测试：`./gradlew.bat :app:testDebugUnitTest --no-daemon`，成功。
- 修改后应重复运行上述单元测试，并构建 debug APK；可用 `android-debug` 技能将 APK 安装到 `Pixel_6_API_30` 做 UI/事件验证。
