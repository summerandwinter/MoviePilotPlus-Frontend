<script lang="ts" setup>
import api from '@/api'
import type { DownloadTask, Progress } from '@/api/types'
import { formatFileSize } from '@/@core/utils/formatters'

// 输入参数
const props = defineProps({
  info: Object as PropType<DownloadTask>,
  progress: Array as PropType<Progress[]>,
})

// 是否显示卡片
const cardState = ref(true)

// 进度条
function getPercentage() {
  for (const progress of props.progress) {
    if (progress.task_id === props.info?.id) return progress.percent ?? 0
  }
  return 0
}

// 速度
function getSpeedText() {
  for (const progress of props.progress) {
    if (progress.task_id === props.info?.id) 
    return `${formatFileSize(progress?.downloaded_size || 0)} / ${formatFileSize(progress?.total_size || 0)}  ↓ ${formatFileSize(progress?.speed)}/s ${
      progress?.percent
  }`
  }
  return `${formatFileSize(props.info?.downloaded_size || 0)} / ${formatFileSize(props.info?.total_size || 0)}  ↓ ${formatFileSize(props.info?.speed)}/s ${
    props.info?.percent
  }`
}

// 下载状态
const isDownloading = ref(props.info?.status === 'downloading')

// 监听props.info?.state的变化
watch(
  () => props.info?.status,
  newValue => {
    isDownloading.value = newValue === 'downloading'
  },
)

// 图片是否加载完成
const imageLoaded = ref(false)

// 图片加载完成响应
function imageLoadHandler() {
  imageLoaded.value = true
}

// 计算文本类
function getTextClass() {
  return imageLoaded.value ? 'text-white' : ''
}

// 下载状态控制
async function toggleDownload() {
  const operation = isDownloading.value ? 'stop' : 'start'
  try {
    const result: { [key: string]: any } = await api.get(`task/${operation}/${props.info?.id}`)
    if (result.success) isDownloading.value = !isDownloading.value
  } catch (error) {
    console.error(error)
  }
}

// 删除下截
async function deleteDownload() {
  try {
    await api.delete(`download/${props.info?.id}`)
    cardState.value = false
  } catch (error) {
    console.error(error)
  }
}
</script>

<template>
  <VCard v-if="cardState" :key="props.info?.id">
    <template #image>
      <VImg :src="props.info?.poster" aspect-ratio="2/3" cover class="brightness-50" @load="imageLoadHandler" />
    </template>

    <VCardTitle class="break-words whitespace-normal" :class="getTextClass()">
      {{ props.info?.name }}
    </VCardTitle>

    <VCardSubtitle class="break-words whitespace-normal" :class="getTextClass()">
      {{ props.info?.name }}
    </VCardSubtitle>

    <VCardText class="text-subtitle-1 pt-3 pb-1" :class="getTextClass()">
      {{ getSpeedText() }}
    </VCardText>

    <VCardText v-if="getPercentage() > 0" :class="getTextClass()">
      <VProgressLinear :model-value="getPercentage()" />
    </VCardText>

    <VCardActions class="justify-space-between">
      <VBtn :icon="`${isDownloading ? 'mdi-pause' : 'mdi-play'}`" @click="toggleDownload" />
      <VBtn color="error" icon="mdi-trash-can-outline" @click="deleteDownload" />
    </VCardActions>
  </VCard>
</template>
