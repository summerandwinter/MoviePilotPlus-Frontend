<script setup lang="ts">
import NoDataFound from '@/components/NoDataFound.vue'
import api from '@/api'
import type { Collect, DownloadTask } from '@/api/types'
import TaskCardListView from '@/views/collect/TaskCardListView.vue'
import TaskRowListView from '@/views/collect/TaskRowListView.vue'
import { useDisplay } from 'vuetify'

// APP
const display = useDisplay()
const appMode = inject('pwaMode') && display.mdAndDown.value

// 路由参数
const route = useRoute()

// 查询TMDBID或标题
const keyword = route.query?.keyword?.toString() ?? ''

// 查询类型
const type = route.query?.type?.toString() ?? ''

// 搜索字段
const area = route.query?.area?.toString() ?? ''

// 搜索标题
const title = route.query?.title?.toString() ?? ''

// 搜索年份
const year = route.query?.year

// 搜索季
const season = route.query?.season?.toString() ?? ''

// 搜索站点，以,分离多个
const sites = route.query?.sites?.toString() ?? ''

// 视图类型，从localStorage中读取
const viewType = ref<string>(localStorage.getItem('MPTaskViewType') ?? 'card')

// 数据列表
const collectList = ref<Array<Collect>>([])

const taskList = ref<Array<DownloadTask>>([])

// 是否刷新过
const isRefreshed = ref(false)

// 加载进度文本
const progressText = ref('')

// 加载进度
const progressValue = ref(0)

// 加载进度SSE
const progressEventSource = ref<EventSource>()

// 错误标题
const errorTitle = ref('没有数据')

// 错误描述
const errorDescription = ref('未搜索到任何资源')

// 使用SSE监听加载进度
function startLoadingProgress() {
  progressText.value = '正在搜索，请稍候...'
  progressEventSource.value = new EventSource(`${import.meta.env.VITE_API_BASE_URL}system/progress/search`)
  progressEventSource.value.onmessage = event => {
    const progress = JSON.parse(event.data)
    if (progress) {
      progressText.value = progress.text
      progressValue.value = progress.value
    }
  }
}

// 停止监听加载进度
function stopLoadingProgress() {
  if (progressEventSource.value) progressEventSource.value?.close()
}

// 设置视图类型
function setViewType(type: string) {
  localStorage.setItem('MPTaskViewType', type)
  viewType.value = type
}

// 获取搜索列表数据
async function fetchData() {
  try {
    // 查询上次搜索结果
    collectList.value = await api.get('collect/') ?? []
    taskList.value = await api.get('task/') ?? []
    // 标记已刷新
    isRefreshed.value = true
  } catch (error) {
    console.error(error)
    return Promise.reject(error)
  }
}

// 加载数据
onMounted(() => {
  fetchData()
})

// 卸载时停止加载进度
onUnmounted(() => {
  stopLoadingProgress()
})
</script>

<template>
  <LoadingBanner v-if="!isRefreshed" class="mt-12" :text="progressText" :progress="progressValue" />
  <div v-if="viewType === 'list'">
    <NoDataFound v-if="collectList.length === 0 && isRefreshed" :error-title="errorTitle"
      :error-description="errorDescription" />
    <div v-if="collectList.length > 0 && isRefreshed">
      <TaskRowListView :items="collectList" />
    </div>
  </div>
  <div v-else>
    <NoDataFound v-if="taskList.length === 0 && isRefreshed" :error-title="errorTitle"
      :error-description="errorDescription" />
    <div v-if="taskList.length > 0 && isRefreshed">
      <TaskCardListView :items="taskList" />
    </div>
  </div>

  <!-- 视图切换 -->
  <div v-if="isRefreshed">
    <VFab v-if="viewType === 'list'" icon="mdi-view-grid" location="bottom" size="x-large" absolute app appear
      @click="setViewType('card')" :class="{ 'mb-12': appMode }" />
    <VFab v-else icon="mdi-view-list" location="bottom" size="x-large" fixed app appear @click="setViewType('list')"
      :class="{ 'mb-12': appMode }" />
  </div>
</template>
