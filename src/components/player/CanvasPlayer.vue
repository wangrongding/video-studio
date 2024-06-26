<script setup lang="ts">
import ContextMenu from './ContextMenu.vue'
import movie from '/bird.mp4'
import { fabric } from 'fabric'
import { storeToRefs } from 'pinia'
import { onMounted, ref, toRaw, watch } from 'vue'
import Logo from '~/assets/icons/icon-github.svg'
import { usePlayerStore, type ElementItem } from '~/stores/player'
import emitter, { BusEvent } from '~/utils/eventBus'
import { initFabricControlCustomStyle } from './customFabricControl'
import { registerHotkey } from './operate'

defineProps<{
  msg: string
}>()
const container = ref<HTMLElement | null>(null)
let videoRef: HTMLVideoElement
let canvasRef: HTMLCanvasElement
const playerStore = usePlayerStore()
const { togglePlay, addElement, removeElement, setFocusElements } = playerStore
const { playStatus, currentTime, duration, elementList } = storeToRefs(playerStore)
const menuShow = ref(false)
const contextMenuPosition = ref({ x: 0, y: 0 })
const contextMenu = ref<typeof ContextMenu | null>(null)
let canvas: fabric.Canvas
let ctx: CanvasRenderingContext2D
let clipboard: fabric.Object | null = null
initFabricControlCustomStyle() // 初始化 fabric 控件自定义样式

const menuList = [
  { key: 'flipX', shortkey: '⌘+C', text: '复制', callback: onCopy },
  { key: 'flipX', shortkey: '⌘+V', text: '粘贴', callback: onPaste },
  { key: 'flipX', shortkey: '⌘+X', text: '剪切', callback: onCut },
  { key: 'flipX', shortkey: 'Del', text: '删除', callback: onDelete },
  { key: 'flipY', shortkey: '⌘+]', text: '上移一层', callback: () => setElementLayer('up') },
  { key: 'flipY', shortkey: '⌘+[', text: '下移一层', callback: () => setElementLayer('down') },
  { key: 'flipY', shortkey: '', text: '置于顶层', callback: () => setElementLayer('top') },
  { key: 'flipY', shortkey: '', text: '置于底层', callback: () => setElementLayer('bottom') },
  { key: 'flipX', shortkey: '', text: '旋转90°', callback: rotate },
  { key: 'flipX', shortkey: '', text: '左右翻转', callback: () => flip('x') },
  { key: 'flipY', shortkey: '', text: '垂直翻转', callback: () => flip('y') },
]

emitter.on(BusEvent.ElementCopy, onCopy)
emitter.on(BusEvent.ElementPaste, onPaste)
emitter.on(BusEvent.ElementDelete, onDelete)
emitter.on(BusEvent.ElementAdd, onAdd)
emitter.on(BusEvent.ElementAlign, setElementAlign)
emitter.on(BusEvent.ElementLayer, setElementLayer)
emitter.on(BusEvent.ElementSelect, selectElement)
emitter.on(BusEvent.ElementSelectAll, selectAll)
emitter.on(BusEvent.CanvasFullScreen, toggleCanvasFullScreen)
emitter.on(BusEvent.CanvasExportCurrentFrame, onExportCurrentFrame)
emitter.on(BusEvent.VideoSkip, (time: number) => (videoRef.currentTime += time))

watch(playStatus, (v) => videoRef[v ? 'play' : 'pause']())

// 初始化画布
function initCanvas() {
  canvas = new fabric.Canvas('canvas', {
    fireRightClick: true, // 启用右键，button的数字为 3
    stopContextMenu: true, // 禁止默认右键菜单
    selection: true, // 是否开启选择
    hoverCursor: 'pointer', // 鼠标悬停时的样式
    preserveObjectStacking: true, // 保持对象的堆叠顺序(选中时不会置顶)
    backgroundColor: '#000', // 画布背景色
    selectionBorderColor: 'red', // 选中边框颜色
    selectionLineWidth: 2, // 选中边框宽度
    selectionDashArray: [10], // 选中边框虚线
    selectionFullyContained: true, // 精准选择(不包含空白区域，真正选中图形才算选中 )
  })
  canvasRef = canvas.getElement()
  ctx = canvas.getContext()
  resizePlayer()
  initCanvasEvent()
  drawElements()
}

