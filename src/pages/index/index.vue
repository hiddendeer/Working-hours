<!-- 使用 type="home" 属性设置首页，其他页面不需要设置，默认为page；推荐使用json5，更强大，且允许注释 -->
<route lang="json5">
  {
    style: {
      navigationBarTitleText: '通勤记录',
    },
  }
  </route>
<template>
  <view class="h-screen flex flex-col p-[48rpx] box-border" :style="{ paddingTop: safeAreaTop + 'px' }">
    <!-- 统计数据区域 -->
    <view class="flex-1">
      <!-- 今日统计 -->
      <view class="text-2xl font-bold mb-[32rpx] text-gray-800">📊 今日统计</view>
      <view class="grid grid-cols-2 gap-[32rpx] mb-[64rpx]">
        <view
          class="p-[32rpx] rounded-xl bg-gradient-to-br from-blue-50 to-indigo-100 border border-blue-200 shadow-sm">
          <view class="text-blue-600 font-medium flex items-center">
            💼 <text class="ml-[8rpx]">工作时长</text>
          </view>
          <view class="text-3xl font-bold mt-[16rpx] text-indigo-700">{{ formattedWorkTime }}</view>
        </view>
        <view
          class="p-[32rpx] rounded-xl bg-gradient-to-br from-emerald-50 to-teal-100 border border-emerald-200 shadow-sm">
          <view class="text-emerald-600 font-medium flex items-center">
            🚗 <text class="ml-[8rpx]">通勤时长</text>
          </view>
          <view class="text-3xl font-bold mt-[16rpx] text-teal-700">{{ formattedCommuteTime }}</view>
        </view>
      </view>

      <!-- 本月统计 -->
      <view class="text-xl font-bold mb-[32rpx] text-gray-800">📈 本月统计</view>
      <view class="grid grid-cols-2 gap-[32rpx]">
        <view
          class="p-[32rpx] rounded-xl bg-gradient-to-br from-purple-50 to-violet-100 border border-purple-200 shadow-sm">
          <view class="text-purple-600 font-medium flex items-center">
            ⏰ <text class="ml-[8rpx]">平均工作</text>
          </view>
          <view class="text-2xl font-bold mt-[16rpx] text-violet-700">8h 15m</view>
        </view>
        <view class="p-[32rpx] rounded-xl bg-gradient-to-br from-rose-50 to-pink-100 border border-rose-200 shadow-sm">
          <view class="text-rose-600 font-medium flex items-center">
            🛣️ <text class="ml-[8rpx]">平均通勤</text>
          </view>
          <view class="text-2xl font-bold mt-[16rpx] text-pink-700">1h 5m</view>
        </view>
      </view>
    </view>

    <!-- 操作按钮区域 -->
    <view class="flex justify-center items-center py-[64rpx]">
      <button v-if="currentAction" @click="currentAction.handler"
        :class="`w-[320rpx] h-[320rpx] rounded-full ${currentAction.bgColor} ${currentAction.textColor} flex justify-center items-center text-xl font-bold ${currentAction.shadowColor} shadow-xl border-4 ${currentAction.borderColor} transform transition-all duration-200 hover:scale-105 active:scale-95`">
        {{ currentAction.text }}
      </button>

      <!-- 午休状态 -->
      <view v-if="appState === 'LUNCH_BREAK'"
        class="text-center p-[48rpx] rounded-2xl bg-gradient-to-br from-yellow-50 to-orange-100 border border-yellow-200">
        <view class="text-2xl mb-[16rpx]">🥪</view>
        <view class="text-orange-600 text-xl font-semibold mb-[8rpx]">午休时间</view>
        <view class="text-sm text-orange-500">13:30 自动开始工作</view>
      </view>

      <!-- 结束状态 -->
      <view v-if="appState === 'DAY_ENDED'"
        class="text-center p-[48rpx] rounded-2xl bg-gradient-to-br from-green-50 to-emerald-100 border border-green-200">
        <view class="text-3xl mb-[24rpx]">🎉</view>
        <view class="text-emerald-600 mb-[32rpx] text-xl font-semibold">今天辛苦了!</view>
        <button @click="resetDay"
          class="bg-gradient-to-r from-emerald-100 to-teal-200 text-emerald-800 px-[48rpx] py-[24rpx] rounded-xl shadow-xl border-2 border-emerald-400 font-bold transform transition-all duration-200 hover:scale-105 active:scale-95">
          ✨ 新的一天
        </button>
      </view>
    </view>
  </view>
