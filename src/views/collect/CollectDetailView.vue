<script setup lang="ts">
import { useToast } from 'vue-toast-notification'
import PersonCardSlideView from './PersonCardSlideView.vue'
import MediaCardSlideView from './MediaCardSlideView.vue'
import api from '@/api'
import type { Collect, CollectCreate, DownloadTask, SiteSeed, Subscribe, TmdbEpisode } from '@/api/types'
import NoDataFound from '@/components/NoDataFound.vue'
import TaskCard from '@/components/cards/TaskCard.vue'
import SlideView from '@/components/slide/SlideView.vue'
import { doneNProgress, startNProgress } from '@/api/nprogress'
import { seedStatus } from '@/api/constants'
import { formatSeason } from '@/@core/utils/formatters'
import router from '@/router'
import VideoMediaInfoDialog from '@/components/dialog/VideoMediaInfoDialog.vue'
import VideoDescInfoDialog from '@/components/dialog/VideoDescInfoDialog.vue'
import ProgressInfoDialog from '@/components/dialog/ProgressInfoDialog.vue'
import AddSiteSeedDialog from '@/components/dialog/AddSiteSeedDialog.vue'
import SiteSeedInfoDialog from '@/components/dialog/SiteSeedInfoDialog.vue'
import CollectOperationDialog from '@/components/dialog/CollectOperationDialog.vue'


import { isNullOrEmptyObject } from '@/@core/utils'
import { useUserStore } from '@/stores'

// 输入参数
const collectProps = defineProps({
  id: String
})

// 从 provide 中获取全局设置
const globalSettings: any = inject('globalSettings')

// 用户 Store
const userStore = useUserStore()

// 提示框
const $toast = useToast()

// 媒体详情
const collectDetail = ref<Collect>({} as Collect)

const taskList = ref<DownloadTask[]>([])
// 站点列表
const siteSeedList = ref<SiteSeed[]>([])
const seedInfo = ref<SiteSeed>({} as SiteSeed)
const showMediaInfo = ref(false)
const showDescInfo = ref(false)
const showProgressInfo = ref(false)
const showSiteSeedInfo = ref(false)
const showAddSiteSedd = ref(false)
const showCollectOperation = ref(false)
const operationType = ref('')
// 本地是否存在，存在则包括Item信息
const existsItemId = ref('1')

// 是否已订阅
const isSubscribed = ref(false)

// 是否已加载完成
const isRefreshed = ref(false)


// 采集任务添加表单
const addForm = ref<CollectCreate>({
  cid: "",
  defn: "",
  douban_id: "",
  cn_title: "",
  year: "",
  type: "",
  site: "",
  auto_download: true,
  auto_publish: true,
  anon_publish: true,
  source: "WEB-DL",
  tags: [],
  episode_list: [],
  site_list: []
})

// 调用API查询详情
async function getDetail() {
  if (collectProps.id) {
    collectDetail.value = await api.get(`collect/${collectProps.id}`)
    taskList.value = await api.get(`collect/task/${collectProps.id}`)
    console.log('taskList', taskList.value)

    addForm.value.douban_id = collectDetail.value.douban_id ?? ''
    addForm.value.cn_title = collectDetail.value.title ?? ''
    addForm.value.year = collectDetail.value.year ?? ''


    isRefreshed.value = true
  }
}

async function getSiteSeedList() {
  try {
    siteSeedList.value = await api.get(`collect/seed/${collectProps.id}`)
  } catch (error) {
    console.error(error)
  }
}
function showMediaInfoDialog(operation: string) {
  if (collectDetail.value.mediainfo_collected) {
    showMediaInfo.value = true
  } else {
    operationType.value = operation
    showCollectOperation.value = true
  }
}
function showDescInfoDialog(operation: string) {
  if (collectDetail.value.desc_collected) {
    showDescInfo.value = true
  } else {
    operationType.value = operation
    showCollectOperation.value = true
  }
  
}
function showProgressInfoDialog() {
  console.log('showProgressInfoDialog')
  showProgressInfo.value = true
}

function showSiteSeedInfoDialog(seed: SiteSeed) {
  seedInfo.value = seed
  showSiteSeedInfo.value = true
}

