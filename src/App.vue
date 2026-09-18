<script setup lang="ts">
import { computed, ref } from "vue";
import { cities, type City } from "./data/cities";

// 部署在 GitHub Pages 的子路径下，public 目录里的静态资源需要拼上这个前缀才能正确加载
const base = import.meta.env.BASE_URL;

const audioRef = ref<HTMLAudioElement | null>(null);
const isPlaying = ref(false);

function toggleMusic() {
  const audio = audioRef.value;
  if (!audio) return;

  if (audio.paused) {
    audio.play();
    isPlaying.value = true;
  } else {
    audio.pause();
    isPlaying.value = false;
  }
}

// selectedCity 在弹窗关闭动画播完之前都保留数据，避免面板在滑出过程中内容跳变
const selectedCity = ref<City | null>(null);
const panelOpen = ref(false);

function selectCity(city: City) {
  selectedCity.value = city;
  panelOpen.value = true;
}
function closePanel() {
  panelOpen.value = false;
}
function clearSelectedCity() {
  selectedCity.value = null;
}

// 君临是铁王座所在地，给统治家族的英文名单独做花体点缀、
// 整个简介区域套上一层皇家配色，其余城市维持原有的朴素风格
const isRoyalCapital = computed(() => selectedCity.value?.id === "kings-landing");

// ruler 字段格式统一为“中文家族名 English House Name”，按第一个空格拆开
const rulerCn = computed(() => selectedCity.value?.ruler.split(" ")[0] ?? "");
const rulerEn = computed(() =>
  selectedCity.value?.ruler.split(" ").slice(1).join(" ") ?? ""
);
</script>

<template>
  <main
    class="map-container"
    :style="{
      backgroundImage: `linear-gradient(rgba(15, 23, 42, 0.6), rgba(15, 23, 42, 0.6)), url(${base}background.jpg)`,
    }"
  >
    <h1 class="map-title">WESTEROS</h1>

    <!-- map-frame 的宽高比锁定为原图比例，这样城市坐标的百分比定位才能精确对齐图片像素 -->
    <div class="map-frame">
      <img :src="`${base}westeros.jpg`" alt="维斯特洛地图" class="map-image" />

      <button
        v-for="city in cities"
        :key="city.id"
        type="button"
        class="city-marker"
        :style="{ left: city.x + '%', top: city.y + '%' }"
        :aria-label="`查看${city.name}的介绍`"
        @click="selectCity(city)"
      >
        <span class="city-marker-dot"></span>
        <span class="city-marker-label">{{ city.name }}</span>
      </button>
    </div>

    <!-- 把 background_bgm.mp3 换成你放进 public 文件夹的音乐文件名 -->
    <audio ref="audioRef" :src="`${base}background_bgm.mp3`" loop preload="auto"></audio>
    <button
      class="bgm-toggle"
      type="button"
      :aria-label="isPlaying ? '暂停背景音乐' : '播放背景音乐'"
      @click="toggleMusic"
    >
      {{ isPlaying ? "🔊" : "🔈" }}
    </button>

    <Transition
      :name="selectedCity?.side === 'right' ? 'slide-right' : 'slide-left'"
      @after-leave="clearSelectedCity"
    >
      <aside
        v-if="panelOpen && selectedCity"
        class="city-panel"
        :class="selectedCity.side"
      >
        <button
          class="city-panel-close"
          type="button"
          aria-label="关闭"
          @click="closePanel"
        >
          ×
        </button>
        <!-- 独立的可滚动内容层：金框（::before）和关闭按钮留在 city-panel 本身，
             不随内容滚动，避免长正文滚动时边框跟着挪位、露出面板外的问题 -->
        <div class="city-panel-scroll">
          <img
            :src="selectedCity.image"
            :alt="`${selectedCity.name}街景`"
            class="city-panel-image"
          />
          <div class="city-panel-body" :class="{ 'city-panel-body--royal': isRoyalCapital }">
            <h2>{{ selectedCity.name }}</h2>
            <img
              :src="selectedCity.sigil"
              :alt="`${selectedCity.ruler}的纹章`"
              class="city-panel-sigil"
            />
            <p class="city-panel-house">
              统治：{{ rulerCn }}
              <span v-if="isRoyalCapital" class="house-fleur" aria-hidden="true">⚜</span>
              <span :class="{ 'house-cursive': isRoyalCapital }">{{ rulerEn }}</span>
              <span v-if="isRoyalCapital" class="house-fleur" aria-hidden="true">⚜</span>
            </p>

            <template
              v-for="(block, index) in selectedCity.description"
              :key="index"
            >
              <h3
                v-if="block.kind === 'heading' && block.level === 2"
                class="city-panel-h2"
              >
                {{ block.text }}
              </h3>
              <h4
                v-else-if="block.kind === 'heading' && block.level === 3"
                class="city-panel-h3"
              >
                {{ block.text }}
              </h4>
              <p v-else-if="block.kind === 'paragraph'" class="city-panel-desc">
                {{ block.text }}
              </p>
              <ul v-else-if="block.kind === 'list'" class="city-panel-list">
                <li v-for="(item, itemIndex) in block.items" :key="itemIndex"><strong v-if="item.label">{{ item.label }}：</strong>{{ item.text }}</li>
              </ul>
            </template>
          </div>
        </div>
      </aside>
    </Transition>
  </main>
