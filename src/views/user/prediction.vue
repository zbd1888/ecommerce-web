<template>
  <div class="prediction-page">
    <!-- 综合预测输入 -->
    <el-card>
      <template #header><span>综合商品预测</span></template>
      <el-form :model="form" label-width="100px" :inline="false">
        <el-row :gutter="20">
          <el-col :span="8">
            <el-form-item label="商品品类" required>
              <el-select v-model="form.keyword" placeholder="请选择品类" filterable style="width: 100%">
                <el-option v-for="k in keywords" :key="k" :label="k" :value="k" />
              </el-select>
            </el-form-item>
          </el-col>
          <el-col :span="8">
            <el-form-item label="商品价格" required>
              <el-input-number v-model="form.price" :min="1" :max="99999" :precision="2" style="width: 100%" />
            </el-form-item>
          </el-col>
          <el-col :span="8">
            <el-form-item label="发货地">
              <el-input v-model="form.location" placeholder="如：广东 深圳" />
            </el-form-item>
          </el-col>
        </el-row>
        <el-row :gutter="20">
          <el-col :span="8">
            <el-form-item label="店铺标签">
              <el-select v-model="form.shopTag" placeholder="选择店铺类型" clearable style="width: 100%">
                <el-option label="品牌旗舰店" value="品牌旗舰店" />
                <el-option label="官方旗舰店" value="官方旗舰店" />
                <el-option label="品牌专卖店" value="品牌专卖店" />
                <el-option label="企业店铺" value="企业店铺" />
                <el-option label="个人店铺" value="个人店铺" />
                <el-option label="无标签" value="" />
              </el-select>
            </el-form-item>
          </el-col>
          <el-col :span="16">
            <el-form-item label="商品标题">
              <el-input v-model="form.title" placeholder="输入商品标题（可选）" />
            </el-form-item>
          </el-col>
        </el-row>
        <el-form-item>
          <el-button type="primary" size="large" @click="predict" :loading="loading">
            <el-icon><Search /></el-icon> 开始综合预测
          </el-button>
        </el-form-item>
      </el-form>
    </el-card>

    <!-- 预测结果 -->
    <template v-if="result.success">
      <el-row :gutter="20" style="margin-top: 20px">
        <!-- 综合评分 -->
        <el-col :span="8">
          <el-card class="score-card">
            <div class="score-circle" :style="{ borderColor: getScoreColor(result.successScore) }">
              <div class="score-value">{{ result.successScore }}</div>
              <div class="score-label">成功率评分</div>
            </div>
            <div class="recommendation">{{ result.overallRecommendation }}</div>
          </el-card>
        </el-col>

        <!-- 各维度评分 -->
        <el-col :span="16">
          <el-card>
            <template #header><span>各维度评分</span></template>
            <el-row :gutter="10">
              <el-col :span="6">
                <div class="dim-score">
                  <el-progress type="circle" :percentage="result.priceScore" :color="getScoreColor(result.priceScore)" :width="100" />
                  <div class="dim-label">价格竞争力</div>
                  <div class="dim-tip">{{ result.priceSuggestion }}</div>
                </div>
              </el-col>
              <el-col :span="6">
                <div class="dim-score">
                  <el-progress type="circle" :percentage="result.tagScore" :color="getScoreColor(result.tagScore)" :width="100" />
                  <div class="dim-label">店铺标签</div>
                  <div class="dim-tip">{{ result.tagSuggestion }}</div>
                </div>
              </el-col>
              <el-col :span="6">
                <div class="dim-score">
                  <el-progress type="circle" :percentage="result.locationScore" :color="getScoreColor(result.locationScore)" :width="100" />
                  <div class="dim-label">发货地优势</div>
                  <div class="dim-tip">{{ result.locationSuggestion }}</div>
                </div>
              </el-col>
              <el-col :span="6">
                <div class="dim-score">
                  <el-progress type="circle" :percentage="result.hotScore" :color="getScoreColor(result.hotScore)" :width="100" />
                  <div class="dim-label">品类热度</div>
                  <div class="dim-tip">样本数: {{ result.sampleSize }}</div>
                </div>
              </el-col>
            </el-row>
          </el-card>
        </el-col>
      </el-row>

      <el-row :gutter="20" style="margin-top: 20px">
        <!-- 销量预测 -->
        <el-col :span="12">
          <el-card>
            <template #header>
              <div class="card-header-flex">
                <span>销量预测</span>
                <el-tag v-if="result.predictionConfidence" :type="result.predictionConfidence >= 70 ? 'success' : 'warning'" size="small">
                  置信度: {{ result.predictionConfidence }}%
                </el-tag>
              </div>
            </template>
            <div class="sales-prediction-main">
              <div class="sales-value-box">
                <div class="sales-label">预测销量</div>
                <div class="sales-value">{{ formatSales(result.predictedSalesMid) }}</div>
                <div class="sales-range">区间: {{ formatSales(result.predictedSalesLow) }} ~ {{ formatSales(result.predictedSalesHigh) }}</div>
              </div>
            </div>
            <el-descriptions :column="2" border size="small" style="margin-top: 15px">
              <el-descriptions-item label="品类均价">¥{{ result.categoryAvgPrice }}</el-descriptions-item>
              <el-descriptions-item label="最优价格区间">{{ result.bestPriceRange || '-' }}</el-descriptions-item>
              <el-descriptions-item label="热门发货地">{{ result.topLocation }}</el-descriptions-item>
              <el-descriptions-item label="推荐发货地">{{ result.bestSalesLocation || '-' }}</el-descriptions-item>
              <el-descriptions-item label="相似样本数">{{ result.similarSampleCount || result.sampleSize }}</el-descriptions-item>
              <el-descriptions-item label="竞品数量">
                <el-tag :type="result.competitionLevel === '激烈' ? 'danger' : result.competitionLevel === '中等' ? 'warning' : 'success'" size="small">
                  {{ result.competitorCount || 0 }} ({{ result.competitionLevel || '未知' }})
                </el-tag>
              </el-descriptions-item>
            </el-descriptions>
          </el-card>
        </el-col>

        <!-- 优化建议 -->
        <el-col :span="12">
          <el-card>
            <template #header><span>优化建议</span></template>
            <div class="suggestion-list">
              <div v-for="(s, i) in result.suggestions" :key="i" class="suggestion-item" :class="'suggestion-' + s.status">
                <div class="suggestion-header">
                  <el-icon v-if="s.status === 'good'" color="#67C23A"><CircleCheckFilled /></el-icon>
                  <el-icon v-else-if="s.status === 'normal'" color="#E6A23C"><WarningFilled /></el-icon>
                  <el-icon v-else color="#F56C6C"><CircleCloseFilled /></el-icon>
                  <span class="suggestion-text">{{ s.text }}</span>
                  <el-tag :type="s.status === 'good' ? 'success' : s.status === 'normal' ? 'warning' : 'danger'" size="small" style="margin-left: auto">
                    {{ s.score }}分
                  </el-tag>
                </div>
                <div class="suggestion-detail">{{ s.detail }}</div>
              </div>
            </div>
          </el-card>
        </el-col>
      </el-row>

      <!-- 标题关键词建议 -->
      <el-row :gutter="20" style="margin-top: 20px" v-if="result.titleKeywords && result.titleKeywords.length > 0">
        <el-col :span="24">
          <el-card>
            <template #header>
              <div class="card-header-flex">
                <span>标题关键词建议</span>
                <el-tag type="info" size="small">基于爆款商品标题分析</el-tag>
              </div>
            </template>
            <div class="title-keyword-tip" v-if="result.titleKeywordSuggestion">
              <el-alert :title="result.titleKeywordSuggestion" type="success" :closable="false" show-icon />
            </div>
            <div class="keyword-tags" style="margin-top: 12px">
              <el-tag
                v-for="(kw, i) in result.titleKeywords"
                :key="i"
                :type="i < 3 ? '' : 'info'"
                :effect="i < 3 ? 'dark' : 'light'"
                size="large"
                class="keyword-tag"
              >
                {{ kw.word }}
                <span class="keyword-count">{{ kw.count }}次</span>
              </el-tag>
            </div>
          </el-card>
        </el-col>
      </el-row>

    </template>

    <!-- 报告下载 -->
    <el-card style="margin-top: 20px">
      <template #header><span>报告下载</span></template>
      <el-space>
        <el-button type="primary" @click="downloadProducts">下载商品数据 (Excel)</el-button>
        <el-button type="success" @click="downloadMarketAnalysis">下载市场分析报告 (Excel)</el-button>
        <el-button type="warning" @click="refreshCache" :loading="refreshing">刷新分析缓存</el-button>
      </el-space>
    </el-card>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { Search, CircleCheckFilled, WarningFilled, CircleCloseFilled } from '@element-plus/icons-vue'
