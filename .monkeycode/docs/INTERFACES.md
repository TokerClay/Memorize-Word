# 接口文档

CET 词汇是一个纯前端 PWA，无 HTTP API。本文档描述其内部接口：IndexedDB 数据契约、JSON 词库格式、DOM 事件系统、CSS 变量接口。

---

## IndexedDB 数据契约

### 数据库配置

| 属性 | 值 |
|------|-----|
| 数据库名称 | `cet4_study` |
| 版本 | `3` |
| Object Stores | `profile`, `data`, `settings` |

### Object Store: `profile`

**Key: `active`**

```typescript
type ActiveKey = 'active'
type ActiveValue = string  // 当前活跃用户的 uid
```

**Key: `user:{uid}`**

```typescript
interface UserProfile {
  username: string       // 用户显示的昵称
  uid: string            // 用户唯一标识
  activeBank: string     // 当前选中的词库 key，如 'cet4'
  createdAt: string      // ISO 时间戳
}
```

### Object Store: `data`

**Key: `data:{uid}:{bank}`**

```typescript
interface StudyData {
  today: string                    // 今日日期，ISO 格式 YYYY-MM-DD
  todaySwiped: number              // 今日已刷词数
  totalSwiped: number              // 累计刷词数（所有模式）
  rounds: number                   // 完成轮次数
  streakDays: number               // 连续打卡天数
  checkInDates: string[]           // 打卡日期列表 ISO YYYY-MM-DD
  orderSwiped: number              // 顺序模式下累计刷词数
  shuffleSwiped: number            // 乱序模式下累计刷词数
  bookmarks: string[]              // 生词本单词列表
  progress: StudyProgress          // 各模式刷词位置
}

interface StudyProgress {
  orderIndex: number               // 顺序模式当前 index
  shuffleIndex: number             // 乱序模式当前 index
  shuffleOrder: number[]           // 当前乱序排列
  bookmarkIndex?: number           // 生词本当前 index
}
```

### Object Store: `settings`

**Key: `settings:{uid}:{bank}`**

```typescript
interface UserSettings {
  trackMode: 'both' | 'order' | 'shuffle'  // 统计追踪模式
  dailyGoal: number                          // 每日目标词数，默认 50
  ttsEnabled: boolean                        // TTS 朗读开关
  goalSet: boolean                           // 是否已完成首次目标设置
}
```

**Key: `theme:{uid}`**

```typescript
type ThemeValue = 'dark' | 'light'
```

---

## JSON 词库格式

词库文件（`words_cet4.json`、`words_cet6.json`）为 JSON 数组，每个元素结构如下：

```typescript
interface WordEntry {
  word: string       // 单词，如 "abandon"
  pos: string        // 词性，如 "v."
  meaning: string    // 中文释义，如 "放弃"
  ukphone: string    // 英式发音，如 "'ə'bændən"
  usphone?: string   // 美式发音（可选）
}
```

**示例**:

```json
{
  "word": "abandon",
  "pos": "v.",
  "meaning": "放弃",
  "ukphone": "'ə'bændən"
}
```

---

## DOM 事件系统

### 页面导航

| 触发元素 | 事件类型 | 处理函数 | 行号 |
|---------|---------|---------|------|
| `.tab-item` 点击 | `click` | `switchPage(data-page)` | 868 |
| 返回按钮 `#backBtn` | `click` | `switchPage('home')` | 824 |

### 学习页交互

| 触发元素 | 事件类型 | 处理函数 | 并发保护 |
|---------|---------|---------|---------|
| `#nextBtn` 点击 | `click` | `nextWord()` | `nextWordBusy` 互斥锁 |
| `#prevBtn` 点击 | `click` | `prevWord()` | `prevWordBusy` 互斥锁 |
| 学习页触摸 swipe | `touchstart/move/end` | 滑动方向判断 | `spTouchActive` 去抖 |
| 学习页点击 | `click` | 左半屏→prev, 右半屏→next | `spTouchActive` 防重复 |
| 键盘 Space/ArrowRight | `keydown` | `nextWord()` | 检查 modal 是否显示 |
| 键盘 ArrowLeft | `keydown` | `prevWord()` | 同上 |

**触摸交互规则**:

| 手势 | 条件 | 动作 |
|------|------|------|
| 点击（<300ms，<15px 移动） | 左半屏 | `prevWord()` |
| 点击（<300ms，<15px 移动） | 右半屏 | `nextWord()` |
| 左滑（dx < 0，|dx| > |dy|，>30px） | — | `nextWord()` |
| 右滑（dx > 0，|dx| > |dy|，>30px，且起手 x > 15） | — | `prevWord()` |
| 上滑（dy < 0，|dy| > |dx|，>30px） | — | `toggleBookmark()` |

> 右滑起手 x > 15 的限制用于避免与 iOS 左边缘返回手势冲突（约 15px 安全区）。

