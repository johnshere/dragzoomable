<template>
    <div
        ref="wrap"
        title="按住拖动，滚轮缩放"
        class="draggable-zoomable"
        @mousedown="startDrag"
        @mousemove="drag"
        @mouseup="stopDrag"
        @mouseleave="stopDrag"
        @wheel="zoom"
    >
        <div ref="container" class="draggable-zoomable-container">
            <slot></slot>
        </div>
    </div>
</template>

<script lang="ts" setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'

const wrap = ref<HTMLElement>()
const container = ref<HTMLElement>()
const isDragging = ref(false)
const mouseStartPos = ref({ x: 0, y: 0 })
const position = ref({ x: 0, y: 0 })
const scale = ref(1)

const transform = (x = 0, y = 0, _scale = 1) => {
    if (!container.value) return
    container.value.style.transform = `translate(${x}px, ${y}px) scale(${_scale}, ${_scale})`
}
onMounted(transform)

const isPressed = ref(false)
function startDrag(event: MouseEvent) {
    isPressed.value = true
    if (event.button !== 0) return
    if (!isDragging.value) {
        console.log('start drag')
        isDragging.value = true
        mouseStartPos.value = { x: event.clientX, y: event.clientY }
    }
}
function drag(event: MouseEvent) {
    if (isDragging.value) {
        const x = event.clientX - mouseStartPos.value.x + position.value.x
        const y = event.clientY - mouseStartPos.value.y + position.value.y
        transform(x, y, scale.value)
        return [x, y]
    }
}
function stopDrag(event: MouseEvent) {
    isPressed.value = false
    if (isDragging.value) {
        // 弥补鼠标移动时，鼠标位置与元素位置的偏差
        const [x, y] = drag(event) || []
        position.value.x = x
        position.value.y = y
        isDragging.value = false
    }
}

function mark(x: number, y: number, color: string) {
    if (!wrap.value) return
    const div = document.createElement('div')
    div.style.position = 'absolute'
    div.style.left = `${x}px`
    div.style.top = `${y}px`
    div.style.width = '2px'
    div.style.height = '2px'
    div.style.borderRadius = '50%'
    div.style.pointerEvents = 'none'
    div.style.zIndex = '9999'
    div.style.transform = 'translate(-50%, -50%)'
    div.style.backgroundColor = color
    wrap.value.appendChild(div)
    setTimeout(() => {
        div.remove()
    }, 10000)
}
let isWheelLocked = false
async function zoom(event: WheelEvent) {
    if (isPressed.value) return
    if (isWheelLocked) return
    isWheelLocked = true
    stopDrag(event)
    const newScale = Math.max(0.3, Math.min(4, scale.value + event.deltaY * -0.01))
    if (newScale === scale.value) return (isWheelLocked = false)
    // 鼠标位置相对于wrap是固定的

    const el = container.value!

    const toContainerLeft = event.clientX - el.getBoundingClientRect().left
    const toContainerTop = event.clientY - el.getBoundingClientRect().top
    const wrate = toContainerLeft / el.getBoundingClientRect().width
    const hrate = toContainerTop / el.getBoundingClientRect().height

    scale.value = newScale
    transform(position.value.x, position.value.y, newScale)
    await nextTick()
    const x = event.clientX - wrap.value!.getBoundingClientRect().left - el.getBoundingClientRect().width * wrate
    const y = event.clientY - wrap.value!.getBoundingClientRect().top - el.getBoundingClientRect().height * hrate
    position.value = { x, y }
    transform(x, y, newScale)
    await nextTick()
    isWheelLocked = false
}

onBeforeUnmount(() => {
    isDragging.value = false
})

async function reset(x = 0, y = 0, _scale = 1) {
    position.value = { x, y }
    scale.value = _scale
    transform(position.value.x, position.value.y, _scale)
    await nextTick()
}
async function center() {
    const wrapWidth = wrap.value!.getBoundingClientRect().width
    const wrapHeight = wrap.value!.getBoundingClientRect().height
    const elWidth = container.value!.getBoundingClientRect().width
    const elHeight = container.value!.getBoundingClientRect().height
    position.value = { x: (wrapWidth - elWidth) / 2, y: (wrapHeight - elHeight) / 2 }
    transform(position.value.x, position.value.y, scale.value)
    await nextTick()
}
async function resize() {
    if (!container.value || !wrap.value) return
    const wrapWidth = wrap.value!.getBoundingClientRect().width
    const wrapHeight = wrap.value!.getBoundingClientRect().height
    await reset(-wrapWidth * 2, -wrapHeight * 2, 1)
    const elWidth = container.value!.getBoundingClientRect().width
    const elHeight = container.value!.getBoundingClientRect().height
    let _scale
    if (elWidth > wrapWidth || elHeight > wrapHeight) {
        _scale = Math.min(wrapHeight / elHeight, wrapWidth / elWidth)
    } else if (elWidth < wrapWidth && elHeight < wrapHeight) {
        _scale = Math.max(wrapHeight / elHeight, wrapWidth / elWidth)
    } else {
        _scale = 1
    }
    scale.value = _scale
    transform(0, 0, _scale)
    await nextTick()
    await center()
}
defineExpose({ reset, resize, center })
</script>

<style lang="scss" scoped>
.draggable-zoomable {
    width: 100%;
    height: 100%;
    cursor: grab;
    user-select: none;
    overflow: hidden;
    &-container {
        position: relative;
        transform-origin: left top;
        transition: none;
    }
}
</style>