// 初始化 Canvas 事件
function initCanvasEvent() {
  // 画布鼠标按下事件
  canvas.on('mouse:down', canvasOnMouseDown)
  // 选中元素时
  canvas.on('selection:created', onElementSelected)
  canvas.on('selection:updated', onElementSelected)
  // 取消选中元素时
  canvas.on('selection:cleared', onElementDeselected)
  // 目标移动中
  canvas.on('object:moving', onElementMoving)
  // 目标修改后
  canvas.on('object:modified', onElementModified)
}

// 绘制元素
async function drawElements() {
  await drawVideo(movie)
  addSVG(Logo)
  addText('侧边栏，添加素材...', { left: 100, top: 100, fill: 'red' })
  addText('右键点击元素，进行操作', { left: 400, top: 150 })
  addText('双击修改文字', { left: 400, top: 500, angle: 45, fill: 'blue', fontSize: 30 })
  canvas.renderAll()
}

// 删除元素
function onDelete(obj: ElementItem | void) {
  const targetElement: ElementItem | null = obj ? toRaw(obj) : canvas.getActiveObject()
  if (!targetElement) return
  const { elementId } = targetElement
  const activeObject = targetElement instanceof fabric.Object ? targetElement : null
  if (!activeObject) return
  canvas.remove(activeObject)
  canvas.requestRenderAll()
  if (menuShow.value) menuShow.value = false
  elementId && removeElement(elementId)
}

// 添加元素
// TODO 后面需要支持传入添加元素的配置
function onAdd({ type, value }: { type: string; value: string }) {
  switch (type) {
    case 'video':
      drawVideo(value)
      break
    case 'image':
      addImage(value)
      break
    case 'svg':
      addSVG(value)
      break
    case 'text':
      addText(value)
      break
    case 'rect':
      addRect()
      break
    default:
      break
  }
}

// 绘制 svg
function addSVG(url: string) {
  fabric.loadSVGFromURL(url, (objects, options) => {
    const svgElement = fabric.util.groupSVGElements(objects, options)
    const scale = 5
    svgElement.set({
      scaleX: scale,
      scaleY: scale,
      left: canvas.width! / 2 - (svgElement.width! * scale) / 2,
      top: canvas.height! / 2 - (svgElement.height! * scale) / 2,
      angle: 0,
    })
    canvas.add(svgElement)
    addElement('svg', svgElement)
  })
}

// 绘制矩形
function addRect() {
  const rectElement = new fabric.Rect({
    left: canvas.width! / 2,
    top: canvas.height! / 2,
    fill: 'red',
    width: 100,
    height: 100,
  })
  canvas.add(rectElement)
  addElement('svg', rectElement)
}

// 添加文字
function addText(val: string, options?: fabric.ITextboxOptions) {
  const textElement = new fabric.Textbox(val, {
    left: canvas.width! / 2,
    top: canvas.height! / 2,
    fill: 'white',
    fontSize: 20,
    ...options,
    // fontStyle: 'italic', // 整体斜体
    // splitByGrapheme: true // 自动换行
  })
  // textElement.splitByGrapheme = true // 自动换行
  canvas.add(textElement)
  addElement('text', textElement)
}

// 绘制视频
// TODO 需要实现通过 webcodecs 进行视频解码后绘制
async function drawVideo(url: string) {
  videoRef = document.createElement('video')
  videoRef.src = url
  videoRef.loop = true
  videoRef.preload = 'auto' // 不加会导致未播放时元素黑屏
  videoRef.crossOrigin = 'anonymous' // 跨域的视频会污染画布，导致无法导出

  await new Promise<void>((resolve) => {
    videoRef.addEventListener('loadedmetadata', () => {
      // 视频源宽高
      videoRef.width = videoRef.videoWidth
      videoRef.height = videoRef.videoHeight
      resolve()
    })
  })

  const videoWidth = videoRef.width
  const videoHeight = videoRef.height
  const canvasWidth = canvas.width!
  const canvasHeight = canvas.height!
  // 适配画布大小（如果 宽>高，以宽为准,反之以高为准）
  const scale = Math.min(canvasWidth / videoWidth, canvasHeight / videoHeight)
  const videoElement = new fabric.Image(videoRef, {
    scaleX: scale,
    scaleY: scale,
    // 水平垂直居中
    left: canvasWidth / 2 - (videoWidth * scale) / 2,
    top: canvasHeight / 2 - (videoHeight * scale) / 2,
    angle: 0,
    originX: 'left',
    originY: 'top',
  })
  canvas.add(videoElement)
  continuouslyRepaint()
  duration.value = videoRef.duration
  addElement('video', videoElement)
  canvas.setActiveObject(videoElement)

  videoRef.addEventListener('timeupdate', () => {
    currentTime.value = videoRef.currentTime
  })
}

