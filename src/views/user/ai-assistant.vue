<template>
  <div class="ai-assistant">
    <div class="chat-header">
      <div class="header-left">
        <el-icon :size="24" color="#409EFF"><ChatDotRound /></el-icon>
        <span>选品分析助手</span>
        <el-tag type="info" size="small">用户分析版</el-tag>
      </div>
      <el-button text type="primary" @click="startNewSession">新会话</el-button>
    </div>

    <div class="chat-body" ref="chatBodyRef">
      <div class="msg-row ai">
        <el-avatar :size="36" class="avatar ai-avatar"><el-icon><ChatDotRound /></el-icon></el-avatar>
        <div class="bubble ai-bubble">
          你好，我会基于当前系统里的真实商品数据，帮你做商品、品类、销量和价格分析。
          <br />
          <span class="hint">例如：当前商品总数是多少？销量最高的前 5 个商品是什么？我刚刚的问题是什么？</span>
        </div>
      </div>

      <div v-for="(msg, idx) in messages" :key="idx" :class="['msg-row', msg.role]">
        <el-avatar v-if="msg.role === 'ai'" :size="36" class="avatar ai-avatar">
          <el-icon><ChatDotRound /></el-icon>
        </el-avatar>

        <div :class="['bubble', msg.role === 'ai' ? 'ai-bubble' : 'user-bubble']">
          <div class="msg-content">{{ msg.content }}</div>
          <div v-if="msg.role === 'ai' && msg.summary" class="summary-block">
            <div class="block-title">一句话结论</div>
            <div>{{ msg.summary }}</div>
          </div>
          <div v-if="msg.role === 'ai' && msg.usedMemory && msg.resolvedQuestion && msg.resolvedQuestion !== msg.originalQuestion" class="memory-block">
            <div class="block-title">上下文补全</div>
            <div>已按上文理解为：{{ msg.resolvedQuestion }}</div>
          </div>
          <div v-if="msg.role === 'ai' && msg.businessHint" class="hint-block">
            <div class="block-title">业务解读</div>
            <div>{{ msg.businessHint }}</div>
          </div>
          <div v-if="msg.role === 'ai' && msg.suggestion" class="hint-block">
            <div class="block-title">建议</div>
            <div>{{ msg.suggestion }}</div>
          </div>
          <div v-if="msg.role === 'ai'" class="msg-meta">
            <el-tag size="small" :type="sourceType(msg.source)" effect="plain">
              {{ msg.source === 'knowledge' ? '知识检索' : sourceLabel(msg.source) }}
            </el-tag>
            <el-tag v-if="msg.route" size="small" effect="plain">{{ msg.route === 'knowledge' ? '知识路由' : routeLabel(msg.route) }}</el-tag>
            <el-tag v-if="msg.tablesQueried" size="small" effect="plain">表：{{ msg.tablesQueried }}</el-tag>
            <span class="query-time">{{ msg.queryTime }} 秒</span>
          </div>
        </div>

        <el-avatar v-if="msg.role === 'user'" :size="36" class="avatar user-avatar">
          <el-icon><User /></el-icon>
        </el-avatar>
      </div>

      <div v-if="loading" class="msg-row ai">
        <el-avatar :size="36" class="avatar ai-avatar"><el-icon><ChatDotRound /></el-icon></el-avatar>
        <div class="bubble ai-bubble loading-bubble">
          <span class="dot"></span><span class="dot"></span><span class="dot"></span>
        </div>
      </div>
    </div>

    <div class="chat-footer">
        <el-input
          v-model="inputText"
          placeholder="输入问题，例如：某个品类的平均价格是多少？这个品类值得做吗？"
          :disabled="loading"
          @keyup.enter="sendMessage"
          clearable
      />
      <el-button type="primary" :loading="loading" @click="sendMessage" :disabled="!inputText.trim()">
        发送
      </el-button>
    </div>
  </div>
</template>

<script setup>
import { computed, nextTick, ref } from 'vue'
import { askAI } from '@/api/index.js'
import { useCacheStore } from '@/store/cache.js'

const messages = ref([])
const inputText = ref('')
const loading = ref(false)
const chatBodyRef = ref(null)
const sessionId = ref('')

const cacheStore = useCacheStore()
const contextData = computed(() => {
  const dashboard = cacheStore.dashboardData
  const overview = dashboard?.stats
  if (!overview) return null
  return {
    module: 'user_dashboard',
    pageTitle: '用户总览',
    totalProducts: overview.totalProducts,
    totalCategories: overview.totalCategories,
    totalSales: overview.totalSales,
    avgPrice: overview.avgPrice,
    topProducts: dashboard?.topProducts || [],
    storeRanking: dashboard?.storeRanking || [],
    rankHotlist: dashboard?.rankHotlist || []
  }
})

const scrollToBottom = async () => {
  await nextTick()
  if (chatBodyRef.value) {
    chatBodyRef.value.scrollTop = chatBodyRef.value.scrollHeight
  }
}

