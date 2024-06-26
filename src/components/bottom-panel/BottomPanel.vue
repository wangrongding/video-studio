<script setup lang="ts">
import TrackList from './TrackList.vue'
import { storeToRefs } from 'pinia'
import {
  IconCopy,
  IconCut,
  IconDelete,
  IconNext,
  IconPause,
  IconPlay,
  IconPrev,
  IconRedo,
  IconUndo,
  IconZoomToFit,
  IconZoomin,
  IconZoomout,
} from '~/assets/icons/index'
import { usePlayerStore } from '~/stores/player'
import emitter, { BusEvent } from '~/utils/eventBus'
import { formatSeconds } from '~/utils/index'

const playerStore = usePlayerStore()
// 将 store 中的 playStatus 转换为 ref
const { playStatus, currentTime, duration } = storeToRefs(playerStore)

function skipVideo(time: number) {
  emitter.emit(BusEvent.VideoSkip, time)
}
function toggleCanvasFullScreen() {
  emitter.emit(BusEvent.CanvasFullScreen, true)
}
</script>
<template>
  <div class="bottom-panel flex h-[300px] flex-col bg-[#17171e] px-4 pb-2 text-white">
    <!-- 刻度尺 -->
    <div class="flex h-[40px] items-center justify-between gap-2 border-y border-[#272836]">
      <div class="flex gap-2">
        <button class="btn-control tooltip" data-tip="撤销"><IconUndo /></button>
        <button class="btn-control tooltip" data-tip="恢复"><IconRedo /></button>
        <button class="btn-control tooltip" data-tip="分割"><IconCut /></button>
        <button class="btn-control tooltip" data-tip="删除"><IconDelete /></button>
        <button class="btn-control tooltip" data-tip="复制"><IconCopy /></button>
      </div>
      <div class="flex items-center gap-2">
        <button class="btn-control" @click="skipVideo(-5)"><IconPrev /></button>
        <button class="btn-control" v-if="playStatus" @click="() => playerStore.togglePlay()">
          <IconPause />
        </button>
        <button class="btn-control" v-else @click="() => playerStore.togglePlay()">
          <IconPlay />
        </button>
        <button class="btn-control" @click="skipVideo(5)"><IconNext /></button>
        <span>{{ `${formatSeconds(currentTime)} / ${formatSeconds(duration)}` }}</span>
      </div>
      <div class="flex gap-4">
        <button class="btn-control tooltip" data-tip="放大"><IconZoomout /></button>
        <div class="flex items-center gap-2">
          <input
            type="range"
            min="0"
            max="100"
            value="40"
            class="range range-xs [--range-shdw:#00a7ea]"
          />
          {{ '40%' }}
        </div>
        <button class="btn-control tooltip" data-tip="缩小"><IconZoomin /></button>
        <button class="tooltip" data-tip="适配到合适的大小">
          <IconZoomToFit class="btn-control" />
        </button>
        <button @click="toggleCanvasFullScreen">预览</button>
      </div>
    </div>

    <!-- 轨道列表 -->
    <TrackList />
  </div>
</template>
<style scoped>
.btn-control {
  @apply rounded-sm hover:bg-[#ee2788];
}
</style>
