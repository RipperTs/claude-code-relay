<template>
  <div class="api-key-query-container">
    <!-- API Key输入区域 -->
    <div class="api-key-input-section">
      <t-form :model="form">
        <t-form-item label="API Key" required>
          <t-input
            v-model="form.apiKey"
            :placeholder="loading ? '正在查询中...' : '请输入API Key (sk-...)'"
            clearable
            :disabled="loading"
            @enter="search"
            @blur="onBlur"
            @clear="clear"
          >
            <template #suffix-icon>
              <t-loading v-if="loading" size="small" />
            </template>
          </t-input>
        </t-form-item>
      </t-form>
    </div>

    <div v-if="data" class="stats-section">
      <!-- API Key基本信息 -->
      <t-card class="api-key-info-card" title="API Key 信息">
        <div class="api-key-info">
          <div class="info-item">
            <span class="label">名称：</span>
            <span class="value">{{ data.api_key_info.name }}</span>
          </div>
          <div class="info-item">
            <span class="label">状态：</span>
            <t-tag v-if="data.api_key_info.status === 1" theme="success" variant="light"> 启用 </t-tag>
            <t-tag v-else theme="danger" variant="light"> 禁用 </t-tag>
          </div>
        </div>
      </t-card>

      <t-card class="overview-card" title="最近30天统计概览">
        <div class="overview-stats">
          <div class="stat-item">
            <div class="stat-value">{{ formatNumber(data.stats.summary.total_requests) }}</div>
            <div class="stat-label">总请求数</div>
          </div>
          <div class="stat-item">
            <div class="stat-value">{{ formatNumber(data.stats.summary.total_tokens) }}</div>
            <div class="stat-label">总Token数</div>
          </div>
          <div class="stat-item">
            <div class="stat-value">${{ formatCost(data.stats.summary.total_cost) }}</div>
            <div class="stat-label">总费用</div>
          </div>
          <div class="stat-item">
            <div class="stat-value">{{ formatDuration(data.stats.summary.avg_duration) }}</div>
            <div class="stat-label">平均响应时间</div>
          </div>
        </div>
      </t-card>

      <t-card v-if="chartData.length > 0" class="chart-card" title="使用趋势">
        <div ref="chartRef" class="chart-container"></div>
      </t-card>
    </div>

    <div v-if="logs" class="logs-section">
      <t-card class="logs-card" title="最近调用日志">
        <t-table
          :data="logs.list.slice(0, 10)"
          :columns="logColumns"
          :loading="loading"
          :pagination="false"
        >
          <template #model_name="{ row }">
            <t-tag theme="primary" variant="outline">{{ row.model_name }}</t-tag>
          </template>

          <template #tokens="{ row }">
            <div class="tokens-info">
              <p><strong>输入:</strong> {{ formatNumber(row.input_tokens) }}</p>
              <p><strong>输出:</strong> {{ formatNumber(row.output_tokens) }}</p>
            </div>
          </template>

          <template #cost="{ row }">
            <div class="cost-info">
              <p class="total-cost">
                <strong>${{ row.total_cost.toFixed(4) }}</strong>
              </p>
            </div>
          </template>

          <template #duration="{ row }">
            <t-tag theme="default" variant="light">{{ formatDuration(row.duration) }}</t-tag>
          </template>

          <template #created_at="{ row }">
            <span>{{ formatDateTime(row.created_at) }}</span>
          </template>
        </t-table>
      </t-card>
    </div>

    <t-empty v-if="!loading && !data && attempted" description="请输入有效的API Key进行查询" />
  </div>
</template>

<script setup lang="ts">
import { LineChart } from 'echarts/charts';
import { GridComponent, LegendComponent, TooltipComponent } from 'echarts/components';
import * as echarts from 'echarts/core';
import { CanvasRenderer } from 'echarts/renderers';
import type { PrimaryTableCol, TableRowData } from 'tdesign-vue-next';
import { MessagePlugin } from 'tdesign-vue-next';
import { computed, nextTick, onUnmounted, reactive, ref, watch } from 'vue';

import { getApiKeyStats } from '@/api/stats';

const emit = defineEmits<{
  'has-data': [hasData: boolean];
  'clear-data': [];
}>();

echarts.use([LineChart, GridComponent, TooltipComponent, LegendComponent, CanvasRenderer]);

const loading = ref(false);
const attempted = ref(false);
const chartRef = ref<HTMLElement>();
let chart: echarts.ECharts | null = null;

const form = reactive({ apiKey: '' });
const data = ref<any>(null);
const logs = ref<any>(null);

const logColumns: PrimaryTableCol<TableRowData>[] = [
  {
    title: '模型名称',
    colKey: 'model_name',
    width: 140,
  },
  {
    title: 'Token使用',
    colKey: 'tokens',
    width: 160,
  },
  {
    title: '费用',
    colKey: 'cost',
    width: 100,
  },
  {
    title: '耗时',
    colKey: 'duration',
    width: 80,
  },
  {
    title: '时间',
    colKey: 'created_at',
    width: 140,
  },
];

const chartData = computed(() => data.value?.stats?.trend_data || []);

watch(data, (newData) => {
  const hasData = !!newData;
  emit('has-data', hasData);
  
  if (hasData) {
    setTimeout(renderChart, 450);
  }
}, { immediate: true });

