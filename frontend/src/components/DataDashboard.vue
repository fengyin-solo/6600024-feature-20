<template>
  <div class="data-dashboard" :class="{ 'frozen-mode': store.isFrozen }">
    <!-- 冻结遮罩提示 -->
    <div v-if="store.isFrozen" class="frozen-overlay">
      <el-icon :size="32" color="#e6a23c"><Lock /></el-icon>
      <span class="frozen-text">数据已冻结</span>
      <span class="frozen-hint">
        连接已断开，显示为断开前快照数据
        <span v-if="store.lastDisconnectTime">
          ({{ formatTime(store.lastDisconnectTime) }})
        </span>
      </span>
    </div>

    <!-- 仪表盘区域 -->
    <div class="gauges-section">
      <h3 class="section-title">
        实时仪表
        <el-tag v-if="store.isFrozen" type="warning" size="small" effect="dark" class="ml-2">
          <el-icon><Lock /></el-icon>
          已冻结
        </el-tag>
      </h3>
      <div class="gauges-grid">
        <!-- 温度仪表 -->
        <div class="gauge-card" :class="{ 'gauge-frozen': store.isFrozen, 'gauge-changed': hasNodeDiff('temp_sensor') }">
          <div class="gauge-header">
            <div class="gauge-label">温度</div>
            <el-tag v-if="store.isFrozen" type="warning" size="small" effect="plain">
              <el-icon :size="12"><Lock /></el-icon>
            </el-tag>
            <el-tag v-else-if="hasNodeDiff('temp_sensor')" :type="getDiffTagType('temp_sensor')" size="small" effect="dark">
              {{ getDiffLabel('temp_sensor') }}
            </el-tag>
          </div>
          <div class="gauge-value" :class="getTempClass(temperature)">
            {{ temperature.toFixed(1) }}
            <span class="gauge-unit">°C</span>
          </div>
          <el-progress
            :percentage="Math.max(0, Math.min((temperature / 50) * 100, 100))"
            :color="getTempColor(temperature)"
            :stroke-width="8"
          />
          <div class="gauge-quality">
            <span class="quality-dot" :class="getQualityClass('temp_sensor')"></span>
            {{ getNodeQuality('temp_sensor') }}
          </div>
        </div>

        <!-- 压力表 -->
        <div class="gauge-card" :class="{ 'gauge-frozen': store.isFrozen, 'gauge-changed': hasNodeDiff('pressure_transmitter') }">
          <div class="gauge-header">
            <div class="gauge-label">压力</div>
            <el-tag v-if="store.isFrozen" type="warning" size="small" effect="plain">
              <el-icon :size="12"><Lock /></el-icon>
            </el-tag>
            <el-tag v-else-if="hasNodeDiff('pressure_transmitter')" :type="getDiffTagType('pressure_transmitter')" size="small" effect="dark">
              {{ getDiffLabel('pressure_transmitter') }}
            </el-tag>
          </div>
          <div class="gauge-value" :class="getPressureClass(pressure)">
            {{ pressure.toFixed(2) }}
            <span class="gauge-unit">MPa</span>
          </div>
          <el-progress
            :percentage="Math.max(0, Math.min((pressure / 6) * 100, 100))"
            :color="getPressureColor(pressure)"
            :stroke-width="8"
          />
          <div class="gauge-quality">
            <span class="quality-dot" :class="getQualityClass('pressure_transmitter')"></span>
            {{ getNodeQuality('pressure_transmitter') }}
          </div>
        </div>

        <!-- 流量计 -->
        <div class="gauge-card" :class="{ 'gauge-frozen': store.isFrozen, 'gauge-changed': hasNodeDiff('flow_meter') }">
          <div class="gauge-header">
            <div class="gauge-label">流量</div>
            <el-tag v-if="store.isFrozen" type="warning" size="small" effect="plain">
              <el-icon :size="12"><Lock /></el-icon>
            </el-tag>
            <el-tag v-else-if="hasNodeDiff('flow_meter')" :type="getDiffTagType('flow_meter')" size="small" effect="dark">
              {{ getDiffLabel('flow_meter') }}
            </el-tag>
          </div>
          <div class="gauge-value text-blue-400">
            {{ flow.toFixed(1) }}
            <span class="gauge-unit">L/min</span>
          </div>
          <el-progress
            :percentage="Math.max(0, Math.min((flow / 300) * 100, 100))"
            color="#60a5fa"
            :stroke-width="8"
          />
          <div class="gauge-quality">
            <span class="quality-dot" :class="getQualityClass('flow_meter')"></span>
            {{ getNodeQuality('flow_meter') }}
          </div>
        </div>

        <!-- 阀门开度 -->
        <div class="gauge-card" :class="{ 'gauge-frozen': store.isFrozen, 'gauge-changed': hasNodeDiff('valve_position') }">
          <div class="gauge-header">
            <div class="gauge-label">阀门开度</div>
            <el-tag v-if="store.isFrozen" type="warning" size="small" effect="plain">
              <el-icon :size="12"><Lock /></el-icon>
            </el-tag>
            <el-tag v-else-if="hasNodeDiff('valve_position')" :type="getDiffTagType('valve_position')" size="small" effect="dark">
              {{ getDiffLabel('valve_position') }}
            </el-tag>
          </div>
          <div class="gauge-value text-purple-400">
            {{ valvePosition.toFixed(0) }}
            <span class="gauge-unit">%</span>
          </div>
          <el-progress
            :percentage="Math.max(0, Math.min(valvePosition, 100))"
            color="#a78bfa"
            :stroke-width="8"
          />
          <div class="gauge-quality">
            <span class="quality-dot" :class="getQualityClass('valve_position')"></span>
            {{ getNodeQuality('valve_position') }}
          </div>
        </div>

        <!-- 电机转速 -->
        <div class="gauge-card" :class="{ 'gauge-frozen': store.isFrozen, 'gauge-changed': hasNodeDiff('motor_speed') }">
          <div class="gauge-header">
            <div class="gauge-label">电机转速</div>
            <el-tag v-if="store.isFrozen" type="warning" size="small" effect="plain">
              <el-icon :size="12"><Lock /></el-icon>
            </el-tag>
            <el-tag v-else-if="hasNodeDiff('motor_speed')" :type="getDiffTagType('motor_speed')" size="small" effect="dark">
              {{ getDiffLabel('motor_speed') }}
            </el-tag>
          </div>
          <div class="gauge-value" :class="getSpeedClass(motorSpeed)">
            {{ motorSpeed }}
            <span class="gauge-unit">RPM</span>
          </div>
          <el-progress
            :percentage="Math.max(0, Math.min((motorSpeed / 2000) * 100, 100))"
            :color="getSpeedColor(motorSpeed)"
            :stroke-width="8"
          />
          <div class="gauge-quality">
            <span class="quality-dot" :class="getQualityClass('motor_speed')"></span>
            {{ getNodeQuality('motor_speed') }}
          </div>
        </div>

        <!-- 泵状态 -->
        <div class="gauge-card" :class="{ 'gauge-frozen': store.isFrozen, 'gauge-changed': hasNodeDiff('pump_status') }">
          <div class="gauge-header">
            <div class="gauge-label">泵运行状态</div>
            <el-tag v-if="store.isFrozen" type="warning" size="small" effect="plain">
              <el-icon :size="12"><Lock /></el-icon>
            </el-tag>
            <el-tag v-else-if="hasNodeDiff('pump_status')" :type="getDiffTagType('pump_status')" size="small" effect="dark">
              {{ getDiffLabel('pump_status') }}
            </el-tag>
          </div>
          <div class="gauge-value" :class="pumpStatus ? 'text-green-400' : 'text-red-400'">
            {{ pumpStatus ? '运行中' : '已停止' }}
          </div>
          <div class="pump-indicator" :class="pumpStatus ? 'pump-on' : 'pump-off'">
            <el-icon :size="32"><CircleCheckFilled v-if="pumpStatus" /><CircleCloseFilled v-else /></el-icon>
          </div>
          <div class="gauge-quality">
            <span class="quality-dot" :class="getQualityClass('pump_status')"></span>
            {{ getNodeQuality('pump_status') }}
          </div>
        </div>
      </div>
    </div>

    <!-- 趋势图区域 -->
    <div class="charts-section">
      <h3 class="section-title">数据趋势</h3>
      <div class="charts-grid">
        <div class="chart-card">
          <v-chart :option="tempChartOption" autoresize class="chart" />
        </div>
        <div class="chart-card">
          <v-chart :option="pressureChartOption" autoresize class="chart" />
        </div>
        <div class="chart-card">
          <v-chart :option="flowChartOption" autoresize class="chart" />
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import VChart from 'vue-echarts'
import { use } from 'echarts/core'
import { CanvasRenderer } from 'echarts/renderers'
import { LineChart } from 'echarts/charts'
import { GridComponent, TooltipComponent, TitleComponent } from 'echarts/components'
import { CircleCheckFilled, CircleCloseFilled, Lock } from '@element-plus/icons-vue'
import { useOpcuaStore } from '../store/opcua'
import type { NodeDataDiff } from '../types'