// 添加图片
function addImage(url: string) {
  const activeObject = canvas.getActiveObject()
  fabric.Image.fromURL(url, (imgElement) => {
    const scale = 0.5
    imgElement.set({
      scaleX: scale,
      scaleY: scale,
      // 存在选中元素时，以选中元素为基准添加图片，否则以画布中心为基准
      left: activeObject
        ? activeObject.left! + 30
        : canvas.width! / 2 - (imgElement.width! * scale) / 2,
      top: activeObject
        ? activeObject.top! + 30
        : canvas.height! / 2 - (imgElement.height! * scale) / 2,
      angle: 0,
    })
    canvas.add(imgElement)
    addElement('image', imgElement)
    canvas.setActiveObject(imgElement)
  })
}

// 持续重绘
function continuouslyRepaint() {
  canvas.renderAll()
  fabric.util.requestAnimFrame(continuouslyRepaint)
}

// 设置选中文本的斜体
function italic() {
  let activeTxt = canvas.getActiveObject() as fabric.Textbox
  if (!activeTxt) return

  if (activeTxt.isEditing) {
    // 编辑状态，将被选中的文本进行斜体或恢复默认的操作
    const state = activeTxt.getSelectionStyles().find((item) => item.fontStyle !== 'italic')

    if (!state || (JSON.stringify(state) === '{}' && activeTxt['fontStyle'] === 'italic')) {
      activeTxt.setSelectionStyles({ fontStyle: 'normal' })
    } else {
      activeTxt.setSelectionStyles({ fontStyle: 'italic' })
    }
  } else {
    // 选择状态，全文进行斜体或恢复默认操作
    if (activeTxt['fontStyle'] === 'italic') {
      activeTxt.fontStyle = 'normal'
      let s = activeTxt.styles
      for (let i in s) {
        for (let j in s[i]) {
          s[i][j].fontStyle = 'normal'
        }
      }
    } else {
      activeTxt.fontStyle = 'italic'
      let s = activeTxt.styles
      for (let i in s) {
        for (let j in s[i]) {
          s[i][j].fontStyle = 'italic'
        }
      }
    }
  }
  canvas.renderAll()
}

// 设置选中文本的删除线
function linethrough() {
  let activeTxt = canvas.getActiveObject() as fabric.Textbox // 获取当前选中的文字
  // 如果当前没选中文字，那什么都不操作
  if (!activeTxt) return
  // 判断当前是否进入编辑状态
  if (activeTxt.isEditing) {
    // 编辑状态
    const state = activeTxt.getSelectionStyles().find((item) => item.linethrough !== true)
    // 如果当前
    if (!state || (JSON.stringify(state) === '{}' && activeTxt['linethrough'] === true)) {
      // 如果当前已经设置了中划线，那就把全局中划线取消
      activeTxt.setSelectionStyles({ linethrough: false })
    } else {
      // 如果当前没设置中划线，那就添加上中划线
      activeTxt.setSelectionStyles({ linethrough: true })
    }
  } else {
    // 选择状态
    if (activeTxt['linethrough'] === true) {
      activeTxt.linethrough = false
      activeTxt.dirty = true
      let s = activeTxt.styles
      for (let i in s) {
        for (let j in s[i]) {
          s[i][j].linethrough = false
        }
      }
    } else {
      activeTxt.linethrough = true
      activeTxt.dirty = true
      let s = activeTxt.styles
      for (let i in s) {
        for (let j in s[i]) {
          s[i][j].linethrough = true
        }
      }
    }
  }
  canvas.renderAll()
}

