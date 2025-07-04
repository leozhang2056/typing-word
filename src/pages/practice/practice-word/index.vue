<script setup lang="ts">

import TypingWord from "@/pages/practice/practice-word/TypingWord.vue";
import {$ref} from "vue/macros";
import {cloneDeep} from "lodash-es";
import {useBaseStore} from "@/stores/base.ts";
import {onMounted, onUnmounted, ref, watch} from "vue";
import {useRuntimeStore} from "@/stores/runtime.ts";
import {ShortcutKey, Word} from "@/types.ts";
import {emitter, EventKey} from "@/utils/eventBus.ts";
import {useSettingStore} from "@/stores/setting.ts";
import {syncMyDictList} from "@/hooks/dict.ts";

const store = useBaseStore()
const runtimeStore = useRuntimeStore()
const settingStore = useSettingStore()

let wordData = $ref({
  words: [],
  index: -1
})

const typingWordRef = ref();
const carouselEnabled = ref(false);
const carouselInterval = ref(3); // 默认3秒
let carouselTimer: any = null;

function getCurrentPractice() {
  if (store.chapter.length) {
    wordData.words = store.chapter
    wordData.index = 0

    store.chapter.map((w: Word) => {
      if (!w.trans.length) {
        let res = runtimeStore.translateWordList.find(a => a.name === w.name)
        if (res) w = Object.assign(w, res)
      }
    })

    wordData.words = cloneDeep(store.chapter)
    emitter.emit(EventKey.resetWord)
  }
}

function sort(list: Word[]) {
  store.currentDict.chapterWords[store.currentDict.chapterIndex] = wordData.words = list
  wordData.index = 0
  syncMyDictList(store.currentDict)
}

function next() {
  if (store.currentDict.chapterIndex >= store.currentDict.chapterWords.length - 1) {
    store.currentDict.chapterIndex = 0
  } else store.currentDict.chapterIndex++

  getCurrentPractice()
}

function startCarousel() {
  console.log('startCarousel called, carouselEnabled:', carouselEnabled.value, 'interval:', carouselInterval.value)
  stopCarousel();
  if (carouselEnabled.value) {
    console.log('Starting carousel timer...')
    carouselTimer = setInterval(() => {
      console.log('Carousel timer triggered, calling next...')
      if (typingWordRef.value && typingWordRef.value.next) {
        typingWordRef.value.next(false);
      } else {
        console.log('typingWordRef or next method not available')
      }
    }, carouselInterval.value * 1000);
  }
}

function stopCarousel() {
  console.log('stopCarousel called')
  if (carouselTimer) {
    clearInterval(carouselTimer);
    carouselTimer = null;
  }
}

watch([carouselEnabled, carouselInterval], () => {
  console.log('Watch triggered - carouselEnabled:', carouselEnabled.value, 'carouselInterval:', carouselInterval.value)
  if (carouselEnabled.value) {
    startCarousel();
  } else {
    stopCarousel();
  }
});

onMounted(() => {
  getCurrentPractice()
  emitter.on(EventKey.changeDict, getCurrentPractice)
  emitter.on(EventKey.next, next)
  emitter.on(ShortcutKey.NextChapter, next)
  emitter.on('getCarouselSetting', () => {
    console.log('getCarouselSetting event received')
    emitter.emit('setCarouselSetting', {enabled: carouselEnabled.value, interval: carouselInterval.value})
  })
  emitter.on('updateCarouselSetting', (payload: {enabled: boolean, interval: number}) => {
    console.log('updateCarouselSetting event received:', payload)
    carouselEnabled.value = payload.enabled
    carouselInterval.value = payload.interval
  })
  emitter.on(EventKey.chapterFinished, () => {
    if (carouselEnabled.value) {
      console.log('章节结束，自动进入下一章')
      next()
    }
  })
})

onUnmounted(() => {
  emitter.off(EventKey.changeDict, getCurrentPractice)
  emitter.off(EventKey.next, next)
  emitter.off(ShortcutKey.NextChapter, next)
  emitter.off('getCarouselSetting', () => {})
  emitter.off('updateCarouselSetting', () => {})
  emitter.off(EventKey.chapterFinished, () => {})
  stopCarousel();
})

defineExpose({getCurrentPractice})

</script>

<template>
  <div class="practice">
    <TypingWord
        ref="typingWordRef"
        @sort="sort"
        v-model:words="wordData.words"
        :index="wordData.index"/>
  </div>
</template>

<style scoped lang="scss">
.practice {
  //height: 100%;
  flex: 1;
}
</style>