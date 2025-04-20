<script setup lang="ts">

import api from '@/api'
import { CollectProgress, Collect, TorrentInfo } from '@/api/types'
import { formatDateToMonthShort, formatDate, formatDateDifference } from '@core/utils/formatters'
import AddDownloadDialog from '../dialog/AddDownloadDialog.vue'
import BbcodeParser from '@/components/render/BbcodeParser.vue'

// 输入参数
const props = defineProps({
  collect_id: Number,
  title: String,
})
const tab = ref('option-1')

// 注册事件
const emit = defineEmits(['close'])


// 加载状态
const resourceLoading = ref(false)

const collect = ref<Collect>()
const progress = ref<Array<CollectProgress>>([])


// 种子元数据
const torrent = ref<TorrentInfo>()

// 添加下载对话框
const addDownloadDialog = ref(false)

// 添加下载成功
function addDownloadSuccess(url: string) {
  addDownloadDialog.value = false
}

// 添加下载失败
function addDownloadError(error: string) {
  addDownloadDialog.value = false
}

// 调用API，查询站点资源
async function getDetailInfo() {
  resourceLoading.value = true
  try {
    collect.value = await api.get(`collect/${props?.collect_id}`)
  } catch (error) {
    console.error(error)
  }
  resourceLoading.value = false
}

async function getProgressInfo() {
  resourceLoading.value = true
  try {
    progress.value = await api.get(`collect/progress/${props?.collect_id}`)
  } catch (error) {
    console.error(error)
  }
  resourceLoading.value = false
}

// 装载时查询站点图标
onMounted(() => {
  getDetailInfo()
  getProgressInfo()
})
</script>
<template>
  <VDialog scrollable fullscreen :scrim="false" transition="dialog-bottom-transition">
    <VCard>
      <!-- Toolbar -->
      <div>
        <VToolbar color="primary">
          <VToolbarTitle>{{ `浏览 - ${collect?.name}` }}</VToolbarTitle>
          <VSpacer />
          <VToolbarItems>
            <VBtn icon variant="plain" @click="emit('close')" class="me-3">
              <VIcon size="large" color="white" icon="ri-close-line" />
            </VBtn>
          </VToolbarItems>
        </VToolbar>
      </div>
  <VCardItem>  
    1111
  <div class="max-w-8xl mx-auto px-4">
    <template>
      ccc
      <div class="vue-media-back absolute left-0 top-0 w-full h-96">ccc
        <VImg class="h-96" position="top" src="https://vcover-hz-pic.puui.qpic.cn/vcover_hz_pic/0/mzc00200fhd6k651732848671980/0" cover />
      </div>
      <div class="vue-media-back absolute left-0 top-0 w-full h-96" />
    </template>
    
  </div>
  2222
</VCardItem>  
    </VCard>
    <!-- 添加下载对话框 -->
    
    
    <AddDownloadDialog
      v-if="addDownloadDialog"
      v-model="addDownloadDialog"
      :torrent="torrent"
      @done="addDownloadSuccess"
      @error="addDownloadError"
      @close="addDownloadDialog = false"
    />
  </VDialog>
</template>

<style lang="scss" scoped>
.v-table th {
  white-space: nowrap;
}
</style>
