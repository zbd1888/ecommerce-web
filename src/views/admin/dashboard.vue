<template>
  <div class="admin-dashboard">
    <h2 class="page-title">数据质量监控大屏</h2>

    <!-- 核心指标卡片 -->
    <el-row :gutter="16" class="stat-cards">
      <el-col :span="6">
        <el-card shadow="hover" class="stat-card">
          <div class="stat-icon" style="background: linear-gradient(135deg, #409EFF, #66b1ff)"><el-icon><Document /></el-icon></div>
          <div class="stat-info">
            <div class="stat-value">{{ stats.totalProducts?.toLocaleString() }}</div>
            <div class="stat-label">商品总数</div>
          </div>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card shadow="hover" class="stat-card">
          <div class="stat-icon" style="background: linear-gradient(135deg, #67C23A, #85ce61)"><el-icon><FolderOpened /></el-icon></div>
          <div class="stat-info">
            <div class="stat-value">{{ stats.totalCategories }}</div>
            <div class="stat-label">品类数量</div>
          </div>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card shadow="hover" class="stat-card">
          <div class="stat-icon" style="background: linear-gradient(135deg, #E6A23C, #f0c78a)"><el-icon><Files /></el-icon></div>
          <div class="stat-info">
            <div class="stat-value">{{ stats.pendingFiles }}</div>
            <div class="stat-label">待清洗文件</div>
          </div>
        </el-card>
      </el-col>
      <el-col :span="6">
        <el-card shadow="hover" class="stat-card">
          <div class="stat-icon" style="background: linear-gradient(135deg, #F56C6C, #f89898)"><el-icon><User /></el-icon></div>
          <div class="stat-info">
            <div class="stat-value">{{ stats.totalUsers }}</div>
            <div class="stat-label">用户数量</div>
          </div>
        </el-card>
      </el-col>
    </el-row>

    <!-- 上下结构：最近清洗日志（上）+ 数据质量指标（下） -->
    <div class="content-stack">
      <!-- 最近清洗日志（上）：可折叠下拉滚动区域 -->
      <el-card class="log-card">
        <template #header><span class="card-title">最近清洗日志</span></template>
        <el-collapse v-model="logCollapseActive">
          <el-collapse-item name="log">
            <template #title>
              <span class="collapse-title">点击展开/收起查看清洗日志</span>
            </template>
            <div class="log-scroll-wrap">
              <el-table :data="cleanLogs" size="default" max-height="360" stripe>
            <el-table-column prop="fileName" label="文件名" show-overflow-tooltip min-width="280" />
            <el-table-column prop="insertedCount" label="新增" width="90" align="center">
              <template #default="{ row }"><span class="count-success">+{{ row.insertedCount }}</span></template>
            </el-table-column>
            <el-table-column prop="skippedCount" label="跳过" width="90" align="center">
              <template #default="{ row }"><span class="count-skip">{{ row.skippedCount }}</span></template>
            </el-table-column>
            <el-table-column prop="status" label="状态" width="100" align="center">
              <template #default="{ row }">
                <el-tag :type="row.status === 'SUCCESS' ? 'success' : 'danger'" effect="dark">
                  {{ row.status === 'SUCCESS' ? '成功' : '失败' }}
                </el-tag>
              </template>
            </el-table-column>
            <el-table-column prop="createdAt" label="处理时间" width="180" />
          </el-table>
            </div>
          </el-collapse-item>
        </el-collapse>
      </el-card>

      <!-- 数据质量指标（下） -->
      <el-card class="quality-card">
          <template #header><span class="card-title">数据质量指标</span></template>
          <div class="quality-item">
            <div class="quality-header">
              <span class="quality-name">数据完整性</span>
              <span class="quality-percent" :style="{ color: getColor(quality.completeness) }">{{ quality.completeness }}%</span>
            </div>
            <el-progress :percentage="quality.completeness" :color="getColor(quality.completeness)" :stroke-width="18" :show-text="false" />
          </div>
          <div class="quality-item">
            <div class="quality-header">
              <span class="quality-name">价格有效率</span>
              <span class="quality-percent" :style="{ color: getColor(quality.priceValid) }">{{ quality.priceValid }}%</span>
            </div>
            <el-progress :percentage="quality.priceValid" :color="getColor(quality.priceValid)" :stroke-width="18" :show-text="false" />
          </div>
          <div class="quality-item">
            <div class="quality-header">
              <span class="quality-name">销量有效率</span>
              <span class="quality-percent" :style="{ color: getColor(quality.saleValid) }">{{ quality.saleValid }}%</span>
            </div>
            <el-progress :percentage="quality.saleValid" :color="getColor(quality.saleValid)" :stroke-width="18" :show-text="false" />
          </div>
          <div class="quality-item">
            <div class="quality-header">
              <span class="quality-name">图片有效率</span>
              <span class="quality-percent" :style="{ color: getColor(quality.imageValid) }">{{ quality.imageValid }}%</span>
            </div>
            <el-progress :percentage="quality.imageValid" :color="getColor(quality.imageValid)" :stroke-width="18" :show-text="false" />
          </div>
          <div class="quality-summary">
            <div class="summary-item">
              <span class="summary-label">综合评分</span>
              <span class="summary-value" :style="{ color: getColor(avgQuality) }">{{ avgQuality }}%</span>
            </div>
          </div>
        </el-card>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import request from '@/api/request'

