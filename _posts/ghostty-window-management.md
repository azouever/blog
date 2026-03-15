---

title: Ghostty + 窗口管理：我的多任务并行工作流

date: 2026-03-15 10:00:00

---

用 Ghostty + Hammerspoon + Karabiner 并行处理多个 AI 任务的实践

## 为什么写这篇

用 Claude Code / Codex 这种 AI 编程助手，一个很爽的场景是**多任务并行**：

- 窗口 A：让它帮我重构一段代码
- 窗口 B：让另一个 Agent 查文档
- 窗口 C：跑测试看结果
- 窗口 D：开着备用，随时有新需求

**但问题来了**：窗口一多，管理就成了瓶颈。

- 切来切去累眼睛
- 分屏找不到快捷键
- 伸手摸鼠标 —— 节奏就断了

于是就需要一套**键盘驱动的窗口管理方案**。主流有两种：

| 方案 | 平台 | 特点 |
|------|------|------|
| **Tmux** | 跨平台（Linux/macOS/终端） | 终端内分屏，会话持久化 |
| **Hammerspoon** | macOS 原生 | 系统级窗口管理，灵活性更强 |

我选 **Hammerspoon**，因为它是**系统级窗口管理**——不只是终端，任意应用都能分屏、切换。而 Tmux 只能管终端内的 session。

> 如果你主要在 Linux 服务器上工作，或者需要在多台机器上保持一致的体验，Tmux 是更通用的选择。它的 sessions / windows / panes 模型也很强大，配合 prefix 键（默认 Ctrl+b）可以做到：
> - `Ctrl+b %` 垂直分屏
> - `Ctrl+b "` 水平分屏
> - `Ctrl+b 数字` 切换窗口
> - `Ctrl+b d`  detach 会话

但对于 macOS 用户来说，Hammerspoon + Karabiner 的组合更贴合系统，体验也更顺滑。

---

## 为什么是 Ghostty？

用过的终端不少：iTerm2、Alacritty、Kitty……最后锁定 **Ghostty**，原因很简单：

- **快**：Rust 实现，启动即用
- **原生 macOS**：Metal 渲染，续航友好
- **配置简单**：一个文件走天下

但真正让我离不开的，是围绕 Ghostty 搭建的**「无鼠标工作流」**。

---

## 我的 Ghostty 配置哲学

### 1. 禁用 Tab，永远新窗口

```bash
# 不需要 Tab 栏
window-show-tab-bar = never

# Cmd+N 新窗口，Cmd+T 禁用（不用 Tab）
keybind = cmd+n=new_window
keybind = cmd+t=unbind
```

**核心理念**：一个任务一个窗口，用完即关。不需要 Ctrl+Tab 切换得眼都花。

### 2. 视觉舒适度

```bash
# 字体：FiraCode Nerd Font，14px
font-family = "FiraCode Nerd Font"
font-size = 14

# 轻微透明，视觉更柔和
background-opacity = 0.96

# 窗口内边距
window-padding-x = 12
window-padding-y = 10
```

### 3. 光标与交互

```bash
cursor-style = block
cursor-style-blink = false
copy-on-select = clipboard  # 选中即复制
```

---

## 窗口管理：Hyper + HJKL

不用鼠标，怎么分屏？**Hammerspoon**。

我的核心快捷键：

| 快捷键 | 功能 |
|--------|------|
| `Hyper + H` | 左半屏 |
| `Hyper + J` | 下半屏 |
| `Hyper + K` | 上半屏 |
| `Hyper + L` | 右半屏 |
| `Hyper + Y/U/I/O` | 四角布局 |
| `Hyper + M` | 最大化 |
| `Hyper + N` | 居中 |

> Hyper = Shift + Ctrl + Alt + Cmd（右键 Command 映射）

**效果**：
```
┌─────────────┬─────────────┐
│             │             │
│   Hyper+L   │   Hyper+L   │
│             │             │
├─────────────┼─────────────┤
│             │             │
│   Hyper+J   │   Hyper+J   │
│             │             │
└─────────────┴─────────────┘
```

一键切换，干净利落。

---

## HHKB + SpaceFN 键位映射

键盘是 **HHKB BLE**（YDKB 改），Karabiner 配置：

### 核心映射

| 按键 | 映射 |
|------|------|
| Right Command | Hyper |
| Right Shift | Small Hyper |
| Tab（按住） | Left Hyper |
| Space（按住） | SpaceFN 模式 |

### SpaceFN 键位

```
Space + H/J/K/L   → 方向键 ←↓↑→
Space + U/O       → Home/End
Space + Y/N       → Page Up/Down
Space + W/B       → 词尾/词首（IDE 神技）
Space + D         → Delete
Space + E         → Escape
Space + Enter     → Option+Enter（IDE 快速修复）
```

**按一下空格 = 普通空格，按住空格 = 进入方向键模式**。这才是真正的「手不离位」。

### 输入法切换：Control 单击

- **单击** `Left Control` = 切换输入法（Rime 小鹤音形）
- **按住** = 仍是 Control 键

MacBook 内置键盘同理：`Caps Lock` 单击 = 切输入法。

---

## 语音输入：Typeless + Spokenly

偶尔想解放双手？**双语音方案**：

| 工具 | 场景 |
|------|------|
| **Typeless** | 语音触发快捷键，说"新建窗口"自动执行 |
| **Spokenly** | 实时语音转文字，写文档时直接口述 |

Typeless 配合 Hammerspoon可以做很多事，比如：
- 「打开 Safari」
- 「切换到左边窗口」
- 「最大化窗口」

---

## 完整工作流一览

```
┌─────────────────────────────────────────────────┐
│  Ghostty（终端）                                 │
│  ├── 禁用 Tab，全部新窗口                        │
│  ├── Cmd+N 新窗口                               │
│  └── 选中即复制                                 │
├─────────────────────────────────────────────────┤
│  Hammerspoon（窗口管理）                        │
│  ├── Hyper + HJKL 分屏                         │
│  ├── Hyper + YUO 角落布局                      │
│  └── Hyper + M 最大化                          │
├─────────────────────────────────────────────────┤
│  Karabiner（键盘映射）                          │
│  ├── Right Cmd → Hyper                        │
│  ├── Space → SpaceFN（方向键）                │
│  └── Control 单击 → 切输入法                  │
├─────────────────────────────────────────────────┤
│  语音输入（Typeless + Spokenly）               │
│  ├── 语音触发快捷键                            │
│  └── 实时语音转文字                            │
└─────────────────────────────────────────────────┘
```

---

## 总结

这套组合下来，多窗口并行变得更顺手：

1. **键盘分屏** —— Hyper + HJKL 搞定，不用伸手摸鼠标
2. **方向键随身** —— SpaceFN 模式，手不用往右下挪
3. **输入法切换不打断** —— Control 单击切换，手不离位
4. **偶尔动口** —— 语音输入兜底

Ghostty 是入口，Hammerspoon 是骨架，Karabiner 是肌肉，语音是 Bonus。

---

> 工具不重要，习惯才是关键。

## 配置位置

- Ghostty: `~/macos-automation/terminal/ghostty/config`
- Hammerspoon: `~/.hammerspoon` → `~/macos-automation/hammerspoon/`
- Karabiner: `~/.config/karabiner/karabiner.json`