</template>

<script lang="ts" setup>
import { ref, computed } from 'vue'

defineOptions({
  name: 'Home',
})

// 常量配置
const WORK_START_TIME = { hour: 9, minute: 0 }
const LUNCH_START_TIME = { hour: 12, minute: 0 }
const LUNCH_END_TIME = { hour: 13, minute: 30 }
const STORAGE_KEYS = {
  STATE: 'commute_state_v2',
  WORK_TIME: 'commute_work_v2',
  COMMUTE_TIME: 'commute_travel_v2',
  START_TIME: 'commute_start_v2',
} as const

// 状态类型定义
type AppState = 'IDLE' | 'COMMUTING_TO_WORK' | 'WORKING' | 'LUNCH_BREAK' | 'COMMUTING_HOME' | 'DAY_ENDED'

// 状态管理
const appState = ref<AppState>('IDLE')
const workSeconds = ref(0)
const commuteSeconds = ref(0)
const sessionStartTime = ref(0)
const timeCheckInterval = ref<number>()
const autoSaveCounter = ref(0)

// 获取安全区域
const safeAreaTop = (() => {
  try {
    return uni.getSystemInfoSync().safeArea?.top || 0
  } catch {
    return 0
  }
})()

// 时间格式化
const formatTime = (seconds: number): string => {
  const h = Math.floor(seconds / 3600).toString().padStart(2, '0')
  const m = Math.floor((seconds % 3600) / 60).toString().padStart(2, '0')
  const s = Math.floor(seconds % 60).toString().padStart(2, '0')
  return `${h}:${m}:${s}`
}

// 计算属性
const formattedWorkTime = computed(() => formatTime(workSeconds.value))
const formattedCommuteTime = computed(() => formatTime(commuteSeconds.value))

// 当前操作按钮配置
const currentAction = computed(() => {
  const actions = {
    IDLE: {
      text: '🚗 开始通勤',
      bgColor: 'bg-gradient-to-r from-emerald-100 to-teal-200',
      textColor: 'text-emerald-800',
      shadowColor: 'shadow-emerald-400',
      borderColor: 'border-emerald-400',
      handler: handleStartCommute
    },
    COMMUTING_TO_WORK: {
      text: '💼 到达公司',
      bgColor: 'bg-gradient-to-r from-blue-100 to-indigo-200',
      textColor: 'text-blue-800',
      shadowColor: 'shadow-blue-400',
      borderColor: 'border-blue-400',
      handler: handleStartWork
    },
    WORKING: {
      text: '🏠 准备下班',
      bgColor: 'bg-gradient-to-r from-rose-100 to-pink-200',
      textColor: 'text-rose-800',
      shadowColor: 'shadow-rose-400',
      borderColor: 'border-rose-400',
      handler: handleEndWork
    },
    COMMUTING_HOME: {
      text: '🎉 到家啦',
      bgColor: 'bg-gradient-to-r from-orange-100 to-amber-200',
      textColor: 'text-orange-800',
      shadowColor: 'shadow-orange-400',
      borderColor: 'border-orange-400',
      handler: handleEndCommute
    },
  } as const

  return actions[appState.value as keyof typeof actions]
})