use([CanvasRenderer, LineChart, GridComponent, TooltipComponent, TitleComponent])

const store = useOpcuaStore()

function formatTime(timestamp: number): string {
  return new Date(timestamp).toLocaleTimeString('zh-CN', { hour12: false })
}

function getNodeDiff(nodeId: string): NodeDataDiff | undefined {
  if (!store.dataDiff) return undefined
  return store.dataDiff.changedNodes.find(d => d.nodeId === nodeId)
}

function hasNodeDiff(nodeId: string): boolean {
  return !!getNodeDiff(nodeId)
}

function getDiffTagType(nodeId: string): 'success' | 'warning' | 'danger' | 'info' {
  const diff = getNodeDiff(nodeId)
  if (!diff) return 'info'
  if (diff.qualityChanged && diff.newQuality !== 'Good') return 'danger'
  if (diff.valueChanged) {
    if (typeof diff.oldValue === 'number' && typeof diff.newValue === 'number') {
      const pct = diff.oldValue !== 0 ? Math.abs((diff.newValue - diff.oldValue) / Math.abs(diff.oldValue)) : 0
      if (pct > 0.1) return 'warning'
    }
    return 'success'
  }
  return 'info'
}

function getDiffLabel(nodeId: string): string {
  const diff = getNodeDiff(nodeId)
  if (!diff) return ''
  if (typeof diff.oldValue === 'number' && typeof diff.newValue === 'number') {
    const delta = diff.newValue - diff.oldValue
    const pct = diff.oldValue !== 0 ? ((delta / Math.abs(diff.oldValue)) * 100).toFixed(1) : '0.0'
    return delta > 0 ? `↑ +${pct}%` : `↓ ${pct}%`
  }
  if (diff.valueChanged) return '已变更'
  if (diff.qualityChanged) return `质量: ${diff.newQuality}`
  return '变化'
}

