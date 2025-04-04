<script lang="ts" setup>
import { useToast } from 'vue-toast-notification'
import MediaIdSelector from '../misc/MediaIdSelector.vue'
import api from '@/api'
import { storageOptions, transferTypeOptions } from '@/api/constants'
import { numberValidator } from '@/@validators'
import { useDisplay } from 'vuetify'
import ProgressDialog from './ProgressDialog.vue'
import { FileItem, TransferDirectoryConf, TransferForm } from '@/api/types'

// 显示器宽度
const display = useDisplay()

// 输入参数
const props = defineProps({
  logids: Array<number>,
  items: Array<FileItem>,
  target_storage: String,
  target_path: String,
})

// 从 provide 中获取全局设置
const globalSettings: any = inject('globalSettings')

// 当前识别类型
const mediaSource = ref(globalSettings.data?.RECOGNIZE_SOURCE || 'themoviedb')

// 定义事件
const emit = defineEmits(['done', 'close'])

// 生成1到100季的下拉框选项
const seasonItems = ref(
  Array.from({ length: 101 }, (_, i) => i).map(item => ({
    title: `第 ${item} 季`,
    value: item,
  })),
)

// 提示框
const $toast = useToast()

// TMDB选择对话框
const mediaSelectorDialog = ref(false)

// 加载进度SSE
const progressEventSource = ref<EventSource>()

// 整理进度条
const progressDialog = ref(false)

// 整理进度文本
const progressText = ref('正在处理 ...')

// 整理进度
const progressValue = ref(0)

// 标题
const dialogTitle = computed(() => {
  if (props.items) {
    if (props.items.length > 1) return `整理 - 共 ${props.items.length} 项`
    return `整理 - ${props.items[0].path}`
  } else if (props.logids) {
    return `整理 - 共 ${props.logids.length} 项`
  }
  return '手动整理'
})

// 禁用指定集数
const disableEpisodeDetail = computed(() => {
  if (props.items) {
    if (transferForm.episode_format) return false
    return !(props.items.length === 1 && props.items[0].type !== 'dir')
  }
})

// 表单
const transferForm = reactive<TransferForm>({
  fileitem: {} as FileItem,
  logid: 0,
  target_storage: props.target_storage ?? 'local',
  transfer_type: '',
  target_path: '',
  min_filesize: 0,
  scrape: false,
  from_history: false,
})

// 所有媒体库目录
const directories = ref<TransferDirectoryConf[]>([])
// 查询目录
async function loadDirectories() {
  try {
    const result: { [key: string]: any } = await api.get('system/setting/Directories')
    directories.value = result.data?.value ?? []
  } catch (error) {
    console.log(error)
  }
}

// 目的目录下拉框
const targetDirectories = computed(() => {
  const libraryDirectories = directories.value.map(item => item.library_path)
  return [...new Set(libraryDirectories)]
})

// 监听目的路径变化，配置默认值
watch(
  () => transferForm.target_path,
  async newPath => {
    if (newPath) {
      const directory = directories.value.find(item => item.library_path === newPath)
      if (directory) {
        transferForm.target_storage = directory.library_storage ?? 'local'
        transferForm.transfer_type = transferForm.transfer_type || directory.transfer_type
        transferForm.scrape = directory.scraping ?? false
        transferForm.library_category_folder = directory.library_category_folder ?? false
        transferForm.library_type_folder = directory.library_type_folder ?? false
      } else {
        transferForm.transfer_type = transferForm.transfer_type || 'copy'
        transferForm.scrape = false
        transferForm.library_category_folder = false
        transferForm.library_type_folder = false
      }
    } else {
      // 路径为空时, 恢复到`自动`条件
      transferForm.transfer_type = ''
      transferForm.library_type_folder = undefined
      transferForm.library_category_folder = undefined
    }
  },
)

// 整理文件
async function handleTransfer(item: FileItem, background: boolean = false) {
  transferForm.fileitem = item
  transferForm.logid = 0
  try {
    const result: { [key: string]: any } = await api.post(`transfer/manual?background=${background}`, transferForm)
    if (!result.success) $toast.error(result.message)
    else if (background) $toast.success(`文件 ${item.name} 已加入整理队列！`)
  } catch (e) {
    console.log(e)
  }
}

