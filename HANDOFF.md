# 项目开发进度与上下文交接文档

**项目**：维斯特洛互动地图（westeros-interactive-map）
**技术栈**：Vite + Vue 3（`<script setup>` + TypeScript）
**日期**：2026-08-12（延续自上一份 HANDOFF；今天新增君临城，并围绕它的专属弹窗主题做了好几轮样式排查与重构）

---

## 一、今天完成的功能

### 1. 新增城市：君临 King's Landing（坦格利安家族）—— 完整录入
- **坐标**：`x: 59.9, y: 71.3`，`side: "right"`。
- **正文**：参照已有城市的写法，涵盖红堡与铁王座、龙窝、白书塔、密道与地牢、贝勒大圣堂、城中风貌（河间集市/跳蚤窝/丝绸街/七座城门）、历史四条（伊耿登陆、铁王座锻造、红堡营建、贝勒大圣堂落成）。
- **素材**：家徽图 `public/sigils/kings-landing.png`、街景图 `public/cities/kings-landing.jpg` 已就位（这两个文件目前还是 git 未跟踪状态，记得下次提交时 `git add`）。

### 2. 新增 `"royal"` 弹窗主题——君临专属的羊皮纸 + 胡桃木风格
`City` 接口新增了一个可选字段 `theme?: "royal"`，君临是目前唯一用到它的城市。视觉上和其余城市的深色羊皮纸面板区分开：
- 米黄色羊皮纸底色 + 四角渍染渐变 + 极淡的噪点纹理（`data:image/svg+xml` 内联生成，不是外部图片，符合"暂时不加图片纹理"的要求）。
- 首段落做首字下沉（Drop Cap），标题两侧缀 ⚜ 花纹。
- 面板最外沿有一圈**双线胡桃木画框**（`::before` 伪元素，`inset: 10px`，颜色 `#3b2219`）。
- 顶部城市插图带细描边 + 内阴影，做出"镶在羊皮纸卡槽里"的凹陷质感，而不是图片剪影/照片纹理。

### 3. 清理了一处过期注释
`cities.ts` 里君临条目的 `sigil`/`image` 字段原本各带一条 `// TODO: 把图放进去` 的注释，图片其实已经放进 `public/` 了，注释已经顺手删掉（这是延续上一份 HANDOFF 里提到的"老问题"，之前临冬城/凯岩城已经清过一次，君临新增时又带出来了同样的模式）。

---

## 二、当前代码架构

```
westeros-interactive-map/
├── index.html              # 引入 Google Fonts（UnifrakturMaguntia）
├── src/
│   ├── main.ts              # Vue 应用入口，未改动
│   ├── style.css             # 全局样式（Vite 默认模板残留，和地图页基本无关，见"遗留问题"）
│   ├── App.vue               # 唯一的组件，页面全部逻辑和样式都在这里（现在约 500 行）
│   └── data/
│       └── cities.ts        # 城市数据表（唯一的数据源，现已收录 6 座城市）
└── public/
    ├── westeros.jpg / background.jpg / background_bgm.mp3
    ├── sigils/   # hightower / tyrell / stark / lannister / nightswatch / kings-landing.png
    └── cities/   # oldtown / highgarden / winterfell / castle-black / casterly-rock / kings-landing.jpg
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
  ruler: string;
  x: number;              // 地图图片上的横向百分比坐标
  y: number;              // 地图图片上的纵向百分比坐标
  side: "left" | "right"; // 弹窗从哪一侧滑出
  sigil: string;
  image: string;
  theme?: "royal";         // 新增字段：君临用，不填则用默认深色羊皮纸风格
  description: DescriptionBlock[];
}
```
加新城市依旧只需要往 `cities` 数组追加一条数据，`App.vue` 的渲染逻辑是完全数据驱动的，不用改。想让新城市也用 royal 风格，加一行 `theme: "royal"` 就行——但注意 royal 主题目前只有君临一个样本，配色/间距是照着君临的图片尺寸（1920×1080 街景图）调出来的，换一个长宽比差异很大的图可能需要重新看一眼效果。

