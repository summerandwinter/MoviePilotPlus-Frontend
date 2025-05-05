<script lang="ts" setup>
import api from '@/api'
import type { DownloadTask, Progress } from '@/api/types'
import { formatFileSize } from '@/@core/utils/formatters'
import { downloadStatus} from '@/api/constants'
import tencentImage from '@images/logos/tencent-white.png'
import ProgressInfoDialog from '@/components/dialog/ProgressInfoDialog.vue'
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
  width: String,
  height: String,
})

// 是否显示卡片
const cardState = ref(true)
const showProgressInfo = ref(false)

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
// 来源角标字典
const siteIconDict: { [key: string]: any } = {
  Tencent: tencentImage
}
function showProgressInfoDialog() {
  showProgressInfo.value = true
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
// 角标颜色
function getChipColor() {
  const status = getSatus()
  if (status === downloadStatus.DownloadError) {
    return 'border-red-500 bg-red-600'
  } else if (status === downloadStatus.Downloading) {
    return 'bg-green-500 border-green-600'
  } else {
    return 'border-purple-600 bg-purple-600'
  }
}
function getSatus() {
  if (props.progress?.length) {
    for (const progress of props.progress) {
      if (Number(progress.task_id) === props.info?.id) {
        return downloadStatus[progress.state as keyof typeof downloadStatus]
      }
    }
  }
  return downloadStatus[props.info.status as keyof typeof downloadStatus]
}

function showToggleBtn() {
  const status = getSatus()
  if (status === downloadStatus.DownloadCreated) {
    return true
  } else if (getSatus() === downloadStatus.DownloadPending) {
    return true
  } else if (getSatus() === downloadStatus.DownloadPending) {
    return true
  } else if (getSatus() === downloadStatus.Downloading) {
    return true
  } else if (getSatus() === downloadStatus.DownloadError) {
    return true
  } else if (getSatus() === downloadStatus.DownloadStop) {
    return true
  } else {
    return false
  }
}

function downloadBtnIcon() {
  const showBtn = showToggleBtn()
  const status = getSatus()
  if (showBtn) {
    return status === downloadStatus.Downloading ? 'mdi-pause' : 'mdi-play'
  } else {
    return ''
  }
}

function downloading() {
  const status = getSatus()
  return status === downloadStatus.Downloading
}
// 下载状态
const isDownloading = ref(props.info?.status === downloadStatus.Downloading)

// 监听props.info?.state的变化
watch(
  () => props.info?.status,
  newValue => {
    isDownloading.value = newValue === downloadStatus.Downloading
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
const taskCardRef = ref<HTMLElement | null>(null)
</script>

<template>
  <VHover>
    
    <template #default="hover">
      <div ref="taskCardRef">
      <VCard v-if="cardState" :key="props.info?.id" class="glass-card" v-bind="hover.props" 
      :height="props.height" :width="props.width" @click.stop="showProgressInfoDialog"> 
        <VChip
              variant="elevated"
              size="small"
              :class="getChipColor()"
              class="absolute right-2 top-2 bg-opacity-50 shadow-md text-white font-bold"
            >
              {{ getSatus() }}
            </VChip>
            <VAvatar 
                size="24"
                density="compact"
                class="absolute top-1 left-4"
                tile
                v-show="hover.isHovering"
              >
                <VImg cover :src="siteIconDict[props.info.site]" class="shadow-lg" />
              </VAvatar>
        <template #image>
          <VImg :src="props.info?.poster" aspect-ratio="2/3" cover class="brightness-50 card-bg" @load="imageLoadHandler" />
        </template>
        <VCardItem class="card-content pt-7">
          <VCardTitle class="truncate" :class="getTextClass()">
          {{ props.info?.name }}
          <VTooltip
            activator="parent"
            location="bottom"
          >{{ props.info?.name }}</VTooltip>
        </VCardTitle>

        <!-- <VCardSubtitle class="break-words whitespace-normal" :class="getTextClass()">
          {{ props.info?.name }}
        </VCardSubtitle> -->
        <template v-if="isDownloading">
          <VCardItem  class="text-subtitle-2 pt-3 pb-1 pl-0 pr-0" :class="getTextClass()">
            {{ getSpeedText() }}
            <VProgressLinear :model-value="getPercentage()" />
          </VCardItem>
        </template>
        <template v-else>
          <VCardItem  class="text-subtitle-2 pt-4 pb-1 pl-0 pr-0" :class="getTextClass()">
            <VChip variant="outlined" size="x-small" label class="mr-1 text-white font-bold" v-if="props.info.type">{{props.info.type}}</VChip>
            <VChip variant="outlined" size="x-small" label class="mr-1 text-white font-bold" v-if="props.info.video_codec">{{props.info.video_codec}}</VChip>
            <VChip variant="outlined" size="x-small" label class="mr-1 text-white font-bold" v-if="props.info.hdr_format">{{props.info.hdr_format}}</VChip>
            <VChip variant="outlined" size="x-small" label class="mr-1 text-white font-bold" v-if="props.info.audio_codec">{{props.info.audio_codec}}</VChip>
            <VChip variant="outlined" size="x-small" label class="mr-1 text-white font-bold" v-if="props.info.bit_depth">{{props.info.bit_depth}}bits</VChip>
            
          </VCardItem>
        </template>
        

        <!-- <VCardItem v-if="getPercentage() > 0" class="text-subtitle-2 pt-0 pb-0 pl-0 pr-0" :class="getTextClass()">
          <VProgressLinear :model-value="getPercentage()" />
        </VCardItem> -->
        <!-- <VCardText v-if="getPercentage() > 0" class="pt-3 pb-1 pl-0 pr-0" :class="getTextClass()">
          <VProgressLinear :model-value="getPercentage()" />
        </VCardText> -->

        <VCardActions class="justify-space-between  pt-2 pb-0 pl-0 pr-0">
          <VBtn :readonly="!showToggleBtn()" :icon="downloadBtnIcon()" @click.stop="toggleDownload" />
          <VBtn color="error" icon="mdi-trash-can-outline" @click.stop="deleteDownload" />
        </VCardActions>
        </VCardItem>
        
      </VCard>
      <ProgressInfoDialog v-if="showProgressInfo" v-model="showProgressInfo" type="task" :id="props.info.id" :name="props.info.name"
      @close="showProgressInfo = false" />
    </div>
    </template>
  
  </VHover>
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