**键盘事件屏蔽条件**: 当以下任一 Modal 显示时，键盘事件不触发学习页动作：
- `#completeModal`（完成弹窗）
- `#goalModal`（目标设置弹窗）
- `#settingsModal`（设置弹窗）

### 学习功能按钮

| 按钮 | 事件 | 处理函数 | 行号 |
|------|------|---------|------|
| `#ttsBtn`（朗读） | `click` | `speakWord()` | 827 |
| `#bookmarkBtn` / `#favBtn`（收藏） | `click` | `toggleBookmark()` | 828-833 |

### 首页

| 触发元素 | 事件 | 处理函数 | 行号 |
|---------|------|---------|------|
| `#btnOrder`（顺序） | `click` | `startStudy('order')` | 822 |
| `#btnShuffle`（乱序） | `click` | `startStudy('shuffle')` | 823 |
| `#bookmarkStudyBtn`（生词本） | `click` | `startStudy('bookmark')` | 879-881 |
| `#homeWord`（首页单词） | `click` | 展开词性和释义 | 866 |
| `#home-page` 背景点击 | `click` | `showHomeWord()` 随机换词 | 865 |

### 主题

| 触发元素 | 事件 | 处理函数 | 行号 |
|---------|------|---------|------|
| `#themeBtn`（主题按钮） | `click` | 切换 .dark class + 保存 | 887-893 |
| `window.matchMedia` 监听 | `change` | 跟随系统主题（仅当未手动设置） | 1087-1091 |

### 设置

| 触发元素 | 事件 | 处理函数 |
|---------|------|---------|
| `#settingsBtn` | `click` | 打开设置弹窗，渲染目标选择器 |
| `#modalClose` | `click` | 关闭设置弹窗 |
| `.modal-option`（追踪模式） | `click` | 切换 trackMode 并保存 |
| `#ttsToggle` | `click` | 切换 TTS 开关并保存 |
| 目标选择器拖拽 | `touch/mouse` | `bindGoalPicker` 滚动选择目标值 |

### 词库

| 触发元素 | 事件 | 处理函数 | 行号 |
|---------|------|---------|------|
| `#bankTitle` | `click` | 打开词库选择弹窗 | 1044 |
| 词库选择按钮 | `click` | `bankSelected(bank)` | 1003-1005 |
| `#bankSelectConfirm` | `click` | 确认切换/新用户创建 | 1022-1035 |

### 全局

| 事件 | 处理 | 目的 |
|------|------|------|
| `gesturestart` | `e.preventDefault()` | 禁用 iOS 双指缩放和手势 |

---

## Voice (Web Speech API)

```typescript
function speakWord(): void
```

- 使用 `window.speechSynthesis`
- 朗读语言: `en-US`
- 语速: `0.85`
- 调用前先 `cancel()` 清除队列中待朗读的语音
- 依赖 `settings.ttsEnabled` 开关控制是否自动朗读

---

## CSS 变量接口 (主题 Token)

```css
:root {
  --bg: #fff;               /* 页面背景 */
  --text: #000;             /* 主文字颜色 */
  --text-secondary: #888;   /* 次要文字 */
  --text-tertiary: #aaa;    /* 辅助文字 */
  --border: #e0e0e0;        /* 边框和分隔 */
  --btn-bg: #fff;           /* 按钮背景 */
  --btn-hover: #f5f5f5;     /* 按钮悬停 */
  --btn-active: #e8e8e8;    /* 按钮按下 */
  --accent: #000;           /* 强调色 */
}

.dark {
  --bg: #1a1a1a;
  --text: #e0e0e0;
  --text-secondary: #777;
  --text-tertiary: #555;
  --border: #333;
  --btn-bg: #1a1a1a;
  --btn-hover: #2a2a2a;
  --btn-active: #333;
  --accent: #e0e0e0;
}
```

所有组件颜色通过 `var(--xxx)` 引用这些 token，切换主题只需在 `<html>` 元素上添加/移除 `.dark` class。

---

## XSS 防护

**escapeHTML() 函数**:

```javascript
function escapeHTML(s) {
  var d = document.createElement('div');
  d.textContent = s;
  return d.innerHTML;
}
```

通过 DOM `textContent` 自动转义 `<`、`>`、`&`、`"`、`'` 五个字符，不依赖正则替换，防止遗漏。

**适用位置**:

| 注入点 | 转义内容 | 行号 |
|--------|---------|------|
| `#homeDetail` | `w.pos`, `w.meaning` | 866 |
| `#bankTitle` | `BANKS[bank].name` | 713 |
| `#bookmarkTags` | 生词本单词名 | 739 |

**CSP 头**:

```
default-src 'self' 'unsafe-inline' 'unsafe-eval';
img-src data:;
connect-src 'self';
media-src 'self' blob:;
```

禁止外部脚本/样式/资源加载，只允许自域内的 fetch 请求（词库 JSON）。
