# 业火五笔输入法 (Fire) 功能资产文档

> 版本：v0.18.0 | Bundle ID: com.qwertyyb.inputmethod.Fire  
> 仓库：https://github.com/qwertyyb/Fire

---

## 一、项目概述

业火五笔是一款运行在 macOS 13.0+ 平台的开源五笔输入法，支持 **86版/98版/06版** 五笔编码方案及拼音/五笔拼音混合模式。使用 Apple InputMethodKit 框架深度集成系统输入法机制，以 SwiftUI 构建现代化候选词界面。

---

## 二、技术栈

| 层级 | 技术 |
|------|------|
| 语言 | **Swift 5.9+** (主程序)、**C++** (码表构建工具 TableBuilder)、**Bash** (构建/打包/签名/公证脚本) |
| UI 框架 | **SwiftUI** (候选词视图、偏好设置面板)、**AppKit/NSWindow** (候选窗浮层)、**NSHostingView** 桥接 |
| 输入法框架 | **Apple InputMethodKit** (`IMKInputController`) |
| 数据库 | **SQLCipher** (AES-256 加密 SQLite)，通过 `sqlite3.h` C API 操作 |
| 状态持久化 | **Defaults** (基于 UserDefaults 的类型安全键值存储，支持 Codable 类型) |
| 偏好设置 | **原生 SwiftUI** (NavigationSplitView 标签页布局，替换了 Settings SPM 库) |
| 统计图表 | **Swift Charts** (`Chart { LineMark + AreaMark + PointMark }`，macOS 13+ 原生框架) |
| 自动更新 | **Sparkle** (`SUFeedURL` + `SUPublicEDKey`) |
| 安全密钥 | **KeychainSwift** + **NanoID** (生成统计数据库加密密钥) |
| 输入源管理 | **Carbon/TIS** API (TextInputSources 注册/启用/选择/监听) |
| 构建系统 | Xcode + 自制 Bash 脚本链 (编译 SQLCipher → 构建 .app → 构建 .pkg → 公证 → 生成 appcast) |

---

## 三、主要功能模块

### 1. 输入事件处理链（核心引擎）

**文件**: `Fire/FireInputController.swift`  
**类型**: `IMKInputController` 子类

按键事件通过 `Utils.shared.processHandlers()` 串联为 **责任链模式**，每个 handler 返回 `Bool?`：

- `true` = 已消费事件
- `false` = 需放行给系统
- `nil` = 交给下一 handler

**完整处理链顺序 (15 步)**：

| # | Handler | 功能 |
|---|---------|------|
| 1 | **deleteConfirmHandler** | 删除二次确认态：Enter 确认删除、Esc 取消、其他键取消后继续处理 |
| 2 | **combineHandler** | 快速组词模式：←键增字、→键减字、Enter 确认组词、Esc 退出 |
| 3 | **hotkeyHandler** | 快捷键：`Ctrl+Shift+数字` 删除候选词、`Ctrl+数字` 调序、`Ctrl+=` 进入组词模式 |
| 4 | **flagChangedHandler** | Shift/CapsLock 等修饰键：检测键弹起切换中/英文模式 |
| 5 | **enModeHandler** | 英文模式：直接放行事件给系统 |
| 6 | **predictorHandler** | 智能预测：数字后 `.` 转小数点、`:` 转半角冒号、中英文间智能空格 |
| 7 | **pageKeyHandler** | 翻页：`+/-`、方向键(按候选方向自适应)翻页 |
| 8 | **deleteKeyHandler** | Backspace 删除原码最后一字符 |
| 9 | **charKeyHandler** | 字母键：附加到原码末尾 |
| 10 | **numberKeyHandlder** | 数字键：选择对应序号的候选词；无原码时记录"上次输入为数字" |
| 11 | **escKeyHandler** | Esc 清空原码并关闭候选窗 |
| 12 | **enterKeyHandler** | 回车上屏原码字符（或临时英文模式下上屏候选） |
| 13 | **spaceKeyHandler** | 空格键上屏第一个候选词 |
| 14 | **extraCandidateKeyHandler** | 额外选键：可用 `;'` 或 `,.` 选择第 2/3 个候选 |
| 15 | **punctuationKeyHandler** | 标点符号：全角转换 + 临时英文模式（;键触发） |

**输入模式**：全局 `Fire.shared.inputMode` 管理，支持 `.zhhans`（中文）和 `.enUS`（英文）切换，切换时可显示 Toast 或光标跟随提示。

---

### 2. 码表引擎 (DictManager)