const stats = ref({ totalProducts: 0, totalCategories: 0, pendingFiles: 0, totalUsers: 0 })
const cleanLogs = ref([])
const quality = ref({ completeness: 0, priceValid: 0, saleValid: 0, imageValid: 0 })
const logCollapseActive = ref(['log'])

const getColor = (val) => val >= 90 ? '#67C23A' : val >= 70 ? '#E6A23C' : '#F56C6C'
const avgQuality = computed(() => Math.round((quality.value.completeness + quality.value.priceValid + quality.value.saleValid + quality.value.imageValid) / 4))

const fetchData = async () => {
  try {
    const [statsRes, logsRes, qualityRes] = await Promise.all([
      request.get('/admin/stats'),
      request.get('/admin/clean-logs', { params: { page: 1, size: 50 } }),
      request.get('/admin/quality')
    ])
    stats.value = statsRes.data || stats.value
    cleanLogs.value = logsRes.data?.records || []
    quality.value = qualityRes.data || quality.value
  } catch (e) { console.error('加载数据失败', e) }
}

onMounted(fetchData)
</script>

<style scoped>
.admin-dashboard { padding: 12px 14px; background: #f5f7fa; min-height: calc(100vh - 60px); display: flex; flex-direction: column; }
.page-title { margin: 0 0 10px; color: #303133; font-size: 26px; font-weight: 700; }
.stat-cards { margin-bottom: 12px; }
.stat-card { display: flex; align-items: center; padding: 20px; border-radius: 8px; }
.stat-card:hover { transform: translateY(-2px); box-shadow: 0 4px 12px rgba(0,0,0,0.1); transition: all 0.3s; }
.stat-icon { width: 72px; height: 72px; border-radius: 12px; display: flex; align-items: center; justify-content: center; margin-right: 16px; box-shadow: 0 4px 8px rgba(0,0,0,0.15); }
.stat-icon .el-icon { font-size: 36px; color: #fff; }
.stat-value { font-size: 38px; font-weight: 700; color: #303133; line-height: 1.2; }
.stat-label { color: #606266; font-size: 17px; margin-top: 4px; }
.content-stack { flex: 1; display: flex; flex-direction: column; gap: 12px; margin-bottom: 0; min-height: 0; }
.card-title { font-size: 20px; font-weight: 600; color: #303133; }
.log-card { flex-shrink: 0; }
.log-card :deep(.el-card__body) { padding: 10px 12px; }
.log-card :deep(.el-collapse-item__header) { font-size: 17px; }
.collapse-title { font-size: 17px; color: #606266; }
.log-scroll-wrap { overflow: hidden; }
.log-card :deep(.el-table) { font-size: 18px; }
.log-card :deep(.el-table th) { font-size: 18px; font-weight: 600; }
.log-card :deep(.el-table td) { font-size: 18px; }
.count-success { color: #67C23A; font-weight: 600; font-size: 18px; }
.count-skip { color: #909399; font-size: 18px; }
.quality-card { flex: 1; min-height: 0; }
.quality-card :deep(.el-card__body) { padding: 16px 20px; }
.quality-item { margin-bottom: 20px; }
.quality-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px; }
.quality-name { font-size: 19px; color: #606266; font-weight: 500; }
.quality-percent { font-size: 26px; font-weight: 700; }
.quality-summary { margin-top: 18px; padding-top: 16px; border-top: 1px solid #ebeef5; }
.summary-item { display: flex; justify-content: space-between; align-items: center; }
.summary-label { font-size: 21px; color: #303133; font-weight: 600; }
.summary-value { font-size: 40px; font-weight: 700; }
</style>