function showAddSiteSeddoDialog() {
  console.log('AddSiteSeedDialog')
  showAddSiteSedd.value = true
}

function showCollectOperationDialog(operation: string) {
  console.log('showCollectOperationDialog')
  operationType.value = operation
  showCollectOperation.value = true
}

// 调用API添加采集任务
async function publish(id: number) {
  try {
    startNProgress()
    // 请求API
    const result: { [key: string]: any } = await api.get('collect/torrent_publish/' + id)
    // 添加采集任务状态
    if (result.success) {
      // 成功
      $toast.success(`发布成功！`)

    } else {
      $toast.error(`发布失败`)
    }
  } catch (error) {
    console.error(error)
  }
  doneNProgress()
}

async function deleteSeed(id: number) {
  try {
    startNProgress()
    // 请求API
    // const result: { [key: string]: any } = await api.get('torrent_seed/' + id)
    // // 添加采集任务状态
    // if (result.success) {
    //   // 成功
    //   $toast.success(`发布成功！`)

    // } else {
    //   $toast.error(`发布失败`)
    // }
    $toast.success(`删除成功！`)
  } catch (error) {
    console.error(error)
  }
  doneNProgress()
}
// 表单校验
function validateForm() {
  // 清空旧数据
  const errors = []

  if (!addForm.value.cid) {
    errors.push('媒体ID不能为空！')
  }
  if (!addForm.value.defn) {
    errors.push('请选择清晰度！')
  }

  if (addForm.value.site_list.length === 0) {
    errors.push('请至少选择一个站点！')
  }

  if (errors.length > 0) {
    errors.forEach(msg => $toast.error(msg))
    return false
  }
  return true
}
// 弹出添加订阅提示
function showCollectAddToast(result: boolean, title: string, message: string) {
  let subname = '采集任务'
  if (!result) $toast.error(`${title} 添加${subname}失败：${message}！`)
}

// TMDB图片转换为w500大小
function getW500Image(url = '') {
  if (!url) return ''
  url = url.replace('original', 'w500')
  // 使用图片缓存
  if (globalSettings.GLOBAL_IMAGE_CACHE)
    return `${import.meta.env.VITE_API_BASE_URL}system/cache/image?url=${encodeURIComponent(url)}`
  return url
}

// 计算Poster地址
const getPosterUrl: Ref<string> = computed(() => {
  const url = collectDetail.value.cover ?? ''
  // 使用图片缓存
  if (globalSettings.GLOBAL_IMAGE_CACHE)
    return `${import.meta.env.VITE_API_BASE_URL}system/cache/image?url=${encodeURIComponent(url)}`
  // 如果地址中包含douban则使用中转代理
  if (url.includes('doubanio.com'))
    return `${import.meta.env.VITE_API_BASE_URL}system/img/0?imgurl=${encodeURIComponent(url)}`
  return url
})

// 计算backdrop地址
const getBackdropUrl: Ref<string> = computed(() => {
  const url = collectDetail.value.poster ?? ''
  // 使用图片缓存
  if (globalSettings.GLOBAL_IMAGE_CACHE)
    return `${import.meta.env.VITE_API_BASE_URL}system/cache/image?url=${encodeURIComponent(url)}`
  return url
})

function getSeedStatus(status: string) {
  return seedStatus[status as keyof typeof seedStatus]
}
// 添加做种任务成功
function addSiteSeedSuccess(url: string) {
  showAddSiteSedd.value = false
  // 重新加载做种列表
  getSiteSeedList()
}

// 添加做种任务失败
function addSiteSeedError(error: string) {
  showAddSiteSedd.value = false
}
onBeforeMount(() => {
  getDetail()
  getSiteSeedList()
})
</script>

