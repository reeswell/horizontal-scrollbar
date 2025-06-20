<script setup lang="ts">
import { throttle } from 'lodash-es'
import { ref, computed, nextTick, onMounted, onUnmounted, readonly } from 'vue'
import type {
  ElementSelector,
  HorizontalScrollbarProps,
  HorizontalScrollbarEmits,
  HorizontalScrollbarExpose,
  ScrollInfo,
} from '../types'

const props = withDefaults(defineProps<HorizontalScrollbarProps>(), {
  autoShow: true,
  minScrollDistance: 50,
  height: 16,
  enableKeyboard: true,
  scrollStep: 50,
  minThumbWidth: 30,
  teleportTo: 'body',
  throttleDelay: 16,
  zIndex: 9999,
  disabled: false,
  ariaLabel: 'Horizontal scrollbar',
})

const emit = defineEmits<HorizontalScrollbarEmits>()

const scrollLeft = ref(0)
const maxScroll = ref(0)
const scrollPercent = ref(0)
const showScrollbar = ref(false)
const isVisible = ref(false)

const customScrollbarRef = ref<HTMLElement>()
const customScrollbarThumbRef = ref<HTMLElement>()

const isDragging = ref(false)
const dragStartX = ref(0)
const dragStartScrollLeft = ref(0)

const targetElement = ref<HTMLElement | null>(null)
const contentElement = ref<HTMLElement | null>(null)
let resizeObserver: ResizeObserver | null = null

const thumbStyle = computed(() => {
  const targetWidth = targetElement?.value?.clientWidth || 0
  const contentWidth = contentElement?.value?.scrollWidth || 1
  const scrollbarWidth = customScrollbarRef.value?.offsetWidth || 0

  const thumbWidth = Math.max(
    props.minThumbWidth,
    (targetWidth / contentWidth) * scrollbarWidth
  )

  const thumbPosition = maxScroll.value > 0
    ? (scrollLeft.value / maxScroll.value) * (scrollbarWidth - thumbWidth)
    : 0

  return {
    width: `${thumbWidth}px`,
    left: `${thumbPosition}px`,
  }
})

const throttledHandleScroll = throttle(() => {
  if (!isDragging.value) {
    updateScrollInfo()
  }
}, props.throttleDelay)

function getElement(selector: ElementSelector): HTMLElement | null {
  try {
    if (typeof selector === 'string') {
      const element = document.querySelector<HTMLElement>(selector)
      if (element) {
        return element
      } else {
        console.warn(`[VueHorizontalScrollbar] Element not found: ${selector}`)
        emit('error', new Error(`Element not found: ${selector}`))
      }
    } else if (typeof selector === 'function') {
      return selector() || null
    }
    return null
  } catch (error) {
    console.error('[VueHorizontalScrollbar] Error getting element:', error)
    emit('error', error as Error)
    return null
  }
}

async function initializeTarget(): Promise<void> {
  try {
    await nextTick()

    targetElement.value = getElement(props.targetSelector)
    contentElement.value = getElement(props.contentSelector)

    if (!targetElement.value || !contentElement.value) {
      throw new Error('Target or content element not found')
    }

    setupScrollListener()
    setupResizeObserver()
    updateScrollInfo()
    isVisible.value = true

    emit('ready')
  } catch (error) {
    console.error('[VueHorizontalScrollbar] Initialization error:', error)
    emit('error', error as Error)
  }
}

function setupResizeObserver(): void {
  if (!targetElement.value || !contentElement.value) return

  resizeObserver = new ResizeObserver(() => {
    updateScrollInfo()
  })

  resizeObserver.observe(targetElement.value)
  resizeObserver.observe(contentElement.value)
}

function setupScrollListener(): void {
  if (!targetElement.value) return
  targetElement.value.addEventListener('scroll', throttledHandleScroll, { passive: true })
}

function updateScrollInfo(): void {
  if (!targetElement.value || !contentElement.value) return

  const newScrollLeft = targetElement.value.scrollLeft
  const newMaxScroll = Math.max(0, contentElement.value.scrollWidth - targetElement.value.clientWidth)
  const newScrollPercent = newMaxScroll > 0 ? (newScrollLeft / newMaxScroll) * 100 : 0

  scrollLeft.value = newScrollLeft
  maxScroll.value = newMaxScroll
  scrollPercent.value = newScrollPercent

  if (props.autoShow) {
    showScrollbar.value = maxScroll.value > props.minScrollDistance
  } else {
    showScrollbar.value = true
  }

  const scrollInfo: ScrollInfo = {
    scrollLeft: newScrollLeft,
    maxScroll: newMaxScroll,
    scrollPercent: newScrollPercent,
  }
  emit('scroll', scrollInfo)
}

function scrollToPosition(position: number): void {
  if (!targetElement.value) return

  const clampedPosition = Math.max(0, Math.min(maxScroll.value, position))
  targetElement.value.scrollLeft = clampedPosition
}

function scrollToEnd(): void {
  scrollToPosition(maxScroll.value)
}