import request from '@/api/request'
import { ElMessage } from 'element-plus'
import { useCacheStore } from '@/store/cache'

const cacheStore = useCacheStore()
const keywords = computed(() => cacheStore.keywords)
const form = ref({ keyword: '', price: 100, location: '', shopTag: '', title: '' })
const result = ref({})
const loading = ref(false)
const refreshing = ref(false)

const formatSales = (val) => {
  if (val == null) return '-'
  return val >= 10000 ? (val / 10000).toFixed(1) + '万' : val.toLocaleString()
}
const getScoreColor = (score) => score >= 70 ? '#67C23A' : score >= 50 ? '#E6A23C' : '#F56C6C'

const fetchKeywords = async () => {
  await cacheStore.fetchKeywords()
}

const predict = async () => {
  if (!form.value.keyword) return ElMessage.warning('请选择品类')
  if (!form.value.price || form.value.price <= 0) return ElMessage.warning('请输入有效价格')
  loading.value = true
  try {
    const res = await request.get('/analysis/predict/comprehensive', {
      params: {
        keyword: form.value.keyword,
        price: form.value.price,
        location: form.value.location || '',
        shopTag: form.value.shopTag || '',
        title: form.value.title || ''
      }
    })
    result.value = res.data || {}
    if (!result.value.success) {
      ElMessage.warning(result.value.message || '预测失败')
    }
  } catch (e) {
    ElMessage.error('预测请求失败')
  }
  loading.value = false
}