### 弹窗结构与滚动机关（`App.vue`）
- `.city-panel` 是 `position: fixed` 的整个弹窗容器，`.city-panel-scroll` 是它内部**唯一真正滚动**的内容层（`overflow-y: auto`）。这样设计是为了让金框（royal 主题的 `::before`）和关闭按钮**不随内容滚动**，永远贴在面板可视边缘。
- `selectedCity` 和 `panelOpen` 两个变量配合，保证关闭动画播完之前面板内容不会跳变（`@after-leave` 才清空 `selectedCity`）。
- `<Transition>` 的 `slide-left` / `slide-right` 由 `selectedCity.side` 动态决定。

### royal 主题的最终实现方案（重要，下面这节详细写了为什么是这样）
`.city-panel.royal .city-panel-image` **不再做"通栏出血 + 负 margin 精确对齐画框"**这种需要手算像素的写法，而是老老实实做一个 `.city-panel-scroll` 内边距（32px/24px）里的普通区块（`width: 100%; margin: 0 0 24px; box-sizing: border-box; overflow: hidden;`），天然和 10px 内缘的金框画框之间留了 22px/14px 安全距离，物理上不可能再压住边框。双线画框 `.city-panel.royal::before` 的两条线颜色统一为 `#3b2219`，中间的缝是纯 `transparent`（透出面板自己的羊皮纸底色），不再额外引入第三种颜色。

---

## 三、今天踩的坑（对以后改这块 CSS 有参考价值，建议别删）

这部分是今天来回调试最费时间的地方，记下来避免以后重复踩：

1. **`:first-of-type` 选错了元素，导致首字下沉一直没生效。**
   `.city-panel.royal .city-panel-desc:first-of-type::first-letter` 里 `:first-of-type` 是"同标签同级中的第一个"，不看 class。而 `.city-panel-house`（"统治：xxx"那一行）本身也是个 `<p>`，且排在所有 `.city-panel-desc` 之前，所以真正的 `p:first-of-type` 是它，不是任何一条正文段落——这条规则永远匹配不到东西。**修法**：改用相邻兄弟选择器 `.city-panel.royal .city-panel-house + .city-panel-desc::first-letter`，精确锁定"统治："后面紧跟着的第一段正文，不受列表/标题顺序影响。

2. **四角"铆钉"圆点定位错乱，压在正文文字上。**
   原来用 4 个 `radial-gradient(circle, ...)` 当 `background-image` 做四角小圆点装饰，但没写 `background-size`。渐变默认按整个 `::before` 盒子（几乎是面板全高）铺开，`background-position` 的 `-4px` 偏移量只是把这个巨大渐变层挪了几像素，圆点实际停在框体纵向中部附近——面板一旦内容变长，圆点就会压在正文中间的文字上（用户反馈是压住了"城"字）。这不是简单的坐标错位，是这个写法本身没法做出贴角的效果，所以**直接删掉了，没有重做**。

3. **图片穿透边框，反复出现了好几次，最后才找到根本解法。**
   最早的写法是图片通栏铺满到面板真正的边缘（`margin: -32px -24px`），而金框在 `inset: 10px` 处，两者之间那 10px 的"卡纸"区域被图片盖住，边框看起来在图片处断开。中途试过"精确计算负 margin 让图片刚好收在金框内缘"（`margin: -22px -14px`），理论上没问题，但只要有一点像素误差就会重新出问题，反复出现"顶部图片和边框交界处不齐"的反馈。**最终解法**：图片完全不做出血，就是内边距里的一个普通区块，天然留出安全距离，不需要精算像素（见上面"核心架构"一节）。

4. **调试红色背景没有显现，一度被误判为"浏览器没读到新 CSS"（缓存问题），其实是我的测试方法选错了属性。**
   `.city-panel.royal` 同时设置了 `background-color` 和 `background-image`（羊皮纸渐变+噪点纹理）。CSS 里 `background-image` 图层天生盖在 `background-color` 之上，且那层渐变不透明，所以 `background-color: red !important` 就算生效了，也会被同一元素自己的 `background-image` 完全盖住，跟浏览器有没有读到新 CSS 毫无关系。后来改用 `outline`（画在盒子外面，不会被任何 background 属性挡住）才验证成功。**教训**：以后要验证"这条 CSS 规则有没有生效"，优先用 `outline` 而不是 `background-color`，尤其是当元素本身还设了 `background-image` 的时候。