async function search() {
  if (!form.apiKey.trim()) {
    MessagePlugin.warning('请输入API Key');
    return;
  }

  loading.value = true;
  attempted.value = true;

  try {
    const response = await getApiKeyStats({
      api_key: form.apiKey,
      page: 1,
      limit: 20,
    });

    data.value = response;
    logs.value = response.logs;
    MessagePlugin.success('查询成功');
  } catch (error: any) {
    console.error('查询失败:', error);
    MessagePlugin.error(error.message || '查询失败');
    data.value = null;
    logs.value = null;
    emit('has-data', false);
  } finally {
    loading.value = false;
  }
}

function onBlur() {
  if (form.apiKey.trim()) {
    search();
  }
}

function clear() {
  loading.value = false;
  data.value = null;
  logs.value = null;
  attempted.value = false;
  emit('clear-data');
}

function renderChart() {
  if (!chartRef.value || chartData.value.length === 0) return;

  const rect = chartRef.value.getBoundingClientRect();
  if (rect.width === 0 || rect.height === 0) {
    setTimeout(renderChart, 100);
    return;
  }

  if (chart) {
    chart.dispose();
  }

  chart = echarts.init(chartRef.value);

  const option = {
    tooltip: {
      trigger: 'axis',
      axisPointer: {
        type: 'cross',
      },
    },
    legend: {
      data: ['请求数', 'Token数', '费用'],
    },
    grid: {
      left: '3%',
      right: '4%',
      bottom: '3%',
      containLabel: true,
    },
    xAxis: {
      type: 'category',
      data: chartData.value.map((item: any) => item.date),
    },
    yAxis: [
      {
        type: 'value',
        name: '请求数/Token数',
        position: 'left',
      },
      {
        type: 'value',
        name: '费用($)',
        position: 'right',
      },
    ],
    series: [
      {
        name: '请求数',
        type: 'line',
        data: chartData.value.map((item: any) => item.requests),
        smooth: true,
      },
      {
        name: 'Token数',
        type: 'line',
        data: chartData.value.map((item: any) => item.tokens),
        smooth: true,
      },
      {
        name: '费用',
        type: 'line',
        yAxisIndex: 1,
        data: chartData.value.map((item: any) => item.cost),
        smooth: true,
      },
    ],
  };

  chart.setOption(option);
  
  nextTick(() => {
    chart?.resize();
  });
}

// 格式化数字
function formatNumber(value: number): string {
  if (value >= 1000000) {
    return `${(value / 1000000).toFixed(1)}M`;
  }
  if (value >= 1000) {
    return `${(value / 1000).toFixed(1)}K`;
  }
  return value.toString();
}

// 格式化费用
function formatCost(value: number): string {
  return value.toFixed(4);
}

// 格式化时长
function formatDuration(duration: number): string {
  if (duration < 1000) return `${duration}ms`;
  return `${(duration / 1000).toFixed(2)}s`;
}

// 格式化日期时间
function formatDateTime(dateStr: string): string {
  if (!dateStr) return '-';
  const date = new Date(dateStr);
  if (isNaN(date.getTime())) return '-';
  return date.toLocaleString('zh-CN', {
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
  });
}

onUnmounted(() => {
  chart?.dispose();
});
</script>

<style lang="less" scoped>
.api-key-query-container {
  height: 100%;
  display: flex;
  flex-direction: column;
  
  .api-key-input-section {
    margin-bottom: var(--td-comp-margin-xl);
    flex-shrink: 0;

    :deep(.t-form-item) {
      margin-bottom: 0;
    }

    :deep(.t-input) {
      &.t-is-disabled {
        .t-input__inner {
          background-color: var(--td-bg-color-component);
          cursor: not-allowed;
          opacity: 0.7;
        }
      }
    }

    :deep(.t-loading) {
      color: var(--td-color-primary);
    }
  }

  .stats-section {
    flex-shrink: 0;
    margin-bottom: var(--td-comp-margin-xl);

    .api-key-info-card {
      margin-bottom: var(--td-comp-margin-l);

      .api-key-info {
        display: flex;
        gap: var(--td-comp-margin-xl);

        .info-item {
          .label {
            color: var(--td-text-color-secondary);
          }

          .value {
            font-weight: 500;
            color: var(--td-text-color-primary);
          }
        }
      }
    }

    .overview-card {
      margin-bottom: var(--td-comp-margin-l);

      .overview-stats {
        display: grid;
        grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
        gap: var(--td-comp-margin-l);

        .stat-item {
          text-align: center;
          padding: var(--td-comp-paddingTB-l) var(--td-comp-paddingLR-l);
          background: var(--td-bg-color-container-hover);
          border-radius: var(--td-radius-medium);

          .stat-value {
            font-size: 20px;
            font-weight: 600;
            color: var(--td-color-primary);
            margin-bottom: 4px;
          }

          .stat-label {
            color: var(--td-text-color-secondary);
            font-size: 12px;
          }
        }
      }
    }

    .chart-card {
      .chart-container {
        height: 300px;
        width: 100%;
      }
    }
  }

  .logs-section {
    flex-shrink: 0;
    margin-bottom: var(--td-comp-margin-xl);
    
    .logs-card {
      :deep(.t-table) {
        .t-table__cell {
          padding: 8px 12px;
        }
      }
    }

    .tokens-info,
    .cost-info {
      font-size: 11px;
      line-height: 1.3;

      p {
        margin: 1px 0;
      }
    }

    .total-cost {
      color: var(--td-text-color-primary);
      font-weight: 600;
    }
  }
}

@media (max-width: 768px) {
  .api-key-query-container {
    .stats-section {
      .api-key-info {
        flex-direction: column;
        gap: 12px;
      }

      .overview-stats {
        grid-template-columns: repeat(2, 1fr);
        gap: 12px;

        .stat-item {
          padding: 10px;
        }
      }
    }
  }
}
</style>