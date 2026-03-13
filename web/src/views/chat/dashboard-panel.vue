<script lang="ts" setup>
import MarkdownAntv from '@/components/MarkdownPreview/markdown-antv.vue'
import type { DashboardChartItem } from '@/store/business'

const businessStore = useBusinessStore()
const refreshMap = ref<Record<string, number>>({})

const allCharts = computed(() => businessStore.dashboardCharts || [])
const visibleCharts = computed(() => allCharts.value.filter(item => !item.hidden))
const hiddenCount = computed(() => allCharts.value.filter(item => item.hidden).length)

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

const getChartKey = (item: DashboardChartItem) => {
  const refreshVersion = refreshMap.value[item.id] || 0
  return `${item.id}-${refreshVersion}`
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

      <div v-else class="dashboard-grid">
        <div
          v-for="item in visibleCharts"
          :key="item.id"
          class="dashboard-grid-item"
          :class="{ 'dashboard-grid-item--wide': visibleCharts.length > 2 && item === visibleCharts[0] }"
        >
          <div class="dashboard-card-header">
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
            :key="getChartKey(item)"
            :chart-id="`dashboard-${item.id}`"
            :chart-data="item.chartData"
            :qa-type="item.qaType"
            :record-id="item.recordId || undefined"
            :allow-pin="false"
          />
        </div>
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.dashboard-page {
  background: #f6f8fc;
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
  margin-bottom: 12px;
  gap: 12px;
}

.dashboard-title-wrap {
  display: flex;
  align-items: center;
  gap: 8px;
}

.dashboard-title {
  font-size: 22px;
  font-weight: 700;
  color: #111;
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
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 16px;
}

.dashboard-grid-item {
  background: #fff;
  border-radius: 12px;
  border: 1px solid #e9edf7;
  box-shadow: 0 4px 16px rgba(24, 39, 75, 0.06);
  padding: 10px;
}

.dashboard-grid-item--wide {
  grid-column: 1 / -1;
}

.dashboard-card-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin: 2px 4px 10px;
}

.dashboard-card-title {
  color: #222;
  font-size: 15px;
  font-weight: 600;
  line-height: 1.2;
}

@media (max-width: 1100px) {
  .dashboard-grid {
    grid-template-columns: 1fr;
  }

  .dashboard-grid-item--wide {
    grid-column: auto;
  }
}
</style>
