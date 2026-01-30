<script setup lang="ts">
// [WHY] 根组件，包含路由视图和底部导航
// [WHAT] 使用 Vant Tabbar 实现底部导航切换
// [NOTE] 公告和更新检查已移至 Home.vue 中处理
import { ref, watch, computed, onMounted, onUnmounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { showToast } from 'vant'

const route = useRoute()
const router = useRouter()

// [WHY] 处理 Android 返回键，防止直接退出应用
// [WHAT] 在主页时需要双击才能退出
let lastBackTime = 0
let backButtonHandler: ((e: any) => void) | null = null

onMounted(async () => {
  // [WHAT] 动态导入 Capacitor App 插件（如果可用）
  try {
    const { App: CapApp } = await import('@capacitor/app')
    
    // [WHAT] 监听 Capacitor 返回键事件
    await CapApp.addListener('backButton', ({ canGoBack }) => {
      // [WHY] 如果不在主页，正常返回上一页
      const mainPages = ['home', 'market', 'holding', 'analysis']
      const isMainPage = mainPages.includes(route.name as string)
      
      if (!isMainPage && window.history.length > 1) {
        router.back()
        return
      }
      
      // [WHY] 在主页时，双击退出
      const now = Date.now()
      if (now - lastBackTime < 2000) {
        // 2秒内双击返回键，退出应用
        CapApp.exitApp()
      } else {
        lastBackTime = now
        showToast('再按一次退出应用')
      }
    })
    
    backButtonHandler = () => CapApp.removeAllListeners()
  } catch {
    // [EDGE] Web 环境或 Capacitor 未安装，忽略
    console.log('Capacitor App plugin not available')
  }
})

onUnmounted(() => {
  if (backButtonHandler) {
    backButtonHandler(null)
  }
})

// [WHAT] 当前激活的 tab
const activeTab = ref('home')

// [WHAT] 需要隐藏底部导航的页面
const hiddenTabbarPages = ['search', 'detail', 'trades']
const showTabbar = computed(() => !hiddenTabbarPages.includes(route.name as string))

// [WHY] 路由变化时同步更新 tab 状态
watch(
  () => route.name,
  (name) => {
    const tabMap: Record<string, string> = {
      home: 'home',
      market: 'market',
      holding: 'holding',
      analysis: 'analysis'
    }
    if (name && tabMap[name as string]) {
      activeTab.value = tabMap[name as string]
    }
  },
  { immediate: true }
)

// [WHAT] 切换 tab 时跳转路由
function onTabChange(name: string | number) {
  const routeMap: Record<string, string> = {
    home: '/',
    market: '/market',
    holding: '/holding',
    analysis: '/analysis'
  }
  if (routeMap[name as string]) {
    router.push(routeMap[name as string])
  }
}
</script>

<template>
  <div class="app-container">
    <!-- 路由视图 -->
    <!-- [WHY] 暂时禁用 keep-alive 避免页面缓存混乱 -->
    <router-view />

    <!-- 底部导航栏 -->
    <van-tabbar
      v-if="showTabbar"
      v-model="activeTab"
      @change="onTabChange"
    >
      <van-tabbar-item name="holding" icon="balance-list-o">持仓</van-tabbar-item>
      <van-tabbar-item name="market" icon="chart-trending-o">行情</van-tabbar-item>
      <van-tabbar-item name="home" icon="home-o">自选</van-tabbar-item>
      <van-tabbar-item name="analysis" icon="bar-chart-o">分析</van-tabbar-item>
    </van-tabbar>
  </div>
</template>

<style scoped>
.app-container {
  min-height: 100vh;
  /* [WHY] 使用主题变量 */
  background: var(--bg-primary);
  padding-bottom: 50px;
  transition: background-color 0.3s;
}
</style>