function handleScrollbarClick(e: MouseEvent): void {
  if (!customScrollbarRef.value || !customScrollbarThumbRef.value || props.disabled) return
  if (e.target === customScrollbarThumbRef.value) return

  const rect = customScrollbarRef.value.getBoundingClientRect()
  const clickX = e.clientX - rect.left
  const scrollbarWidth = customScrollbarRef.value.offsetWidth
  const thumbWidth = Number.parseFloat(thumbStyle.value.width)

  const targetScrollLeft = ((clickX - thumbWidth / 2) / (scrollbarWidth - thumbWidth)) * maxScroll.value
  scrollToPosition(targetScrollLeft)

  emit('click', e)
}

function handleThumbMouseDown(e: MouseEvent): void {
  if (props.disabled) return

  startDragging(e.clientX)

  document.addEventListener('mousemove', handleMouseMove, { passive: true })
  document.addEventListener('mouseup', handleMouseUp, { passive: true })

  e.preventDefault()
  emit('dragStart', e)
}

function handleThumbTouchStart(e: TouchEvent): void {
  if (props.disabled) return

  startDragging(e.touches[0].clientX)

  document.addEventListener('touchmove', handleTouchMove, { passive: false })
  document.addEventListener('touchend', handleTouchEnd, { passive: true })

  e.preventDefault()
  emit('dragStart', e)
}

function startDragging(clientX: number): void {
  isDragging.value = true
  dragStartX.value = clientX
  dragStartScrollLeft.value = scrollLeft.value
}

function handleMouseMove(e: MouseEvent): void {
  if (!isDragging.value) return
  updateDragPosition(e.clientX)
}

function handleTouchMove(e: TouchEvent): void {
  if (!isDragging.value) return
  updateDragPosition(e.touches[0].clientX)
  e.preventDefault()
}

function updateDragPosition(clientX: number): void {
  if (!customScrollbarRef.value) return

  const deltaX = clientX - dragStartX.value
  const scrollbarWidth = customScrollbarRef.value.offsetWidth
  const thumbWidth = Number.parseFloat(thumbStyle.value.width)

  const scrollRatio = deltaX / (scrollbarWidth - thumbWidth)
  const newScrollLeft = dragStartScrollLeft.value + (scrollRatio * maxScroll.value)

  scrollToPosition(newScrollLeft)
  updateScrollInfo()
}

function handleMouseUp(): void {
  stopDragging()
  document.removeEventListener('mousemove', handleMouseMove)
  document.removeEventListener('mouseup', handleMouseUp)
  emit('dragEnd', new MouseEvent('mouseup'))
}

function handleTouchEnd(): void {
  stopDragging()
  document.removeEventListener('touchmove', handleTouchMove)
  document.removeEventListener('touchend', handleTouchEnd)
  emit('dragEnd', new TouchEvent('touchend'))
}

function stopDragging(): void {
  isDragging.value = false
}

function handleKeyDown(e: KeyboardEvent): void {
  if (!props.enableKeyboard || !targetElement.value || props.disabled) return

  let handled = false

  switch (e.key) {
    case 'ArrowLeft':
      scrollToPosition(scrollLeft.value - props.scrollStep)
      handled = true
      break
    case 'ArrowRight':
      scrollToPosition(scrollLeft.value + props.scrollStep)
      handled = true
      break
    case 'Home':
      scrollToPosition(0)
      handled = true
      break
    case 'End':
      scrollToEnd()
      handled = true
      break
  }

  if (handled) {
    e.preventDefault()
    emit('keydown', e)
  }
}

function cleanup(): void {
  if (targetElement.value) {
    targetElement.value.removeEventListener('scroll', throttledHandleScroll)
  }

  if (resizeObserver) {
    resizeObserver.disconnect()
  }

  document.removeEventListener('keydown', handleKeyDown)
  document.removeEventListener('mousemove', handleMouseMove)
  document.removeEventListener('mouseup', handleMouseUp)
  document.removeEventListener('touchmove', handleTouchMove)
  document.removeEventListener('touchend', handleTouchEnd)
}

onMounted(() => {
  initializeTarget()

  if (props.enableKeyboard) {
    document.addEventListener('keydown', handleKeyDown)
  }
})

onUnmounted(() => {
  cleanup()
})

const exposedAPI: HorizontalScrollbarExpose = {
  scrollToPosition,
  scrollToEnd,
  updateScrollInfo,
  scrollLeft: readonly(scrollLeft),
  maxScroll: readonly(maxScroll),
  scrollPercent: readonly(scrollPercent),
}

defineExpose(exposedAPI)
</script>

<template>
  <Teleport :to="teleportTo">
    <Transition name="vue-horizontal-scrollbar-fade">
      <div v-if="isVisible" class="vue-horizontal-scrollbar-container" :class="{
        'show': showScrollbar,
        'disabled': props.disabled
      }" :style="{
        height: `${props.height}px`,
        zIndex: props.zIndex
      }" role="scrollbar" :aria-valuenow="scrollPercent" aria-valuemin="0" aria-valuemax="100"
        :aria-label="props.ariaLabel">
        <div ref="customScrollbarRef" class="vue-horizontal-scrollbar" @click="handleScrollbarClick">
          <div ref="customScrollbarThumbRef" class="vue-horizontal-scrollbar-thumb" :style="thumbStyle" role="slider"
            tabindex="0" :aria-valuenow="scrollPercent" aria-valuemin="0" aria-valuemax="100"
            @mousedown="handleThumbMouseDown" @touchstart="handleThumbTouchStart" @keydown="handleKeyDown" />
        </div>
      </div>
    </Transition>
  </Teleport>
</template>
