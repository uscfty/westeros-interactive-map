# 项目开发进度与上下文交接文档

**项目**：维斯特洛互动地图（westeros-interactive-map）
**技术栈**：Vite + Vue 3（`<script setup>` + TypeScript）
**日期**：2026-09-18（延续自上一份 HANDOFF——2026-08-12 那份君临城的记录；今天没有再录入新城市，主要是把部署这块彻底修好，再补了移动端适配）

---

## 一、今天完成的功能

### 1. 修好了线上网站——它其实从来没有真正 build 过
上线地址 https://uscfty.github.io/westeros-interactive-map/ 之前一直是 Pages 的 legacy 模式，直接拿 `main` 分支根目录当静态站点源，而根目录的 `index.html` 是**没编译过的源文件**（`<script type="module" src="/src/main.ts">`，直接引用 `.vue`/`.ts`），浏览器根本没法执行——线上页面此前是打不开的，仓库里也从来没有提交过任何构建产物。

修复分两部分：
- **接入 GitHub Actions 自动构建部署**：新增 `.github/workflows/deploy.yml`，push 到 `main` 时用 `vue-tsc -b && vite build` 构建，再用官方 `actions/deploy-pages` 发布；同时把仓库 Pages 设置的 build source 从 "branch: main / legacy" 切到 "workflow" 模式（通过 `gh api -X PUT repos/.../pages -f build_type=workflow`）。
- **修掉子路径部署下的资源 404**：项目页面部署在 `uscfty.github.io/westeros-interactive-map/` 这个子路径下，但 `App.vue`/`cities.ts`/`index.html` 里所有静态资源路径都是硬编码的**域名根路径**（`/sigils/xxx.png`、`/westeros.jpg`、`/background.jpg`、`/favicon.png` 等）。这类写死的 public 目录绝对路径 Vite 不会自动帮你加 base 前缀（这是 Vite 的既定行为，不是 bug）。改法：
  - `vite.config.ts` 设置 `base: '/westeros-interactive-map/'`
  - `index.html` 里的 favicon 改用 Vite 支持的 `%BASE_URL%` 占位符
  - `App.vue`、`cities.ts` 里新增 `const base = import.meta.env.BASE_URL`（或 `BASE`），所有资源路径改成模板字符串拼接，例如 `` `${base}westeros.jpg` ``
  - `.map-container` 原来在 CSS `<style>` 里用 `background-image: url("background.jpg")`（相对路径，且没有 `/` 前缀，Vite 会把它当模块导入去解析，根本解析不到 `public/` 下的文件）——这个此前也是坏的。改成不在 CSS 里写死，通过 `:style` 内联绑定 `` url(${base}background.jpg) ``，CSS 规则里只保留 `background-size`/`position`/`repeat`。
  - 本地用 `vite build` + `vite preview` 实测了一遍，`curl` 检查了所有资源路径（favicon/地图图/背景图/音乐/家徽/街景图）在子路径下全部返回 200，确认没漏。

### 2. 把上次遗留的未提交文件一起提交了
上一份 HANDOFF 里提到的 `public/cities/kings-landing.jpg`、`public/sigils/kings-landing.png`（当时是 `??` 未跟踪状态）这次一起 `git add` 提交了，连同 `favicon.png`（新的）替换掉了旧的 `favicon.svg`。

**`public/aged_parchment_scroll.png`（10.8MB）目前依然是未跟踪状态，故意没提交**——全仓库搜索了一遍，代码里没有任何地方引用它，不确定是不是当时做 royal 主题时试验性下载但没用上的素材。**下次开工第一步建议先问一下这文件到底要不要用**，要用就接进代码，不用就直接删掉，别让它继续留着占用未跟踪状态。

### 3. 移动端 / iPhone 竖屏适配
用户反馈说要在 iPhone 上也能正常看，测下来主要是两个硬伤：

