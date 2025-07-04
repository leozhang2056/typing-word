<script setup lang="ts">
import {$computed, $ref} from "vue/macros";
import TypingArticle from "./TypingArticle.vue";
import {
  Article,
  ArticleItem,
  ArticleWord,
  DefaultArticle,
  DisplayStatistics,
  ShortcutKey,
  TranslateType,
  Word
} from "@/types.ts";
import {cloneDeep} from "lodash-es";
import TypingWord from "@/pages/practice/practice-word/TypingWord.vue";
import Panel from "../Panel.vue";
import {onMounted, onUnmounted, watch} from "vue";
import {renewSectionTexts, renewSectionTranslates} from "@/hooks/translate.ts";
import {MessageBox} from "@/utils/MessageBox.tsx";
import {useBaseStore} from "@/stores/base.ts";
import EditSingleArticleModal from "@/components/article/EditSingleArticleModal.vue";
import {usePracticeStore} from "@/stores/practice.ts";
import {emitter, EventKey} from "@/utils/eventBus.ts";
import IconWrapper from "@/components/IconWrapper.vue";
import {Icon} from "@iconify/vue";
import Tooltip from "@/components/Tooltip.vue";
import {useRuntimeStore} from "@/stores/runtime.ts";
import {useSettingStore} from "@/stores/setting.ts";
import BaseIcon from "@/components/BaseIcon.vue";
import {syncMyDictList, useArticleOptions} from "@/hooks/dict.ts";
import ArticleList from "@/components/list/ArticleList.vue";
import {useOnKeyboardEventListener} from "@/hooks/event.ts";

const store = useBaseStore()
const practiceStore = usePracticeStore()
const runtimeStore = useRuntimeStore()

let tabIndex = $ref(0)
let wordData = $ref({
  words: [],
  index: -1
})
let articleData = $ref({
  articles: [],
  article: cloneDeep(DefaultArticle),
  sectionIndex: 0,
  sentenceIndex: 0,
  wordIndex: 0,
  stringIndex: 0,
})
let showEditArticle = $ref(false)
let typingArticleRef = $ref<any>()
let editArticle = $ref<Article>(cloneDeep(DefaultArticle))
let articleIsActive = $computed(() => tabIndex === 0)

function next() {
  if (!articleIsActive) return
  if (store.currentDict.chapterIndex >= articleData.articles.length - 1) {
    store.currentDict.chapterIndex = 0
  } else store.currentDict.chapterIndex++

  emitter.emit(EventKey.resetWord)
  getCurrentPractice()
}

function init() {
  if (!store.currentDict.articles.length) return
  articleData.articles = cloneDeep(store.currentDict.articles)
  getCurrentPractice()
}

function setArticle(val: Article) {
  let tempVal = cloneDeep(val)
  articleData.articles[store.currentDict.chapterIndex] = tempVal
  articleData.article = tempVal
  practiceStore.inputWordNumber = 0
  practiceStore.wrongWordNumber = 0
  practiceStore.repeatNumber = 0
  practiceStore.total = 0
  practiceStore.wrongWords = []
  practiceStore.startDate = Date.now()
  articleData.article.sections.map((v, i) => {
    v.map((w, j) => {
      w.words.map(s => {
        if (!store.skipWordNamesWithSimpleWords.includes(s.name.toLowerCase()) && !s.isSymbol) {
          practiceStore.total++
        }
      })
    })
  })
}

function getCurrentPractice() {
  // console.log('store.currentDict',store.currentDict)
  // return
  tabIndex = 0
  articleData.article = cloneDeep(DefaultArticle)

  let currentArticle = articleData.articles [store.currentDict.chapterIndex]
  let tempArticle = {...DefaultArticle, ...currentArticle}
  // console.log('article', tempArticle)
  if (tempArticle.sections.length) {
    setArticle(tempArticle)
  } else {
    if (tempArticle.useTranslateType === TranslateType.none) {
      renewSectionTexts(tempArticle)
      setArticle(tempArticle)
    } else {
      if (tempArticle.useTranslateType === TranslateType.custom) {
        if (tempArticle.textCustomTranslate.trim()) {
          if (tempArticle.textCustomTranslateIsFormat) {
            renewSectionTexts(tempArticle)
            renewSectionTranslates(tempArticle, tempArticle.textCustomTranslate)
            setArticle(tempArticle)
          } else {
            //说明有本地翻译，但是没格式化成一行一行的
            MessageBox.confirm('检测到存在本地翻译，但未格式化，是否进行编辑?',
                '提示',
                () => {
                  editArticle = tempArticle
                  showEditArticle = true
                },
                () => {
                  renewSectionTexts(tempArticle)
                  tempArticle.useTranslateType = TranslateType.none
                  setArticle(tempArticle)
                }, null,
                {
                  confirmButtonText: '去编辑',
                  cancelButtonText: '不需要翻译',
                })
          }
        } else {
          //没有本地翻译
          MessageBox.confirm(
              '没有本地翻译，是否进行编辑?',
              '提示',
              () => {
                editArticle = tempArticle
                showEditArticle = true
              },

              () => {
                renewSectionTexts(tempArticle)
                tempArticle.useTranslateType = TranslateType.none
                setArticle(tempArticle)
              }, null,
              {
                confirmButtonText: '去编辑',
                cancelButtonText: '不需要翻译',
              })
        }
      }

      if (tempArticle.useTranslateType === TranslateType.network) {
        renewSectionTexts(tempArticle)
        renewSectionTranslates(tempArticle, tempArticle.textNetworkTranslate)
        setArticle(tempArticle)
      }
    }
  }
}

