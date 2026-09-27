<script setup lang="ts">
import { computed, nextTick, ref } from "vue";
import { cities, type City } from "./data/cities";

// 部署在 GitHub Pages 的子路径下，public 目录里的静态资源需要拼上这个前缀才能正确加载
const base = import.meta.env.BASE_URL;

const audioRef = ref<HTMLAudioElement | null>(null);
const isPlaying = ref(false);

const audioSource = ref<string>();
const musicLoading = ref(false);
const musicError = ref("");
const closeButton = ref<HTMLButtonElement | null>(null);
let cityTrigger: HTMLElement | null = null;

async function toggleMusic() {
  const audio = audioRef.value;
  if (!audio || musicLoading.value) return;
  musicError.value = "";
  if (!audio.paused) {
    audio.pause();
    return;
  }
  musicLoading.value = true;
  try {
    if (!audioSource.value) {
      audioSource.value = `${base}background_bgm.mp3`;
      // Assign immediately to preserve the tap gesture required by iOS audio.
      audio.src = audioSource.value;
    }
    await audio.play();
  } catch {
    musicError.value = "音乐暂时无法播放，请再试一次";
  } finally {
    musicLoading.value = false;
  }
}

// selectedCity 在弹窗关闭动画播完之前都保留数据，避免面板在滑出过程中内容跳变
const selectedCity = ref<City | null>(null);
const panelOpen = ref(false);
const panelExpanded = ref(false);

async function selectCity(city: City, event: MouseEvent) {
  cityTrigger = event.currentTarget as HTMLElement;
  selectedCity.value = city;
  panelExpanded.value = false;
  panelOpen.value = true;
  await nextTick();
  closeButton.value?.focus({ preventScroll: true });
}
function closePanel() {
  panelOpen.value = false;
  cityTrigger?.focus({ preventScroll: true });
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
      backgroundImage: `linear-gradient(rgba(15, 23, 42, 0.6), rgba(15, 23, 42, 0.6)), url(${base}background.webp)`,
    }"
  >
    <h1 class="map-title">WESTEROS</h1>

    <!-- map-frame 的宽高比锁定为原图比例，这样城市坐标的百分比定位才能精确对齐图片像素 -->
    <div class="map-frame">
      <picture>
      <source media="(max-width: 950px)" :srcset="`${base}westeros-640.webp 640w, ${base}westeros-960.webp 960w`" sizes="calc(100vw - 24px)" />
      <img
        :src="`${base}westeros-960.webp`"
        :srcset="`${base}westeros-640.webp 640w, ${base}westeros-960.webp 960w, ${base}westeros-1920.webp 1920w`"
        sizes="(max-width: 600px) calc(100vw - 24px), 600px"
        width="1920" height="2716" fetchpriority="high"
        alt="维斯特洛地图" class="map-image"
      />
      </picture>

      <button
        v-for="city in cities"
        :key="city.id"
        type="button"
        class="city-marker"
        :style="{ left: city.x + '%', top: city.y + '%' }"
        :aria-label="`查看${city.name}的介绍`"
        @click="selectCity(city, $event)"
      >
        <span class="city-marker-dot"></span>
        <span class="city-marker-label">{{ city.name.split(" ")[0] }}</span>
      </button>
    </div>

    <!-- 音乐仅在点击播放后设置 src，避免首屏下载。 -->
    <audio ref="audioRef" :src="audioSource" loop preload="none"
      @play="isPlaying = true" @pause="isPlaying = false" @error="isPlaying = false"
    ></audio>
    <button
      class="bgm-toggle"
      :disabled="musicLoading"
      :aria-busy="musicLoading"
      :aria-pressed="isPlaying"
      type="button"
      :aria-label="isPlaying ? '暂停背景音乐' : '播放背景音乐'"
      @click="toggleMusic"
    >
      {{ musicLoading ? "…" : isPlaying ? "🔊" : "🔈" }}
    </button>

    <p v-if="musicError" class="music-error" role="status">{{ musicError }}</p>

    <Transition
      :name="selectedCity?.side === 'right' ? 'slide-right' : 'slide-left'"
      @after-leave="clearSelectedCity"
    >
      <aside
        v-if="panelOpen && selectedCity"
        class="city-panel"
        aria-labelledby="city-heading"
        @keydown.esc="closePanel"
        :class="[selectedCity.side, { 'city-panel--expanded': panelExpanded }]"
      >
        <button
          ref="closeButton"
          class="city-panel-close"
          type="button"
          aria-label="关闭"
          @click="closePanel"
        >
          ×
        </button>
        <button class="city-panel-expand" type="button" :aria-expanded="panelExpanded" @click="panelExpanded = !panelExpanded">
          {{ panelExpanded ? "收起" : "展开阅读" }}
        </button>
        <!-- 独立的可滚动内容层：金框（::before）和关闭按钮留在 city-panel 本身，
             不随内容滚动，避免长正文滚动时边框跟着挪位、露出面板外的问题 -->
        <div class="city-panel-scroll">
          <img
            :src="selectedCity.image"
            :alt="`${selectedCity.name}街景`"
            class="city-panel-image"
            :class="{ 'city-panel-image--complete': selectedCity.imageCredit }"
            decoding="async"
          />
          <p v-if="selectedCity.imageCredit" class="city-panel-credit">{{ selectedCity.imageCredit }}</p>
          <div class="city-panel-body" :class="{ 'city-panel-body--royal': isRoyalCapital }">
            <h2 id="city-heading">{{ selectedCity.name }}</h2>
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
            <footer v-if="selectedCity.sources?.length" class="city-panel-sources">
              <p>参考资料</p>
              <a v-for="source in selectedCity.sources" :key="source.url" :href="source.url" target="_blank" rel="noopener noreferrer">{{ source.title }}</a>
            </footer>
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
  height: 100dvh; /* iOS Safari 地址栏收起/展开时 100vh 会跳动，100dvh 跟手；不支持的浏览器回退到上一行的 100vh */
  overflow: hidden;
  font-family: system-ui, sans-serif;
  color: #e5e7eb;
  background: #0f172a;
}
.map-container {
  text-align: center;
  width: 100%;
  height: 100dvh;
  padding: 16px 12px;
  gap: 12px;
  box-sizing: border-box;
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
}
.map-title {
  /* 哥特花体由 src/style.css 中的本地 @font-face 提供。 */
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
  width: min(100%, calc((100dvh - 100px) * 1920 / 2716));
  aspect-ratio: 1920 / 2716;
  flex-shrink: 0;
}
.map-image {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: cover;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
  box-sizing: border-box;
  border: 2px solid #333;
}