- **整个页面被强制锁成桌面横屏的 16:9 "信封盒子"**（`.map-container` 用 `width: min(100vw, 177.7778vh); height: min(100vh, 56.25vw)`）。这套公式在竖屏手机上会把可用高度压缩到很离谱的程度（算了一下 iPhone SE 375×667 的视口，算出来容器只有 375×211，地图被挤成屏幕正中间一条很窄的横条，上下一大片全是留黑）。这层信封盒子本来是为了让容器比例始终匹配 16:9 的 `background.jpg` 桌面壁纸，横屏桌面端没问题，但完全不适合竖屏手机。
  **修法**：加了 `@media (max-aspect-ratio: 1/1)`（视口宽 ≤ 高，覆盖绝大多数竖屏场景，不是拍脑袋定一个手机宽度断点），命中时 `.map-container` 改成直接撑满视口。之所以选用宽高比而不是固定像素宽度做断点，是因为问题根源就是"容器比例和视口比例不匹配"，用 aspect-ratio 判断更贴合本质，顺带能覆盖竖屏平板这类没有专门测过但同理会中招的场景。改完之后地图本身是竖版比例（`aspect-ratio: 1920/2716`，图源本来就是竖的），天然适合竖屏，不需要再靠信封盒子对齐背景图。
- **城市标记的名字标签只在 `:hover` 时显示，触屏设备没有 hover 状态，永远看不到城市名，只能瞎点。**14px 的圆点热区也偏小（Apple HIG 建议最小触控尺寸 44px）。
  **修法**：加了 `@media (hover: none)`，命中时标签默认常显、圆点放大到 16px、外层 padding 加到 14px（视觉热区接近 44px）；另外给 `:active` 也加上了原来只有 hover/focus-visible 才有的放大反馈，弥补触屏点击时没有视觉确认的问题。

其余顺手改的：
- `main` 高度加一层 `height: 100dvh`（写在 `height: 100vh` 后面做渐进增强，不支持的浏览器自动忽略回退到 vh），避免 iOS Safari 地址栏收起/展开导致 `100vh` 跳动。
- `.city-marker`、`.bgm-toggle`、`.city-panel-close` 加 `-webkit-tap-highlight-color: transparent`，去掉 iOS Safari 点击时那个丑的蓝色高亮闪烁。
- `.city-panel-scroll` 加 `-webkit-overflow-scrolling: touch`，iOS 惯性滚动。
- `.city-panel` 宽度 `min(420px, 88vw)` → `min(420px, 92vw)`，窄屏下多一点可用宽度。

---

## 二、当前代码架构

```
westeros-interactive-map/
├── .github/workflows/deploy.yml  # push main 后自动 vite build + 部署到 GitHub Pages
├── vite.config.ts           # base: '/westeros-interactive-map/'（项目页面部署在子路径下，必须设）
├── index.html                # 引入 Google Fonts；favicon 用 %BASE_URL% 占位符
├── src/
│   ├── main.ts               # Vue 应用入口，未改动
│   ├── style.css              # 全局样式（上一轮已经清理掉 Vite 默认模板残留）
│   ├── App.vue                # 唯一的组件，页面全部逻辑和样式都在这里（现在约 520 行）
│   └── data/
│       └── cities.ts         # 城市数据表（唯一的数据源，现收录 6 座城市）
└── public/
    ├── westeros.jpg / background.jpg / background_bgm.mp3 / favicon.png
    ├── sigils/   # hightower / tyrell / stark / lannister / nightswatch / kings-landing
    ├── cities/   # oldtown / highgarden / winterfell / castle-black / casterly-rock / kings-landing
    └── aged_parchment_scroll.png  # 未跟踪、代码里没引用，见上面"遗留问题"
```

### 静态资源路径的新约定（这次改动引入，以后加资源要照着写）
项目部署在 GitHub Pages 的子路径下，`public/` 目录里的文件**不能再写死域名根路径**（`/xxx.png` 这种），必须拼上 `import.meta.env.BASE_URL`：
- `.vue`/`.ts` 里：文件顶部/`<script setup>` 里定义 `const base = import.meta.env.BASE_URL`（`cities.ts` 里叫 `BASE`），资源路径写成 `` `${base}sigils/xxx.png` ``
- `index.html` 里：用 Vite 内置的 `%BASE_URL%` 占位符，例如 `href="%BASE_URL%favicon.png"`
- CSS `<style>` 块里**不要**直接写 `url("xxx.jpg")` 引用 `public/` 下的图（Vite 会当模块导入去解析，大概率找不到文件）——需要用到背景图的话通过 `:style` 内联绑定，把拼好 base 前缀的 URL 传进去（参考 `.map-container` 的 `background-image` 做法）。