// 设置选中文本的字号
function changeSize(value: number) {
  let activeTxt = canvas.getActiveObject() as fabric.Textbox
  if (!activeTxt) return
  if (activeTxt.isEditing) {
    // 编辑状态
    activeTxt.setSelectionStyles({ fontSize: value })
  } else {
    activeTxt.fontSize = value
    let s = activeTxt.styles
    // 遍历行和列
    for (let i in s) {
      for (let j in s[i]) {
        s[i][j].fontSize = value // 针对每个字设置字号
      }
    }
    activeTxt.dirty = true
  }
  canvas.renderAll()
}

// 水平居中选中元素
function setElementAlign(align: 'left' | 'center' | 'right' | 'top' | 'middle' | 'bottom') {
  let activeObj = canvas.getActiveObject()
  if (!activeObj) return
  switch (align) {
    case 'center':
      activeObj.centerH()
      break
    case 'middle':
      activeObj.centerV()
      break
    case 'left':
      activeObj.set('left', 0)
      break
    case 'right':
      activeObj.set('left', canvas.width! - activeObj.width! * activeObj.scaleX!)
      break
    case 'top':
      activeObj.set('top', 0)
      break
    case 'bottom':
      activeObj.set('top', canvas.height! - activeObj.height! * activeObj.scaleY!)
      break
    default:
      break
  }
  // canvas.viewportCenterObject(activeObj) // 画布水平垂直居中
  canvas.renderAll()
}

// TODO 需完善窗口大小变化时，画布保持当前宽高比进行缩放
function resizePlayer() {
  const width = container.value!.clientWidth
  const height = container.value!.clientHeight
  canvas.setDimensions({
    width,
    height,
  })
}

// 设置元素层级
function setElementLayer(type: 'up' | 'down' | 'top' | 'bottom') {
  const activeObject = canvas.getActiveObject()
  if (!activeObject) return
  switch (type) {
    case 'up':
      // 上移一层
      canvas.bringForward(activeObject)
      break
    case 'down':
      // 下移一层
      canvas.sendBackwards(activeObject)
      break
    case 'top':
      // 置于顶层
      canvas.bringToFront(activeObject)
      break
    case 'bottom':
      // 置于底层
      canvas.sendToBack(activeObject)
      break
    default:
      break
  }
  menuShow.value = false
}

// 切换画布全屏
function toggleCanvasFullScreen(fullscreen?: boolean) {
  if (fullscreen === undefined) {
    if (document.fullscreenElement) {
      document.exitFullscreen()
    } else {
      canvasRef.requestFullscreen()
    }
  } else if (!document.fullscreenElement && fullscreen) {
    canvasRef.requestFullscreen()
  } else if (document.fullscreenElement && !fullscreen) {
    document.exitFullscreen()
  }
}

// 导出图片
function onExportCurrentFrame() {
  const dataURL = canvas.toDataURL({
    width: canvas.width,
    height: canvas.height,
    left: 0,
    top: 0,
    format: 'png',
  })
  const link = document.createElement('a')
  link.download = 'canvas.png'
  link.href = dataURL
  document.body.appendChild(link)
  link.click()
  document.body.removeChild(link)
}

// 复制元素
function onCopy() {
  const activeObject = canvas.getActiveObject()
  if (!activeObject) return
  activeObject.clone((cloned: fabric.Object) => {
    clipboard = cloned
  })
  menuShow.value = false
}

// 剪切元素
function onCut() {
  const activeObject = canvas.getActiveObject()
  if (!activeObject) return
  onCopy()
  canvas.remove(activeObject)
}

// TODO 视频元素为什么无法复制
// 粘贴元素
function onPaste() {
  if (clipboard === null) return
  // clone again, so you can do multiple copies.
  clipboard!.clone((clonedObj: any) => {
    canvas.discardActiveObject()
    clonedObj.set({
      left: clonedObj.left + 30,
      top: clonedObj.top + 30,
      evented: true,
    })
    if (clonedObj.type === 'activeSelection') {
      // active selection needs a reference to the canvas.
      clonedObj.canvas = canvas
      clonedObj.forEachObject((obj: any) => {
        canvas.add(obj)
      })
      // this should solve the unselectability
      clonedObj.setCoords()
    } else {
      canvas.add(clonedObj)
    }
    clipboard!.top! += 30
    clipboard!.left! += 30
    canvas.setActiveObject(clonedObj)
    canvas.requestRenderAll()
    menuShow.value = false
  })
}