</template>
<style scoped>
main {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  font-family: system-ui, sans-serif;
  color: #e5e7eb;
  /* 当窗口比例不是 16:9 时，.map-container 会小于整个视口，
     多出来的部分露出这个深色背景，形成有意为之的“letterbox”边框，
     而不是背景图被 cover 硬裁切变形 */
  background: #0f172a;
}
.map-container {
  text-align: center;
  padding: 12px;
  /* 用 min() 在宽、高两个方向分别求出不超出视口的极限值，
     两者天然保持 1920:1080（16:9），与 background.jpg 的原始尺寸完全一致，
     容器形状永远和图片形状相同，background-size: cover 就不会再裁掉画面内容 */
  width: min(100vw, 177.7778vh);
  height: min(100vh, 56.25vw);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 8px;
  overflow: hidden; /* 防止图片超出边界 */
  box-sizing: border-box;

  /* background-image 通过内联样式绑定（见 template 里的 :style），
     这样 public 目录下的图片在部署到子路径时也能正确拼出 base 前缀 */
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
}
.map-title {
  /* 哥特花体，来自 index.html 里引入的 Google Fonts */
  font-family: "UnifrakturMaguntia", "Times New Roman", serif;
  font-weight: 400;
  font-size: 2.75rem;
  letter-spacing: 4px;
  margin: 0;
  line-height: 1;
  flex-shrink: 0;
  color: #f5f0e6; /* 米白色，比纯白更衬托古旧质感 */
  text-shadow:
    0 0 12px rgba(0, 0, 0, 0.8),
    0 2px 4px rgba(0, 0, 0, 0.9);
}
.map-frame {
  position: relative;
  /* 与 westeros.jpg 的原始宽高比一致，图片在框内不会有留白，
     百分比坐标才能精确对应图片上的像素位置。
     width/height 都保持 auto，只用 max-height/max-width 限制，
     aspect-ratio 会在两者间自动取更小的一边，不会拉伸变形 */
  aspect-ratio: 1920 / 2716;
  /* 减去标题高度、上下 padding 和 gap，避免总高度超出 .map-container 后被 overflow:hidden 裁掉；
     改用相对 .map-container 自身尺寸的百分比，letterbox 之后容器变小时地图也跟着等比缩小 */
  max-height: calc(100% - 90px);
  max-width: 92%;
  flex-shrink: 0;
}
.map-image {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: cover;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
  border: 2px solid #333;
}

