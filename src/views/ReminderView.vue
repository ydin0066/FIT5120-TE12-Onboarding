<template>
  <div class="page-container">
    <div class="top-nav">
      <div class="nav-title">Sunscreen Guide</div>
      <div class="nav-buttons">
        <router-link to="/" class="nav-button">
          <i class="fas fa-umbrella-beach"></i> Sun Protector
        </router-link>
        <router-link to="/cancer-chart" class="nav-button">
          <i class="fas fa-chart-bar"></i> Cancer Chart
        </router-link>
        <router-link to="/uv-indicator" class="nav-button">
          <i class="fas fa-sun"></i> UV Indicator
        </router-link>
        <router-link to="/recommendation" class="nav-button">
          <i class="fas fa-lightbulb"></i> Recommendation
        </router-link>
        <router-link to="/reminder" class="nav-button active">
          <i class="fas fa-bell"></i> Set Reminders
        </router-link>
      </div>
    </div>

    <div class="content">
      <div class="reminder-section">
        <h2><i class="fas fa-bell icon-title"></i> Set Reminders</h2>

        <div class="sub-description">
          <p>
            💡 You can use browser notifications to get timely reminders for reapplying sunscreen
          </p>
        </div>

        <div class="notification-toggle">
          <label class="switch">
            <input type="checkbox" v-model="notificationsEnabled" @change="toggleNotifications" />
            <span class="slider round"></span>
          </label>
          <span class="toggle-label">Enable Browser Notifications</span>
        </div>

        <div class="reminder-interval">
          <span>Remind me every:</span>
          <select v-model="reminderInterval" :disabled="!notificationsEnabled">
            <option value="1">1 hour</option>
            <option value="2">2 hours</option>
            <option value="3">3 hours</option>
            <option value="4">4 hours</option>
          </select>
        </div>

        <div class="progress-bar">
          <div class="progress" :style="{ width: progressWidth + '%' }"></div>
        </div>

        <div class="time-display">
          {{ nextReminderTime }}
        </div>
      </div>

      <NextPageArrow nextRoute="/" nextPageName="Home" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from 'vue'
import NextPageArrow from '../components/NextPageArrow.vue'

const notificationsEnabled = ref(false)
const reminderInterval = ref('2') // 默认为2小时
const nextReminderTime = ref('')
const progressWidth = ref(0)
let reminderTimer: number | null = null
let progressTimer: number | null = null

// 检查浏览器是否支持通知
const checkNotificationSupport = () => {
  return 'Notification' in window
}

// 请求通知权限
const requestNotificationPermission = async () => {
  if (!checkNotificationSupport()) {
    alert('Your browser does not support notifications')
    return false
  }

  // 如果已经获得了权限，直接返回成功
  if (Notification.permission === 'granted') {
    return true
  }

  // 如果权限状态是默认的（未决定），则请求权限
  if (Notification.permission !== 'denied') {
    try {
      // 显示浏览器原生的权限请求对话框
      // 对话框文本会自动翻译成用户浏览器的语言
      // 通常会显示："Allow [网站名] to send notifications?" 并有 "Allow" 和 "Block" 按钮
      const permission = await Notification.requestPermission()
      return permission === 'granted'
    } catch (error) {
      console.error('Error requesting notification permission:', error)
      return false
    }
  }

  // 如果权限已被拒绝，返回失败
  return false
}

// 切换通知状态
const toggleNotifications = async () => {
  if (notificationsEnabled.value) {
    // 用户尝试开启通知
    const permissionGranted = await requestNotificationPermission()
    if (!permissionGranted) {
      // 如果用户拒绝了权限请求，将开关切回关闭状态
      notificationsEnabled.value = false

      // 显示提示信息
      if (Notification.permission === 'denied') {
        alert('Notification permission was denied. You can enable it in your browser settings.')
      }
      return
    }

    // 权限获取成功，开始计时器
    startReminderTimer()
  } else {
    // 用户关闭了通知
    stopReminderTimer()
  }

  // 保存设置
  saveSettings()
}

// 发送通知
const sendNotification = () => {
  if (!notificationsEnabled.value) return

  const notification = new Notification('Sun Protector Reminder', {
    body: 'Time to reapply your sunscreen!',
    icon: '/favicon.ico',
  })

  notification.onclick = () => {
    window.focus()
    notification.close()
  }

  // 重置进度条并开始新的计时
  resetProgressBar()
}

// 开始提醒计时器
const startReminderTimer = () => {
  stopReminderTimer() // 先停止现有的计时器

  const intervalMs = parseInt(reminderInterval.value) * 60 * 60 * 1000
  reminderTimer = window.setTimeout(sendNotification, intervalMs)

  // 设置下次提醒时间
  const nextTime = new Date()
  nextTime.setTime(nextTime.getTime() + intervalMs)
  nextReminderTime.value = nextTime.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })

  // 开始进度条动画
  startProgressBar(intervalMs)
}

