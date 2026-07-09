# 开发者指南

## 环境搭建

### 前置条件

- 现代浏览器（Chrome 90+ / Safari 14+ / Firefox 90+ / Edge 90+）
- 任意 HTTP 服务器（用于本地开发，因为 `fetch()` 需 HTTP 协议）

### 启动开发服务器

```bash
# Python 内置 HTTP 服务器（推荐）
python3 -m http.server 8000

# 或使用 Node.js
npx serve .

# 或使用 PHP
php -S localhost:8000
```

访问 `http://localhost:8000` 即可。

### 无需构建

项目零依赖、零构建。修改 `index.html` 后刷新浏览器即可看到效果。

---

## 编码规范

### 全局变量命名

所有全局变量均为 `var` 声明，位于 IIFE 闭包内，不会泄漏到全局作用域。

| 用途 | 命名风格 | 示例 |
|------|---------|------|
| 模块级常量 | UPPER_SNAKE | `BANKS`, `DB_NAME`, `DB_VERSION` |
| 状态变量 | camelCase | `currentBank`, `studyMode`, `words` |
| 布尔标志 | camelCase + 描述性前缀 | `isInitialized`, `nextWordBusy`, `spTouchActive` |
| 函数 | camelCase | `loadWords`, `startStudy`, `checkNewDay` |
| DOM ID 引用 | kebab-case | `study-page`, `bookmarkBtn` |

### 代码组织

函数按功能域分组，以注释分隔：

```
// === IndexedDB ===        → 数据库操作
// === Data load/save ===   → 数据存取
// === New day check ===    → 打卡逻辑
// === Word loading ===     → 词库加载
// === Study ===            → 刷词引擎
// === My page ===          → 统计页面
// === Settings ===         → 设置系统
// === Event bindings ===   → 事件绑定
// Theme                    → 主题
```

### 并发保护

涉及异步 `saveData()` 和 `fetch()` 的场景，使用互斥锁模式：

```javascript
// nextWord() 开头
if(nextWordBusy)return;nextWordBusy=true;
try {
    // ... 异步操作 ...
} finally {
    nextWordBusy=false;
}
```

`nextWord` 和 `prevWord` 各有一个独立的互斥锁。

### 错误处理

- IndexedDB 读写失败：静默捕获，`console.warn` 记录，不阻断用户操作
- `fetch()` 词库失败：显示"加载失败，请刷新重试"
- 用户输入验证失败：显示错误信息，不执行保存
- 触摸/键盘事件异常：try/catch 包裹，防崩溃

### XSS 防注入

所有包含用户数据或外部数据（词库 JSON）的 innerHTML 注入点，必须使用 `escapeHTML()` 转义：

```javascript
// 正确
el.innerHTML = '<div>' + escapeHTML(userData) + '</div>';

// 错误
el.innerHTML = '<div>' + userData + '</div>';
```

内置常量（日期数字、CSS 类名、HTML 实体如 `&#9662;`）不需要转义。

---

## 常见任务

### 添加新词库

**需修改的文件**:
1. `index.html` — BANKS 常量（行 375）
2. 创建新 JSON 词库文件

**步骤**:
1. 在 BANKS 对象中添加新条目：
   ```javascript
   var BANKS = {
     cet4: {name:'四级核心词汇', file:'words_cet4.json'},
     cet6: {name:'六级核心词汇', file:'words_cet6.json'},
     // 新增
     newbank: {name:'新词库名称', file:'words_new.json'}
   };
   ```
2. 按 [JSON 词库格式](./专有概念/词库系统.md) 创建 `words_new.json`
3. 同一域名下部署新 JSON 文件

无需修改切换逻辑、数据隔离逻辑或 UI。词库选择弹窗自动渲染所有 BANKS 键。

### 修改 IndexedDB Schema

**需修改的文件**: `index.html`

**步骤**:
1. 递增 `DB_VERSION`（行 379）
2. 在 `openDB()` 的 `onupgradeneeded` 中添加 `e.oldVersion` 条件判断
3. 如需迁移，在 `loadAll()` 之后调用迁移函数

```javascript
// 示例：DB v3 → v4，新增字段
var DB_VERSION = 4;

req.onupgradeneeded = function(e) {
    // ... 现有逻辑 ...
    if (e.oldVersion < 4) {
        // v4 迁移逻辑
    }
};
```

> 绝不修改已部署的旧版本迁移代码，只追加新版本逻辑。

### 添加新功能和配置项

**添加新设置项（如每日提醒时间）**:
1. 在 `defaultSettings()` 中添加默认值
2. 在设置弹窗 DOM 中添加对应 UI 控件
3. 添加事件绑定和保存逻辑
4. 在 `updateMyPage()` 中读取并应用

**添加新刷词模式**:
1. 在 `startStudy(mode)` 中添加新 `if` 分支
2. 在 `currentStudyList()` 中返回对应列表
3. 在 `nextWord()` 和 `prevWord()` 中处理新模式下的进度保存
4. 在首页添加新按钮

### 添加 Service Worker（离线支持）

```javascript
// 注册 SW
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js');
}
```

Service Worker 文件（`sw.js`）需缓存 HTML 和 JSON 文件，实现离线访问。SW 文件必须与页面同源。

### 部署到生产

1. 将 `/workspace/index.html`、`/workspace/words_cet4.json`、`/workspace/words_cet6.json` 部署到任意静态托管
2. 确保所有文件在同一域名下，词库 JSON 路径正确
3. 验证 HTTPS（IndexedDB 和 `navigator.mediaDevices` 在部分浏览器中要求安全上下文）

---

## 关键边界情况

### 词库切换期间

- `switchBank()` 先 `switchPage('home')` 切回首页避免旧数据残留
- 切换到同一词库且 `words.length > 0` 时直接返回（幂等）
- `saveData()` 失败不阻断切换（try/catch）

### 轮次完成

- 顺序/乱序模式：显示完成弹窗，index 归零，保存进度，不重复计数首词
- 生词本模式：显示"生词本刷完"弹窗

### 跨日边界

- `nextWord()` 中 `checkNewDay()` 会在每日首刷时将 `todaySwiped` 归零
- 打卡日期用 UTC ISO 字符串 `YYYY-MM-DD`，避免时区问题

### 空状态

- 词库未加载：`words.length === 0` 时刷词按钮显示"词库加载中..."
- 生词本为空：`bookmarkStudyBtn` 点击无反应

### 多用户隔离

- 每个用户拥有独立的 `uid`
- 活跃用户记录在 `profile` store 的 `active` key 中
- 同一浏览器不同用户数据完全隔离

### 旧版 localStorage 清理

- `init()` 末尾执行 `localStorage.removeItem('cet4_study_data')`，清理 v1 遗留数据
