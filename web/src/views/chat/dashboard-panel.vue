<script lang="ts" setup>
import MarkdownAntv from '@/components/MarkdownPreview/markdown-antv.vue'
import type { DashboardChartItem } from '@/store/business'

const businessStore = useBusinessStore()
const refreshMap = ref<Record<string, number>>({})
const gridRef = ref<HTMLElement | null>(null)
const draggingId = ref('')
const dropTargetId = ref('')
const currentColumns = ref(12)
const rowHeight = 72
const gridGap = 16

const allCharts = computed(() => businessStore.dashboardCharts || [])
const visibleCharts = computed(() => allCharts.value.filter(item => !item.hidden))
const hiddenCount = computed(() => allCharts.value.filter(item => item.hidden).length)
const isSingleColumn = computed(() => currentColumns.value === 1)
const gridTemplateColumns = computed(() => `repeat(${currentColumns.value}, minmax(0, 1fr))`)

const updateColumns = () => {
  const width = window.innerWidth
  if (width <= 900) {
    currentColumns.value = 1
    return
  }
  if (width <= 1280) {
    currentColumns.value = 6
    return
  }
  currentColumns.value = 12
}

onMounted(() => {
  updateColumns()
  window.addEventListener('resize', updateColumns)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', updateColumns)
})

const getMenuOptions = (item: DashboardChartItem) => [
  {
    label: item.hidden ? '显示图表' : '隐藏图表',
    key: `toggle-${item.id}`,
  },
  {
    label: '刷新',
    key: `refresh-${item.id}`,
  },
  {
    label: '删除',
    key: `delete-${item.id}`,
  },
]

const bumpRefreshVersion = (chartIds: string[]) => {
  const uniqueIds = Array.from(new Set(chartIds.filter(Boolean)))
  if (uniqueIds.length === 0) {
    return
  }
  uniqueIds.forEach((chartId) => {
    const current = refreshMap.value[chartId] || 0
    refreshMap.value[chartId] = current + 1
  })
}

const getAffectedChartIdsByMove = (sourceId: string, targetId: string) => {
  const currentVisibleIds = visibleCharts.value.map((item) => item.id)
  const sourceIndex = currentVisibleIds.indexOf(sourceId)
  const targetIndex = currentVisibleIds.indexOf(targetId)
  if (sourceIndex === -1 || targetIndex === -1) {
    return [sourceId, targetId]
  }
  const startIndex = Math.min(sourceIndex, targetIndex)
  const endIndex = Math.max(sourceIndex, targetIndex)
  return currentVisibleIds.slice(startIndex, endIndex + 1)
}

const getItemLayout = (item: DashboardChartItem) => {
  return {
    w: item.layout?.w || 6,
    h: item.layout?.h || 6,
    minW: item.layout?.minW || 3,
    minH: item.layout?.minH || 5,
  }
}

const getGridItemStyle = (item: DashboardChartItem) => {
  const layout = getItemLayout(item)
  const colSpan = Math.max(1, Math.min(currentColumns.value, layout.w))
  const rowSpan = Math.max(1, layout.h)
  return {
    gridColumn: `span ${colSpan}`,
    gridRow: `span ${rowSpan}`,
  }
}

const handleMenuSelect = (key: string) => {
  if (key.startsWith('toggle-')) {
    const chartId = key.replace('toggle-', '')
    businessStore.toggle_dashboard_chart_hidden(chartId)
    return
  }

  if (key.startsWith('refresh-')) {
    const chartId = key.replace('refresh-', '')
    const current = refreshMap.value[chartId] || 0
    refreshMap.value[chartId] = current + 1
    return
  }

  if (key.startsWith('delete-')) {
    const chartId = key.replace('delete-', '')
    businessStore.remove_dashboard_chart(chartId)
  }
}

const handleRestoreHidden = () => {
  businessStore.clear_dashboard_hidden()
}

const handleDragStart = (event: DragEvent, chartId: string) => {
  if (isSingleColumn.value) {
    return
  }
  draggingId.value = chartId
  dropTargetId.value = chartId
  if (event.dataTransfer) {
    event.dataTransfer.effectAllowed = 'move'
  }
}

const handleDragOver = (event: DragEvent, chartId: string) => {
  if (isSingleColumn.value || !draggingId.value || draggingId.value === chartId) {
    return
  }
  event.preventDefault()
  dropTargetId.value = chartId
}