// 时间计算逻辑
const updateTime = () => {
  if (!sessionStartTime.value) return

  const now = Date.now()
  const deltaSeconds = Math.floor((now - sessionStartTime.value) / 1000)

  if (appState.value === 'WORKING') {
    workSeconds.value += deltaSeconds
  } else if (['COMMUTING_TO_WORK', 'COMMUTING_HOME'].includes(appState.value)) {
    commuteSeconds.value += deltaSeconds
  }

  sessionStartTime.value = now
}

// 处理包含午休时间的工作时间计算
const updateTimeWithLunchBreak = () => {
  if (!sessionStartTime.value) return

  const now = Date.now()
  const startTime = new Date(sessionStartTime.value)
  const endTime = new Date(now)
  
  // 计算总的时间差（秒）
  let totalSeconds = Math.floor((now - sessionStartTime.value) / 1000)
  
  // 检查是否跨越了午休时间（12:00-13:30）
  const lunchStart = new Date(startTime)
  lunchStart.setHours(LUNCH_START_TIME.hour, LUNCH_START_TIME.minute, 0, 0)
  
  const lunchEnd = new Date(startTime)
  lunchEnd.setHours(LUNCH_END_TIME.hour, LUNCH_END_TIME.minute, 0, 0)
  
  // 如果时间段跨越了午休时间，需要扣除午休时间
  if (startTime < lunchEnd && endTime > lunchStart) {
    const actualLunchStart = startTime < lunchStart ? lunchStart : startTime
    const actualLunchEnd = endTime > lunchEnd ? lunchEnd : endTime
    
    if (actualLunchEnd > actualLunchStart) {
      const lunchSeconds = Math.floor((actualLunchEnd.getTime() - actualLunchStart.getTime()) / 1000)
      totalSeconds -= lunchSeconds
      console.log(`扣除午休时间 ${lunchSeconds} 秒`)
    }
  }
  
  // 只有正数才累加到工作时间
  if (totalSeconds > 0) {
    workSeconds.value += totalSeconds
  }
  
  sessionStartTime.value = now
}

// 状态转换函数
const startSession = (newState: AppState) => {
  updateTime() // 保存当前会话时间
  appState.value = newState
  sessionStartTime.value = Date.now()
  saveState()
}

const stopSession = () => {
  updateTime()
  sessionStartTime.value = 0
  saveState()
}

// 操作处理函数
const handleStartCommute = () => startSession('COMMUTING_TO_WORK')
const handleStartWork = () => startSession('WORKING')
const handleEndWork = () => {
  stopSession()
  startSession('COMMUTING_HOME')
}
const handleEndCommute = () => {
  stopSession()
  appState.value = 'DAY_ENDED'
  saveState()
}

const resetDay = () => {
  stopSession()
  appState.value = 'IDLE'
  workSeconds.value = 0
  commuteSeconds.value = 0
  clearStorage()
}

// 存储操作
const saveState = () => {
  const today = new Date().toDateString()
  try {
    uni.setStorageSync(STORAGE_KEYS.STATE, { state: appState.value, date: today })
    uni.setStorageSync(STORAGE_KEYS.WORK_TIME, workSeconds.value)
    uni.setStorageSync(STORAGE_KEYS.COMMUTE_TIME, commuteSeconds.value)
    uni.setStorageSync(STORAGE_KEYS.START_TIME, sessionStartTime.value)
  } catch (error) {
    console.error('保存状态失败:', error)
  }
}

