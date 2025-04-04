<script setup lang="ts">
import { debounce } from 'lodash-es'
import { VForm } from 'vuetify/components/VForm'
import { useAuthStore, useUserStore } from '@/stores'
import { authState, userState } from '@/stores/types'
import { requiredValidator } from '@/@validators'
import api from '@/api'
import router from '@/router'
import logo from '@images/logo.png'
import { useTheme } from 'vuetify'
import { checkPrefersColorSchemeIsDark } from '@/@core/utils'
import { urlBase64ToUint8Array } from '@/@core/utils/navigator'
import { saveLocalTheme } from '@/@core/utils/theme'

// 主题
const { global: globalTheme } = useTheme()
// 认证 Store
const authStore = useAuthStore()
//用户 Store
const userStore = useUserStore()

// 表单
const form = ref({
  username: '',
  password: '',
  otp_password: '',
  remember: true,
})

const refForm = ref<InstanceType<typeof VForm> | null>(null)

// 密码输入
const isPasswordVisible = ref(false)

// 错误信息
const errorMessage = ref('')

// 背景图片 URL 和预加载 URL
const backgroundImages = ref<string[]>([])
const activeImageIndex = ref(0)

// 是否开启双重验证
const isOTP = ref(false)

// 用户名称输入框
const usernameInput = ref()

// Interval定时器
let intervalTimer: NodeJS.Timeout | null = null

// 获取背景图片
async function fetchBackgroundImage() {
  try {
    backgroundImages.value = await api.get('/login/wallpapers')
  } catch (e) {
    console.log(e)
  }
}

// 查询是否开启双重验证
const fetchOTP = debounce(async () => {
  const userid = usernameInput.value?.value
  if (!userid) {
    isOTP.value = false
    return
  }
  api
    .get(`/user/otp/${userid}`)
    .then((response: any) => {
      isOTP.value = response.success
    })
    .catch((error: any) => {
      console.log(error)
    })
}, 500)

// 获取用户主题配置
async function fetchThemeConfig() {
  const response = await api.get('/user/config/Layout')
  if (response && response.data && response.data.value) {
    return response.data.value?.theme
  }
  return null
}

// 生效主题
async function setTheme() {
  let themeValue = (await fetchThemeConfig()) || localStorage.getItem('theme') || 'light'
  const autoTheme = checkPrefersColorSchemeIsDark() ? 'dark' : 'light'
  globalTheme.name.value = themeValue === 'auto' ? autoTheme : themeValue
  // 存储主题到本地
  saveLocalTheme(themeValue, globalTheme)
}

// 订阅推送通知
async function subscribeForPushNotifications() {
  if ('serviceWorker' in navigator && 'PushManager' in window) {
    const registration = await navigator.serviceWorker.ready
    // 获取订阅信息
    const subscription = await registration.pushManager.getSubscription().then(function (subscription) {
      if (subscription === null) {
        const convertedVapidKey = urlBase64ToUint8Array(import.meta.env.VITE_PUBLIC_VAPID_KEY)
        return registration.pushManager.subscribe({
          userVisibleOnly: true,
          applicationServerKey: convertedVapidKey,
        })
      } else {
        return subscription
      }
    })
    // 发送订阅请求
    try {
      await api.post('/message/webpush/subscribe', subscription)
    } catch (e) {
      console.log(e)
    }
  }
}

// 登录后处理
async function afterLogin(superuser: boolean) {
  // 生效主题配置
  await setTheme()
  // 跳转到首页或回原始页面
  router.push(authStore.originalPath ?? '/')
  // 订阅推送通知
  if (superuser) await subscribeForPushNotifications()
}

// 登录获取token事件
function login() {
  errorMessage.value = ''

  // 进行表单校验
  if (!form.value.username || !form.value.password || (isOTP.value && !form.value.otp_password)) {
    return
  }
  // 用户名密码
  const formData = new FormData()

  formData.append('username', form.value.username)
  formData.append('password', form.value.password)
  formData.append('otp_password', form.value.otp_password)

  // 请求token
  api
    .post('/login/access-token', formData, {
      headers: {
        Accept: 'application/json', // 设置 Accept 类型
      },
    })
    .then((response: any) => {
      const authPayLoad: authState = {
        token: response.access_token,
        remember: form.value.remember,
      }

      const userPayload: userState = {
        superUser: response.super_user,
        userID: response.user_id,
        userName: response.user_name,
        avatar: response.avatar,
        level: response.level,
        permissions: response.permissions,
      }

      authStore.login(authPayLoad)
      userStore.loginUser(userPayload)

      // 登录后处理
      afterLogin(userPayload.superUser)
    })
    .catch((error: any) => {
      // 登录失败，显示错误提示
      if (!error.response) errorMessage.value = '登录失败，请检查网络连接！'
      else if (error.response.status === 401) errorMessage.value = '登录失败，请检查用户名、密码或双重验证是否正确！'
      else if (error.response.status === 403) errorMessage.value = '登录失败，您没有权限访问！'
      else if (error.response.status === 500) errorMessage.value = '登录失败，服务器错误！'
      else errorMessage.value = `登录失败 ${error.response.status}，请检查用户名、密码或双重验证码是否正确！`
    })
}

