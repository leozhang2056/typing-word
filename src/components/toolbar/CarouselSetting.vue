<script setup lang="ts">
import MiniDialog from "@/components/dialog/MiniDialog.vue";
import {Icon} from "@iconify/vue";
import IconWrapper from "@/components/IconWrapper.vue";
import Tooltip from "@/components/Tooltip.vue";
import {ref, onMounted} from "vue";
import {useWindowClick} from "@/hooks/event.ts";
import {emitter} from "@/utils/eventBus.ts";

let show = ref(false)

useWindowClick(() => show.value = false)

function toggle2() {
  show.value = !show.value
}

// 轮播设置的本地状态
const carouselEnabled = ref(false)
const carouselInterval = ref(3)

// 与 practice-word/index.vue 通信
onMounted(() => {
  // 确保 practice-word/index.vue 已挂载后再发送事件
  console.log('CarouselSetting mounted, sending getCarouselSetting event')
  emitter.emit('getCarouselSetting')
  emitter.on('setCarouselSetting', (payload: {enabled: boolean, interval: number}) => {
    console.log('setCarouselSetting received in CarouselSetting:', payload)
    carouselEnabled.value = payload.enabled
    carouselInterval.value = payload.interval
  })
})

function updateSetting() {
  console.log('updateSetting called - enabled:', carouselEnabled.value, 'interval:', carouselInterval.value)
  emitter.emit('updateCarouselSetting', {
    enabled: carouselEnabled.value,
    interval: carouselInterval.value
  })
}
</script>

<template>
  <div class="setting" @click.stop="null">
    <Tooltip title="轮播设置">
      <IconWrapper>
        <Icon icon="mdi:play-speed" @click="toggle2"/>
      </IconWrapper>
    </Tooltip>
    <MiniDialog
        width="220rem"
        v-model="show">
      <div class="mini-row-title">轮播设置</div>
      <div class="mini-row">
        <span>开启轮播</span>
        <el-switch v-model="carouselEnabled" @change="updateSetting"/>
      </div>
      <div class="mini-row">
        <span>间隔时长(秒)</span>
        <el-input-number v-model="carouselInterval" :min="1" :max="30" @change="updateSetting"/>
      </div>
    </MiniDialog>
  </div>
</template>

<style scoped lang="scss">
@import "@/assets/css/style";

.setting {
  position: relative;
}
</style> 