<template>
  <LoadingBanner v-if="!isRefreshed" class="mt-12" />
  <div class="max-w-8xl mx-auto px-4">
    <template v-if="getBackdropUrl || getPosterUrl">
      <div class="vue-media-back absolute left-0 top-0 w-full h-96">
        <VImg class="h-96" position="top" :src="getBackdropUrl || getPosterUrl" cover />
      </div>
      <div class="vue-media-back absolute left-0 top-0 w-full h-96" />
    </template>
    <div class="media-page">
      <div class="media-header">
        <div class="media-poster">
          <VImg :src="getW500Image(getPosterUrl)" cover class="object-cover aspect-w-2 aspect-h-3 ring-1 ring-gray-500">
            <template #placeholder>
              <div class="w-full h-full">
                <VSkeletonLoader class="object-cover aspect-w-2 aspect-h-3" />
              </div>
            </template>
          </VImg>
        </div>
        <div class="media-title">
          <div v-if="existsItemId" class="media-status">
            <span
              class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full whitespace-nowrap transition !no-underline bg-green-500 bg-opacity-80 border border-green-500 !text-green-100 hover:bg-green-500 hover:bg-opacity-100 false overflow-hidden">
              <div class="relative z-20 flex items-center false"><span>已入库</span></div>
            </span>
          </div>
          <h1 class="d-flex flex-column flex-lg-row align-baseline justify-center justify-lg-start">
            <div class="align-self-center align-self-lg-end">
              {{ collectDetail.name }}
            </div>
          </h1>

        </div>
        <div class="media-actions">

          <VBtn class="ms-2 mb-2" color="primary" variant="tonal" @click="showProgressInfoDialog()">
            <template #prepend>
              <VIcon icon="mdi-timetable" />
            </template>
            进度
          </VBtn>
        </div>
      </div>
      <div class="media-overview">
        <div class="media-overview-left">
          <div class="tagline">
            tagline
          </div>
          <div class="mt-6">
            <v-stepper bg-color="rgba(255, 255, 255, 0.1)" complete-icon="mdi-check-circle" edit-icon="mdi-checkbox-blank-circle">
              <v-stepper-header>
                <v-stepper-item title="媒体下载" value="1" :color="collectDetail.is_downloaded ? 'success' : ''"
                  :complete="collectDetail.is_downloaded"
                  :disabled="collectDetail.is_downloaded"
                  @click.stop="showCollectOperationDialog('start_download_by_collect')">
                </v-stepper-item>
                <v-divider></v-divider>
                <v-stepper-item title="媒体信息采集" value="2" :color="collectDetail.mediainfo_collected ? 'success' : ''"
                  :complete="collectDetail.mediainfo_collected" editable
                  @click.stop="showMediaInfoDialog('metadata_by_collect')"></v-stepper-item>
                <v-divider></v-divider>
                <v-stepper-item title="截图" value="3" :color="collectDetail.image_collected ? 'success' : ''"
                  :complete="collectDetail.image_collected" editable :disabled="collectDetail.image_collected"
                  @click.stop="showCollectOperationDialog('screenshot_by_collect')"></v-stepper-item>
                <v-divider></v-divider>
                <v-stepper-item title="简介采集" value="4" :color="collectDetail.desc_collected ? 'success' : ''"
                  :complete="collectDetail.desc_collected" editable
                  @click.stop="showDescInfoDialog('collect_desc_by_collect')"></v-stepper-item>
                <v-divider></v-divider>
                <v-stepper-item title="重命名" value="5" :color="collectDetail.is_renamed ? 'success' : ''"
                  :complete="collectDetail.is_renamed" editable
                  :disabled="collectDetail.is_renamed"
                  @click.stop="showCollectOperationDialog('collect_move')"></v-stepper-item>
                <v-divider></v-divider>
                <v-stepper-item title="制作种子" value="6" :color="collectDetail.torrent_created ? 'success' : ''"
                  :complete="collectDetail.torrent_created" :editable="!collectDetail.torrent_created"
                  @click.stop="showCollectOperationDialog('torrent_create')"></v-stepper-item>
              </v-stepper-header>
            </v-stepper>
          </div>
          <div class="mt-6">
            <v-row>
              <v-col cols="2">
                <v-switch v-model="addForm.auto_download" :label="`自动下载`" hide-details>
                </v-switch>
              </v-col>
              <v-col cols="2">
                <v-switch v-model="addForm.auto_publish" :label="`自动发布`" hide-details>
                </v-switch>
              </v-col>
            </v-row>
          </div>

          <div class="mt-6">
            <VChipGroup class="p-3" column>
              <VChip v-for="(item, index) in siteSeedList" :key="index" @click.stop="showSiteSeedInfoDialog(item)">
                <template #append>
                  <VBadge color="primary" :content="getSeedStatus(item.status)" inline size="small" />
                </template>
                {{ item.site_name }}
              </VChip>
              <VBtn class="ms-2 mb-2" color="success" variant="tonal" @click="showAddSiteSeddoDialog()">
            <template #prepend>
              <VIcon icon="mdi-plus" />
            </template>
            添加站点
          </VBtn>
            </VChipGroup>
            
          </div>
        </div>


      </div>
      <div v-if="taskList && taskList.length > 0">
        <SlideView>
          <template #content>
            <template v-for="data in taskList" :key="data.id">
              <TaskCard :info="data" height="11rem" width="20rem" />
            </template>
          </template>
        </SlideView>
      </div>

    </div>
    <VideoMediaInfoDialog v-if="showMediaInfo" v-model="showMediaInfo" :collect="collectDetail"
      @close="showMediaInfo = false" />
    <VideoDescInfoDialog v-if="showDescInfo" v-model="showDescInfo" :collect="collectDetail"
      @close="showDescInfo = false" />
    <ProgressInfoDialog v-if="showProgressInfo" v-model="showProgressInfo" type="collect" :id="collectDetail.id" :name="collectDetail.name" @close="showProgressInfo = false" />
    <SiteSeedInfoDialog v-if="showSiteSeedInfo" v-model="showSiteSeedInfo" :seed="seedInfo"
      @close="showSiteSeedInfo = false" />
    <AddSiteSeedDialog v-if="showAddSiteSedd" v-model="showAddSiteSedd" :collect="collectDetail" :siteSeedList="siteSeedList"
      @done="addSiteSeedSuccess"
      @error="addSiteSeedError"
      @close="showAddSiteSedd = false" />
    <CollectOperationDialog v-if="showCollectOperation" v-model="showCollectOperation" :collect="collectDetail" :operation="operationType"
     @close="showCollectOperation = false" />
      
  </div>
  <NoDataFound v-if="!collectDetail.id && isRefreshed" error-code="500" error-title="出错啦！"
    error-description="未识别到媒体信息。" />