// 获取节点当前值
function getNodeValue(nodeId: string): number | boolean {
  const data = store.realTimeData.get(nodeId)
  if (data) return data.value as number | boolean
  const node = findNodeById(nodeId)
  return node?.value ?? 0
}

function getNodeQuality(nodeId: string): string {
  const data = store.realTimeData.get(nodeId)
  if (data) return data.quality
  const node = findNodeById(nodeId)
  return node?.quality ?? 'Unknown'
}

function findNodeById(id: string) {
  function search(nodes: any[]): any {
    for (const node of nodes) {
      if (node.id === id) return node
      if (node.children) {
        const found = search(node.children)
        if (found) return found
      }
    }
    return null
  }
  return search(store.nodeTree)
}

function getQualityClass(nodeId: string): string {
  const quality = getNodeQuality(nodeId)
  return quality === 'Good' ? 'quality-good' : quality === 'Bad' ? 'quality-bad' : 'quality-uncertain'
}

// 实时数值
const temperature = computed(() => getNodeValue('temp_sensor') as number || 25)
const pressure = computed(() => getNodeValue('pressure_transmitter') as number || 3.5)
const flow = computed(() => getNodeValue('flow_meter') as number || 150)
const valvePosition = computed(() => getNodeValue('valve_position') as number || 75)
const motorSpeed = computed(() => getNodeValue('motor_speed') as number || 1480)
const pumpStatus = computed(() => getNodeValue('pump_status') as boolean)

// 温度和颜色判断
function getTempClass(val: number) {
  if (val > 30) return 'text-red-400'
  if (val > 28) return 'text-yellow-400'
  return 'text-green-400'
}

function getTempColor(val: number) {
  if (val > 30) return '#f56c6c'
  if (val > 28) return '#e6a23c'
  return '#67c23a'
}

function getPressureClass(val: number) {
  if (val > 4.5) return 'text-red-400'
  if (val > 4.0) return 'text-yellow-400'
  return 'text-cyan-400'
}

function getPressureColor(val: number) {
  if (val > 4.5) return '#f56c6c'
  if (val > 4.0) return '#e6a23c'
  return '#06b6d4'
}

function getSpeedClass(val: number) {
  if (val > 1600) return 'text-red-400'
  if (val > 1550) return 'text-yellow-400'
  return 'text-emerald-400'
}

function getSpeedColor(val: number) {
  if (val > 1600) return '#f56c6c'
  if (val > 1550) return '#e6a23c'
  return '#34d399'
}