.city-marker {
  position: absolute;
  transform: translate(-50%, -50%);
  display: grid;
  place-items: center;
  width: 44px;
  height: 44px;
  padding: 0;
  border: 0;
  border-radius: 50%;
  background: transparent;
  cursor: pointer;
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
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
  position: absolute;
  top: 34px;
  font-size: 13px;
  line-height: 18px;
  color: #f5f0e6;
  text-shadow: 0 1px 3px rgba(0, 0, 0, 0.9);
  white-space: nowrap;
  opacity: 0;
  transition: opacity 0.2s;
  pointer-events: none;
}
.city-marker:hover .city-marker-dot,
.city-marker:focus-visible .city-marker-dot,
.city-marker:active .city-marker-dot {
  transform: scale(1.3);
}
.city-marker:hover .city-marker-label,
.city-marker:focus-visible .city-marker-label {
  opacity: 1;
}
@media (hover: none) {
  .city-marker-label { opacity: 1; }
}
.city-marker:focus-visible, .bgm-toggle:focus-visible, .city-panel-close:focus-visible {
  outline: 2px solid #f5f0e6;
  outline-offset: 2px;
}

.city-panel {
  position: fixed;
  top: 0;
  bottom: 0;
  width: min(420px, 92vw);
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
  -webkit-overflow-scrolling: touch; /* iOS Safari 惯性滚动 */
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
  width: 44px;
  height: 44px;
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
  -webkit-tap-highlight-color: transparent;
  /* 提高到 10：确保始终悬浮在通栏图片之上，不会被压住 */
  z-index: 10;
}
.city-panel-image {
  display: block;
  /* 让街景图片撑满面板顶部两侧的 padding，做出通栏 banner 的效果 */
  width: calc(100% + 48px);
  margin: -32px -24px 24px;
  height: auto;
  aspect-ratio: 16 / 9;
  object-fit: cover;
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
  font-size: 1rem;
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
  font-size: 1rem;
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
  -webkit-tap-highlight-color: transparent;
  backdrop-filter: blur(4px);
  transition:
    background 0.2s,
    transform 0.2s;
}
.bgm-toggle:hover {
  background: rgba(15, 23, 42, 0.85);
  transform: scale(1.08);
}

