<template>
  <div class="app-container">
    <!-- 顶部导航栏 -->
    <header class="app-header">
      <div class="header-left">
        <el-icon :size="24" class="text-cyan-400"><Monitor /></el-icon>
        <h1 class="app-title">OPC-UA 工业节点浏览与数据采集</h1>
      </div>
      <div class="header-right">
        <el-badge :value="store.activeAlarmsCount" :max="99" class="alarm-badge">
          <el-icon :size="20" class="text-yellow-400"><Bell /></el-icon>
        </el-badge>
        <el-tag type="success" v-if="store.isConnected" class="status-tag">
          <el-icon><CircleCheck /></el-icon>
          在线
        </el-tag>
        <el-tag type="danger" v-else class="status-tag">
          <el-icon><CircleClose /></el-icon>
          离线
        </el-tag>
        <el-tag type="warning" v-if="store.isFrozen" class="status-tag frozen-tag" effect="dark">
          <el-icon><Snowflake /></el-icon>
          数据已冻结
        </el-tag>
        <el-button
          v-if="store.dataDiff && !store.showDiffDialog"
          type="warning"
          size="small"
          plain
          @click="store.showDiffDialog = true"
        >
          <el-icon><DataAnalysis /></el-icon>
          查看差异 ({{ store.dataDiff.changedCount }})
        </el-button>
        <el-button
          :type="store.isConnected ? 'danger' : 'success'"
          size="small"
          @click="toggleConnection"
        >
          {{ store.isConnected ? '断开' : '连接' }}
        </el-button>
      </div>
    </header>

    <!-- 主内容区域 -->
    <div class="app-main">
      <!-- 左侧面板: 节点树 -->
      <aside class="left-panel">
        <NodeTree />
      </aside>

      <!-- 中央区域: 仪表盘 -->
      <main class="center-panel">
        <DataDashboard />
      </main>

      <!-- 右侧面板: 报警列表 -->
      <aside class="right-panel">
        <div class="alarm-panel">
          <div class="alarm-header">
            <h3 class="text-lg font-bold text-yellow-400">报警事件</h3>
            <el-button type="danger" size="small" text @click="store.clearAlarms()" v-if="store.alarms.length > 0">
              清空
            </el-button>
          </div>

          <div class="alarm-stats">
            <el-tag type="danger" size="small">严重: {{ criticalCount }}</el-tag>
            <el-tag type="warning" size="small">活跃: {{ store.activeAlarmsCount }}</el-tag>
            <el-tag type="info" size="small">总计: {{ store.alarms.length }}</el-tag>
          </div>

          <div class="alarm-list">
            <div
              v-for="alarm in store.alarms"
              :key="alarm.id"
              class="alarm-item"
              :class="{
                'alarm-critical': alarm.severity === 'Critical',
                'alarm-high': alarm.severity === 'High',
                'alarm-medium': alarm.severity === 'Medium',
                'alarm-acknowledged': alarm.acknowledged
              }"
            >
              <div class="alarm-item-header">
                <el-tag
                  :type="getSeverityType(alarm.severity)"
                  size="small"
                  effect="dark"
                >
                  {{ getSeverityLabel(alarm.severity) }}
                </el-tag>
                <span class="alarm-time">{{ formatTime(alarm.timestamp) }}</span>
              </div>
              <div class="alarm-item-body">
                <span class="alarm-node">{{ alarm.nodeName }}</span>
                <p class="alarm-message">{{ alarm.message }}</p>
              </div>
              <el-button
                v-if="!alarm.acknowledged"
                type="primary"
                size="small"
                text
                @click="store.acknowledgeAlarm(alarm.id)"
              >
                确认
              </el-button>
            </div>

            <div v-if="store.alarms.length === 0" class="no-alarms">
              <el-empty description="暂无报警" :image-size="60" />
            </div>
          </div>
        </div>
      </aside>
    </div>

    <!-- 数据差异对话框 -->
    <el-dialog
      v-model="store.showDiffDialog"
      title="恢复连接 - 数据差异提示"
      width="640px"
      class="diff-dialog"
      :close-on-click-modal="false"
      @close="store.clearDataDiff"
    >
      <template v-if="store.dataDiff">
        <div class="diff-summary">
          <el-alert
            :title="`连接已恢复，断开时长: ${formatDuration(store.dataDiff.duration)}`"
            type="warning"
            :closable="false"
            show-icon
          />
          <div class="diff-stats">
            <el-statistic title="监控节点总数" :value="store.dataDiff.totalNodes" />
            <el-statistic title="发生变化节点" :value="store.dataDiff.changedCount" />
            <el-statistic title="未变化节点" :value="store.dataDiff.totalNodes - store.dataDiff.changedCount" />
          </div>
        </div>

        <el-divider />

        <div v-if="store.dataDiff.changedCount === 0" class="no-diff">
          <el-empty description="所有节点数据无变化" :image-size="80" />
        </div>

        <div v-else class="diff-list">
          <div
            v-for="diff in store.dataDiff.changedNodes"
            :key="diff.nodeId"
            class="diff-item"
          >
            <div class="diff-item-header">
              <span class="diff-node-name">{{ diff.nodeName }}</span>
              <el-tag size="small" type="info">{{ diff.nodeId }}</el-tag>
            </div>
            <div class="diff-item-body">
              <div v-if="diff.valueChanged" class="diff-row">
                <span class="diff-label">数值:</span>
                <span class="diff-old">{{ formatValue(diff.oldValue, diff.unit) }}</span>
                <el-icon class="diff-arrow" color="#e6a23c"><Right /></el-icon>
                <span class="diff-new">{{ formatValue(diff.newValue, diff.unit) }}</span>
                <span
                  class="diff-change-badge"
                  :class="getChangeClass(diff.oldValue, diff.newValue)"
                >
                  {{ getChangeLabel(diff.oldValue, diff.newValue) }}
                </span>
              </div>
              <div v-if="diff.qualityChanged" class="diff-row">
                <span class="diff-label">质量:</span>
                <el-tag size="small" :type="getQualityType(diff.oldQuality)">{{ diff.oldQuality }}</el-tag>
                <el-icon class="diff-arrow" color="#e6a23c"><Right /></el-icon>
                <el-tag size="small" :type="getQualityType(diff.newQuality)">{{ diff.newQuality }}</el-tag>
              </div>
            </div>
          </div>
        </div>
      </template>

      <template #footer>
        <el-button type="primary" @click="store.clearDataDiff">
          确认并同步数据
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref } from 'vue'
import { Monitor, Bell, CircleCheck, CircleClose, Snowflake, DataAnalysis, Right } from '@element-plus/icons-vue'
import { ElMessage } from 'element-plus'
import { useOpcuaStore } from './store/opcua'
import NodeTree from './components/NodeTree.vue'
import DataDashboard from './components/DataDashboard.vue'
import type { AlarmEvent } from './types'