const handleDrop = (event: DragEvent, targetId: string) => {
  if (isSingleColumn.value) {
    return
  }
  event.preventDefault()
  const sourceId = draggingId.value
  if (!sourceId || sourceId === targetId) {
    draggingId.value = ''
    dropTargetId.value = ''
    return
  }
  const affectedChartIds = getAffectedChartIdsByMove(sourceId, targetId)
  businessStore.reorder_dashboard_charts(sourceId, targetId)
  bumpRefreshVersion(affectedChartIds)
  draggingId.value = ''
  dropTargetId.value = ''
}

const handleDragEnd = () => {
  draggingId.value = ''
  dropTargetId.value = ''
}

const resizingState = ref<{
  chartId: string
  startX: number
  startY: number
  startW: number
  startH: number
  minW: number
  minH: number
  colWidth: number
} | null>(null)

const handleResizeMove = (event: PointerEvent) => {
  if (!resizingState.value) {
    return
  }
  const state = resizingState.value
  const deltaX = event.clientX - state.startX
  const deltaY = event.clientY - state.startY
  const deltaW = Math.round(deltaX / state.colWidth)
  const deltaH = Math.round(deltaY / rowHeight)
  const maxW = currentColumns.value
  const nextW = Math.max(state.minW, Math.min(maxW, state.startW + deltaW))
  const nextH = Math.max(state.minH, state.startH + deltaH)
  businessStore.update_dashboard_chart_layout(state.chartId, { w: nextW, h: nextH })
}

const handleResizeEnd = () => {
  if (!resizingState.value) {
    return
  }
  const chartId = resizingState.value.chartId
  const current = refreshMap.value[chartId] || 0
  refreshMap.value[chartId] = current + 1
  resizingState.value = null
  window.removeEventListener('pointermove', handleResizeMove)
  window.removeEventListener('pointerup', handleResizeEnd)
}

const startResize = (event: PointerEvent, item: DashboardChartItem) => {
  if (isSingleColumn.value || !gridRef.value) {
    return
  }
  event.preventDefault()
  event.stopPropagation()
  const layout = getItemLayout(item)
  const colWidth = (gridRef.value.clientWidth - gridGap * (currentColumns.value - 1)) / currentColumns.value
  resizingState.value = {
    chartId: item.id,
    startX: event.clientX,
    startY: event.clientY,
    startW: layout.w,
    startH: layout.h,
    minW: layout.minW,
    minH: layout.minH,
    colWidth,
  }
  window.addEventListener('pointermove', handleResizeMove)
  window.addEventListener('pointerup', handleResizeEnd)
}
</script>

<template>
  <div class="dashboard-page h-full w-full overflow-y-auto">
    <div class="dashboard-inner">
      <div class="dashboard-header">
        <div class="dashboard-title-wrap">
          <div class="i-hugeicons:dashboard-square-02 text-20 text-[#3B5CFF]"></div>
          <span class="dashboard-title">Dashboard</span>
        </div>
        <div v-if="hiddenCount > 0" class="hidden-tip">
          <span>已隐藏 {{ hiddenCount }} 个图表</span>
          <n-button text type="primary" @click="handleRestoreHidden">
            全部恢复
          </n-button>
        </div>
      </div>

      <div v-if="visibleCharts.length === 0" class="dashboard-empty">
        <div class="i-hugeicons:chart text-54 text-[#c8cde6]"></div>
        <div class="empty-title">Dashboard 暂无图表</div>
        <div class="empty-subtitle">在图表右上角点击“添加到 Dashboard”即可收藏到这里</div>
      </div>

      <div
        v-else
        ref="gridRef"
        class="dashboard-grid"
        :style="{ gridTemplateColumns }"
      >
        <div
          v-for="item in visibleCharts"
          :key="item.id"
          class="dashboard-grid-item"
          :class="{
            'dashboard-grid-item--dragging': draggingId === item.id,
            'dashboard-grid-item--drop-target': dropTargetId === item.id && draggingId !== item.id,
          }"
          :style="getGridItemStyle(item)"
          @dragover="(event) => handleDragOver(event, item.id)"
          @drop="(event) => handleDrop(event, item.id)"
        >
          <div class="dashboard-card-header">
            <div
              class="dashboard-card-drag"
              :draggable="!isSingleColumn"
              @dragstart="(event) => handleDragStart(event, item.id)"
              @dragend="handleDragEnd"
            >
              <div class="i-hugeicons:move text-16"></div>
            </div>
            <div class="dashboard-card-title">{{ item.title }}</div>
            <n-dropdown
              trigger="click"
              :options="getMenuOptions(item)"
              placement="bottom-end"
              @select="handleMenuSelect"
            >
              <div class="i-hugeicons:more-vertical text-18 text-[#8A8A8A] hover:text-[#333] cursor-pointer"></div>
            </n-dropdown>
          </div>
          <MarkdownAntv
            :key="item.id"
            :chart-id="`dashboard-${item.id}`"
            :chart-data="item.chartData"
            :qa-type="item.qaType"
            :record-id="item.recordId || undefined"
            :refresh-version="refreshMap[item.id] || 0"
            :allow-pin="false"
          />
          <div
            v-if="!isSingleColumn"
            class="dashboard-resize-handle"
            @pointerdown="(event) => startResize(event, item)"
          >
            <div class="i-hugeicons:drag-04 text-14"></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.dashboard-page {
  background: #f3f5f9;
}

