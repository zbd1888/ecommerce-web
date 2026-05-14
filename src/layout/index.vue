<template>
  <el-container class="layout-container">
    <!-- 侧边栏 -->
    <el-aside :width="isCollapse ? '64px' : '260px'" class="layout-aside">
      <div class="logo-area">
        <el-icon :size="32" color="#fff"><DataAnalysis /></el-icon>
        <span v-show="!isCollapse" class="logo-text">{{ isAdmin ? '电商爆款分析后台管理系统' : '电商爆款分析与可视化系统' }}</span>
      </div>
      <el-menu :default-active="activeMenu" :collapse="isCollapse" background-color="#1d1e1f" text-color="#bfcbd9" active-text-color="#409EFF" router>
        <!-- ========== 管理员菜单 ========== -->
        <template v-if="isAdmin">
          <el-menu-item index="/admin/dashboard">
            <el-icon><Monitor /></el-icon>
            <span>数据质量大屏</span>
          </el-menu-item>
          <el-menu-item index="/admin/crawl">
            <el-icon><Download /></el-icon>
            <span>数据采集</span>
          </el-menu-item>
          <el-menu-item index="/admin/clean">
            <el-icon><Brush /></el-icon>
            <span>数据清洗</span>
          </el-menu-item>
          <el-menu-item index="/admin/product">
            <el-icon><Goods /></el-icon>
            <span>商品管理</span>
          </el-menu-item>
          <el-menu-item index="/admin/user">
            <el-icon><UserFilled /></el-icon>
            <span>用户管理</span>
          </el-menu-item>
          <el-menu-item index="/admin/rule">
            <el-icon><Setting /></el-icon>
            <span>爆品规则配置</span>
          </el-menu-item>
          <el-menu-item index="/admin/monitor">
            <el-icon><DataLine /></el-icon>
            <span>系统监控</span>
          </el-menu-item>
          <el-menu-item index="/admin/ai-assistant">
            <el-icon><ChatDotRound /></el-icon>
            <span>AI 助手</span>
          </el-menu-item>
        </template>
        <!-- ========== 普通用户菜单 ========== -->
        <template v-else>
          <el-menu-item index="/user/dashboard">
            <el-icon><DataAnalysis /></el-icon>
            <span>爆品总览</span>
          </el-menu-item>
          <el-menu-item index="/user/product">
            <el-icon><Search /></el-icon>
            <span>爆品发现</span>
          </el-menu-item>
          <el-menu-item index="/user/hot-analysis">
            <el-icon><TrendCharts /></el-icon>
            <span>爆品详情分析</span>
          </el-menu-item>
          <el-menu-item index="/user/category">
            <el-icon><PieChart /></el-icon>
            <span>品类分析</span>
          </el-menu-item>
          <el-menu-item index="/user/region">
            <el-icon><Location /></el-icon>
            <span>地域可视化</span>
          </el-menu-item>
          <el-menu-item index="/user/prediction">
            <el-icon><Aim /></el-icon>
            <span>选品助手</span>
          </el-menu-item>
          <el-menu-item index="/user/ai-assistant">
            <el-icon><ChatDotRound /></el-icon>
            <span>AI 助手</span>
          </el-menu-item>
        </template>
      </el-menu>
    </el-aside>

    <!-- 主内容区 -->
    <el-container>
      <el-header class="layout-header">
        <div class="header-left">
          <el-icon class="collapse-btn" @click="isCollapse = !isCollapse">
            <Fold v-if="!isCollapse" />
            <Expand v-else />
          </el-icon>
          <el-breadcrumb separator="/">
            <el-breadcrumb-item>首页</el-breadcrumb-item>
            <el-breadcrumb-item>{{ currentTitle }}</el-breadcrumb-item>
          </el-breadcrumb>
        </div>
        <div class="header-right">
          <el-tag v-if="isAdmin" type="danger" size="small" class="role-tag">管理员</el-tag>
          <el-tag v-else type="info" size="small" class="role-tag">普通用户</el-tag>
          <el-dropdown @command="handleCommand">
            <div class="user-info">
              <el-avatar :size="32" icon="User" />
              <span class="username">{{ user?.nickname || user?.username }}</span>
              <el-icon><ArrowDown /></el-icon>
            </div>
            <template #dropdown>
              <el-dropdown-menu>
                <el-dropdown-item disabled>
                  <el-icon><User /></el-icon>{{ user?.username }}
                </el-dropdown-item>
                <el-dropdown-item divided command="logout">
                  <el-icon><SwitchButton /></el-icon>退出登录
                </el-dropdown-item>
              </el-dropdown-menu>
            </template>
          </el-dropdown>
        </div>
      </el-header>
      <el-main class="layout-main">
        <router-view />
      </el-main>
    </el-container>
  </el-container>
</template>

<script setup>
import { ref, computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'

const route = useRoute()
const router = useRouter()
const isCollapse = ref(false)

// 从localStorage或sessionStorage获取用户信息
const getStorageItem = (key) => localStorage.getItem(key) || sessionStorage.getItem(key)

const user = computed(() => {
  try {
    const u = getStorageItem('user')
    if (!u) return null
    const parsed = JSON.parse(u)
    // 确保用户名有值
    if (!parsed.nickname && !parsed.username) {
      parsed.nickname = parsed.role === 'admin' ? '管理员' : '用户'
    }
    return parsed
  } catch (e) {
    console.error('解析用户信息失败', e)
    return null
  }
})

const isAdmin = computed(() => {
  return user.value?.role === 'admin'
})

const activeMenu = computed(() => route.path)
const currentTitle = computed(() => route.meta?.title || '爆品总览')

const handleCommand = (cmd) => {
  if (cmd === 'logout') {
    // 清除所有存储的登录信息
    localStorage.removeItem('token')
    localStorage.removeItem('user')
    localStorage.removeItem('rememberedUser')
    localStorage.removeItem('tokenExpire')
    sessionStorage.removeItem('token')
    sessionStorage.removeItem('user')
    router.push('/login')
  }
}
</script>

<style scoped>
.layout-container { height: 100vh; }

.layout-aside {
  background: #1d1e1f;
  transition: width 0.3s;
  overflow: hidden;
}

.logo-area {
  height: 70px;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  padding: 0 12px;
  border-bottom: 1px solid #2d2e2f;
}

.logo-text {
  color: #fff;
  font-size: 16px;
  font-weight: 600;
  white-space: nowrap;
  line-height: 1.4;
  overflow: visible;
}

.el-menu {
  border-right: none;
}

.el-menu :deep(.el-menu-item) {
  height: 52px;
  line-height: 52px;
  padding: 0 20px;
  margin: 2px 0;
  font-size: 17px;
}

.el-menu :deep(.el-menu-item .el-icon) {
  font-size: 21px;
}

.layout-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #fff;
  border-bottom: 1px solid #ebeef5;
  box-shadow: 0 1px 4px rgba(0, 21, 41, 0.08);
}

.header-left {
  display: flex;
  align-items: center;
  gap: 16px;
}

.collapse-btn {
  font-size: 22px;
  cursor: pointer;
  color: #606266;
}

.collapse-btn:hover { color: #409EFF; }

.header-right {
  display: flex;
  align-items: center;
  gap: 12px;
}

.role-tag { margin-right: 8px; font-size: 14px; }

.user-info {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  padding: 4px 8px;
  border-radius: 4px;
}

.user-info:hover { background: #f5f7fa; }

.username {
  color: #606266;
  font-size: 15px;
}

.layout-main {
  background: #f0f2f5;
  padding: 0;
  overflow-y: auto;
}
</style>