const sourceLabel = (source) => ({
  context: '页面上下文',
  template: '模板查询',
  memory: '会话记忆',
  agent: '长尾查询',
  guardrail: '受控回答',
  error: '异常'
}[source] || '数据库查询')

const sourceType = (source) => ({
  context: 'success',
  template: 'primary',
  knowledge: 'info',
  agent: 'warning',
  guardrail: 'danger',
  error: 'danger'
}[source] || 'info')

const routeLabel = (route) => ({
  context: '上下文路由',
  template: '模板路由',
  memory: '记忆路由',
  agent: '长尾兜底',
  guardrail: '精确模式',
  agent_unavailable: 'Agent 未启用'
}[route] || route)

const createSessionId = () => {
  if (window.crypto?.randomUUID) {
    return window.crypto.randomUUID()
  }
  return 'user-ai-' + Date.now() + '-' + Math.random().toString(16).slice(2)
}

const ensureSession = () => {
  const existing = sessionStorage.getItem('user_ai_session_id')
  if (existing) {
    sessionId.value = existing
    return existing
  }
  const created = createSessionId()
  sessionStorage.setItem('user_ai_session_id', created)
  sessionId.value = created
  return created
}

const startNewSession = () => {
  messages.value = []
  const created = createSessionId()
  sessionStorage.setItem('user_ai_session_id', created)
  sessionId.value = created
}

const sendMessage = async () => {
  const q = inputText.value.trim()
  if (!q || loading.value) return

  messages.value.push({ role: 'user', content: q })
  inputText.value = ''
  loading.value = true
  await scrollToBottom()

  try {
    const res = await askAI({ question: q, context: contextData.value, sessionId: ensureSession() })
    const payload = res?.data?.data || res?.data
    if (!payload?.answer) {
      throw new Error(res?.data?.message || 'AI 返回数据格式异常')
    }
    messages.value.push({
      role: 'ai',
      content: payload.answer,
      summary: payload.summary,
      businessHint: payload.businessHint,
      suggestion: payload.suggestion,
      source: payload.source,
      route: payload.route,
      tablesQueried: payload.tablesQueried,
      queryTime: payload.queryTime,
      originalQuestion: payload.originalQuestion,
      resolvedQuestion: payload.resolvedQuestion,
      usedMemory: payload.usedMemory
    })
  } catch (e) {
    const errorMessage = e?.response?.data?.message || e?.message || '查询失败，请稍后重试。'
    messages.value.push({
      role: 'ai',
      content: errorMessage,
      source: 'error',
      queryTime: 0
    })
  } finally {
    loading.value = false
    await scrollToBottom()
  }
}

ensureSession()
</script>

<style scoped>
.ai-assistant {
  display: flex;
  flex-direction: column;
  height: calc(100vh - 120px);
  max-width: 920px;
  margin: 20px auto;
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.08);
  overflow: hidden;
}

.chat-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 24px;
  border-bottom: 1px solid #ebeef5;
  font-size: 17px;
  font-weight: 600;
  color: #303133;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 10px;
}

.chat-body {
  flex: 1;
  overflow-y: auto;
  padding: 24px 20px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.msg-row {
  display: flex;
  align-items: flex-start;
  gap: 10px;
}

.msg-row.user {
  flex-direction: row-reverse;
}

.avatar {
  flex-shrink: 0;
}

.ai-avatar {
  background: #ecf5ff;
  color: #409eff;
}

.user-avatar {
  background: #f0f9eb;
  color: #67c23a;
}

.bubble {
  max-width: 76%;
  padding: 12px 16px;
  border-radius: 12px;
  font-size: 14px;
  line-height: 1.7;
  word-break: break-word;
}

.ai-bubble {
  background: #f4f4f5;
  color: #303133;
  border-top-left-radius: 2px;
}

.user-bubble {
  background: #409eff;
  color: #fff;
  border-top-right-radius: 2px;
}

.hint {
  color: #909399;
  font-size: 13px;
}

.summary-block,
.memory-block,
.hint-block {
  margin-top: 10px;
  padding: 10px 12px;
  border-radius: 8px;
  background: rgba(64, 158, 255, 0.08);
}

.block-title {
  font-size: 12px;
  color: #606266;
  margin-bottom: 4px;
}

.msg-meta {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-top: 10px;
  flex-wrap: wrap;
}

.query-time {
  font-size: 12px;
  color: #909399;
}

.loading-bubble {
  display: flex;
  gap: 5px;
  align-items: center;
  padding: 14px 18px;
}

.dot {
  width: 8px;
  height: 8px;
  background: #909399;
  border-radius: 50%;
  animation: bounce 1.2s infinite;
}

.dot:nth-child(2) {
  animation-delay: 0.2s;
}

.dot:nth-child(3) {
  animation-delay: 0.4s;
}

@keyframes bounce {
  0%, 80%, 100% { transform: translateY(0); }
  40% { transform: translateY(-8px); }
}

.chat-footer {
  display: flex;
  gap: 10px;
  padding: 16px 20px;
  border-top: 1px solid #ebeef5;
  background: #fafafa;
}

.chat-footer .el-input {
  flex: 1;
}
</style>