function saveArticle(val: Article) {
  console.log('saveArticle', val)
  showEditArticle = false
  let rIndex = store.currentDict.articles.findIndex(v => v.id === val.id)
  if (rIndex > -1) {
    store.currentDict.articles[rIndex] = cloneDeep(val)
  }
  setArticle(val)
}

function edit(val: Article = articleData.article) {
  if (!articleIsActive) return
  // tabIndex = 1
  // wordData.words = [
  //   {
  //     ...cloneDeep(DefaultWord),
  //     name: 'test'
  //   }
  // ]
  // wordData.index = 0
  // return
  editArticle = val
  showEditArticle = true
}

function wrong(word: Word) {
  let lowerName = word.name.toLowerCase();
  if (!store.wrong.originWords.find((v: Word) => v.name.toLowerCase() === lowerName)) {
    store.wrong.originWords.push(word)
  }
  if (!store.skipWordNamesWithSimpleWords.includes(lowerName)) {
    if (!practiceStore.wrongWords.find((v) => v.name.toLowerCase() === lowerName)) {
      practiceStore.wrongWords.push(word)
      practiceStore.wrongWordNumber++
    }
  }
}

function collect(word: Word) {
  let lowerName = word.name.toLowerCase();
  if (!store.collect.originWords.find((v: Word) => v.name.toLowerCase() === lowerName)) {
    store.collect.originWords.push(word)
  }
}

function simple(word: Word) {
  let lowerName = word.name.toLowerCase();
  if (!store.simple.originWords.find((v: Word) => v.name.toLowerCase() === lowerName)) {
    store.simple.originWords.push(word)
  }
}

function shortcutKeyEdit() {
  if (!articleIsActive) return
  edit()
}

onMounted(() => {
  init()
  emitter.on(EventKey.changeDict, init)
  emitter.on(EventKey.next, next)
  emitter.on(ShortcutKey.NextChapter, next)
  emitter.on(ShortcutKey.ToggleCollect, collect)
  emitter.on(ShortcutKey.EditArticle, shortcutKeyEdit)
})

onUnmounted(() => {
  emitter.off(EventKey.changeDict, init)
  emitter.off(EventKey.next, next)
  emitter.off(ShortcutKey.NextChapter, next)
  emitter.off(ShortcutKey.ToggleCollect, collect)
  emitter.off(ShortcutKey.EditArticle, shortcutKeyEdit)
})

</script>

<template>
  <div class="practice-article">
    <div class="content">
      <div class="article-wrapper" v-if="articleIsActive">
        <TypingArticle
            ref="typingArticleRef"
            :article="articleData.article"
            :section-index="articleData.sectionIndex"
            :sentence-index="articleData.sentenceIndex"
            :word-index="articleData.wordIndex"
            :string-index="articleData.stringIndex"
            :active="articleIsActive"
            @wrong="wrong"
            @next-word="(val) => { }"
            @over="next"
            @edit="edit"
        />
      </div>
      <div class="word-wrapper" v-else>
        <TypingWord
            v-model:words="wordData.words"
            :index="wordData.index"
            @wrong="wrong"
            @collect="collect"
            @simple="simple"
        />
      </div>
    </div>
    <div class="panel-wrapper">
      <Panel>
        <template v-slot="{active}">
          <div class="panel-page-item">
            <div class="list-header">
              <div class="left">
                <div class="title">
                  {{ store.chapterName }}
                </div>
                <BaseIcon title="切换词典"
                          @click="emitter.emit(EventKey.openDictModal,'list')"
                          icon="carbon:change-catalog"/>
                <BaseIcon icon="bi:arrow-right"
                          @click="emitter.emit(EventKey.next)"
                          :title="`下一章(快捷键：${settingStore.shortcutKeyMap[ShortcutKey.NextChapter]})`"
                          v-if="store.currentDict.chapterIndex < store.currentDict.articles.length - 1"/>
              </div>
              <div class="right">
                {{ articleData.articles.length }}篇文章
              </div>
            </div>
            <ArticleList
                v-if="articleData.articles.length"
                :is-active="active"
                :static="false"
                :list="articleData.articles"
                :activeIndex="store.currentDict.chapterIndex"
                @click="(val:any) => { store.currentDict.chapterIndex = val.index; getCurrentPractice() }"
            >
              <template v-slot:suffix="{item,index}">
                <BaseIcon
                    v-if="!isArticleCollect(item)"
                    class="collect"
                    @click="toggleArticleCollect(item)"
                    title="收藏" icon="ph:star"/>
                <BaseIcon
                    v-else
                    class="fill"
                    @click="toggleArticleCollect(item)"
                    title="取消收藏" icon="ph:star-fill"/>
                <BaseIcon
                    :title="`编辑(快捷键：${settingStore.shortcutKeyMap[ShortcutKey.EditArticle]})`"
                    icon="tabler:edit"
                    @click="edit(item)"/>
              </template>
            </ArticleList>
          </div>
        </template>
      </Panel>
    </div>
  </div>
  <EditSingleArticleModal v-if="showEditArticle" @close="showEditArticle = false" @save="saveArticle" :article="editArticle"/>
</template>

<style scoped lang="scss">
@import "@/assets/css/style";

.practice-article {
  height: 100%;
  display: flex;
  gap: var(--space);

  .content {
    flex: 1;
    display: flex;
    flex-direction: column;
  }

  .panel-wrapper {
    width: var(--panel-width);
  }
}
</style>