.city-marker {
  position: absolute;
  transform: translate(-50%, -50%);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  background: none;
  border: none;
  padding: 6px;
  cursor: pointer;
}
.city-marker-dot {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background: radial-gradient(circle at 30% 30%, #ffe9a8, #c9962c 70%, #7a5a12);
  border: 2px solid #f5f0e6;
  box-shadow: 0 0 8px rgba(255, 215, 120, 0.9);
  transition: transform 0.2s;
}
.city-marker-label {
  font-size: 0.7rem;
  color: #f5f0e6;
  text-shadow: 0 1px 3px rgba(0, 0, 0, 0.9);
  white-space: nowrap;
  opacity: 0;
  transition: opacity 0.2s;
  pointer-events: none;
}
.city-marker:hover .city-marker-dot,
.city-marker:focus-visible .city-marker-dot {
  transform: scale(1.3);
}
.city-marker:hover .city-marker-label,
.city-marker:focus-visible .city-marker-label {
  opacity: 1;
}

.city-panel {
  position: fixed;
  top: 0;
  bottom: 0;
  width: min(420px, 88vw);
  background: rgba(15, 23, 42, 0.94);
  color: #f5f0e6;
  box-shadow: 0 0 30px rgba(0, 0, 0, 0.6);
  z-index: 20;
  text-align: left;
}
/* 真正滚动的内容层：面板本身（含金框、关闭按钮）不再滚动，
   只有这一层滚动，边框永远贴在面板可视边缘，不会被卷动带偏 */
.city-panel-scroll {
  height: 100%;
  overflow-y: auto;
  overflow-x: hidden; /* 保险：防止通栏图片的负 margin 计算稍有偏差时溢出面板、压住金框 */
  box-sizing: border-box;
  padding: 32px 24px;
}
.city-panel.left {
  left: 0;
  border-right: 1px solid rgba(245, 240, 230, 0.25);
}
.city-panel.right {
  right: 0;
  border-left: 1px solid rgba(245, 240, 230, 0.25);
}
.city-panel-close {
  position: absolute;
  top: 12px;
  right: 16px;
  width: 28px;
  height: 28px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  background: rgba(15, 23, 42, 0.55);
  border: none;
  color: #f5f0e6;
  font-size: 20px;
  line-height: 1;
  cursor: pointer;
  /* 提高到 10：确保始终悬浮在通栏图片之上，不会被压住 */
  z-index: 10;
}
.city-panel-image {
  display: block;
  /* 让街景图片撑满面板顶部两侧的 padding，做出通栏 banner 的效果 */
  width: calc(100% + 48px);
  margin: -32px -24px 24px;
  /* 高度不再写死，改成 auto：每张图按自己的原始宽高比、跟着通栏宽度等比缩放，
     图片本身多高，相框就多高，不再有裁切或者固定 280px 导致的上下留白 */
  height: auto;
  background: rgba(15, 23, 42, 0.94);
  border-bottom: 1px solid rgba(245, 240, 230, 0.25);
}
.city-panel-sigil {
  display: block;
  width: 72px;
  height: 72px;
  object-fit: contain;
  margin: 4px auto 12px;
  filter: drop-shadow(0 4px 10px rgba(0, 0, 0, 0.6));
}
.city-panel h2 {
  font-family: 'UnifrakturMaguntia', 'Times New Roman', serif;
  font-size: 2rem;
  text-align: center;
  margin: 0 0 4px;
  color: #f5f0e6;
}
.city-panel-house {
  text-align: center;
  font-style: italic;
  color: #d8c9a3;
  margin: 0 0 16px;
  font-size: 0.9rem;
}
/* 君临简介全文卡片：羊皮纸金黄渐变 + 深金描边，从标题一路铺到最后一段，
   衬出铁王座所在地九五至尊的皇家气派（其余城市不加这层 --royal class，维持原本朴素配色） */
.city-panel-body--royal {
  margin: -8px -16px -8px;
  padding: 20px 16px 22px;
  border-radius: 8px;
  background:
    radial-gradient(120% 45% at 50% 0%, rgba(255, 244, 214, 0.55), transparent 65%),
    linear-gradient(165deg, #f3dfa3 0%, #e6c565 40%, #d4a944 70%, #b8873a 100%);
  border: 1px solid rgba(120, 78, 15, 0.65);
  box-shadow:
    inset 0 0 30px rgba(120, 78, 15, 0.25),
    0 4px 18px rgba(0, 0, 0, 0.45);
}
/* 背景转为浅色羊皮纸调后，卡片内文字统一换成墨褐色系，确保可读性 */
.city-panel-body--royal h2,
.city-panel-body--royal .city-panel-h2 {
  color: #4a2f0d;
}
.city-panel-body--royal .city-panel-h2 {
  border-bottom-color: rgba(120, 78, 15, 0.4);
}
.city-panel-body--royal .city-panel-h3 {
  color: #6b4a15;
}
.city-panel-body--royal .city-panel-house {
  color: #5c3d10;
}
.city-panel-body--royal .city-panel-desc {
  color: #3b2a12;
}
.city-panel-body--royal .city-panel-list {
  color: #3b2a12;
}
.city-panel-body--royal .city-panel-list li::marker {
  color: #8a5a12;
}
.city-panel-body--royal .city-panel-list strong {
  color: #4a2f0d;
}
/* 英文家族名花体点缀：与页面顶部 WESTEROS 标题呼应的哥特花体 */
.house-cursive {
  font-family: "UnifrakturMaguntia", "Times New Roman", serif;
  font-style: normal;
  font-size: 1.15em;
  letter-spacing: 1px;
  color: #7a4e0a;
  text-shadow: 0 0 6px rgba(120, 78, 15, 0.25);
}
/* 家族名两侧的鸢尾花装饰 */
.house-fleur {
  color: #8a5a12;
  font-size: 0.85em;
  margin: 0 6px;
  vertical-align: middle;
}
.city-panel-desc {
  font-size: 0.95rem;
  line-height: 1.7;
  margin: 0 0 14px;
}
.city-panel-h2 {
  font-family: 'UnifrakturMaguntia', 'Times New Roman', serif;
  font-size: 1.4rem;
  font-weight: 400;
  letter-spacing: 1px;
  color: #f5f0e6;
  margin: 24px 0 10px;
  padding-bottom: 6px;
  border-bottom: 1px solid rgba(245, 240, 230, 0.3);
}
.city-panel-h3 {
  font-size: 1.05rem;
  font-weight: 600;
  color: #d8c9a3;
  margin: 18px 0 8px;
}
.city-panel-list {
  margin: 0 0 14px;
  padding-left: 20px;
  font-size: 0.95rem;
  line-height: 1.7;
}
.city-panel-list li {
  margin-bottom: 8px;
}
.city-panel-list li::marker {
  color: #c9962c;
}
.city-panel-list strong {
  color: #f5f0e6;
}
.city-panel-desc:last-child,
.city-panel-list:last-child {
  margin-bottom: 0;
}

.slide-left-enter-active,
.slide-left-leave-active,
.slide-right-enter-active,
.slide-right-leave-active {
  transition: transform 0.35s ease;
}
.slide-left-enter-from,
.slide-left-leave-to {
  transform: translateX(-100%);
}
.slide-right-enter-from,
.slide-right-leave-to {
  transform: translateX(100%);
}
.bgm-toggle {
  position: fixed;
  top: 20px;
  right: 20px;
  width: 48px;
  height: 48px;
  border-radius: 50%;
  border: 1px solid rgba(245, 240, 230, 0.4);
  background: rgba(15, 23, 42, 0.6);
  color: #f5f0e6;
  font-size: 20px;
  line-height: 1;
  cursor: pointer;
  backdrop-filter: blur(4px);
  transition:
    background 0.2s,
    transform 0.2s;
}
.bgm-toggle:hover {
  background: rgba(15, 23, 42, 0.85);
  transform: scale(1.08);
}
</style>
