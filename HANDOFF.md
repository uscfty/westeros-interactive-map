# 项目开发进度与上下文交接文档

**项目**：维斯特洛互动地图（westeros-interactive-map）
**技术栈**：Vite + Vue 3（`<script setup>` + TypeScript）
**日期**：截至今天收工时（延续自上一份 HANDOFF，之前已完成地图展示、BGM、城市点击面板等基础功能，录入了旧镇、高庭、黑城堡三座城市）

---

## 一、今天完成的功能

1. **新增城市：临冬城 Winterfell（史塔克家族）—— 完整录入**
   - **坐标定位**：用 Python/Pillow 对 `public/westeros.jpg` 原图逐步裁图打网格，精确找到地图上"Winterfell"标记圆点的像素中心，换算成百分比坐标 `x: 49.5, y: 33.7`，`side: "left"`。
   - **介绍正文**：参照高庭/黑城堡的写法，用 paragraph / heading（二三级）/ list 内容块写了完整介绍，涵盖建城传说（"筑城者"布兰登）、内外城墙与首堡/大堡/图书塔/钟塔等主要建筑、天然温泉供暖与玻璃花园、神木林与地下墓室、以及"北境之王"到伊耿征服（托伦·史塔克屈膝）的历史。
   - **素材**：家徽图 `public/sigils/stark.png`、街景图 `public/cities/winterfell.jpg` 已经就位（图片是后来补上的，代码里原先留的 TODO 注释可以清理掉了，见下方遗留问题）。

2. **新增城市：凯岩城 Casterly Rock（兰尼斯特家族）—— 仅完成坐标与骨架，正文待补**
   - **坐标定位**：同样用网格法精确定位，中心像素约 (615.5, 1874.5)，换算成 `x: 32.1, y: 69.0`，`side: "left"`。
   - **数据骨架**：在 `cities.ts` 里新增了 `id: "casterly-rock"` 条目，`ruler` 填了"兰尼斯特家族 House Lannister"。
   - **素材**：家徽图 `public/sigils/lannister.png`、街景图 `public/cities/casterly-rock.jpg` 已经放进项目了。
   - **⚠️ 正文尚未填写**：`description` 目前只有一句占位文字 `"TODO: 补充凯岩城简介正文。"`，不是真实介绍内容，是**明天要做的第一件事**（详见第三节）。

3. **验证**：`vue-tsc --noEmit` 和 `npm run build` 均通过，无类型/构建错误。

---

## 二、当前代码架构

```
westeros-interactive-map/
├── index.html              # 引入 Google Fonts（UnifrakturMaguntia）
├── src/
│   ├── main.ts              # Vue 应用入口，未改动
│   ├── style.css             # 全局样式（Vite 默认模板残留，目前和地图页基本无关）
│   ├── App.vue               # 唯一的组件，页面全部逻辑和样式都在这里
│   └── data/
│       └── cities.ts        # 城市数据表（唯一的数据源，现已收录 5 座城市）
└── public/
    ├── westeros.jpg          # 地图底图
    ├── background.jpg        # 页面背景图
    ├── background_bgm.mp3    # 背景音乐
    ├── sigils/                # 家族/组织纹章图
    │   ├── hightower.png
    │   ├── tyrell.png
    │   ├── stark.png          # 今天新增
    │   ├── lannister.png      # 今天新增
    │   └── nightswatch.png
    └── cities/                # 城市街景图
        ├── oldtown.jpg
        ├── highgarden.jpg
        ├── winterfell.jpg     # 今天新增
        ├── casterly-rock.jpg  # 今天新增
        └── castle-black.jpg
```

### 核心数据结构（`src/data/cities.ts`）

```ts
type DescriptionBlock =
  | { kind: "paragraph"; text: string }
  | { kind: "heading"; level: 2 | 3; text: string }
  | { kind: "list"; items: { label?: string; text: string }[] };

interface City {
  id: string;
  name: string;
  ruler: string;        // 统治家族或组织
  x: number;             // 地图图片上的横向百分比坐标
  y: number;             // 地图图片上的纵向百分比坐标
  side: "left" | "right"; // 弹窗从哪一侧滑出
  sigil: string;          // 纹章图路径
  image: string;          // 街景图路径
  description: DescriptionBlock[];
}
```

**加一个新城市只需要往 `cities` 数组里追加一条数据，不用改 `App.vue` 的任何逻辑**（标记点渲染是 `v-for="city in cities"`，完全数据驱动）。