</template>

<style lang="scss">
.vue-media-back {
  background-image: linear-gradient(180deg,
      rgba(var(--v-theme-background), 0) 50%,
      rgba(var(--v-theme-background), 1) 100%),
    linear-gradient(90deg, rgba(var(--v-theme-background), 0) 50%, rgba(var(--v-theme-background), 1) 100%),
    linear-gradient(270deg, rgba(var(--v-theme-background), 0) 50%, rgba(var(--v-theme-background), 1) 100%);
  box-shadow: 0 0 0 2px rgb(var(--v-theme-background));
  margin-block-start: calc(-70px - env(safe-area-inset-top));
}

.media-page {
  position: relative;
  background-position: 50%;
  background-size: cover;
  margin-block-start: calc(-4rem - env(safe-area-inset-top));
  margin-inline: -1rem;
  padding-block-start: calc(4rem + env(safe-area-inset-top));
  padding-inline: 1rem;
}

.media-header {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding-block-start: 1rem;
}

@media (width >=1280px) {
  .media-header {
    flex-direction: row;
    align-items: flex-end;
  }
}

.media-overview {
  display: flex;
  flex-direction: column;
  padding-block: 2rem 1rem;
}

@media (width >=1024px) {
  .media-overview {
    flex-direction: row;
  }
}