const loadState = () => {
  const today = new Date().toDateString()
  try {
    const savedMeta = uni.getStorageSync(STORAGE_KEYS.STATE)

    if (!savedMeta || savedMeta.date !== today) {
      resetDay()
      return
    }

    appState.value = savedMeta.state
    workSeconds.value = uni.getStorageSync(STORAGE_KEYS.WORK_TIME) || 0
    commuteSeconds.value = uni.getStorageSync(STORAGE_KEYS.COMMUTE_TIME) || 0
    const savedStartTime = uni.getStorageSync(STORAGE_KEYS.START_TIME) || 0

    // 恢复中断的会话，需要处理午休时间
    if (savedStartTime > 0 && ['WORKING', 'COMMUTING_TO_WORK', 'COMMUTING_HOME'].includes(appState.value)) {
      sessionStartTime.value = savedStartTime
      
      // 如果是工作状态，需要扣除午休时间
      if (appState.value === 'WORKING') {
        updateTimeWithLunchBreak() // 使用新的午休时间处理函数
      } else {
        updateTime() // 通勤时间正常补充
      }
      
      sessionStartTime.value = Date.now() // 重新开始计时
    }
    
    // 检查当前是否应该在午休状态
    if (isLunchTime() && appState.value === 'WORKING') {
      stopSession()
      appState.value = 'LUNCH_BREAK'
      saveState()
      console.log('恢复时检测到午休时间，切换到午休模式')
    }
  } catch (error) {
    console.error('加载状态失败:', error)
    resetDay()
  }
}

const clearStorage = () => {
  try {
    Object.values(STORAGE_KEYS).forEach(key => uni.removeStorageSync(key))
  } catch (error) {
    console.error('清理存储失败:', error)
  }
}

// 检查是否在午休时间内
const isLunchTime = () => {
  const now = new Date()
  const currentMinutes = now.getHours() * 60 + now.getMinutes()
  const lunchStartMinutes = LUNCH_START_TIME.hour * 60 + LUNCH_START_TIME.minute
  const lunchEndMinutes = LUNCH_END_TIME.hour * 60 + LUNCH_END_TIME.minute
  
  return currentMinutes >= lunchStartMinutes && currentMinutes < lunchEndMinutes
}

// 自动时间检查
const checkAutoTriggers = () => {
  const now = new Date()
  const { hour, minute } = { hour: now.getHours(), minute: now.getMinutes() }

  // 9:00 自动开始工作
  if (hour === WORK_START_TIME.hour && minute === WORK_START_TIME.minute) {
    if (['IDLE', 'COMMUTING_TO_WORK'].includes(appState.value)) {
      handleStartWork()
    }
  }

  // 12:00 自动进入午休
  if (hour === LUNCH_START_TIME.hour && minute === LUNCH_START_TIME.minute) {
    if (appState.value === 'WORKING') {
      stopSession()
      appState.value = 'LUNCH_BREAK'
      saveState()
      console.log('自动进入午休模式')
    }
  }

  // 13:30 自动结束午休
  if (hour === LUNCH_END_TIME.hour && minute === LUNCH_END_TIME.minute) {
    if (appState.value === 'LUNCH_BREAK') {
      startSession('WORKING')
      console.log('自动结束午休，恢复工作')
    }
  }

  // 如果当前在午休时间内，但状态不是午休，自动切换到午休
  if (isLunchTime() && appState.value === 'WORKING') {
    stopSession()
    appState.value = 'LUNCH_BREAK'
    saveState()
    console.log('检测到午休时间，自动进入午休模式')
  }
}

// 生命周期管理
const startTimeTracking = () => {
  loadState()
  checkAutoTriggers()
  autoSaveCounter.value = 0
  timeCheckInterval.value = setInterval(() => {
    updateTime()
    checkAutoTriggers()
    // 每10秒自动保存一次，确保数据不丢失
    autoSaveCounter.value++
    if (autoSaveCounter.value >= 10) {
      autoSaveCounter.value = 0
      saveState()
    }
  }, 1000)
}

const stopTimeTracking = () => {
  updateTime()
  saveState()
  if (timeCheckInterval.value) {
    clearInterval(timeCheckInterval.value)
    timeCheckInterval.value = undefined
  }
}

// uni-app 生命周期
onLoad(() => {
  startTimeTracking()
})

onUnload(() => {
  stopTimeTracking()
})

onShow(() => {
  loadState()
  if (!timeCheckInterval.value) {
    startTimeTracking()
  }
})

onHide(() => {
  stopTimeTracking()
})
</script>



<style>
/* 保留用于可能的全局样式覆盖 */
</style>