**文件**: `Fire/DictManager.swift`  
**数据结构**: 单例模式

#### 2.1 核心查询

使用 SQLCipher 操作 `wb_py_dict` 表：
- 候选词类型：`wb`(五笔词)、`py`(拼音词)、`user`(用户词)、`blocked`(屏蔽标记)、`placeholder`(运行时占位)
- 查询使用 `GLOB` + `?`/`*` 通配符（z键通配查询用 `?` 代替 `z`）
- 支持精确匹配（`enableExactMatch`）和逐码模糊匹配
- 支持分页（多查一个以判断是否有下一页）
- 满 4 码唯一候选词自动上屏（`wubiAutoCommit`）

**查询 SQL** 动态拼接核心逻辑：
```sql
SELECT max(wbcode), text, type, min(query), max(s86/s98/s06), max(py86/py98/py06)
FROM wb_py_dict
WHERE query GLOB :queryLike
  AND type IN ('wb','user')  -- 依 codeMode 调整
  AND text NOT IN (SELECT text FROM wb_py_dict WHERE type='blocked')
GROUP BY text ORDER BY query, id
LIMIT :offset, :count
```

#### 2.2 用户词管理
- **调序**：用负 ID 插入用户词到最前（`prependCandidate`）
- **组词**（快速加词）：`makeWubiWordCode(for:)` 按五笔词组取码规则：
  - 2 字：字1前2码 + 字2前2码
  - 3 字：字1首码 + 字2首码 + 字3前2码
  - ≥4 字：前三字各首码 + 末字首码
- **删除**：插入 `type='blocked'` 屏蔽 + 删除 `type='user'` 记录
- **导入**：解析 `编码 词1 词2 ...` 格式文本
- **导出**：导出完整码表/用户词库
- **日期变量**：用户词文本中可用 `{yyyy}/{MM}/{dd}/{HH}/{mm}/{ss}` 模板

#### 2.3 拆字数据
- 支持 86/98/06 三种拆字方案（`s86/s98/s06` 列）
- 候选提示可显示拆根字形（需 `黑体字根.ttf` 字体）
- 用户词自动从原词库复制拆字数据，多字词按规则组合生成

---

### 3. 候选窗口系统

#### CandidatesWindow (`Fire/CandidatesWindow.swift`)
- 无边框 NSWindow，`CGShieldingWindowLevel` 浮动层级
- NSHostingView 承载 SwiftUI 视图
- 自动适配屏幕边界（`limitFrameInScreen` / `transformTopLeft`）
- 通过 NOTIFICATION 监听：候选选择、翻页、输入模式切换
- 全局 `flagsChanged` 监视器（处理 LaunchPad 等场景的 Shift 监听缺失）

#### CandidatesView (`Fire/CandidatesView.swift`)
- 支持**横向/竖向**排列（`CandidatesDirection`）
- 候选提示模式：
  - `none`：无提示
  - `wubiCode`：显示剩余编码
  - `spelling`：显示拆字字根（使用黑体字根字体）
  - `pinyin`：显示拼音
- 翻页指示器（已选/未选分页圆点）
- 鼠标可点击选择候选词
- 完整的主题系统支持（颜色、字体、间距）

---

### 4. 偏好设置面板（7 个原生 SwiftUI 面板）

**实现**：`Fire/Preferences/NativePreferencesView.swift` + `Fire/Preferences/FirePreferencesController.swift`  
**说明**：v0.18.0 已从 Settings SPM 库迁移至原生 SwiftUI `NavigationSplitView`，减少外部依赖，提升加载速度和稳定性。

| 面板 | 文件 | 功能说明 |
|------|------|---------|
| **基本** | `GeneralPane.swift` | 编码方案(wubi/pinyin/wubiPinyin)、候选词数(2-9)、翻页键、中英文切换键、中文输出模式（简/繁）、候选方向、额外选键、禁用英文模式、禁用临时英文、z键查询、精确匹配、GBK生僻字、四码唯一自动上屏、中英文间空格 |
| **标点符号** | `PunctuationPane.swift` | 模式(全角/半角/自定义)、自定义标点映射编辑 |
| **高级** | `ThesaurusPane.swift` | 码表文件路径、重建词库、导出完整码表 |
| **主题** | `ThemePane.swift` | 导入/导出/切换主题、Light/Dark 独立配置、`LabeledContent` 布局 |
| **应用** | `ApplicationPane.swift` | 为特定 App 设置强制输入模式（中文/英文/最近使用）、应用输入模式记忆开关、切换提示时机 |
| **用户词库** | `UserDictPane.swift` | 浏览/编辑/导入/导出用户词库 |
| **统计** | `StatisticsPane.swift` | 输入字数统计图表（Swift Charts 框架，替代原自定义 Path 绘图）、总字数、清除数据 |