.media-poster {
  overflow: hidden;
  border-radius: 0.25rem;
  box-shadow: var(--tw-ring-offset-shadow, 0 0 #0000), var(--tw-ring-shadow, 0 0 #0000), var(--tw-shadow);
  inline-size: 8rem;

  --tw-shadow: 0 1px 3px 0 rgba(0, 0, 0, 10%), 0 1px 2px -1px rgba(0, 0, 0, 10%);
  --tw-shadow-colored: 0 1px 3px 0 var(--tw-shadow-color), 0 1px 2px -1px var(--tw-shadow-color);
}

@media (width >=1280px) {
  .media-poster {
    inline-size: 13rem;
    margin-inline-end: 1rem;
  }
}

@media (width >=768px) {
  .media-poster {
    border-radius: 0.5rem;
    box-shadow: var(--tw-ring-offset-shadow, 0 0 #0000), var(--tw-ring-shadow, 0 0 #0000), var(--tw-shadow);
    inline-size: 11rem;

    --tw-shadow: 0 25px 50px -12px rgba(0, 0, 0, 25%);
    --tw-shadow-colored: 0 25px 50px -12px var(--tw-shadow-color);
  }
}

.media-title {
  display: flex;
  flex: 1 1 0%;
  flex-direction: column;
  margin-block-start: 1rem;
  text-align: center;
}

@media (width >=1280px) {
  .media-title {
    margin-block-start: 0;
    margin-inline-end: 1rem;
    text-align: start;
  }
}

.media-title>h1 {
  font-size: 1.5rem;
  font-weight: 700;
  line-height: 2rem;
}

@media (width >=1280px) {
  .media-title>h1 {
    font-size: 2.25rem;
    line-height: 2.5rem;
  }
}

ul.media-crew {
  display: grid;
  gap: 1.5rem;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  margin-block-start: 1.5rem;
}

@media (width >=640px) {
  ul.media-crew {
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }
}

ul.media-crew>li {
  display: flex;
  flex-direction: column;
  font-weight: 700;
  grid-column: span 1 / span 1;
}

a.crew-name {
  font-weight: 400;
}

.media-status {
  margin-block-end: 0.5rem;
}

.media-attributes {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: center;
  margin-block-start: 0.25rem;
}

@media (width >=1280px) {
  .media-attributes {
    justify-content: flex-start;
    font-size: 1rem;
    line-height: 1.5rem;
    margin-block-start: 0;
  }
}

@media (width >=640px) {
  .media-attributes {
    font-size: 0.875rem;
    line-height: 1.25rem;
  }
}

.media-actions {
  position: relative;
  display: flex;
  flex-shrink: 0;
  flex-wrap: wrap;
  align-items: center;
  justify-content: center;
  margin-block-start: 1rem;
}

@media (width >=1280px) {
  .media-actions {
    margin-block-start: 0;
  }
}

@media (width >=640px) {
  .media-actions {
    flex-wrap: nowrap;
    justify-content: flex-end;
  }
}

.media-overview-left {
  flex: 1 1 0%;
}

@media (width >=1024px) {
  .media-overview-left {
    margin-inline-end: 2rem;
  }
}

.media-overview-right {
  inline-size: 100%;
  margin-block-start: 2rem;
}

@media (width >=1024px) {
  .media-overview-right {
    inline-size: 20rem;
    margin-block-start: 0;
  }
}

.media-facts {
  border-width: 1px;
  border-color: rgb(55 65 81 / var(--tw-border-opacity));
  border-radius: 0.5rem;
  font-size: 0.875rem;
  font-weight: 700;
  line-height: 1.25rem;

  --tw-border-opacity: 1;
  --tw-bg-opacity: 1;
  --tw-text-opacity: 1;
}

.media-ratings {
  display: flex;
  align-items: center;
  justify-content: center;
  border-color: rgb(55 65 81 / var(--tw-border-opacity));
  border-block-end-width: 1px;
  font-weight: 500;
  padding-block: 0.5rem;
  padding-inline: 1rem;

  --tw-border-opacity: 1;
}

.media-fact {
  display: flex;
  justify-content: space-between;
  border-color: rgb(55 65 81 / var(--tw-border-opacity));
  border-block-end-width: 1px;
  padding-block: 0.5rem;
  padding-inline: 1rem;

  --tw-border-opacity: 1;
}

.media-overview h2 {
  font-size: 1.25rem;
  font-weight: 700;
  line-height: 1.75rem;
}

@media (width >=640px) {
  .media-overview h2 {
    font-size: 1.5rem;
    line-height: 2rem;
  }
}

.tagline {
  font-size: 1.25rem;
  font-style: italic;
  line-height: 1.75rem;
  margin-block-end: 1rem;
}

@media (width >=1024px) {
  .tagline {
    font-size: 1.5rem;
    line-height: 2rem;
  }
}
</style>