const store = useOpcuaStore()
const updateTimer = ref<number | null>(null)

const criticalCount = computed(() =>
  store.alarms.filter(a => a.severity === 'Critical' && !a.acknowledged).length
)

function toggleConnection() {
  if (store.isConnected) {
    store.disconnect()
    if (updateTimer.value) {
      clearInterval(updateTimer.value)
      updateTimer.value = null
    }
    ElMessage.warning('已断开 OPC-UA 连接')
  } else {
    store.connect()
    startSimulation()
    ElMessage.success('已连接 OPC-UA 服务器')
  }
}

function startSimulation() {
  updateTimer.value = window.setInterval(() => {
    store.simulateDataUpdate()
  }, 1000)
}

function getSeverityType(severity: AlarmEvent['severity']) {
  switch (severity) {
    case 'Critical': return 'danger'
    case 'High': return 'danger'
    case 'Medium': return 'warning'
    case 'Low': return 'info'
    case 'Info': return 'info'
  }
}

function getSeverityLabel(severity: AlarmEvent['severity']) {
  switch (severity) {
    case 'Critical': return '严重'
    case 'High': return '高'
    case 'Medium': return '中'
    case 'Low': return '低'
    case 'Info': return '信息'
  }
}

function formatTime(timestamp: number): string {
  return new Date(timestamp).toLocaleTimeString('zh-CN', { hour12: false })
}

function formatDuration(ms: number): string {
  const seconds = Math.floor(ms / 1000)
  const minutes = Math.floor(seconds / 60)
  const hours = Math.floor(minutes / 60)
  if (hours > 0) {
    return `${hours}小时${minutes % 60}分${seconds % 60}秒`
  }
  if (minutes > 0) {
    return `${minutes}分${seconds % 60}秒`
  }
  return `${seconds}秒`
}

function formatValue(value: any, unit?: string): string {
  if (typeof value === 'boolean') {
    return value ? '运行中' : '已停止'
  }
  if (typeof value === 'number') {
    const formatted = Number.isInteger(value) ? String(value) : value.toFixed(2)
    return unit ? `${formatted} ${unit}` : formatted
  }
  return String(value)
}

function getChangeClass(oldValue: any, newValue: any): string {
  if (typeof oldValue !== 'number' || typeof newValue !== 'number') return 'change-equal'
  const diff = newValue - oldValue
  if (diff > 0) return 'change-up'
  if (diff < 0) return 'change-down'
  return 'change-equal'
}

function getChangeLabel(oldValue: any, newValue: any): string {
  if (typeof oldValue !== 'number' || typeof newValue !== 'number') {
    return oldValue === newValue ? '不变' : '已变更'
  }
  const diff = newValue - oldValue
  const pct = oldValue !== 0 ? ((diff / Math.abs(oldValue)) * 100).toFixed(1) : '0.0'
  if (diff > 0) return `↑ +${diff.toFixed(2)} (${pct}%)`
  if (diff < 0) return `↓ ${diff.toFixed(2)} (${pct}%)`
  return '不变'
}

