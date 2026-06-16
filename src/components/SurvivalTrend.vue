<script setup lang="ts">
import { computed, ref } from 'vue'
import type { StatsSnapshot } from '@/types/game'

interface Props {
  history: StatsSnapshot[]
  highlightedTurn?: number
}

const props = defineProps<Props>()

const emit = defineEmits<{
  (e: 'hover', turn: number | null): void
}>()

const chartWidth = 560
const chartHeight = 180
const paddingTop = 16
const paddingBottom = 28
const paddingLeft = 36
const paddingRight = 12

const innerWidth = chartWidth - paddingLeft - paddingRight
const innerHeight = chartHeight - paddingTop - paddingBottom

const hoveredTurn = ref<number | null>(null)

function getY(value: number): number {
  return paddingTop + innerHeight - (value / 100) * innerHeight
}

function getX(index: number, total: number): number {
  if (total <= 1) return paddingLeft + innerWidth / 2
  return paddingLeft + (index / (total - 1)) * innerWidth
}

function buildPath(key: 'health' | 'hunger' | 'thirst'): string {
  if (props.history.length === 0) return ''
  const points = props.history.map((s, i) => {
    const x = getX(i, props.history.length)
    const y = getY(s[key])
    return `${x},${y}`
  })
  return 'M' + points.join(' L')
}

function buildAreaPath(key: 'health' | 'hunger' | 'thirst'): string {
  if (props.history.length === 0) return ''
  const points = props.history.map((s, i) => {
    const x = getX(i, props.history.length)
    const y = getY(s[key])
    return `${x},${y}`
  })
  const firstX = getX(0, props.history.length)
  const lastX = getX(props.history.length - 1, props.history.length)
  const baseY = paddingTop + innerHeight
  return `M${firstX},${baseY} L` + points.join(' L') + ` L${lastX},${baseY} Z`
}

const yTicks = computed(() => {
  return [0, 25, 50, 75, 100]
})

function handleMouseMove(e: MouseEvent) {
  if (props.history.length === 0) return
  const svg = e.currentTarget as SVGSVGElement
  const rect = svg.getBoundingClientRect()
  const x = ((e.clientX - rect.left) / rect.width) * chartWidth
  const relX = x - paddingLeft
  if (relX < 0 || relX > innerWidth) {
    hoveredTurn.value = null
    emit('hover', null)
    return
  }
  const ratio = relX / innerWidth
  const index = Math.round(ratio * (props.history.length - 1))
  const clampedIndex = Math.max(0, Math.min(props.history.length - 1, index))
  hoveredTurn.value = props.history[clampedIndex].turn
  emit('hover', hoveredTurn.value)
}

function handleMouseLeave() {
  hoveredTurn.value = null
  emit('hover', null)
}

const hoveredSnapshot = computed(() => {
  if (hoveredTurn.value === null) return null
  return props.history.find(s => s.turn === hoveredTurn.value) || null
})

const activeHighlightedTurn = computed(() => {
  return hoveredTurn.value ?? props.highlightedTurn ?? null
})

const highlightedX = computed(() => {
  if (activeHighlightedTurn.value === null || props.history.length === 0) return null
  const index = props.history.findIndex(s => s.turn === activeHighlightedTurn.value)
  if (index === -1) return null
  return getX(index, props.history.length)
})

const lineColors = {
  health: '#ef4444',
  hunger: '#f97316',
  thirst: '#3b82f6',
}

const areaGradients = {
  health: 'url(#healthGradient)',
  hunger: 'url(#hungerGradient)',
  thirst: 'url(#thirstGradient)',
}
</script>