// 翻转元素
function flip(flipType: 'x' | 'y') {
  const activeObject = canvas.getActiveObject()
  if (!activeObject) return
  const type = flipType === 'x' ? 'flipX' : 'flipY'
  activeObject.set({
    [type]: !activeObject.get(type),
  })
  canvas?.renderAll()
}

// 以元素的中心旋转 90°
// BUG 未拖动元素时，旋转会导致元素偏移
function rotate() {
  const activeObject = canvas.getActiveObject()
  if (!activeObject) return
  activeObject.set({
    angle: activeObject.angle! + 90,
    originX: 'center',
    originY: 'center',
  })
  canvas?.renderAll()
}

// 初始化右键菜单
function initContextMenu(e: fabric.IEvent<MouseEvent>) {
  if (!e.target) return
  canvas.setActiveObject(e.target)
  contextMenuPosition.value.x = e.pointer!.x
  contextMenuPosition.value.y = e.pointer!.y
  menuShow.value = true
  const menu = contextMenu.value!.$el
  const contextMenuWidth = menu.offsetWidth || 200
  const contextMenuHeight = menu.offsetHeight || 400

  if (canvas.width! - e.pointer!.x <= contextMenuWidth) {
    contextMenuPosition.value.x = canvas.width! - contextMenuWidth
  } else {
    contextMenuPosition.value.x = e.pointer!.x
  }
  if (canvas.height! - e.pointer!.y <= contextMenuHeight) {
    contextMenuPosition.value.y = canvas.height! - contextMenuHeight
  } else {
    contextMenuPosition.value.y = e.pointer!.y
  }
}

/**
 * 需要在初始化时，设置 fireMiddleClick: true
 * 左键：button 的值为 1
 * 中键（也就是点击滚轮），button 的值为 2
 * 右键：button 的值为 3
 */
function canvasOnMouseDown(e: fabric.IEvent<MouseEvent>) {
  switch (e.button) {
    case 1:
      menuShow.value = false
      break
    case 3:
      initContextMenu(e)
      break

    default:
      break
  }
}

// 选中元素
function selectElement(obj: ElementItem) {
  const targetElement: ElementItem = canvas
    .getObjects()
    .find((item: ElementItem) => item.elementId === obj.elementId)!
  if (!targetElement) return
  canvas.setActiveObject(targetElement)
  canvas.requestRenderAll()
}

// 全选
function selectAll() {
  canvas.discardActiveObject()
  const sel = new fabric.ActiveSelection(canvas.getObjects(), {
    canvas: canvas,
  })
  canvas.setActiveObject(sel)
  canvas.requestRenderAll()
}

// 元素选中时
function onElementSelected(e: fabric.IEvent<MouseEvent>) {
  setFocusElements(e.selected, e.deselected)
}

// 元素取消选中时
function onElementDeselected(e: fabric.IEvent<MouseEvent>) {
  setFocusElements(undefined, e.deselected)
}

// 元素移动时
function onElementMoving(e: fabric.IEvent<MouseEvent>) {
  if (!e.target) return
  e.target.opacity = 0.5
}

// 元素修改后
function onElementModified(e: fabric.IEvent<MouseEvent>) {
  if (!e.target) return
  e.target.opacity = 1
}
onMounted((): void => {
  // 初始化画布
  initCanvas()
  // 注册快捷键
  registerHotkey()

  // oncopy事件禁用复制;
  document.oncopy = onCopy
  // onpaste事件禁用粘贴
  document.onpaste = onPaste
  // oncut事件禁用剪切
  document.oncut = onCut
})
</script>
<template>
  <div class="relative h-full w-full overflow-hidden" ref="container">
    <canvas id="canvas"></canvas>
    <ContextMenu
      v-show="menuShow"
      ref="contextMenu"
      :menuList="menuList"
      :style="{
        top: Math.max(0, contextMenuPosition.y) + 'px',
        left: contextMenuPosition.x + 'px',
        maxHeight: '100%',
        overflow: 'auto',
      }"
      class="context-menu"
    />
  </div>
</template>
<style lang="scss" scoped></style>
