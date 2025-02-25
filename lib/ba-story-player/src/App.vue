<script lang="ts" setup>
import axios from "axios";
import { ref, watch } from "vue";
import BaStoryPlayer from "../lib/BaStoryPlayer.vue";
import eventBus from "../lib/eventBus";
import { eventEmitter, resourcesLoader, storyHandler } from "../lib/index";
import { changeStoryIndex } from "../lib/layers/uiLayer/userInteract";
import { usePlayerStore } from "../lib/stores";
import { TranslatedStoryUnit } from "../lib/types/common";
import { Language } from "../lib/types/store";
import ModifyEmotionOption from "./components/ModifyEmotionOption.vue";
import ModifyLive2DOption from "./components/ModifyLive2DOption.vue";
import TestEffect from "./components/TestEffect.vue";
import UnitTest from "./components/UnitTest.vue";
import yuuka from "./data/yuuka.json";
import { useResizeObserver, useThrottleFn } from "@vueuse/core";
import { Assets } from "pixi.js";

console.log("资源加载: ", resourcesLoader);
console.log("资源调用: ", usePlayerStore());
console.log("剧情进度: ", storyHandler);

eventBus.on("*", (type, e) => {
  if (
    !(
      type === "l2dAnimationDone" &&
      (e as { done: boolean; animation: string }).animation.startsWith(
        "Idle_01"
      )
    )
  )
    console.log("事件类型", type, "值", e);
});

let storySummary = {
  chapterName: "章节名",
  summary:
    "从奇怪的梦中醒来之后的[USERNAME]老师从联邦学生会的干部七神凛那里听到学生会长失踪的消息。由于学生会长失踪，学园城市基沃托斯陷入了混乱。为了解决这场混乱，老师和学生会的干部一同前往夏莱办公室。",
};
let toolType = ref("");
let cacheKey = "toolType";
if (localStorage.getItem(cacheKey)) {
  toolType.value = localStorage.getItem(cacheKey) as string;
}
watch(toolType, () => {
  localStorage.setItem(cacheKey, toolType.value);
});

Reflect.set(window, "baResource", resourcesLoader);
Reflect.set(window, "baStory", storyHandler);
Reflect.set(window, "baStore", usePlayerStore());
Reflect.set(window, "eventBus", eventBus);
Reflect.set(window, "baEvent", eventEmitter);
Reflect.set(window, "Assets", Assets);
Reflect.set(window, "next", () => {
  eventBus.emit("characterDone");
  eventBus.emit("effectDone");
  eventBus.emit("next");
});

let width = ref(1000);
const height = ref(562.5);

const currentStoryIndex = ref(0);
const indexCacheKey = "storyIndex";
const cacheIndex = localStorage.getItem(indexCacheKey);
if (cacheIndex) {
  currentStoryIndex.value = Number(cacheIndex);
  storyHandler.currentStoryIndex = currentStoryIndex.value;
}
/**
 * 设置开始的storyIndex
 */
function setStartIndex() {
  localStorage.setItem(indexCacheKey, currentStoryIndex.value.toString());
}

const story = ref<TranslatedStoryUnit>(yuuka);
const showPlayer = ref(false);
const storyJsonName = ref("0");
const storyCacheKey = "storyJson";
const jsonName = localStorage.getItem(storyCacheKey);

if (jsonName && jsonName !== "0") {
  // 从预览部署获取
  // 200362 → https://preview.blue-archive.io/story/favor/20036/200362.json
  axios
    .get(
      `https://preview.blue-archive.io/story/favor/${jsonName.slice(
        0,
        5
      )}/${jsonName}.json`
    )
    .then(response => {
      if (response.status === 200) {
        story.value = response.data;
        storyJsonName.value = jsonName;
        showPlayer.value = true;
      }
    })
    .catch(() => {
      // 远程获取失败，尝试本地获取
      axios.get(`${jsonName}.json`).then(response => {
        if (response.status === 200) {
          story.value = response.data;
          storyJsonName.value = jsonName;
          showPlayer.value = true;
        }
      });
    });
} else {
  showPlayer.value = true;
}
function changeJSON() {
  localStorage.setItem(storyCacheKey, storyJsonName.value);
  location.reload();
}

// 让播放器可变，方便调试，可以节流但是不流畅
const player = ref<HTMLElement | null>(null);
useResizeObserver(
  player as any,
  useThrottleFn(entries => {
    const entry = entries[0];
    if (document.fullscreenElement === null)
      ({ width: width.value, height: height.value } = entry.contentRect);
  }, 1)
);
const languageCacheKey = "language";
const languageList = ["Cn", "Jp", "Kr", "En", "Tw"];
const language = ref<Language>(
  (localStorage.getItem(languageCacheKey) as never) ?? "Cn"
);
const useSuperSampling = ref<boolean>(
  localStorage.getItem("useSuperSampling") === "true"
);
watch(
  () => [language.value, useSuperSampling.value],
  () => {
    localStorage.setItem(languageCacheKey, language.value);
    localStorage.setItem("useSuperSampling", useSuperSampling.value.toString());
    location.reload();
  }
);
</script>

<template>
  <div style="display: flex; justify-content: center">
    <div v-if="showPlayer">
      <BaStoryPlayer
        :story="story"
        data-url="https://yuuka.cdn.diyigemt.com/image/ba-all-data"
        :width="width"
        :height="height"
        :language="language"
        userName="testUser"
        :story-summary="storySummary"
        style="resize: both; overflow: hidden"
        :exit-fullscreen-time-out="2000"
        :use-super-sampling="useSuperSampling ? '2' : false"
        ref="player"
      />
      <!--其实在左边的剧情json里填入11000就能测试序章, 不需要改动这里-->
    </div>
    <div
      style="
        position: absolute;
        left: 0;
        display: flex;
        flex-direction: column;
        width: 20vh;
        z-index: 100;
      "
    >
      <label>辅助工具选择</label>
      <select v-model="toolType">
        <option value="emotion">人物特效测试</option>
        <option value="effect">特效层特效</option>
        <option value="test">单元测试</option>
        <option value="live2d">live2d测试</option>
        <option value="null">无</option>
      </select>
      <label>storyIndex</label>
      <input v-model="currentStoryIndex" />
      <button @click="setStartIndex">设为故事初始index</button>
      <button @click="changeStoryIndex(currentStoryIndex)">
        更换故事index
      </button>
      <label>故事json</label>
      <input v-model="storyJsonName" @keydown.enter="changeJSON" />
      <button @click="changeJSON">更换故事json</button>
      <button @click="showPlayer = !showPlayer">切换显示状态</button>
      <label>语言</label>
      <select v-model="language">
        <option v-for="(name, index) in languageList" :key="index">
          {{ name }}
        </option>
      </select>
      <label
        >超分
        <input type="checkbox" v-model="useSuperSampling" />
      </label>
    </div>
    <div class="absolute-right-center">
      <ModifyEmotionOption v-if="toolType === 'emotion'" />
      <Suspense>
        <TestEffect
          class="absolute-right-center"
          v-if="toolType === 'effect'"
        />
      </Suspense>
      <UnitTest class="absolute-right-center" v-if="toolType === 'test'" />
      <KeepAlive>
        <ModifyLive2DOption :class="{ hidden: toolType !== 'live2d' }" />
      </KeepAlive>
    </div>
  </div>
</template>

<style scoped>
.absolute-right-center {
  display: flex;
  position: absolute;
  right: 0;
  flex-direction: column;
  z-index: 1000;
  background-color: white;
  width: 180px;
  height: 95vh;
  overflow-y: auto;
  text-align: center;
}

.hidden {
  display: none;
}
</style>
