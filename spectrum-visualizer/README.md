# spectrum-visualizer（Folium 模组）

> 把原 `spectrum-viewer` 项目（独立 Electron 桌面小工具）的频谱显示逻辑搬进
> [Folia](https://github.com/chthollyphile/folia-major) 音乐播放器里，
> 作为**播放页左下角的独立小组件**（stageLayer 图层），不占歌词区，被下一首预告等元素挡住时**自动避让**。

| 形态 | 文件 |
|---|---|
| 入口 | [`client.mjs`](./client.mjs) |
| 清单 | [`mod.json`](./mod.json) |
| 签名说明 | [`SIGNING.md`](./SIGNING.md) |
| 预览图 | `preview.jpg`（**需自备**，详见 SIGNING.md） |

---

## 它是什么

- **形态**：`folium.registries.stageLayers` 图层，slot `player.stage.front`
  （歌词之上、播放器控件之下），默认停在播放页**左下角**（左边距 24 px、底边距 20 px，
  以 `transform` 写入，不占用/不覆盖宿主的布局属性），纯显示、点击穿透，不挡任何播放器操作。
  底边距故意贴到最低，"下一首预告"这类底部元素会真的压住它 —— 正好交给自动避让处理。
- **48 段实时频谱柱**：平方映射到 bin，最高 16 kHz，与原 `viewer.js` 完全一致
  （柱色、2px 空闲底座、120 ms 读数刷新率都一致）。
- **一行迷你读数**：输入功率 dB / 峰值频率 Hz（峰值频率 = 峰值 bin × 每格 Hz，见下面的采样率假定）。
- **自动避让**：用命中测试检测自己被哪些宿主元素压住，被挡就上移或换边（见下节）。
- **背景透明**（可选）：去掉卡片底色 / 毛玻璃 / 描边，只剩频谱柱与读数，直接叠在封面上。

### 假定采样率（默认 44.1 kHz）

mod API 只给 `getPower / getBands / getSpectrum`，**不给采样率**，所以峰值频率与频率轴都得
先假定一个采样率。默认取 **44100** 的硬依据来自 Folia 客户端本身（`resources/app.asar`
里播放器那段）：它划分频段时用的是

```js
const r = (from, to) => { /* 对同一块频谱数组按 Math.floor(freq / 21.5) 取 bin 求均值 */ }
r(20, 150); r(150, 400); r(400, 1200); r(1000, 3500); r(3500, 12000);
```

即 **21.5 Hz/格**；而频谱数组长度 = `frequencyBinCount` = 1024（`fftSize = 2048`），
21.5 × 2048 ≈ 44100 —— 这样峰值频率与频率轴跟宿主自己的频段口径一致。

设置里的「假定采样率（Hz）」是 number 输入，**填多少用多少**（0 / 非法值退回 44100）。
每次生效值会写一行日志（`假定采样率 = 44100 Hz（峰值频率与频率轴按它折算）`）——
宿主会把 number 型设置项写歪（实测 `defaultValue: 44100` 被存成 `75000`），
所以想在日志里核对一下实际吃进去的值是不是你填的那个。

> ⚠️ **不是 visualizer**：`registries.visualizers` 注册的是**全屏歌词动画模式**（替换歌词渲染），
> 0.1.0 版走了这个弯路；0.2.0 起改用 `stageLayers`，才是"播放页上的独立组件"。

---

## 自动避让怎么工作

宿主**没有**暴露"下一首预告"这类元素的句柄，也没有舞台安全区 API，所以避让是**通用启发式**，
不依赖任何写死的选择器（宿主换皮、改类名都不会失效）：

1. **命中测试**：在自己矩形内打 4×3 点阵，用 `container.getRootNode().elementsFromPoint()`
   问"这些点上压着谁"（同时也会问一次 `document`，兜住外部浮层）。组件自身是
   `pointer-events:none`，所以永远不会命中自己。
2. **筛掉不该算的**：容器自身及其祖先（舞台根、播放器根）、面积 ≥ 舞台 60% 的铺满层
   （背景、歌词画布）、小于 20×20 的元素、不可见元素，以及**不画东西的透明布局壳**
   （没有背景色/背景图、不是 img/canvas/video、自己也没有文字 —— 判定见 `paints()`）。
   最后一条很关键：卡片外面常套着带 padding 的包裹层，按外框算的话会让让位幅度虚高很多。
3. **候选位打分**：候选序列 = 左下角默认位 → 同列**每 `AVOID.step`（16 px）上移一格** →
   右侧同样一列，全部用舞台 `getBoundingClientRect()` 夹在容器内，总抬升不超过
   `AVOID.maxLift`（200 px）。按顺序扫、**命中 0 就停**，所以绝大多数情况下只多测 1–3 个位置。
   （0.3.1 这里误写成"每级 `组件高度 + 16px`"，一次让位就抬 116 px，看起来像"躲得老远"。）
4. **触发与去抖**：400 ms 轮询 + 容器 `ResizeObserver` + 容器子树 `childList` 变化（预告滑入、
   歌词换行）即时重算；**连续两轮被挡**（≈ 800 ms）才挪动，挪完 **1.2 s 冷却**，位置干净满
   3 s 才考虑回默认位 —— 都是为了不让歌词逐句刷新把组件晃来晃去。
5. **不改宿主**：整个过程只写自己的 `transform`（带 0.18 s 过渡），不动宿主的 DOM / 样式。
6. **可诊断**：每次挪位都会经 `folium.log.info` 写一行到宿主日志（`%APPDATA%\Folia\logs`），
   形如 `自动避让：相对默认位抬高 16px（步长 16px），挡住原位置的是 div.card / span.title`；
   采样率设置被写歪时也会记一行。让位幅度不对劲时先看日志，不用猜。

**局限与调参**：这套判断是纯几何启发式，舞台上任何"画了东西的紧凑块"都会被当成遮挡物
（包括歌词列、封面、悬浮控件）—— 这是有意为之，初衷就是别压住内容。想让它更"迟钝"或更
"敏感"，改 `client.mjs` 里 `AVOID` 常量的 `bleedRatio`（面积阈值）、`minArea`、`step` 与
`maxLift`（让位步长与上限）、`cooldown`、`smoothBack`；想彻底关掉就用设置面板里的
**自动避让** 开关（关掉后固定停在左下角默认位 `left:24px; bottom:20px`，即不再让位）。

---

## 在 Folia 里加载

### 模式 A：本地 / 开发者模式（最快）

Folia 桌面版 → 模组面板 → 加载本目录（`mods/spectrum-visualizer/`）→ 启用本模组；
播放页左下角就会出现频谱小组件（`stageLayers` 没有"选中"这一步，启用即挂载）。
开发者模式不要求签名。

### 模式 B：从 folium-compound 市场（需先签名）

详见 [`SIGNING.md`](./SIGNING.md)。

---

## ⚠️ 清单字段的坑（务必记住）

**`mod.json` 里的 `name` / `description` 必须是纯字符串，不能是 `{ "zh-CN": …, "en": … }`
双语对象。** 写成对象会让 Folia 加载器报 `mod.name is required`，模组无法启用。

- ❌ `"name": { "zh-CN": "频谱可视化", "en": "Spectrum Visualizer" }`
- ✅ `"name": "频谱可视化"`

**双语对象只用于 `registries.*.register()` 里的 `label` / `description` 字段**（例如
`visualizers.register({ label: { 'zh-CN': …, en: … } })`），清单里一律是字符串。

### 权限：`ui.stage` 必填

注册 `registries.stageLayers` 属于 stage（播放页图层）能力，**清单必须声明**：

```json
"permissions": ["ui.stage"]
```

漏掉会在激活时直接失败，宿主报：

```
界面代码出错
client activate: permission-denied:ui.stage
```

（0.2.0 清单里没有 `permissions` 字段，所以本地加载时也是这个错；0.2.1 补上。）
注意 `visualizers` 不需要这个权限，别照抄 `visualizer52hz/mod.json` 的"无 permissions"写法。

对照官方示例 `visualizer52hz/mod.json`（本仓库 `mods/visualizer52hz/` 里就有）：
`"name": "52Hz"`、`"description": "PixiJS 歌词动画…"` 都是纯字符串。

---

## 与原 `viewer.js` 的关键差异

| 维度 | 原 Electron 版 | 本 Folium 模组 |
|---|---|---|
| 音频来源 | `getDisplayMedia({ audio: 'loopback' })` | 跟宿主流（`ctx.audio.getPower/getSpectrum`） |
| AudioContext 可见 | 是（`actx.sampleRate` 可读） | 否，按 44.1 kHz 假定（设置里可改） |
| FFT 大小 | 硬编码 2048 | 从 `getSpectrum().length` 反推（本机实测 1024 → fftSize 2048） |
| 时域 | 暴露（`getFloatTimeDomainData`） | 不暴露 |
| 输入电平算法 | `20·log10(rms)` | `-60 + 60 * power` |
| 读数 | 输入电平 dB + 峰值 Hz + 采样率 kHz | 输入功率 dB + 峰值频率 Hz（采样率只能假定） |
| 频率轴 | 按采样率动态折算 | 假定采样率（默认 44.1 kHz）+ 反推 fftSize |
| 主题适配 | 仅深色 | 跟随 ctx.getTheme() 切深/亮 |
| 位置 | 独立浮窗固定左下角 | 图层左下角 + 命中测试自动避让（可关） |
| 背景 | 固定半透明卡片 | 可切"背景透明"（去底色/毛玻璃/描边） |
| HUD 模式 | 有（透明 + 鼠标穿透） | 没有（图层本身点击穿透） |

---

## 文件结构

```
spectrum-visualizer/
├── mod.json          ← Folium 1 清单（name/description 是字符串）
├── client.mjs        ← 入口（注册 1 个 stageLayer + 4 个设置项）
├── README.md         ← 本文件
├── SIGNING.md        ← 签名入库流程
├── preview.jpg       ← 自备
└── folium.sig.json   ← folium-compound 维护者签名后自动生成
```

---

## 自定义 / 升级

想加新设置：在 `client.mjs` 的 `settingsSections.register({ settings: [...] })` 数组里追加：

```js
{ key: 'myToggle', type: 'boolean', label: { 'zh-CN': '…', en: '…' }, defaultValue: true }
```

然后在 `paint()` 里 `settings.params.get().myToggle` 读取。

> ⚠️ **number 型设置项不可信**：实测宿主会把 `defaultValue` 写歪（`44100` → `75000`）。
> 真要用数字，就把生效值写进日志方便核对（见 `sampleRate()`），否则读数会无声地偏掉。

想换映射算法：改 `paint()` 里
`Math.floor(Math.pow(i / (BARS - 1), 2) * top)` 这段（平方映射 → 对数 / 线性 / Mel 都可以）。

---

## 版本记录

| 版本 | 改动 |
|---|---|
| `0.1.0` | 初版：注册为全屏 visualizer（**形态选错了**——那是歌词动画模式），且 `querySelector` 选择器写错导致 mount 崩溃 |
| `0.2.0` | 改为 `stageLayers` 播放页左下角小组件；修复 `strong[data-meter="x"]` 选择器空指针；渲染改 rAF 自驱（音频数据每帧刷新无事件通知） |
| `0.2.1` | 清单补上 `"permissions": ["ui.stage"]`，修复激活报 `permission-denied:ui.stage` |
| `0.3.0` | 第三个读数「频段」→「解析度」（每格 Hz，由频谱长度反推，不再逐帧乱跳）；新增命中测试自动避让（+ `autoAvoid` 设置项）；修复读数写值时冲掉 dB / Hz 单位的问题；定位改由 `transform` 驱动 |
| `0.3.1` | 修正假定采样率：48 kHz → 44.1 kHz（依据 Folia 自身按 21.5 Hz/格划频段），解析度 23.4 → 21.5 Hz，并新增「假定采样率」设置项；默认底边距 120 → 20 px 压到最底、避让步长 12 → 16 px |
| `0.3.2` | 修复避让"躲太远"：步长误写成 `组件高度 + step`（116 px）→ 改为纯 `step`（16 px），并加 `maxLift` 200 px 上限；遮挡判定新增 `paints()`，透明布局壳不再算遮挡；采样率假定改用 boolean 开关（number 型被宿主写成 75000 → 36.6 Hz）；挪位与解析度推算写 `folium.log.info` 便于诊断 |
| `0.3.3` | **删掉「解析度」读数**（效果差且宿主无接口可取，只留功率 + 峰值）；「假定采样率」改回 number 输入框，填多少用多少（非正值退回 44100，生效值写日志）；新增「背景透明」开关 |