// 停止提醒计时器
const stopReminderTimer = () => {
  if (reminderTimer) {
    clearTimeout(reminderTimer)
    reminderTimer = null
  }

  if (progressTimer) {
    clearInterval(progressTimer)
    progressTimer = null
  }

  nextReminderTime.value = ''
  progressWidth.value = 0
}

// 开始进度条动画
const startProgressBar = (totalDuration: number) => {
  const startTime = Date.now()
  const updateInterval = 1000 // 每秒更新一次

  progressTimer = window.setInterval(() => {
    const elapsedTime = Date.now() - startTime
    progressWidth.value = Math.min((elapsedTime / totalDuration) * 100, 100)

    if (progressWidth.value >= 100) {
      clearInterval(progressTimer as number)
    }
  }, updateInterval)
}

// 重置进度条
const resetProgressBar = () => {
  progressWidth.value = 0
  startReminderTimer()
}

// 保存设置
const saveSettings = () => {
  localStorage.setItem(
    'sunProtectorNotifications',
    JSON.stringify({
      enabled: notificationsEnabled.value,
      interval: reminderInterval.value,
    }),
  )
}

// 加载设置
const loadSettings = () => {
  const saved = localStorage.getItem('sunProtectorNotifications')
  if (saved) {
    const settings = JSON.parse(saved)
    notificationsEnabled.value = settings.enabled
    reminderInterval.value = settings.interval

    if (notificationsEnabled.value) {
      // 如果通知已启用，则检查权限并开始计时器
      requestNotificationPermission().then((granted) => {
        if (granted && notificationsEnabled.value) {
          startReminderTimer()
        } else {
          notificationsEnabled.value = false
          saveSettings()
        }
      })
    }
  }
}

// 当提醒间隔变化时重启计时器
const restartTimerOnIntervalChange = () => {
  if (notificationsEnabled.value) {
    startReminderTimer()
  }
  saveSettings()
}

// 监听间隔变化并重启计时器
watch(reminderInterval, () => {
  restartTimerOnIntervalChange()
})

// 组件挂载时
onMounted(() => {
  loadSettings()
})

// 组件卸载时
onUnmounted(() => {
  stopReminderTimer()
})
</script>

<style scoped>
.page-container {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  width: 100%;
  background-image: linear-gradient(to bottom, #e0f7fa, #b2ebf2, #80deea);
  background-size: cover;
  background-position: center;
  padding: 0;
  margin: 0;
}

.top-nav {
  background-color: #1976d2;
  color: white;
  padding: 10px 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);
  position: relative;
  z-index: 10;
}

.nav-title {
  font-size: 24px;
  font-weight: bold;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.2);
}

.nav-buttons {
  display: flex;
  gap: 10px;
}

.nav-button {
  color: white;
  text-decoration: none;
  padding: 8px 15px;
  border-radius: 5px;
  display: flex;
  align-items: center;
  gap: 5px;
  transition: all 0.3s;
}

.nav-button:hover {
  background-color: rgba(255, 255, 255, 0.15);
  transform: translateY(-2px);
}

.nav-button.active {
  background-color: rgba(255, 255, 255, 0.25);
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.content {
  flex: 1;
  width: 100%;
  background-color: rgba(255, 255, 255, 0.85);
  display: flex;
  flex-direction: column;
  padding: 30px;
  backdrop-filter: blur(10px);
  position: relative;
  z-index: 5;
}

.reminder-section {
  flex: 1;
  background: transparent;
  text-align: center;
  max-width: 800px;
  width: 100%;
  margin: 0 auto;
  animation: fadeIn 0.8s ease;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

h2 {
  font-size: 36px;
  font-weight: 700;
  margin-bottom: 30px;
  text-align: center;
  color: #1565c0;
  text-shadow: 1px 1px 3px rgba(0, 0, 0, 0.15);
  position: relative;
}

.icon-title {
  color: #ff9800;
  margin-right: 10px;
  filter: drop-shadow(0 0 3px rgba(255, 152, 0, 0.4));
}

.sub-description {
  text-align: center;
  font-size: 18px;
  font-weight: 500;
  color: #555;
  margin-bottom: 30px;
  padding: 18px 25px;
  background: rgba(255, 255, 255, 0.9);
  border-left: 4px solid #3498db;
  border-radius: 12px;
  box-shadow: 0 6px 15px rgba(52, 152, 219, 0.15);
  line-height: 1.6;
  max-width: 800px;
  margin-left: auto;
  margin-right: auto;
  transition: all 0.3s ease;
  position: relative;
  overflow: hidden;
}

.sub-description::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(135deg, rgba(255, 255, 255, 0.4) 0%, rgba(255, 255, 255, 0) 100%);
  pointer-events: none;
  z-index: 1;
}

.sub-description:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 20px rgba(52, 152, 219, 0.25);
}