const downloadProducts = () => { window.open('/api/report/export/products', '_blank') }
const downloadMarketAnalysis = () => { window.open('/api/report/export/market-analysis', '_blank') }

const refreshCache = async () => {
  refreshing.value = true
  await request.post('/report/refresh-cache')
  ElMessage.success('缓存刷新成功')
  refreshing.value = false
}

onMounted(fetchKeywords)
</script>

<style scoped>
.prediction-page { padding: 12px 14px; background: #f5f7fa; min-height: calc(100vh - 60px); }
.highlight { color: #E6A23C; font-weight: bold; font-size: 22px; }

.prediction-page :deep(.el-card__header span) { font-size: 22px; font-weight: 600; }
.prediction-page :deep(.el-form-item__label) { font-size: 19px; }
.prediction-page :deep(.el-input__inner),
.prediction-page :deep(.el-input-number .el-input__inner) { font-size: 17px; }
.prediction-page :deep(.el-descriptions__label) { font-size: 17px; }
.prediction-page :deep(.el-descriptions__content) { font-size: 17px; }
.prediction-page :deep(.el-button) { font-size: 17px; }

.score-card { text-align: center; padding: 20px 0; }
.score-circle {
  width: 160px; height: 160px; border-radius: 50%;
  border: 6px solid #67C23A; margin: 0 auto 15px;
  display: flex; flex-direction: column; justify-content: center; align-items: center;
}
.score-value { font-size: 52px; font-weight: bold; color: #303133; }
.score-label { font-size: 19px; color: #909399; }
.recommendation { font-size: 20px; color: #606266; margin-top: 10px; }

.dim-score { text-align: center; padding: 10px 0; }
.dim-label { margin-top: 10px; font-weight: bold; color: #303133; font-size: 19px; }
.dim-tip { font-size: 17px; color: #909399; margin-top: 5px; min-height: 40px; }

.card-header-flex { display: flex; justify-content: space-between; align-items: center; }

.sales-prediction-main { text-align: center; padding: 15px 0; background: #f5f7fa; border-radius: 8px; }
.sales-value-box .sales-label { font-size: 19px; color: #909399; }
.sales-value-box .sales-value { font-size: 48px; font-weight: bold; color: #409EFF; margin: 8px 0; }
.sales-value-box .sales-range { font-size: 18px; color: #909399; }

.recommendation-box { margin-top: 15px; padding: 10px; background: #f5f7fa; border-radius: 6px; }
.rec-item { font-size: 15px; color: #606266; }
.rec-label { color: #909399; }

.suggestion-list { display: flex; flex-direction: column; gap: 10px; }
.suggestion-item { padding: 14px 16px; border-radius: 6px; border-left: 3px solid #dcdfe6; background: #fafafa; }
.suggestion-good { border-left-color: #67C23A; background: #f0f9eb; }
.suggestion-normal { border-left-color: #E6A23C; background: #fdf6ec; }
.suggestion-warning { border-left-color: #F56C6C; background: #fef0f0; }
.suggestion-header { display: flex; align-items: center; gap: 6px; }
.suggestion-text { font-size: 19px; font-weight: 500; color: #303133; }
.suggestion-detail { font-size: 17px; color: #909399; margin-top: 4px; padding-left: 22px; }

.keyword-tags { display: flex; flex-wrap: wrap; gap: 10px; }
.keyword-tag { cursor: default; font-size: 17px; }
.keyword-count { font-size: 15px; margin-left: 4px; opacity: 0.7; }
.title-keyword-tip { margin-bottom: 8px; }
</style>

