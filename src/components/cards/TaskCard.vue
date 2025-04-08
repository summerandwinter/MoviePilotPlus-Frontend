<script lang="ts" setup>
import api from '@/api'
import type { DownloadTask, Progress } from '@/api/types'
import { formatFileSize } from '@/@core/utils/formatters'

// 输入参数
const props = defineProps({
  info: {
    type: Object as PropType<DownloadTask>,
    default: () => {},
    required: true,
  },
  progress: {
    type: Array as PropType<Progress[]>,
    default: () => []
  },
})

// 是否显示卡片
const cardState = ref(true)

// 进度条
function getPercentage() {
  if (!props.progress?.length) return 0
  for (const progress of props.progress) {
    if (Number(progress.task_id) === props.info?.id) {
      return progress.percent ?? 0
    }
  }
  return 0
}

// 速度
function getSpeedText() {
  if (props.progress?.length) {
    for (const progress of props.progress) {
      if (Number(progress.task_id) === props.info?.id) 
      return `${formatFileSize(progress?.downloaded_size || 0)} / ${formatFileSize(progress?.total_size || 0)}  ↓ ${formatFileSize(progress?.speed)}/s`
    }
  }
  return `${formatFileSize(props.info?.downloaded_size || 0)} / ${formatFileSize(props.info?.total_size || 0)}  ↓ ${formatFileSize(props.info?.speed)}/s`
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
  <VCard v-if="cardState" :key="props.info?.id" class="glass-card">
    <template #image>
      <VImg :src="props.info?.poster" aspect-ratio="2/3" cover class="brightness-50 card-bg" @load="imageLoadHandler" />
    </template>
    <VCardItem class="card-content">
      <VCardTitle class="break-words whitespace-normal" :class="getTextClass()">
      {{ props.info?.name }}
    </VCardTitle>

    <!-- <VCardSubtitle class="break-words whitespace-normal" :class="getTextClass()">
      {{ props.info?.name }}
    </VCardSubtitle> -->

    <VCardItem v-if="getPercentage() > 0" class="text-subtitle-2 pt-3 pb-1 pl-0 pr-0" :class="getTextClass()">
      {{ getSpeedText() }}
    </VCardItem>
    <VCardItem v-if="getPercentage() > 0" class="text-subtitle-2 pt-3 pb-3 pl-0 pr-0" :class="getTextClass()">
      <VProgressLinear :model-value="getPercentage()" />
    </VCardItem>
    <!-- <VCardText v-if="getPercentage() > 0" class="pt-3 pb-1 pl-0 pr-0" :class="getTextClass()">
      <VProgressLinear :model-value="getPercentage()" />
    </VCardText> -->

    <VCardActions class="justify-space-between  pt-2 pb-0 pl-0 pr-0">
      <VBtn :icon="`${isDownloading ? 'mdi-pause' : 'mdi-play'}`" @click="toggleDownload" />
      <VBtn color="error" icon="mdi-trash-can-outline" @click="deleteDownload" />
    </VCardActions>
    </VCardItem>
    
  </VCard>
</template>
<style scoped>
  /* 卡片整体样式 */
  .glass-card {
    position: relative;
    overflow: hidden;
    
    /* 创建伪元素实现玻璃效果 */
    &::before {
      content: '';
      position: absolute;
      inset: 0;
      background: rgba(255, 255, 255, 0.1);
      backdrop-filter: blur(12px);
      -webkit-backdrop-filter: blur(12px);
      border: 1px solid rgba(255, 255, 255, 0.2);
    }
  }
  
  /* 卡片背景图处理 */
  .card-bg {
    position: absolute;
    width: 100%;
    height: 100%;
    filter: blur(8px) brightness(0.8);
    transform: scale(1.05); /* 防止模糊边缘漏出 */
    
    /* 覆盖 Vuetify 图片容器样式 */
    :deep(.v-img__img) {
      object-fit: cover;
    }
  }
  
  /* 卡片内容定位 */
  .card-content {
    position: relative;
    z-index: 2;
    height: 100%;
    /* display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center; */
    /* color: white; */
    /* text-shadow: 0 2px 4px rgba(0, 0, 0, 0.2); */
  }
  
  /* 按钮样式增强 */
  :deep(.v-btn) {
    /* backdrop-filter: blur(4px);
    background: rgba(255, 255, 255, 0.1) !important;
    border: 1px solid rgba(255, 255, 255, 0.3) !important;
     */
    &:hover {
      background: rgba(255, 255, 255, 0.2) !important;
    }
  }

  </style>