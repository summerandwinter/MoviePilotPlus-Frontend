<script lang="ts" setup>
import type { PropType } from 'vue'
import { formatFileSize } from '@/@core/utils/formatters'
import api from '@/api'
import type { Collect } from '@/api/types'
import AddDownloadDialog from '../dialog/AddDownloadDialog.vue'

// 输入参数
const props = defineProps({
  task: Object as PropType<Collect>,
})

// 更多来源界面
const showMoreTorrents = ref(false)

// 任务信息
const task = ref(props.task)

// 站点图标
const siteIcon = ref('')

// 存储是否已经下载过的记录
const downloaded = ref<string[]>([])

// 添加下载对话框
const addDownloadDialog = ref(false)

// 查询站点图标
async function getSiteIcon() {
  try {
    siteIcon.value = task.value?.cover
  } catch (error) {
    console.error(error)
  }
}

// 询问并添加下载
async function handleAddDownload() {
  // 打开下载对话框
  addDownloadDialog.value = true
}

// 添加下载成功
function addDownloadSuccess(url: string) {
  addDownloadDialog.value = false
  // 添加下载成功
  downloaded.value.push(url)
}

// 添加下载失败
function addDownloadError(error: string) {
  addDownloadDialog.value = false
}

// 打开种子详情页面
function openTorrentDetail() {
  window.open(task.value?.page_url, '_blank')
}

// 下载种子文件
async function downloadTorrentFile() {
  window.open(task.value?.enclosure, '_blank')
}

// 促销Chip类
function getVolumeFactorClass(downloadVolume: number, uploadVolume: number) {
  if (downloadVolume === 0) return 'text-white bg-lime-500'
  else if (downloadVolume < 1) return 'text-white bg-green-500'
  else if (uploadVolume !== 1) return 'text-white bg-sky-500'
  else return 'text-white bg-gray-500'
}

// 装载时查询站点图标
onMounted(() => {
  getSiteIcon()
})
</script>

<template>
  <div>
    
    <VListItem
      @click="handleAddDownload"
      :variant="downloaded.includes(task?.enclosure || '') ? 'outlined' : 'flat'"
    >
    <template #image>
      <VImg :src="siteIcon" aspect-ratio="2/3" cover class="brightness-50" />
    </template>

      <template v-if="!showMoreTorrents" #prepend>
        <VListItemMedia class="pr-2"><VImg :src="siteIcon" :width="60" 
  aspect-ratio="9/16"
  cover/></VListItemMedia>
        
      </template>
      <VListItemTitle class="break-words overflow-visible whitespace-break-spaces">
        {{ task?.name }}
        <span class="text-green-700 ms-2 text-sm">↑{{ task?.episodes_downloaded }}</span>
        <span class="text-orange-700 ms-2 text-sm">↓{{ task?.episodes_total }}</span>
      </VListItemTitle>
      <VListItemSubtitle> 【{{ task?.type }}】{{ task?.sub_title }} </VListItemSubtitle>
      <div class="pt-2">
        
        <VChip v-if="task?.team" variant="elevated" size="small" class="me-1 mb-1 text-white bg-red-500">
          {{ task?.team }}
        </VChip>
        <VChip v-if="task?.copyright" variant="elevated" size="small" class="me-1 mb-1 text-white bg-red-500">
          {{ task?.copyright }}
        </VChip>
        
      </div>
      <template #append>
        <div class="me-n3">
          <IconBtn>
            <VIcon icon="mdi-dots-vertical" />
            <VMenu activator="parent" close-on-content-click>
              <VList>
                <VListItem variant="plain" @click="openTorrentDetail()">
                  <template #prepend>
                    <VIcon icon="mdi-information" />
                  </template>
                  <VListItemTitle>查看详情</VListItemTitle>
                </VListItem>
                <VListItem
                  variant="plain"
                  @click="downloadTorrentFile()"
                >
                  <template #prepend>
                    <VIcon icon="mdi-download" />
                  </template>
                  <VListItemTitle>下载种子文件</VListItemTitle>
                </VListItem>
              </VList>
            </VMenu>
          </IconBtn>
        </div>
      </template>
    </VListItem>
    <AddDownloadDialog
      v-if="addDownloadDialog"
      v-model="addDownloadDialog"
      :title="`${task?.title || task?.name} ${task?.episodes_downloaded}/${task?.episodes_total}`"
      :task="task"
      @done="addDownloadSuccess"
      @error="addDownloadError"
      @close="addDownloadDialog = false"
    />
  </div>
</template>