<template>
  <div class="bg-game-card rounded-2xl p-6 border border-game-border shadow-xl">
    <h2 class="text-xl font-bold text-white mb-4 flex items-center gap-2">
      <span>📈</span>
      <span>生存趋势</span>
    </h2>

    <div class="flex gap-4 mb-3 text-xs">
      <div class="flex items-center gap-1.5">
        <span class="w-3 h-0.5 bg-red-500 rounded-full"></span>
        <span class="text-red-400">生命值</span>
      </div>
      <div class="flex items-center gap-1.5">
        <span class="w-3 h-0.5 bg-orange-500 rounded-full"></span>
        <span class="text-orange-400">饥饿值</span>
      </div>
      <div class="flex items-center gap-1.5">
        <span class="w-3 h-0.5 bg-blue-500 rounded-full"></span>
        <span class="text-blue-400">口渴值</span>
      </div>
    </div>

    <div class="relative">
      <svg
        :width="chartWidth"
        :height="chartHeight"
        class="w-full h-auto cursor-crosshair"
        viewBox="0 0 560 180"
        preserveAspectRatio="xMidYMid meet"
        @mousemove="handleMouseMove"
        @mouseleave="handleMouseLeave"
      >
        <defs>
          <linearGradient id="healthGradient" x1="0" y1="0" x2="0" y2="1">
            <stop offset="0%" stop-color="#ef4444" stop-opacity="0.3" />
            <stop offset="100%" stop-color="#ef4444" stop-opacity="0" />
          </linearGradient>
          <linearGradient id="hungerGradient" x1="0" y1="0" x2="0" y2="1">
            <stop offset="0%" stop-color="#f97316" stop-opacity="0.3" />
            <stop offset="100%" stop-color="#f97316" stop-opacity="0" />
          </linearGradient>
          <linearGradient id="thirstGradient" x1="0" y1="0" x2="0" y2="1">
            <stop offset="0%" stop-color="#3b82f6" stop-opacity="0.3" />
            <stop offset="100%" stop-color="#3b82f6" stop-opacity="0" />
          </linearGradient>
        </defs>

        <g class="text-gray-600 text-xs" fill="#4b5563">
          <text
            v-for="tick in yTicks"
            :key="tick"
            :x="paddingLeft - 6"
            :y="getY(tick) + 4"
            text-anchor="end"
            font-size="10"
          >
            {{ tick }}
          </text>
        </g>

        <g stroke="#27272a" stroke-width="1">
          <line
            v-for="tick in yTicks"
            :key="'grid-' + tick"
            :x1="paddingLeft"
            :y1="getY(tick)"
            :x2="chartWidth - paddingRight"
            :y2="getY(tick)"
            stroke-dasharray="2,4"
          />
        </g>

        <path v-if="history.length > 1" :d="buildAreaPath('health')" :fill="areaGradients.health" />
        <path v-if="history.length > 1" :d="buildAreaPath('hunger')" :fill="areaGradients.hunger" />
        <path v-if="history.length > 1" :d="buildAreaPath('thirst')" :fill="areaGradients.thirst" />

        <path
          v-if="history.length > 1"
          :d="buildPath('health')"
          fill="none"
          :stroke="lineColors.health"
          stroke-width="2"
          stroke-linecap="round"
          stroke-linejoin="round"
        />
        <path
          v-if="history.length > 1"
          :d="buildPath('hunger')"
          fill="none"
          :stroke="lineColors.hunger"
          stroke-width="2"
          stroke-linecap="round"
          stroke-linejoin="round"
        />
        <path
          v-if="history.length > 1"
          :d="buildPath('thirst')"
          fill="none"
          :stroke="lineColors.thirst"
          stroke-width="2"
          stroke-linecap="round"
          stroke-linejoin="round"
        />

        <line
          v-if="highlightedX !== null"
          :x1="highlightedX"
          :y1="paddingTop"
          :x2="highlightedX"
          :y2="paddingTop + innerHeight"
          stroke="#a1a1aa"
          stroke-width="1"
          stroke-dasharray="4,4"
        />

        <template v-if="activeHighlightedTurn !== null && hoveredSnapshot">
          <circle
            :cx="highlightedX"
            :cy="getY(hoveredSnapshot.health)"
            r="4"
            :fill="lineColors.health"
            stroke="#fff"
            stroke-width="1.5"
          />
          <circle
            :cx="highlightedX"
            :cy="getY(hoveredSnapshot.hunger)"
            r="4"
            :fill="lineColors.hunger"
            stroke="#fff"
            stroke-width="1.5"
          />
          <circle
            :cx="highlightedX"
            :cy="getY(hoveredSnapshot.thirst)"
            r="4"
            :fill="lineColors.thirst"
            stroke="#fff"
            stroke-width="1.5"
          />
        </template>

        <text
          v-if="history.length > 0"
          :x="paddingLeft"
          :y="chartHeight - 8"
          fill="#71717a"
          font-size="10"
        >
          第 {{ history[0].turn }} 回合
        </text>
        <text
          v-if="history.length > 0"
          :x="chartWidth - paddingRight"
          :y="chartHeight - 8"
          fill="#71717a"
          font-size="10"
          text-anchor="end"
        >
          第 {{ history[history.length - 1].turn }} 回合
        </text>
      </svg>

      <div
        v-if="hoveredSnapshot"
        class="absolute top-2 right-2 bg-game-card/95 backdrop-blur border border-game-border rounded-lg px-3 py-2 text-xs shadow-lg pointer-events-none"
      >
        <div class="text-gray-400 mb-1">第 {{ hoveredSnapshot.turn }} 回合</div>
        <div class="space-y-0.5">
          <div class="flex items-center gap-2">
            <span class="w-2 h-2 rounded-full bg-red-500"></span>
            <span class="text-red-400">生命 {{ Math.round(hoveredSnapshot.health) }}</span>
          </div>
          <div class="flex items-center gap-2">
            <span class="w-2 h-2 rounded-full bg-orange-500"></span>
            <span class="text-orange-400">饥饿 {{ Math.round(hoveredSnapshot.hunger) }}</span>
          </div>
          <div class="flex items-center gap-2">
            <span class="w-2 h-2 rounded-full bg-blue-500"></span>
            <span class="text-blue-400">口渴 {{ Math.round(hoveredSnapshot.thirst) }}</span>
          </div>
        </div>
      </div>

      <div
        v-if="history.length === 0"
        class="absolute inset-0 flex items-center justify-center text-gray-500 text-sm"
      >
        暂无数据，开始行动后将显示趋势
      </div>
    </div>
  </div>
</template>
