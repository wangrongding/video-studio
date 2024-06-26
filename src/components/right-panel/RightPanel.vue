<script setup lang="ts">
import {
  IconDelete,
  IconAlignLeft,
  IconAlignCenter,
  IconAlignRight,
  IconAlignTop,
  IconAlignMiddle,
  IconAlignBottom,
  IconLayerTop,
  IconLayerBottom,
  IconLayerUp,
  IconLayerDown,
} from '~/assets/icons/index'
import { computed } from 'vue'
import { usePlayerStoreWithRefs, type ElementItem } from '~/stores/player'
import emitter, { BusEvent } from '~/utils/eventBus'

const { elementList, focusElements } = usePlayerStoreWithRefs()
const focusElementIds = computed(() => {
  return focusElements.value.map((item) => item.elementId)
})

function deleteElement(item: ElementItem) {
  emitter.emit(BusEvent.ElementDelete, item)
}

function setAlign(align: 'left' | 'center' | 'right' | 'top' | 'middle' | 'bottom') {
  emitter.emit(BusEvent.ElementAlign, align)
}

function setLayer(align: 'up' | 'down' | 'top' | 'bottom') {
  emitter.emit(BusEvent.ElementLayer, align)
}
</script>
<template>
  <div class="right-panel flex w-[400px] justify-between bg-[#272836] text-white">
    <div class="w-full p-4">
      <p class="panel-title">元素层级:</p>
      <div class="grid grid-cols-2 gap-4 rounded-md bg-[#1c1c26] p-2">
        <div class="layer-btn" @click="setLayer('up')">
          <IconLayerUp fill="white" />
          <span>上移一层</span>
        </div>
        <div class="layer-btn" @click="setLayer('down')">
          <IconLayerDown fill="white" />
          <span>下移一层</span>
        </div>
        <div class="layer-btn" @click="setLayer('top')">
          <IconLayerTop fill="white" />
          <span>置于顶层</span>
        </div>
        <div class="layer-btn" @click="setLayer('bottom')">
          <IconLayerBottom fill="white" />
          <span>置于底层</span>
        </div>
      </div>
      <p class="panel-title">对齐方式:</p>
      <div class="flex h-[40px] items-center justify-between gap-4 rounded-md bg-[#1c1c26] p-2">
        <div class="position-btn" @click="setAlign('left')">
          <IconAlignLeft fill="white" />
        </div>
        <div class="position-btn" @click="setAlign('center')">
          <IconAlignCenter fill="white" />
        </div>
        <div class="position-btn" @click="setAlign('right')">
          <IconAlignRight fill="white" />
        </div>
        <div class="position-btn" @click="setAlign('top')">
          <IconAlignTop fill="white" />
        </div>
        <div class="position-btn" @click="setAlign('middle')">
          <IconAlignMiddle fill="white" />
        </div>
        <div class="position-btn" @click="setAlign('bottom')">
          <IconAlignBottom fill="white" />
        </div>
      </div>
      <p class="panel-title">元素列表:</p>
      <div class="bg-[#1c1c26]">
        <div
          class="flex max-h-[400px] flex-col gap-4 overflow-y-auto rounded-md p-2"
          v-if="elementList.length > 0"
        >
          <div
            class="flex cursor-pointer items-center justify-between rounded-md border border-[#3b3b4f] p-2"
            :class="focusElementIds.includes(item.elementId) ? 'bg-black' : ''"
            v-for="(item, index) in elementList"
            :key="index"
            @click="emitter.emit(BusEvent.ElementSelect, item)"
          >
            <p class="m-0 h-[16px] p-0 leading-[16px]">{{ item.elementType }} - {{ index + 1 }}</p>
            <button class="btn btn-error btn-sm" @click="deleteElement(item)">
              <IconDelete fill="white" />
            </button>
          </div>
        </div>
        <div v-else><p class="m-0 max-h-[400px] rounded-md p-2 text-center">暂无元素</p></div>
      </div>
    </div>
    <div class="w-[60px] bg-[#1c1c26] p-8"></div>
  </div>
</template>
<style lang="scss" scoped>
.panel-title {
  @apply my-2 font-black leading-6;
}
.position-btn {
  @apply flex h-[30px] w-[30px] items-center justify-center rounded-md hover:bg-slate-400;
}
.layer-btn {
  @apply flex w-[110px] cursor-pointer items-center justify-between rounded-md px-2 py-1 hover:bg-slate-400;
}
</style>