// 构建趋势图
function buildChartOption(title: string, nodeId: string, color: string, unit: string) {
  const history = store.dataHistory.get(nodeId) || []
  const data = history.map(h => [h.timestamp, h.value])

  return {
    title: { text: title, textStyle: { color: '#e0e0e0', fontSize: 14 }, left: 'center' },
    tooltip: { trigger: 'axis' as const },
    grid: { left: 60, right: 20, top: 40, bottom: 30 },
    xAxis: {
      type: 'time' as const,
      axisLabel: { color: '#999', formatter: '{HH}:{mm}:{ss}' },
      axisLine: { lineStyle: { color: '#444' } }
    },
    yAxis: {
      type: 'value' as const,
      axisLabel: { color: '#999', formatter: `{value} ${unit}` },
      splitLine: { lineStyle: { color: '#333' } }
    },
    series: [{
      type: 'line',
      data,
      smooth: true,
      lineStyle: { color, width: 2 },
      areaStyle: { color: { type: 'linear', x: 0, y: 0, x2: 0, y2: 1, colorStops: [{ offset: 0, color: color + '40' }, { offset: 1, color: color + '05' }] } },
      symbol: 'none'
    }]
  }
}

const tempChartOption = computed(() => buildChartOption('温度趋势', 'temp_sensor', '#67c23a', '°C'))
const pressureChartOption = computed(() => buildChartOption('压力趋势', 'pressure_transmitter', '#06b6d4', 'MPa'))
const flowChartOption = computed(() => buildChartOption('流量趋势', 'flow_meter', '#60a5fa', 'L/min'))
</script>

<style scoped>
.data-dashboard {
  height: 100%;
  overflow-y: auto;
  padding: 12px;
  position: relative;
}

.data-dashboard.frozen-mode {
  filter: saturate(0.7);
}

.frozen-overlay {
  position: sticky;
  top: 0;
  z-index: 10;
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 16px;
  margin-bottom: 12px;
  background: linear-gradient(90deg, rgba(230, 162, 60, 0.15), rgba(230, 162, 60, 0.05));
  border: 1px solid rgba(230, 162, 60, 0.4);
  border-radius: 8px;
  backdrop-filter: blur(4px);
}

.frozen-text {
  color: #e6a23c;
  font-weight: 600;
  font-size: 15px;
}

.frozen-hint {
  color: #94a3b8;
  font-size: 12px;
  margin-left: auto;
}

.section-title {
  font-size: 16px;
  font-weight: bold;
  color: #22d3ee;
  margin-bottom: 12px;
  padding-left: 8px;
  border-left: 3px solid #06b6d4;
  display: flex;
  align-items: center;
}

.gauges-section {
  margin-bottom: 20px;
}

.gauges-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
}

.gauge-card {
  background: rgba(30, 41, 59, 0.8);
  border: 1px solid rgba(71, 85, 105, 0.5);
  border-radius: 8px;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 8px;
  position: relative;
  transition: all 0.3s ease;
}

.gauge-card.gauge-frozen {
  border-color: rgba(230, 162, 60, 0.5);
  background: rgba(30, 41, 59, 0.6);
}

.gauge-card.gauge-frozen::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  border-radius: 8px;
  background: repeating-linear-gradient(
    45deg,
    transparent,
    transparent 10px,
    rgba(230, 162, 60, 0.03) 10px,
    rgba(230, 162, 60, 0.03) 20px
  );
  pointer-events: none;
}

.gauge-card.gauge-changed {
  border-color: rgba(230, 162, 60, 0.6);
  box-shadow: 0 0 12px rgba(230, 162, 60, 0.15);
  animation: changed-pulse 2s ease-in-out;
}

@keyframes changed-pulse {
  0%, 100% { box-shadow: 0 0 12px rgba(230, 162, 60, 0.15); }
  50% { box-shadow: 0 0 20px rgba(230, 162, 60, 0.3); }
}

.gauge-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.gauge-label {
  font-size: 13px;
  color: #94a3b8;
  font-weight: 500;
}

.gauge-value {
  font-size: 28px;
  font-weight: bold;
  font-family: 'Courier New', monospace;
}

.gauge-unit {
  font-size: 14px;
  color: #64748b;
  font-weight: normal;
}

.gauge-quality {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  color: #64748b;
}

.quality-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  display: inline-block;
}

.quality-good { background: #67c23a; }
.quality-bad { background: #f56c6c; }
.quality-uncertain { background: #e6a23c; }

.pump-indicator {
  display: flex;
  justify-content: center;
  padding: 8px 0;
}

.pump-on { color: #67c23a; }
.pump-off { color: #f56c6c; }

.charts-section {
  margin-top: 16px;
}

.charts-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
}

.chart-card {
  background: rgba(30, 41, 59, 0.8);
  border: 1px solid rgba(71, 85, 105, 0.5);
  border-radius: 8px;
  padding: 12px;
}

.chart {
  height: 220px;
  width: 100%;
}

@media (max-width: 1200px) {
  .gauges-grid { grid-template-columns: repeat(2, 1fr); }
  .charts-grid { grid-template-columns: 1fr; }
}
</style>