---

### 5. 主题系统

**文件**: `Fire/Theme/ThemeConfig.swift`

- 完整的 **Light/Dark** 双主题支持
- 可配置项（`ApperanceThemeConfig`）：
  - 窗口背景颜色、内边距(top/left/right/bottom)、圆角半径
  - 原码文字颜色
  - 候选序号颜色、候选文字颜色、候选编码颜色（普通态/选中态）
  - 页面指示器颜色（可用/禁用态）
  - 字体名称、字号（正文/序号/编码）
  - 毛玻璃效果（`enableLiquidGlass`）
- 主题序列化为 JSON，支持通过 **URL Scheme** 导入：`fire://theme/import?data=<base64url-json>`
- `ColorData` 支持 hex 字符串编解码（3/4/6/8 位 hex → RGBA 分量双编码兼容）

---

### 6. 应用输入模式记忆

**文件**: `Fire/FireInputServer.swift` + `Fire/InputModeCache.swift`

- **`activateServer`/`deactivateServer`**：当切换到不同应用时自动保存/恢复该应用上次的输入模式
- **`InputModeCache`**：LRU 缓存（容量 100 个应用），持久化到 UserDefaults
- **`ApplicationSettingItem`**：可为特定 App 手动设置强制中文/英文/最近使用
- 安全输入场景（密码框）：自动切换到英文模式（`IsSecureEventInputEnabled()`）

---

### 7. CLI 接口

**文件**: `Fire/cli/CLI.swift` + `Fire/cli/CLIServer.swift`

通过 `DistributedNotificationCenter` 与主进程通信，支持命令：

| 命令 | 功能 |
|------|------|
| `--install` | 安装输入法（注册 TIS 输入源） |
| `--build-dict` | 重建码表数据库 |
| `--stop` | 停止/卸载输入法 |
| `--get-mode` | 获取当前输入模式（中/英） |
| `--set-mode enUS\|zhhans [showTip]` | 设置输入模式 |

---

### 8. 标点符号处理

**文件**: `Fire/PunctuationConversion.swift`

- 三模式：全角(中文)、半角(英文)、自定义
- 智能引号匹配：首次按 `'` 出 `'`，再按出 `'`；`"` 同理
- 智能方括号匹配：`{` 交替输出 `「`/`『`，`}` 匹配输出 `」`/`』`
- 数字后智能转换：`。` → `.`、`：` → `:`（可独立开关）
- 中英文间智能空格插入（`enableWhitespaceBetweenZhEn`）

---

### 9. 打字统计

**文件**: `Fire/Utils/Statistics.swift`

- 独立 SQLCipher 加密数据库 `statistics.db`（Keychain 存储密钥）
- 监听 `Fire.candidateInserted` 通知，记录每次上屏：`text`、`type`、`code`、`createdAt`
- 按日期范围查询输入字数统计（`sum(length(text))`）
- 统计面板以 **Swift Charts** 框架（`LineMark + AreaMark + PointMark`）绘制每日输入统计折线图，替代原 80 行自定义 Path+GeometryReader 手工绘图方案

---

### 10. 系统集成

#### 输入源管理 (`Fire/InputSource.swift`)
- 通过 Carbon `TISRegisterInputSource`/`TISEnableInputSource`/`TISSelectInputSource` API 管理系统级输入法注册与切换
- 监听 `kTISNotifySelectedKeyboardInputSourceChanged` 系统通知
- `getBoolProperty` 辅助方法消除 4 处重复的 `CFBooleanGetValue`+`Unmanaged` 样板代码

#### 状态栏 (`Fire/StatusBar.swift`)
- 菜单栏显示 `中`/`英` 状态图标，点击切换输入模式
- 只在当前输入法被选中时显示
- 可按 `showInputModeStatus` 开关隐藏

#### 自动更新
- Sparkle 框架集成
- 安装包通过 `scripts/` 构建脚本链完成签名和苹果公证

---

### 11. 码表构建系统

**文件**: `Fire/Table/build.swift` (Swift 入口) + `TableBuilder/main.cpp` (C++ 工具)

`TableBuilder` 子进程支持：

| 命令 | 功能 |
|------|------|
| `--create-dict` | 从 .txt 码表创建 SQLite 表 |
| `--combine-dict` | 合并五笔和拼音码表 |
| `--merge-spelling` | 合并拆字数据（s86/s98/s06） |
| `--build-all` | 一步完成全部构建 |

