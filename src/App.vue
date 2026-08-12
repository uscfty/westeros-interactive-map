<script setup lang="ts">
import { ref } from "vue";
import { cities, type City } from "./data/cities";

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
</script>

<template>
  <main class="map-container">
    <h1 class="map-title">WESTEROS</h1>

    <!-- map-frame 的宽高比锁定为原图比例，这样城市坐标的百分比定位才能精确对齐图片像素 -->
    <div class="map-frame">
      <img src="/westeros.jpg" alt="维斯特洛地图" class="map-image" />

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

    <!-- 把 /background_bgm.mp3 换成你放进 public 文件夹的音乐文件名 -->
    <audio ref="audioRef" src="/background_bgm.mp3" loop preload="auto"></audio>
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
        <img
          :src="selectedCity.image"
          :alt="`${selectedCity.name}街景`"
          class="city-panel-image"
        />
        <h2>{{ selectedCity.name }}</h2>
        <img
          :src="selectedCity.sigil"
          :alt="`${selectedCity.ruler}的纹章`"
          class="city-panel-sigil"
        />
        <p class="city-panel-house">统治：{{ selectedCity.ruler }}</p>

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
  height: 100vh;
  font-family: system-ui, sans-serif;
  color: #e5e7eb;
  background: #0f172a; /* 图片加载出来之前的兜底颜色 */
}
.map-container {
  text-align: center;
  padding: 12px;
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 8px;
  overflow: hidden; /* 防止图片超出边界 */
  box-sizing: border-box;

  /* 把 /background_bgm.mp3 换成你自己放进 public 文件夹的音乐文件名 */
  background-image:
    linear-gradient(rgba(15, 23, 42, 0.6), rgba(15, 23, 42, 0.6)),
    url("background.jpg");
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
  /* 100vh 减去标题高度、上下 padding 和 gap，避免总高度超出屏幕后被 overflow:hidden 裁掉 */
  max-height: calc(100vh - 90px);
  max-width: 92vw;
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
  overflow-y: auto;
  background: rgba(15, 23, 42, 0.94);
  padding: 32px 24px;
  color: #f5f0e6;
  box-shadow: 0 0 30px rgba(0, 0, 0, 0.6);
  z-index: 20;
  text-align: left;
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
  z-index: 2;
}
.city-panel-image {
  display: block;
  /* 让街景图片撑满面板顶部两侧的 padding，做出通栏 banner 的效果 */
  width: calc(100% + 48px);
  margin: -32px -24px 20px;
  height: 280px;
  /* contain 保证整张图完整显示，不会被裁掉；上下留白会露出面板底色，和整体风格一致 */
  object-fit: contain;
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