.dashboard-inner {
  width: min(1380px, 96%);
  margin: 0 auto;
  padding: 20px 0 28px;
}

.dashboard-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 14px;
  gap: 12px;
}

.dashboard-title-wrap {
  display: flex;
  align-items: center;
  gap: 8px;
}

.dashboard-title {
  font-size: 20px;
  font-weight: 700;
  color: #1f2937;
}

.hidden-tip {
  display: flex;
  align-items: center;
  gap: 8px;
  color: #666;
  font-size: 13px;
}

.dashboard-empty {
  height: calc(100vh - 210px);
  min-height: 400px;
  border: 1px dashed #d7ddf2;
  border-radius: 14px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  background: #fff;
  gap: 8px;
}

.empty-title {
  font-size: 18px;
  font-weight: 600;
  color: #3f4b6b;
}

.empty-subtitle {
  color: #7f8aab;
  font-size: 14px;
}

.dashboard-grid {
  display: grid;
  grid-template-columns: repeat(12, minmax(0, 1fr));
  grid-auto-rows: 72px;
  grid-auto-flow: dense;
  gap: 16px;
}

.dashboard-grid-item {
  background: #ffffff;
  border-radius: 10px;
  border: 1px solid #d8dee9;
  box-shadow: 0 1px 3px rgba(16, 24, 40, 0.06);
  padding: 0;
  position: relative;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  transition: box-shadow 0.2s ease, border-color 0.2s ease, opacity 0.2s ease;
}

.dashboard-card-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 0;
  padding: 8px 10px;
  border-bottom: 1px solid #e5eaf3;
  background: linear-gradient(180deg, #f9fafc 0%, #f5f7fb 100%);
  min-height: 42px;
}

.dashboard-card-drag {
  width: 22px;
  height: 22px;
  border-radius: 6px;
  background: #f4f6fb;
  color: #8a8a8a;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: grab;
  flex-shrink: 0;
}

.dashboard-card-drag:active {
  cursor: grabbing;
}

.dashboard-card-title {
  color: #1f2937;
  font-size: 13px;
  font-weight: 600;
  line-height: 1.2;
  flex: 1;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.dashboard-grid-item--dragging {
  opacity: 0.55;
}

.dashboard-grid-item--drop-target {
  border-color: #5794f2;
  box-shadow: 0 0 0 2px rgba(87, 148, 242, 0.2), 0 8px 22px rgba(38, 69, 125, 0.15);
}

.dashboard-resize-handle {
  position: absolute;
  right: 6px;
  bottom: 6px;
  width: 18px;
  height: 18px;
  border-radius: 6px;
  background: #edf2fa;
  color: #6b7280;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: nwse-resize;
  user-select: none;
  z-index: 20;
}

.dashboard-grid-item :deep(.chart-wrapper) {
  flex: 1;
  min-height: 0;
}

.dashboard-grid-item :deep(.modern-chart-card) {
  height: 100%;
  border-radius: 0;
  box-shadow: none;
  border: none;
}

.dashboard-grid-item :deep(.modern-chart-card > .n-card-header) {
  display: none;
}

.dashboard-grid-item :deep(.modern-chart-card > .n-card__content) {
  padding: 0 !important;
  height: 100%;
}

.dashboard-grid-item :deep(.card-content-wrapper) {
  height: 100%;
  min-height: 0;
}

.dashboard-grid-item :deep(.chart-view) {
  height: 100%;
}

.dashboard-grid-item :deep(.chart-container) {
  height: 100% !important;
  min-height: 0 !important;
  border-radius: 0;
  padding: 10px !important;
  background: #ffffff !important;
}

@media (max-width: 1100px) {
  .dashboard-grid {
    grid-template-columns: repeat(6, minmax(0, 1fr));
  }
}
</style>