5. **颜色改了几轮用户还是觉得"边框是金色"，后来发现是漏改了双线中间那道浅色隔线。**
   画框的双线结构其实是"深色线 + 中间一道浅色隔线 + 深色线"，前几轮只改了两条深色线（`#9c7523 → #5d4037 → #3b2219`），中间那道 `rgba(232, 219, 184, 0.9)`（暖米黄色）一直没动过。夹在两条深色线中间的这道浅色线，在缩小的截图里非常容易被看成"金线"。最终把它也统一处理了（现在中间是纯 `transparent`，不再是一个独立的颜色）。

6. **一度加了 `!important` 和 `.city-panel.royal *` 暴力通配符做诊断，事后全部清理掉了。**
   这是为了排除"CSS 权重被覆盖"的可能性加的诊断代码，排查完确认了从始至终都不存在权重冲突（问题 3、4、5 才是真正原因），所以这些 `!important` 和暴力覆盖规则都是没有必要的技术债，已经全部删除，现在整个 royal 主题的 CSS 没有一处用 `!important`。

---

## 四、遗留问题 / 明天第一步建议

### 还没做，下次可以顺手看看
1. **滚动条没有做旧适配**：`.city-panel-scroll` 目前用的是浏览器默认滚动条，在羊皮纸/深色底色的面板里比较跳，可以加一段 `::-webkit-scrollbar` 自定义样式（细窄的胡桃木色滑块）。
2. **颜色目前是硬编码，没有抽成 CSS 变量**：现在有"默认深色羊皮纸"和"royal 胡桃木"两套配色并存，`App.vue` 里散落着大量十六进制色值，以后如果要统一调色或者再加第三套主题，改起来会比较痛苦，建议抽成 CSS 自定义属性（`:root` 或者按主题分组的变量）。
3. **`.map-container` 里有一行注释和代码对不上**（`App.vue` 155 行左右）：注释写的是"把 /background_bgm.mp3 换成……"，但这段代码实际是背景图片 `background-image` 的设置。这是好几份 HANDOFF 之前就记录的老问题，一直没顺手修，这次也还是没碰。
4. **`src/style.css`** 里还残留 Vite 默认模板的样式（`.hero`、`#next-steps` 等），页面用不上，可以清理掉。
5. **移动端/触屏体验未验证**：标记点靠 `hover` 显示城市名，触屏没有 hover 状态；14px 的圆点在触屏上命中率可能偏低。

### 功能待办
1. **还差至少 1 座大家族城市没录入**：鹰巢城 Eyrie（艾林家族）目前还没有标记点和数据，其余像多恩/太阳矛、铁群岛/派克城看情况是否需要覆盖。
2. **君临的家徽/街景图片文件还没提交到 git**：`public/cities/kings-landing.jpg`、`public/sigils/kings-landing.png` 目前是 untracked 状态（`git status` 里显示 `??`），下次记得连同 `App.vue`、`cities.ts` 的改动一起提交。

### 顺手提一句：今天调试过程中电脑上一度并存 3 个 vite dev server 进程（5173/5174/5199，都指向同一份代码），当时已经全部 kill 掉、清了 `node_modules/.vite` 缓存、重新单独起了一个干净的 5173。如果下次发现浏览器强刷也不生效，先 `lsof -iTCP -sTCP:LISTEN | grep node` 看看是不是又开了好几个端口，不一定是缓存问题。

### 明天建议的顺序
1. 先把君临的两个图片文件 `git add` 提交掉，避免继续放在未跟踪状态。
2. 有余力的话，开始鹰巢城的坐标定位和正文录入。
3. 如果还有时间，顺手把滚动条做旧样式和颜色变量抽取这两项做掉——都是低风险、纯视觉/工程质量的小改动。