// 整理日志
async function handleTransferLog(logid: number, background: boolean = false) {
  transferForm.logid = logid
  transferForm.fileitem = {} as FileItem
  try {
    const result: { [key: string]: any } = await api.post(`transfer/manual?background=${background}`, transferForm)
    if (!result.success) $toast.error(result.message)
    else if (background) $toast.success(`历史记录 ${logid} 已加入整理队列！`)
  } catch (e) {
    console.log(e)
  }
}

// 使用SSE监听加载进度
function startLoadingProgress() {
  progressText.value = '请稍候 ...'
  progressEventSource.value = new EventSource(`${import.meta.env.VITE_API_BASE_URL}system/progress/filetransfer`)
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
  progressEventSource.value?.close()
}

// 整理文件
async function transfer(background: boolean = false) {
  if (!props.logids && !props.items) return

  // 显示进度条
  progressDialog.value = true

  if (!background) {
    // 开始监听进度
    startLoadingProgress()
  }

  // 文件整理
  if (props.items) {
    for (const item of props.items) {
      await handleTransfer(item, background)
    }
  }

  // 日志整理
  if (props.logids) {
    for (const logid of props.logids) {
      await handleTransferLog(logid, background)
    }
  }

  if (!background) {
    // 停止监听进度
    stopLoadingProgress()
  }

  // 关闭进度条
  progressDialog.value = false
  // 重新加载
  emit('done')
}

onMounted(() => {
  loadDirectories()
})

onUnmounted(() => {
  stopLoadingProgress()
})
</script>