**构建流程**：备份旧库 → 创建新库 → 合并码表 → 合并拆字 → 迁移用户词 → 替换

---

## 四、数据流图

### 按键处理流程

```
用户按键
  │
  ▼
FireInputController.handle(event:)
  │
  ├── CandidatesWindow.shared.inputController = self  // 绑定控制器
  │
  ▼
Utils.processHandlers([15 handlers])
  │
  ├── 1. deleteConfirmHandler ── 删除二次确认处理
  ├── 2. combineHandler ──────── 快速组词处理
  ├── 3. hotkeyHandler ───────── Ctrl+数字/组词快捷键
  ├── 4. flagChangedHandler ──── Shift切换中英文
  ├── 5. enModeHandler ───────── 英文模式直通
  ├── 6. predictorHandler ────── 智能预测（小数点/冒号/空格）
  ├── 7. pageKeyHandler ──────── +/-/方向键翻页
  ├── 8. deleteKeyHandler ────── Backspace删原码
  ├── 9. charKeyHandler ──────── 字母加原码
  ├── 10. numberKeyHandlder ──── 数字选择候选
  ├── 11. escKeyHandler ──────── Esc清空
  ├── 12. enterKeyHandler ────── 回车上屏
  ├── 13. spaceKeyHandler ────── 空格选第1个
  ├── 14. extraCandidateKeyHandler ─ 额外选键(;',.)
  └── 15. punctuationKeyHandler ─ 标点全角转换
       │
       ▼
  Fire.shared.getCandidates(origin:, page:)
       │
       ├── DictManager.shared.getCandidates(query:)
       │     └── SQLCipher → wb_py_dict 表查询
       │
       └── CFStringTransform 简繁转换 + 去重
            │
            ▼
       CandidatesWindow.shared.setCandidates(data, origin, topLeft)
            │
            ▼
       CandidatesView (SwiftUI) 渲染
```

### 应用切换时的输入模式管理

```
用户切换应用
  │
  ▼
FireInputController.activateServer()
  │
  ├── savePreviousClientInputMode()  // 记住上一个应用的输入模式
  │     └── InputModeCache.put(identifier, mode)
  │
  ├── 安全输入检测 (IsSecureEventInputEnabled) → 强制英文
  │
  ├── restoreCurrentClientInputMode()
  │     ├── 查 appSettings 预设 → 如有则强制
  │     └── 查 InputModeCache → 如有则恢复
  │
  └── 必要时显示输入模式提示 (Toast/Tips)
       │
       ▼
FireInputController.deactivateServer()
  ├── insertOriginText()  // 上屏未提交的原码
  └── clean()              // 清空状态
```

---

## 五、关键快捷键一览

| 快捷键 | 功能 |
|--------|------|
| **Shift** | 切换中/英文输入模式（可配置为其他修饰键） |
| **; 键** | 临时英文模式（输入完英文字符后回车或空格上屏） |
| **空格** | 上屏第一个候选词 |
| **数字 1-9** | 选择对应序号候选词 |
| **+/-** | 下/上一页候选词 |
| **方向键 ↑↓/←→** | 翻页（按候选方向自适应） |
| **;** | 选择第 2 个候选（`extraCandidateSelectKeys=semicolonQuote`） |
| **'** | 选择第 3 个候选（同上） |
| **,** | 选择第 2 个候选（`extraCandidateSelectKeys=commaPeriod`） |
| **.** | 选择第 3 个候选（同上） |
| **Ctrl + 数字(1-9)** | 将对应候选词调序至最前（调频） |
| **Ctrl+Shift+数字(1-9)** | 删除对应候选词（两键确认） |
| **Ctrl+=** | 快速加词组词（基于最近上屏的中文词） |
| **Esc** | 清空当前原码 |
| **Backspace** | 删除原码最后一个字符 |
| **回车** | 上屏原码字符 |

---

## 六、文件结构索引