// 初始化背景图片轮循
function startBackgroundRotation() {
  intervalTimer = setInterval(() => {
    activeImageIndex.value = (activeImageIndex.value + 1) % backgroundImages.value.length
  }, 5000) // 每5秒切换一次图片
}

// 自动登录
onMounted(async () => {
  // 获取token和remember状态
  const token = authStore.token
  const remember = authStore.remember

  // 如果token存在，且保持登录状态为true，则跳转到首页
  if (token && remember) {
    router.push('/')
  } else {
    // 获取背景图片
    await fetchBackgroundImage()
    if (backgroundImages.value.length > 1) {
      startBackgroundRotation()
    }
  }
})

onUnmounted(() => {
  if (intervalTimer) clearInterval(intervalTimer)
})
</script>

<template>
  <!-- 当前背景图片 -->
  <div class="relative flex min-h-screen flex-col bg-gray-900 items-center justify-center">
    <div>
      <div
        v-for="(imageUrl, index) in backgroundImages"
        class="absolute-top-shift absolute inset-0 bg-cover bg-center transition-opacity duration-300 ease-in"
        :class="{ 'opacity-100': index === activeImageIndex, 'opacity-0': index !== activeImageIndex }"
      >
        <VImg :src="imageUrl" class="absolute inset-0 transition-opacity duration-1000" cover position="center top" />
        <div
          class="absolute inset-0"
          style="background-image: linear-gradient(rgba(45, 55, 72, 47%) 0%, rgb(26, 32, 46) 100%)"
        />
      </div>
    </div>
    <!-- 登录表单 -->
    <div class="auth-wrapper d-flex align-center justify-center">
      <VCard class="auth-card px-7 py-3 w-full h-full rounded-lg opacity-85" max-width="24rem">
        <VCardItem class="justify-center">
          <template #prepend>
            <div class="d-flex pe-0">
              <VImg :src="logo" width="64" height="64" />
            </div>
          </template>
          <VCardTitle class="font-weight-bold text-2xl text-uppercase"> MoviePilot </VCardTitle>
        </VCardItem>
        <VCardText>
          <VForm ref="refForm" autocomplete="on" @submit.prevent="() => {}">
            <VRow>
              <!-- username -->
              <VCol cols="12">
                <VTextField
                  ref="usernameInput"
                  v-model="form.username"
                  label="用户名"
                  type="text"
                  name="username"
                  autocomplete="username"
                  :rules="[requiredValidator]"
                  @input="fetchOTP"
                />
              </VCol>
              <!-- password -->
              <VCol cols="12">
                <VTextField
                  v-model="form.password"
                  label="密码"
                  :type="isPasswordVisible ? 'text' : 'password'"
                  name="current-password"
                  autocomplete="current-password"
                  :append-inner-icon="isPasswordVisible ? 'mdi-eye-off-outline' : 'mdi-eye-outline'"
                  :rules="[requiredValidator]"
                  @click:append-inner="isPasswordVisible = !isPasswordVisible"
                />
              </VCol>
              <VCol cols="12">
                <VTextField v-if="isOTP" v-model="form.otp_password" label="双重验证码" type="input" />
                <!-- remember me checkbox -->
                <div class="d-flex align-center justify-space-between flex-wrap">
                  <VCheckbox v-model="form.remember" label="保持登录" required />
                </div>
              </VCol>
              <VCol cols="12">
                <!-- login button -->
                <VBtn block type="submit" @click="login"> 登录 </VBtn>
                <VAlert v-if="errorMessage" type="error" variant="tonal" class="mt-3">
                  {{ errorMessage }}
                </VAlert>
              </VCol>
            </VRow>
          </VForm>
        </VCardText>
      </VCard>
    </div>
  </div>
</template>

<style lang="scss">
@use '@core/scss/pages/page-auth.scss';

.v-card-item__prepend {
  padding-inline-end: 0 !important;
}

.absolute-top-shift {
  inset-block-start: calc(-4rem - env(safe-area-inset-top));
}

.auth-wrapper {
  overflow: hidden;
  block-size: auto;
}
</style>
