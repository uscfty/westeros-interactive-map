# 维斯特洛互动地图 Westeros Interactive Map

一张可点击的《冰与火之歌》维斯特洛地图：地图上标出各大城市，点击标记后，桌面从对应一侧滑出介绍面板，手机则打开可展开的底部面板，展示家徽、街景图和正文介绍；页面自带一段可开关的背景音乐。

在线访问：https://uscfty.github.io/westeros-interactive-map/（push 到 `main` 后由 GitHub Actions 自动构建部署，见 `.github/workflows/deploy.yml`）

## 技术栈

- [Vite](https://vitejs.dev/) + [Vue 3](https://vuejs.org/)（`<script setup>` + TypeScript）
- 无额外 UI 框架，样式全部手写在 `App.vue` 的 `<style scoped>` 里

## 快速开始

```bash
npm install
npm run dev       # 本地开发，默认 http://localhost:5173
npm run build      # 类型检查（vue-tsc）+ 生产构建，产物在 dist/
npm run preview    # 预览构建产物
```

## 项目结构

```
westeros-interactive-map/
├── .github/workflows/deploy.yml  # push 到 main 后自动 build 并部署到 GitHub Pages
├── index.html              # 页面入口与手机视口设置
├── src/
│   ├── main.ts              # Vue 应用入口
│   ├── App.vue               # 唯一的组件，页面全部逻辑和样式都在这里
│   └── data/
│       └── cities.ts        # 城市数据表——新增一座城市只需要在这里加一条数据
└── public/
    ├── westeros.jpg          # 保留原始地图，用于编辑和坐标定位
    ├── westeros-*.webp       # 640/960/1920 宽度的响应式地图
    ├── background.webp       # 优化后的页面背景图
    ├── fonts/                # 本地字体及 OFL 许可证
    ├── background_bgm.mp3    # 背景音乐
    ├── sigils/                # 各城市的家族/组织纹章图
    └── cities/                # 各城市的街景/风貌图
```

## 数据模型（`src/data/cities.ts`）

```ts
type DescriptionBlock =
  | { kind: "paragraph"; text: string }
  | { kind: "heading"; level: 2 | 3; text: string }
  | { kind: "list"; items: { label?: string; text: string }[] };

interface City {
  id: string;
  name: string;
  ruler: string;           // 统治家族或组织
  x: number;                // 地图图片上的横向百分比坐标（0~100）
  y: number;                // 地图图片上的纵向百分比坐标（0~100）
  side: "left" | "right";   // 弹窗从地图哪一侧滑出
  sigil: string;             // 家徽图路径，放在 public/sigils 下
  image: string;             // 街景图路径，放在 public/cities 下
  description: DescriptionBlock[];
}
```

新增城市只需要往 `cities` 数组追加一条数据，标记点和弹窗渲染是 `v-for="city in cities"` 完全数据驱动的，不用改 `App.vue` 里的任何逻辑。

**坐标定位建议做法**：用 Python/Pillow 把 `public/westeros.jpg` 按目标区域裁剪、叠加网格线标注，找到标记点的像素中心，按 `x / 1920 * 100`、`y / 2716 * 100`（原图尺寸 1920×2716）换算成百分比写入 `cities.ts`。

## 目前已收录的城市

旧镇 Oldtown · 高庭 Highgarden · 临冬城 Winterfell · 黑城堡 Castle Black · 凯岩城 Casterly Rock · 君临 King's Landing

## 开发笔记

详细的开发过程、踩过的坑和遗留问题见 [`HANDOFF.md`](./HANDOFF.md)。

## 移动端与资源加载

- 手机竖屏使用完整动态视口高度；矮屏横屏允许上下滚动地图。
- 手机详情面板支持展开阅读、独立滚动、关闭与 Escape 返回；主要按钮点击区域至少 44px。
- 背景音乐仅在用户点击后设置音频地址。播放失败会提示重试。
- 地图通过 `picture` / `srcset` 选择 WebP 尺寸，手机候选图最高 960px；原始 JPEG 保留但不参与页面加载。
- 城市详情使用 800px 宽 WebP，顶部图片作为裁切预览展示。
- 标题字体由本地 `public/fonts` 提供，无需连接 Google Fonts；许可证见 `public/fonts/OFL.txt`。

发布前执行 `npm run build`，再用 `npm run preview` 检查 `/westeros-interactive-map/` 子路径。