**坐标定位的标准做法**（今天两座城市都是这么做的，之后加城市可以复用）：
1. 用 Python/Pillow 把 `public/westeros.jpg` 按目标区域裁剪并叠加网格线和坐标数字标注，导出成图看清楚标记点大致范围。
2. 逐步缩小裁剪范围、放大倍数，直到能精确读出标记圆点的像素边界。
3. 取边界中点作为像素坐标，按 `x / 1920 * 100`、`y / 2716 * 100`（原图尺寸 1920×2716）换算成百分比，写入 `cities.ts`。
4. `side` 按点在地图上大致偏左还是偏右决定（左侧城市 `left`，右侧城市 `right`），目前 5 座城市里只有黑城堡是 `right`。

### 核心交互逻辑（`src/App.vue`）

- **`.map-frame` 用 `aspect-ratio` 锁定原图比例**（1920/2716），这是地图上标记点能精确对位的关键——只要容器比例和图片比例一致，`cities.ts` 里的百分比坐标就能在任意窗口尺寸下对准图片上的真实位置，不会跑偏。
- **弹窗状态用两个变量控制**，避免关闭动画播放过程中内容"跳变"：
  - `selectedCity`：当前展示的城市数据，**关闭动画播完之后才清空**
  - `panelOpen`：控制 `v-if`，决定面板是否在 DOM 里（触发进入/离开动画）
  - `<Transition>` 的 `name` 根据 `selectedCity.side` 动态选择 `slide-left` / `slide-right`
- **面板正文用 `<template v-for> + v-if/else-if` 链**遍历 `description` 数组，按 `block.kind` 分别渲染成 `<h3>`/`<h4>`/`<p>`/`<ul>`。

---

## 三、遗留问题 / 明天第一步建议

### 明天第一件事（优先级最高）
**给凯岩城写正文介绍**。坐标、家徽图、街景图都已经就位，`cities.ts` 里的 `description` 目前只有一句占位文字。参照旧镇/高庭/黑城堡/临冬城的结构（开篇段落 + 分主题的二三级标题 + 段落/列表），把兰尼斯特家族凯岩城的介绍（金矿、狮子雕像、地下水道、伊蒙・兰尼斯特打退铁民入侵的传说等）补充完整。

### 需要清理的小问题
1. **`stark.png` / `casterly-rock.jpg` 等图片的 TODO 注释可以摘掉了**：加临冬城和凯岩城数据时，因为图片还没就位，`sigil`/`image` 字段后面加了 `// TODO` 注释提醒补图；现在图片文件都已经放进 `public/sigils/` 和 `public/cities/` 了，注释已经过时，下次改 `cities.ts` 时顺手删掉。
2. **`.map-container` 里有一行注释写错了**（`App.vue` 第 151 行左右）：注释文字说的是"把 /background_bgm.mp3 换成..."，但这段代码实际是背景图片 `background-image` 的设置，注释内容和代码对不上（应该是之前改 BGM 时误改到了这里）。不影响功能，仍未修复。
3. **仍然没有版本管理**：项目至今不是 git 仓库，累计的工作量已经不小（5 座城市、若干次样式调整），强烈建议尽快 `git init` 并提交一次快照，避免误操作导致工作丢失。

### 功能待办
1. **还有 2 座城市完全没有标记点和数据**：君临、鹰巢城等大家族城市还未录入，需要继续按 `cities.ts` 的结构逐个补充坐标和介绍。（河间城看情况是否需要，取决于是否要覆盖全部大家族。）
2. **移动端/触屏体验未验证**：标记点的悬停显示城市名（`hover`）依赖鼠标，触屏设备没有 hover 状态，直接点击虽然能打开面板，但"先看名字再点"这个引导体验在手机上是缺失的；另外 14px 的圆点在触屏上命中率可能偏低，值得后续加大点击热区或做个响应式的常驻标签。
3. **样式复用问题**：花体标题字体（`UnifrakturMaguntia`）、金色强调色（`#c9962c`）等目前是在 `App.vue` 里散落定义，城市已经有 5 座了，如果要统一改主题色，需要挨个改。建议尽快把这些颜色抽成 CSS 变量集中管理。
4. **`src/style.css`**：还残留 Vite 默认模板的一些样式（`.hero`、`#next-steps` 等），目前页面用不上，可以清理掉减少无用代码。

### 明天建议的顺序
1. 先把凯岩城的介绍正文写完（当前唯一"半成品"状态的城市）。
2. 做 **git 初始化 + 首次提交**，把迄今为止的成果存档。
3. 有余力的话，开始下一座城市（君临或鹰巢城）。
