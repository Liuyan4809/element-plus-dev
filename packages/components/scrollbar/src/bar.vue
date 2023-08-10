<template>
  <thumb :move="moveX" :ratio="ratioX" :size="width" :always="always" />
  <thumb
    :move="moveY"
    :ratio="ratioY"
    :size="height"
    vertical
    :always="always"
  />
</template>
<script lang="ts" setup>
import { ref } from 'vue'
import { GAP } from './util'
import Thumb from './thumb.vue'
import { barProps } from './bar'

const props = defineProps(barProps)

const moveX = ref(0)
const moveY = ref(0)

// 被动滚动
const handleScroll = (wrap: HTMLDivElement) => {
  // 滚动容器对象
  if (wrap) {
    // 去除滚动条大小的区域
    const offsetHeight = wrap.offsetHeight - GAP
    const offsetWidth = wrap.offsetWidth - GAP
    // 移动距离 = （（据顶部距离*100）/容器高度）/比率
    moveY.value = ((wrap.scrollTop * 100) / offsetHeight) * props.ratioY
    moveX.value = ((wrap.scrollLeft * 100) / offsetWidth) * props.ratioX
  }
}

defineExpose({
  handleScroll,
})
</script>