function getQualityType(quality: string): 'success' | 'danger' | 'warning' | 'info' {
  switch (quality) {
    case 'Good': return 'success'
    case 'Bad': return 'danger'
    case 'Uncertain': return 'warning'
    default: return 'info'
  }
}

onMounted(() => {
  store.connect()
  startSimulation()
})

onUnmounted(() => {
  if (updateTimer.value) {
    clearInterval(updateTimer.value)
  }
})
</script>

<style scoped>
.app-container {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: #0f172a;
  overflow: hidden;
}

.app-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 20px;
  height: 56px;
  background: rgba(15, 23, 42, 0.95);
  border-bottom: 1px solid rgba(71, 85, 105, 0.5);
  flex-shrink: 0;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 12px;
}

.app-title {
  font-size: 18px;
  font-weight: bold;
  background: linear-gradient(90deg, #06b6d4, #22d3ee);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.header-right {
  display: flex;
  align-items: center;
  gap: 16px;
}

.status-tag {
  display: flex;
  align-items: center;
  gap: 4px;
}

.app-main {
  flex: 1;
  display: flex;
  overflow: hidden;
}

.left-panel {
  width: 320px;
  background: rgba(30, 41, 59, 0.6);
  border-right: 1px solid rgba(71, 85, 105, 0.5);
  overflow-y: auto;
  flex-shrink: 0;
}

.center-panel {
  flex: 1;
  overflow-y: auto;
  background: #0f172a;
}

.right-panel {
  width: 340px;
  background: rgba(30, 41, 59, 0.6);
  border-left: 1px solid rgba(71, 85, 105, 0.5);
  overflow-y: auto;
  flex-shrink: 0;
}

.alarm-panel {
  padding: 12px;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.alarm-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
}

.alarm-stats {
  display: flex;
  gap: 8px;
  margin-bottom: 12px;
}

.alarm-list {
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.alarm-item {
  background: rgba(30, 41, 59, 0.8);
  border: 1px solid rgba(71, 85, 105, 0.5);
  border-radius: 6px;
  padding: 10px;
  border-left: 3px solid #64748b;
}

.alarm-critical {
  border-left-color: #ef4444;
  background: rgba(239, 68, 68, 0.08);
}

.alarm-high {
  border-left-color: #f97316;
  background: rgba(249, 115, 22, 0.05);
}

.alarm-medium {
  border-left-color: #eab308;
  background: rgba(234, 179, 8, 0.05);
}

.alarm-acknowledged {
  opacity: 0.5;
}

.alarm-item-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 6px;
}

.alarm-time {
  font-size: 11px;
  color: #64748b;
  font-family: monospace;
}

.alarm-node {
  font-size: 12px;
  color: #94a3b8;
  font-weight: 500;
}

.alarm-message {
  font-size: 13px;
  color: #cbd5e1;
  margin-top: 4px;
}

.no-alarms {
  display: flex;
  justify-content: center;
  padding-top: 40px;
}

.alarm-badge {
  cursor: pointer;
}

.frozen-tag {
  animation: frozen-pulse 2s ease-in-out infinite;
}

@keyframes frozen-pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.7; }
}

:deep(.diff-dialog) {
  --el-dialog-bg-color: #1e293b;
  --el-dialog-title-color: #e0e0e0;
}

.diff-summary {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.diff-stats {
  display: flex;
  justify-content: space-around;
  padding: 8px 0;
}

:deep(.diff-stats .el-statistic) {
  --el-statistic-content-color: #22d3ee;
  --el-statistic-head-color: #94a3b8;
  text-align: center;
}

.no-diff {
  padding: 24px 0;
}

.diff-list {
  max-height: 400px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.diff-item {
  background: rgba(30, 41, 59, 0.8);
  border: 1px solid rgba(71, 85, 105, 0.5);
  border-radius: 6px;
  padding: 10px 12px;
}

.diff-item-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 8px;
}

.diff-node-name {
  font-weight: 600;
  color: #e0e0e0;
  font-size: 14px;
}

.diff-item-body {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.diff-row {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  flex-wrap: wrap;
}

.diff-label {
  color: #94a3b8;
  min-width: 40px;
}

.diff-old {
  color: #f56c6c;
  font-family: monospace;
  text-decoration: line-through;
}

.diff-new {
  color: #67c23a;
  font-family: monospace;
  font-weight: 600;
}

.diff-arrow {
  flex-shrink: 0;
}

.diff-change-badge {
  font-size: 11px;
  padding: 2px 6px;
  border-radius: 4px;
  font-weight: 500;
}

.change-up {
  background: rgba(103, 194, 58, 0.15);
  color: #67c23a;
}

.change-down {
  background: rgba(245, 108, 108, 0.15);
  color: #f56c6c;
}

.change-equal {
  background: rgba(148, 163, 184, 0.15);
  color: #94a3b8;
}
</style>