.notification-toggle {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 35px;
  background: white;
  padding: 25px;
  border-radius: 15px;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
  animation: slideIn 0.5s ease;
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateX(-20px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

.notification-toggle:hover {
  transform: translateY(-5px);
  box-shadow: 0 15px 30px rgba(0, 0, 0, 0.15);
}

.toggle-label {
  margin-left: 20px;
  font-size: 18px;
  font-weight: 500;
  color: #37474f;
  text-shadow: 0 1px 1px rgba(0, 0, 0, 0.05);
}

/* 开关样式 */
.switch {
  position: relative;
  display: inline-block;
  width: 70px;
  height: 38px;
  filter: drop-shadow(0 2px 5px rgba(0, 0, 0, 0.1));
}

.switch input {
  opacity: 0;
  width: 0;
  height: 0;
}

.slider {
  position: absolute;
  cursor: pointer;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: #ccc;
  transition: 0.4s;
  overflow: hidden;
}

.slider:before {
  position: absolute;
  content: '';
  height: 30px;
  width: 30px;
  left: 4px;
  bottom: 4px;
  background-color: white;
  transition: 0.4s;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
  z-index: 2;
}

input:checked + .slider {
  background: linear-gradient(135deg, #1976d2, #1565c0);
}

input:focus + .slider {
  box-shadow: 0 0 1px #1976d2;
}

input:checked + .slider:before {
  transform: translateX(32px);
}

.slider.round {
  border-radius: 38px;
}

.slider.round:before {
  border-radius: 50%;
}

.reminder-interval {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 40px;
  gap: 20px;
  background: white;
  padding: 25px;
  border-radius: 15px;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
  animation: slideInRight 0.5s ease;
}

@keyframes slideInRight {
  from {
    opacity: 0;
    transform: translateX(20px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

.reminder-interval:hover {
  transform: translateY(-5px);
  box-shadow: 0 15px 30px rgba(0, 0, 0, 0.15);
}

.reminder-interval span {
  font-size: 18px;
  font-weight: 500;
  color: #37474f;
  text-shadow: 0 1px 1px rgba(0, 0, 0, 0.05);
}

.reminder-interval select {
  padding: 12px 20px;
  border: 2px solid #bbdefb;
  border-radius: 8px;
  font-size: 16px;
  background-color: white;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.08);
  transition: all 0.3s ease;
}

.reminder-interval select:focus {
  border-color: #1976d2;
  outline: none;
  box-shadow: 0 0 0 3px rgba(25, 118, 210, 0.2);
}

.reminder-interval select:disabled {
  background-color: #f5f5f5;
  color: #999;
}

.progress-bar {
  height: 15px;
  background-color: #e0e0e0;
  border-radius: 8px;
  margin-bottom: 20px;
  overflow: hidden;
  max-width: 500px;
  margin-left: auto;
  margin-right: auto;
  box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.1);
  position: relative;
}

.progress-bar::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  animation: shimmer 2s infinite;
  pointer-events: none;
}

@keyframes shimmer {
  0% {
    transform: translateX(-100%);
  }
  100% {
    transform: translateX(100%);
  }
}

.progress {
  height: 100%;
  background: linear-gradient(135deg, #1976d2, #1565c0);
  background-image: linear-gradient(
    45deg,
    rgba(255, 255, 255, 0.15) 25%,
    transparent 25%,
    transparent 50%,
    rgba(255, 255, 255, 0.15) 50%,
    rgba(255, 255, 255, 0.15) 75%,
    transparent 75%,
    transparent
  );
  background-size: 40px 40px;
  border-radius: 8px;
  transition: width 1s linear;
  animation: progress-bar-stripes 2s linear infinite;
  box-shadow: 0 0 10px rgba(25, 118, 210, 0.5);
}

@keyframes progress-bar-stripes {
  from {
    background-position: 40px 0;
  }
  to {
    background-position: 0 0;
  }
}

.time-display {
  font-size: 24px;
  font-weight: 600;
  color: #1565c0;
  margin-top: 20px;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  background: white;
  padding: 15px 25px;
  border-radius: 15px;
  display: inline-block;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.03);
  }
  100% {
    transform: scale(1);
  }
}

.time-display:hover {
  transform: translateY(-5px) scale(1.05);
  box-shadow: 0 15px 30px rgba(0, 0, 0, 0.15);
}
</style>