<template>
  <VDialog scrollable max-width="50rem" :fullscreen="!display.mdAndUp.value">
    <VCard :title="dialogTitle" class="rounded-t">
      <DialogCloseBtn @click="emit('close')" />
      <VDivider />
      <VCardText>
        <VForm @submit.prevent="() => {}">
          <VRow>
            <VCol cols="12" md="6">
              <VSelect
                v-model="transferForm.target_storage"
                :items="storageOptions"
                label="目的存储"
                placeholder="留空自动"
                hint="整理目的存储"
                persistent-hint
              />
            </VCol>
            <VCol cols="12" md="6">
              <VSelect
                v-model="transferForm.transfer_type"
                label="整理方式"
                :items="transferTypeOptions"
                hint="文件操作整理方式"
                persistent-hint
              >
                <template v-slot:selection="{ item }">
                  {{ transferForm.transfer_type === '' ? '自动' : item.title }}
                </template>
              </VSelect>
            </VCol>
            <VCol cols="12">
              <VCombobox
                v-model="transferForm.target_path"
                :items="targetDirectories"
                label="目的路径"
                placeholder="留空自动"
                hint="整理目的路径，留空将自动匹配"
                persistent-hint
              />
            </VCol>
          </VRow>
          <VRow>
            <VCol cols="12" md="4">
              <VSelect
                v-model="transferForm.type_name"
                label="类型"
                :items="[
                  { title: '自动', value: '' },
                  { title: '电影', value: '电影' },
                  { title: '电视剧', value: '电视剧' },
                ]"
                hint="文件的媒体类型"
                persistent-hint
              />
            </VCol>
            <VCol cols="12" md="4">
              <VTextField
                v-if="mediaSource === 'themoviedb'"
                v-model="transferForm.tmdbid"
                :disabled="transferForm.type_name === ''"
                label="TheMovieDb编号"
                placeholder="留空自动识别"
                :rules="[numberValidator]"
                append-inner-icon="mdi-magnify"
                hint="按名称查询媒体编号，留空自动识别"
                persistent-hint
                @click:append-inner="mediaSelectorDialog = true"
              />
              <VTextField
                v-else
                v-model="transferForm.doubanid"
                :disabled="transferForm.type_name === ''"
                label="豆瓣编号"
                placeholder="留空自动识别"
                :rules="[numberValidator]"
                append-inner-icon="mdi-magnify"
                hint="按名称查询媒体编号，留空自动识别"
                persistent-hint
                @click:append-inner="mediaSelectorDialog = true"
              />
            </VCol>
            <VCol cols="12" md="4">
              <VSelect
                v-show="transferForm.type_name === '电视剧'"
                v-model.number="transferForm.season"
                label="季"
                :items="seasonItems"
                hint="指定季数"
                persistent-hint
              />
            </VCol>
          </VRow>
          <VRow>
            <VCol cols="12" md="8">
              <VTextField
                v-model="transferForm.episode_format"
                label="集数定位"
                placeholder="使用{ep}定位集数"
                hint="使用{ep}定位文件名中的集数部分以辅助识别"
                persistent-hint
              />
            </VCol>
            <VCol cols="12" md="4">
              <VTextField
                v-model="transferForm.episode_detail"
                :disabled="disableEpisodeDetail"
                label="指定集数"
                placeholder="起始集,终止集，如1或1,2"
                hint="指定集数或范围，如1或1,2"
                persistent-hint
              />
            </VCol>
            <VCol cols="12" md="4">
              <VTextField
                v-model="transferForm.episode_part"
                label="指定Part"
                placeholder="如part1"
                hint="指定Part，如part1"
                persistent-hint
              />
            </VCol>
            <VCol cols="12" md="4">
              <VTextField
                v-model="transferForm.episode_offset"
                label="集数偏移"
                placeholder="如-10"
                hint="集数偏移运算，如-10或EP*2"
                persistent-hint
              />
            </VCol>
            <VCol cols="12" md="4">
              <VTextField
                v-model.number="transferForm.min_filesize"
                label="最小文件大小（MB）"
                :rules="[numberValidator]"
                placeholder="0"
                hint="只整理大于最小文件大小的文件"
                persistent-hint
              />
            </VCol>
          </VRow>
          <VRow>
            <VCol cols="12" md="6" v-if="transferForm.target_path">
              <VSwitch
                v-model="transferForm.library_type_folder"
                label="按类型分类"
                hint="整理时目的路径下按媒体类型添加子目录"
                persistent-hint
              />
            </VCol>
            <VCol cols="12" md="6" v-if="transferForm.target_path">
              <VSwitch
                v-model="transferForm.library_category_folder"
                label="按类别分类"
                hint="整理时在目的路径下按媒体类别添加子目录"
                persistent-hint
              />
            </VCol>
            <VCol cols="12" md="6">
              <VSwitch
                v-model="transferForm.scrape"
                label="刮削元数据"
                hint="整理完成后自动刮削元数据"
                persistent-hint
              />
            </VCol>
            <VCol cols="12" md="6" v-if="props.logids">
              <VSwitch
                v-model="transferForm.from_history"
                label="复用历史识别信息"
                hint="使用历史整理记录中已识别的媒体信息"
                persistent-hint
              />
            </VCol>
          </VRow>
        </VForm>
      </VCardText>
      <VCardActions class="pt-3">
        <VSpacer />
        <VBtn variant="elevated" color="success" @click="transfer(true)" prepend-icon="mdi-plus" class="px-5">
          加入整理队列
        </VBtn>
        <VBtn variant="elevated" @click="transfer(false)" prepend-icon="mdi-arrow-right-bold" class="px-5">
          立即整理
        </VBtn>
      </VCardActions>
    </VCard>
    <!-- 手动整理进度框 -->
    <ProgressDialog v-if="progressDialog" v-model="progressDialog" :text="progressText" :value="progressValue" />
    <!-- TMDB ID搜索框 -->
    <VDialog v-model="mediaSelectorDialog" width="40rem" scrollable max-height="85vh">
      <MediaIdSelector
        v-if="mediaSource === 'themoviedb'"
        v-model="transferForm.tmdbid"
        @close="mediaSelectorDialog = false"
        :type="mediaSource"
      />
      <MediaIdSelector
        v-else
        v-model="transferForm.doubanid"
        @close="mediaSelectorDialog = false"
        :type="mediaSource"
      />
    </VDialog>
  </VDialog>
</template>