其余核心数据结构（`City` 接口、`royal` 主题、弹窗滚动机关等）跟上一份 HANDOFF 描述的一致，这次没有改动，不再重复贴。

---

## 三、今天踩的坑

1. **线上从来没真正部署成功过，但排查前完全看不出来**——`gh api repos/.../pages` 一查 `status: "built"`，Pages 自己认为部署是成功的（因为它确实原样把 `index.html` 发布出去了，只是发布的是源文件不是构建产物）。**教训**：光看 Pages API 返回的 build 状态不能说明网站真的能跑，得实际 `curl` 页面内容看它引用的是 `/src/main.ts` 还是 `/assets/xxx-hash.js` 才能判断有没有走过构建。

2. **想用本地 headless Chrome 截图验证移动端 CSS，结果这台机器上 Chrome 153 的 `--window-size` 完全不生效**——无论传 `390,844` 还是 `390x844`，实际布局视口跟请求的尺寸对不上（用一个只打印 `window.innerWidth` 的极简测试页反复验证过，同一个 flag 组合每次给出的实际视口都不一样、且和请求值对不上，`--screenshot` 输出的 PNG 像素尺寸虽然符合请求，但那只是导出时被缩放/裁剪过，不代表真实布局视口）。折腾了挺久最后放弃了这条验证路径，改成直接读编译后的 CSS 文本确认 `@media` 规则本身正确（`max-aspect-ratio`/`max-width`/`hover:none` 这些都是很成熟的标准特性，理论上没有兼容性问题），线上效果最终还是需要在真机 Safari 上肉眼确认。**教训**：这个环境里不要再指望 `chrome --headless --window-size=... --screenshot` 这条路径来做视口相关的验证，不可靠；真要截图验证移动布局，得用支持真正设备模拟的工具（比如 Playwright/Puppeteer 的 `page.setViewport`），而不是裸 CLI flag。

---

## 四、遗留问题 / 下次第一步建议

### 还没做，下次可以顺手看看
1. **`public/aged_parchment_scroll.png`（10.8MB）到底要不要用**：目前未跟踪、代码零引用。先问清楚用途，用就接进去，不用就删掉。
2. **移动端适配还没在真机上肉眼确认过**：这次的 CSS 改动是照着标准 media query 语义写的，本地 headless 截图验证工具链失效（见上面"踩的坑"），建议下次开工第一件事是找台 iPhone 或用真正支持视口模拟的工具（Chrome DevTools 手动打开、或 Playwright）看一眼实际效果，尤其确认：竖屏地图有没有被压扁、城市名标签会不会太挤/重叠（6 个城市名同时常显，如果以后新增更多城市可能需要考虑做成分级显示而不是全部常驻）。
3. **滚动条没有做旧适配**：`.city-panel-scroll` 还是浏览器默认滚动条，可以加 `::-webkit-scrollbar` 自定义（细窄胡桃木色滑块）——这条从上一份 HANDOFF 就有，一直没顺手做。
4. **颜色硬编码，没抽成 CSS 变量**——同上，遗留老问题。
5. **`src/style.css`** 上一轮已经清理过 Vite 默认模板残留，这条可以从遗留列表里划掉了。

### 功能待办
1. **还差至少 1 座大家族城市没录入**：鹰巢城 Eyrie（艾林家族）。
2. GitHub Actions workflow 里 Node 版本已经从 20 升到 22（消掉了一次 deprecation 警告），后续 Node 22 也快要到 deprecation 周期时记得再升一次。

### 下次建议的顺序
1. 先确认 `aged_parchment_scroll.png` 的去留。
2. 找机会在真机 iPhone 上过一遍移动端效果，看看竖屏地图、城市标签常显是否符合预期。
3. 有余力的话开始鹰巢城的坐标定位和正文录入；再顺手把滚动条做旧样式和颜色变量抽取这两项低风险小改动做掉。
