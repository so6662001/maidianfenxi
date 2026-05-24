# 设计 Tokens（色彩 / 字体 / 间距）

> 一套统一的视觉变量，前端 CSS / Less / SCSS / TS 共用。

## 一、色彩

### 品牌主色

| Token | Hex | RGB | 用途 |
|---|---|---|---|
| `--primary-50` | #E6F4FF | 230,244,255 | 高亮背景 |
| `--primary-100` | #BAE0FF | — | hover 背景 |
| `--primary-300` | #69B1FF | — | 辅助元素 |
| `--primary-500` | **#1677FF** | 22,119,255 | **主按钮/链接/选中** |
| `--primary-700` | #003EB3 | — | 按钮 hover |
| `--primary-900` | #001D66 | — | 按下态 |

### 功能色

| Token | Hex | 用途 |
|---|---|---|
| `--success-500` | #52C41A | 健康/正向/达成 |
| `--success-50` | #F6FFED | 成功背景 |
| `--warning-500` | #FAAD14 | 关注/黄灯 |
| `--warning-50` | #FFFBE6 | 警告背景 |
| `--danger-500` | #FF4D4F | 异常/红灯 |
| `--danger-50` | #FFF1F0 | 危险背景 |
| `--info-500` | #1677FF | 信息 |

### 中性色

| Token | Hex | 用途 |
|---|---|---|
| `--text-1` | #1F2329 | 主标题/强调 |
| `--text-2` | #4E5969 | 次标题/正文 |
| `--text-3` | #86909C | 说明/辅助 |
| `--text-4` | #C9CDD4 | 占位/禁用 |
| `--border-1` | #E5E6EB | 分割线 |
| `--border-2` | #F2F3F5 | 浅分割 |
| `--bg-page` | #F7F8FA | 页面背景 |
| `--bg-card` | #FFFFFF | 卡片背景 |
| `--bg-hover` | #F2F3F5 | hover 背景 |

### 战略象限色（专用）

| 象限 | 背景 | 边框 | 图标 |
|---|---|---|---|
| ⭐ 明星 | #E8F8F0 | #52C41A | #389E0D |
| ❓ 问题 | #FFF8E1 | #FAAD14 | #D48806 |
| 💰 现金牛 | #E3F2FD | #1677FF | #096DD9 |
| 💀 瘦狗 | #FFEBEE | #FF4D4F | #CF1322 |

### 数据可视化色板（ECharts）

```
主色板（按顺序使用，6 色）:
#1677FF #52C41A #FAAD14 #FF4D4F #722ED1 #13C2C2

热力图渐变:
低 #E6F4FF → 高 #003EB3

留存矩阵（5 级）:
极差 #FF4D4F → 差 #FAAD14 → 中 #FFEC3D → 好 #95DE64 → 极好 #389E0D
```

## 二、字体

### 字号 / 行高 / 字重

| Token | 字体 | 字号 | 行高 | 字重 | 用途 |
|---|---|---|---|---|---|
| `--font-display` | Inter / PingFang SC | 32px | 40px | 600 | 大数字 KPI |
| `--font-h1` | 同上 | 24px | 32px | 600 | 页面标题 |
| `--font-h2` | 同上 | 20px | 28px | 600 | 模块标题 |
| `--font-h3` | 同上 | 16px | 24px | 600 | 卡片标题 |
| `--font-body` | 同上 | 14px | 22px | 400 | 正文 |
| `--font-small` | 同上 | 12px | 20px | 400 | 辅助说明 |
| `--font-mono` | JetBrains Mono | 13px | 20px | 400 | 数据/代码 |

### 字体栈

```css
font-family: 
  Inter,
  "PingFang SC",
  "Microsoft YaHei",
  -apple-system,
  BlinkMacSystemFont,
  "Segoe UI",
  sans-serif;

/* 等宽 */
font-family:
  "JetBrains Mono",
  "Fira Code",
  Consolas,
  Monaco,
  "Courier New",
  monospace;
```

## 三、间距

| Token | 值 | 用途 |
|---|---|---|
| `--space-1` | 4px | 紧凑间隙 |
| `--space-2` | 8px | 标准间隙 |
| `--space-3` | 12px | 内边距 |
| `--space-4` | 16px | 卡片内边距 |
| `--space-5` | 24px | 模块间距 |
| `--space-6` | 32px | 大模块间距 |
| `--space-8` | 48px | 页面顶边距 |

## 四、圆角

| Token | 值 | 用途 |
|---|---|---|
| `--radius-sm` | 4px | 标签/小按钮 |
| `--radius-md` | 8px | 卡片/按钮 |
| `--radius-lg` | 12px | 模态框/大卡片 |
| `--radius-full` | 9999px | 圆形 |

## 五、阴影

| Token | 值 | 用途 |
|---|---|---|
| `--shadow-1` | 0 1px 2px rgba(0,0,0,0.06) | 卡片基础 |
| `--shadow-2` | 0 4px 12px rgba(0,0,0,0.08) | hover 浮起 |
| `--shadow-3` | 0 8px 24px rgba(0,0,0,0.12) | 弹层 |
| `--shadow-4` | 0 16px 48px rgba(0,0,0,0.16) | 模态/抽屉 |

## 六、过渡 / 动效

| Token | 值 | 用途 |
|---|---|---|
| `--ease-out` | cubic-bezier(0.2, 0, 0, 1) | 标准 |
| `--ease-in-out` | cubic-bezier(0.4, 0, 0.2, 1) | 双向 |
| `--duration-fast` | 150ms | 微交互（hover） |
| `--duration-base` | 250ms | 默认 |
| `--duration-slow` | 400ms | 大动效 |

## 七、响应式断点

| 断点 | 范围 |
|---|---|
| `xs` | < 576px |
| `sm` | 576 - 768 |
| `md` | 768 - 992 |
| `lg` | 992 - 1280 |
| `xl` | 1280 - 1920 |
| `xxl` | ≥ 1920 |

## 八、Z-index 层级

| 层级 | 值 | 用途 |
|---|---|---|
| `base` | 0 | 默认 |
| `dropdown` | 1000 | 下拉菜单 |
| `sticky` | 1010 | 吸顶元素 |
| `fixed` | 1020 | 固定元素 |
| `modal-bg` | 1030 | 模态背景 |
| `modal` | 1040 | 模态内容 |
| `popover` | 1050 | 弹层 |
| `tooltip` | 1060 | 提示 |
| `assistant` | 1070 | 决策助理浮窗 |
| `message` | 1080 | toast 消息 |

## 九、暗色模式（可选 V2）

提供 `--primary-500-dark` 等暗色版变量，CSS `[data-theme="dark"]` 切换。

## 十、CSS 变量实现示例

```css
:root {
  /* 颜色 */
  --primary-500: #1677FF;
  --success-500: #52C41A;
  --warning-500: #FAAD14;
  --danger-500: #FF4D4F;
  --text-1: #1F2329;
  --text-2: #4E5969;
  --text-3: #86909C;
  --border-1: #E5E6EB;
  --bg-page: #F7F8FA;
  --bg-card: #FFFFFF;
  
  /* 字体 */
  --font-display: 32px/40px Inter, sans-serif;
  --font-h1: 24px/32px Inter, sans-serif;
  --font-body: 14px/22px Inter, sans-serif;
  
  /* 间距 */
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-5: 24px;
  
  /* 圆角 */
  --radius-md: 8px;
  
  /* 阴影 */
  --shadow-1: 0 1px 2px rgba(0,0,0,0.06);
  --shadow-2: 0 4px 12px rgba(0,0,0,0.08);
}
```