.city-panel-expand { display: none; }
.music-error {
  position: fixed;
  top: 76px;
  right: 12px;
  max-width: calc(100vw - 48px);
  padding: 12px;
  background: #0f172a;
  z-index: 30;
  font-size: 14px;
}
@media (max-width: 600px), (max-height: 500px) and (pointer: coarse) {
  .map-container {
    padding-top: max(12px, env(safe-area-inset-top));
    padding-bottom: max(12px, env(safe-area-inset-bottom));
    gap: 12px;
  }
  .map-title { font-size: 1.75rem; letter-spacing: 2px; }
  .map-frame {
    width: min(100%, calc((100dvh - 88px - env(safe-area-inset-top) - env(safe-area-inset-bottom)) * 1920 / 2716));
  }
  .bgm-toggle {
    width: 44px;
    height: 44px;
    top: max(12px, env(safe-area-inset-top));
    right: max(12px, env(safe-area-inset-right));
  }
  .city-panel.left, .city-panel.right {
    top: auto;
    bottom: 0;
    left: 0;
    right: 0;
    width: 100%;
    height: 72dvh;
    max-height: calc(100dvh - max(16px, env(safe-area-inset-top)));
    border: 1px solid rgba(245, 240, 230, .25);
    border-bottom: 0;
    border-radius: 18px 18px 0 0;
    box-sizing: border-box;
    overflow: hidden;
  }
  .city-panel.city-panel--expanded { height: 94dvh; }
  .city-panel-expand {
    display: block;
    position: absolute;
    top: 10px;
    left: 12px;
    z-index: 10;
    min-height: 44px;
    padding: 0 14px;
    border: 1px solid rgba(245, 240, 230, .4);
    border-radius: 22px;
    color: #f5f0e6;
    background: rgba(15, 23, 42, .85);
    font: inherit;
    font-size: 14px;
    cursor: pointer;
  }
  .city-panel-scroll {
    padding: 24px 20px max(24px, env(safe-area-inset-bottom));
    overscroll-behavior: contain;
  }
  .city-panel-image {
    width: calc(100% + 40px);
    height: clamp(120px, 22dvh, 200px);
    margin: -24px -20px 20px;
  }
  .city-panel h2 { font-size: 1.5rem; }
  .city-panel-sigil { width: 56px; height: 56px; }
  .city-panel-close { top: 10px; right: 12px; }
  .slide-left-enter-from, .slide-left-leave-to,
  .slide-right-enter-from, .slide-right-leave-to { transform: translateY(100%); }
}
@media (max-height: 500px) and (pointer: coarse) {
  .map-container { justify-content: flex-start; overflow-y: auto; }
  .map-frame { width: min(100%, 420px); }
}
.city-panel-image.city-panel-image--complete { height: auto; aspect-ratio: auto; }
.city-panel-credit { font-size: 12px; line-height: 1.5; color: #d8c9a3; margin: -8px 0 16px; }
.city-panel-sources { margin-top: 24px; border-top: 1px solid #f5f0e640; padding-top: 12px; font-size: 13px; }
.city-panel-sources p { margin: 0 0 8px; }
.city-panel-sources a { display: block; color: #d8c9a3; padding: 8px 0; overflow-wrap: anywhere; }
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { transition: none !important; }
}
</style>
