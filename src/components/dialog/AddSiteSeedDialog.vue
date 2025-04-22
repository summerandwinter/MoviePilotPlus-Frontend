<script setup lang="ts">
import { useToast } from 'vue-toast-notification'
import api from '@/api'
import { doneNProgress, startNProgress } from '@/api/nprogress'
import type { DownloaderConf, Collect, SiteSeed, Site, TransferDirectoryConf } from '@/api/types'
import { formatFileSize } from '@/@core/utils/formatters'
import { VCardTitle, VChip } from 'vuetify/lib/components/index.mjs'

// 输入参数
const props = defineProps({
  title: String,
  siteSeedList: Array as PropType<SiteSeed[]>,
  collect: Object as PropType<Collect>
})

// 定义成功和失败事件
const emit = defineEmits(['done', 'error', 'close'])

// 提示框
const $toast = useToast()
// 所有站点
const allSites = ref<Site[]>([])
// 选中的站点
const selectedSites = ref<string[]>([])
// 选择的下载器
const selectedDownloader = ref<string | null>(null)

// 选择的保存目录
const selectedDirectory = ref<string | null>(null)

// 下载器
const downloaders = ref<DownloaderConf[]>([])

// 所有目录设置
const directories = ref<TransferDirectoryConf[]>([])

// 是否正在加载
const loading = ref(false)

// 计算按钮图标
const icon = computed(() => (loading.value ? 'mdi-progress-download' : 'mdi-download'))

// 计算按钮文字
const buttonText = computed(() => (loading.value ? '下载中...' : '开始下载'))

// 加载目录设置
async function loadDirectories() {
  try {
    const result: { [key: string]: any } = await api.get('site/')
    directories.value = result.data?.value ?? []
  } catch (error) {
    console.log(error)
  }
}
async function querySites() {
  try {
    const data: Site[] = await api.get('site/')
    allSites.value = data
    // 过滤站点，只有启用的站点才显示
    // allSites.value = data.filter(item => {
    //   if (props.siteSeedList) {
    //     return props.siteSeedList.forEach(siteSeed => {
    //       if (siteSeed.site_id === item.id) {
    //         return true
    //       }
    //     })
    //   }
    // })
  } catch (error) {
    console.log(error)
  }
}

// 获取保存目录
const targetDirectories = computed(() => {
  const downloadDirectories = directories.value.map(item => item.download_path)
  return [...new Set(downloadDirectories)]
})

// 调用API查询下载器设置
async function loadDownloaderSetting() {
  try {
    downloaders.value = await api.get('download/clients')
  } catch (error) {
    console.log(error)
  }
}

// 下载器可选项
const downloaderOptions = computed(() => {
  return downloaders.value.map(item => ({
    title: item.name,
    value: item.name,
  }))
})

// 添加下载
async function addDownload() {
  startNProgress()
  loading.value = true
  try {
    let result: { [key: string]: any }

  
  } catch (error) {
    console.error(error)
  }
  loading.value = false
  doneNProgress()
}

onMounted(() => {
  querySites()
  
})
</script>
<template>
  <VDialog max-width="45rem" scrollable>
    <VCard>
      <VCardItem>
        <VCardTitle>添加做种任务</VCardTitle>
        <DialogCloseBtn @click="emit('close')" />
      </VCardItem>
      <VDivider />
      <VCardText>
        <v-chip-group
        v-model="selectedSites"
        column
        multiple
      >
        <v-chip v-for="site in allSites" :key="site.id" :value="site.id"
          :text="site.name"
          variant="outlined"
          filter
        ></v-chip>
      </v-chip-group>
      </VCardText>
      <VCardText class="text-center">
        <VBtn
          variant="elevated"
          :disabled="loading"
          @click="addDownload"
          prepend-icon="mdi-progress-upload"
          class="px-5"
          size="large"
        >
          添加任务
        </VBtn>
      </VCardText>
    </VCard>
  </VDialog>
</template>
