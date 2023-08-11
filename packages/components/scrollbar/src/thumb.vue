<template>
  <transition :name="ns.b('fade')">
    <div
      v-show="always || visible"
      ref="instance"
      :class="[ns.e('bar'), ns.is(bar.key)]"
      @mousedown="clickTrackHandler"
    >
      <div
        ref="thumb"
        :class="ns.e('thumb')"
        :style="thumbStyle"
        @mousedown="clickThumbHandler"
      />
    </div>
  </transition>
</template>

<script lang="ts" setup>
import { computed, inject, onBeforeUnmount, ref, toRef } from 'vue'
import { useEventListener } from '@vueuse/core'
import { isClient, throwError } from '@element-plus/utils'
import { useNamespace } from '@element-plus/hooks'
import { scrollbarContextKey } from './constants'
import { BAR_MAP, renderThumbStyle } from './util'
import { thumbProps } from './thumb'

const COMPONENT_NAME = 'Thumb'
const props = defineProps(thumbProps)


const scrollbar = inject(scrollbarContextKey)
const ns = useNamespace('scrollbar')

if (!scrollbar) throwError(COMPONENT_NAME, 'can not inject scrollbar context')

// bar实例
const instance = ref<HTMLDivElement>()
// thumb实例
const thumb = ref<HTMLDivElement>()
// thumb状态
const thumbState = ref<Partial<Record<'X' | 'Y', number>>>({})
const visible = ref(false)
// 光变按下
let cursorDown = false
// 光标离开
let cursorLeave = false
// 最初当选中开始
let originalOnSelectStart:
  | ((this: GlobalEventHandlers, ev: Event) => any)
  | null = isClient ? document.onselectstart : null

const bar = computed(() => BAR_MAP[props.vertical ? 'vertical' : 'horizontal'])

const thumbStyle = computed(() =>{
  return renderThumbStyle({
    size: props.size,
    move: props.move,
    bar: bar.value,
  })
})

// 偏移量比例
const offsetRatio = computed(
  () =>
    // offsetRatioX = original width of thumb / current width of thumb / ratioX
    // offsetRatioY = original height of thumb / current height of thumb / ratioY
    // instance height = wrap height - GAP
    instance.value![bar.value.offset] ** 2 /
    scrollbar.wrapElement![bar.value.scrollSize] /
    props.ratio /
    thumb.value![bar.value.offset]
)

const clickThumbHandler = (e: MouseEvent) => {
  // prevent click event of middle and right button
  // 阻止传播
  e.stopPropagation()
  // ctrl 中间、右键的点击事件
  if (e.ctrlKey || [1, 2].includes(e.button)) return
  // 取消所有的选择
  window.getSelection()?.removeAllRanges()
  // 开始抓取
  startDrag(e)
  // 当前点击元素
  const el = e.currentTarget as HTMLDivElement
  if (!el) return
  // 初始化thumbState// 鼠标位于thumb上的位置
  thumbState.value[bar.value.axis] =
    // 当前元素高度减去
    el[bar.value.offset] -
    // 应用客户端直角坐标系位置 - 距离视口距离
    (e[bar.value.client] - el.getBoundingClientRect()[bar.value.direction])
}

const clickTrackHandler = (e: MouseEvent) => {
  if (!thumb.value || !instance.value || !scrollbar.wrapElement) return

  const offset = Math.abs(
    (e.target as HTMLElement).getBoundingClientRect()[bar.value.direction] -
      e[bar.value.client]
  )
  const thumbHalf = thumb.value[bar.value.offset] / 2
  const thumbPositionPercentage =
    ((offset - thumbHalf) * 100 * offsetRatio.value) /
    instance.value[bar.value.offset]

  scrollbar.wrapElement[bar.value.scroll] =
    (thumbPositionPercentage * scrollbar.wrapElement[bar.value.scrollSize]) /
    100
}
// 开始抓取
const startDrag = (e: MouseEvent) => {
  // 停止立即传播
  e.stopImmediatePropagation()
  // 光标按下
  cursorDown = true
  // 监听鼠标移动事件
  document.addEventListener('mousemove', mouseMoveDocumentHandler)
  // 监听鼠标松开事件
  document.addEventListener('mouseup', mouseUpDocumentHandler)
  // 事件在用户开始一个新的选择时候触发
  originalOnSelectStart = document.onselectstart
  // 重置selectstart
  document.onselectstart = () => false
}

// 鼠标移动记录程序（主动）
const mouseMoveDocumentHandler = (e: MouseEvent) => {
  // bar以及thumb实例是否被创建
  if (!instance.value || !thumb.value) return
  // 如果光标没有被按下
  if (cursorDown === false) return
  // 点击的位置（上）
  const prevPage = thumbState.value[bar.value.axis]
  if (!prevPage) return
  // 偏移量
  const offset =
    // bar实例获取大小及相对于视口的指定方位的距离减去（57）
    (instance.value.getBoundingClientRect()[bar.value.direction] -
      // 事件发生时客户端区域指定方向的坐标
      e[bar.value.client]) *
    -1
  // thumb点击的位置（下）
  const thumbClickPosition = thumb.value[bar.value.offset] - prevPage
  // thumb位置比例
  const thumbPositionPercentage =
    // （偏移量减去 - 之前thumb点击的位置） * 100 * 偏移比例  /
    ((offset - thumbClickPosition) * 100 * offsetRatio.value) /
    // bar实例的大小
    instance.value[bar.value.offset]
  // 改变 包裹的元素内容 scrollTop
  scrollbar.wrapElement[bar.value.scroll] =
    (thumbPositionPercentage * scrollbar.wrapElement[bar.value.scrollSize]) /
    100
}

const mouseUpDocumentHandler = () => {
  cursorDown = false
  thumbState.value[bar.value.axis] = 0
  document.removeEventListener('mousemove', mouseMoveDocumentHandler)
  document.removeEventListener('mouseup', mouseUpDocumentHandler)
  restoreOnselectstart()
  if (cursorLeave) visible.value = false
}

const mouseMoveScrollbarHandler = () => {
  cursorLeave = false
  visible.value = !!props.size
}

const mouseLeaveScrollbarHandler = () => {
  cursorLeave = true
  visible.value = cursorDown
}

onBeforeUnmount(() => {
  restoreOnselectstart()
  document.removeEventListener('mouseup', mouseUpDocumentHandler)
})

const restoreOnselectstart = () => {
  if (document.onselectstart !== originalOnSelectStart)
    document.onselectstart = originalOnSelectStart
}

useEventListener(
  toRef(scrollbar, 'scrollbarElement'),
  'mousemove',
  mouseMoveScrollbarHandler
)
useEventListener(
  toRef(scrollbar, 'scrollbarElement'),
  'mouseleave',
  mouseLeaveScrollbarHandler
)
</script>