```
Fire/
├── Fire/                           # 主程序源码
│   ├── AppDelegate.swift          # 应用委托、URL Scheme 主题导入、CLI 命令路由
│   ├── Fire.swift                 # Fire 单例（核心状态、候选获取、简繁转换）
│   ├── FireInputController.swift  # 输入控制器（15 步处理链）
│   ├── FireInputServer.swift      # activateServer/deactivateServer（应用输入模式记忆）
│   ├── FireMenu.swift             # 输入法菜单栏
│   ├── DictManager.swift          # 码表引擎（SQLCipher 查询/调序/组词/删除/导入导出）
│   ├── types.swift                # 类型定义（enum/struct/protocol）
│   ├── Types/DefaultsKeys.swift   # Defaults.Keys 键定义（迁移自 types.swift）
│   ├── Utils.swift                # 工具类（事件链处理器、中英文空格判断、Toast/Message）
│   ├── PunctuationConversion.swift # 标点符号转换
│   ├── InputSource.swift          # TIS 输入源管理
│   ├── StatusBar.swift            # 菜单栏状态图标
│   ├── StatusBarController.swift  # 状态栏 autosaveName 修复
│   ├── InputModeCache.swift       # 应用输入模式 LRU 缓存
│   ├── CandidatesWindow.swift     # 候选 NSWindow 浮层
│   ├── CandidatesView.swift       # 候选 SwiftUI 视图
│   ├── RadicalFontManager.swift   # 字根字体加载
│   │
│   ├── Preferences/               # 7 个偏好设置面板 + NativePreferencesView（原生 SwiftUI）
│   ├── Theme/ThemeConfig.swift    # 主题配置数据结构
│   ├── Utils/
│   │   ├── ModifierKeyUpChecker.swift # 修饰键弹起检测
│   │   ├── Statistics.swift       # 打字统计（SQLCipher 加密）
│   │   ├── TipsWindow.swift       # 光标跟随提示窗
│   │   └── ToastWindow.swift      # 屏幕中央 Toast 提示窗
│   ├── cli/CLI.swift & CLIServer.swift # CLI 接口
│   └── Table/build.swift         # 码表构建入口
│
├── TableBuilder/main.cpp          # C++ 码表构建工具
├── sqlcipher/                     # SQLCipher 子模块
├── scripts/                       # 构建/打包/签名/公证脚本
├── package/scripts/postinstall    # 安装后脚本
├── docs/                          # 开发文档（含本文件 FEATURES.md）
└── Resources/font/                # 黑体字根字体
```

---

## 七、第三方依赖

| 依赖 | 用途 |
|------|------|
| **Defaults** | SPM 包，类型安全 UserDefaults 存储，支持 Codable 及自定义类型 |
| **Settings** | ~~SPM 包~~ v0.18.0 已替换为原生 SwiftUI NavigationSplitView |
| **Sparkle** | 自动更新框架（通过 XCFramework 集成） |
| **Swift Charts** | macOS 13+ 原生统计图表框架（替换自定义 Path 绘图） |
| **SQLCipher** | Git 子模块，AES-256 加密 SQLite |
| **KeychainSwift** | Keychain 封装，存储统计数据库密钥 |
| **NanoID** | 生成加密密钥的随机 ID |
| **黑体字根.ttf** | 嵌入的自定义字体，用于显示拆字字形 |

---

## 八、架构设计要点

### 8.1 内存管理

| 组件 | 策略 | 说明 |
|------|------|------|
| 字根表窗口 | `weak var` + `contentView = nil` | 单窗口弱引用跟踪，切换版本时先释放图片再关闭，避免 `NSApp.windows` 累积 |
| Dock 图标 | `setActivationPolicy(.accessory)` | 关闭偏好窗口后自动隐藏 Dock 图标；`activateAsAccessory` 仅在偏好窗口未显示时切换 |
| Toast/Tips 窗口 | `deinit { timer?.invalidate() }` | 确保对象释放后定时器不会触发 |
| InputModeCache | 读路径不写 UserDefaults | 仅在 `put`（数据变更）时持久化，`get` 不触发 I/O |

### 8.2 线程安全

| 场景 | 措施 |
|------|------|
| CLI 命令（DistributedNotification） | `DispatchQueue.main.async` 包裹，确保 UI 操作在主线程 |
| `Bundle.main.bundleIdentifier` | `??` 兜底替代强制解包（3 处） |
| NSRange 长度 | `string.utf16.count` 替代 `string.count`，消除 emoji/组合字符匹配偏移 |

### 8.3 代码质量

- **11 个单例**：输入法进程生命周期 == 应用生命周期，singleton 模式合理
- **责任链模式**：15 个 handler 串接按键事件处理，每 handler 返回 `Bool?`
- **-200 行代码**：通过 Swift Charts、`#Preview` 宏、`LabeledContent` 等新技术消除冗余代码
- **协议抽象**：`Conversion`（标点转换）、`ToastWindowProtocol`（提示窗口）
- **公共 SQLite 辅助函数**：`optString`、`dbErrMsg`、`sqlitePrepare`、`sqliteQuery` 消除模板代码

---
