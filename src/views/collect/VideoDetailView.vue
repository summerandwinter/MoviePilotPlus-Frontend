<script setup lang="ts">
import { useToast } from 'vue-toast-notification'
import PersonCardSlideView from './PersonCardSlideView.vue'
import MediaCardSlideView from './MediaCardSlideView.vue'
import api from '@/api'
import type { VideoInfo, CollectCreate, NotExistMediaInfo, Site, Subscribe, TmdbEpisode } from '@/api/types'
import NoDataFound from '@/components/NoDataFound.vue'
import EpisodeCard from '@/components/cards/EpisodeCard.vue'
import SlideView from '@/components/slide/SlideView.vue'
import { doneNProgress, startNProgress } from '@/api/nprogress'
import { formatSeason } from '@/@core/utils/formatters'
import router from '@/router'
import SubscribeEditDialog from '@/components/dialog/SubscribeEditDialog.vue'
import { isNullOrEmptyObject } from '@/@core/utils'
import { useUserStore } from '@/stores'

// 输入参数
const mediaProps = defineProps({
  source: String,
  mediaid: String,
  title: String,
  year: String,
  type: String,
})

// 从 provide 中获取全局设置
const globalSettings: any = inject('globalSettings')

// 用户 Store
const userStore = useUserStore()

// 提示框
const $toast = useToast()

// 媒体详情
const mediaDetail = ref<VideoInfo>({} as VideoInfo)
// 站点列表
const siteList = ref<Site[]>([])

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
    source: "WEB-DL",
    episode_list: [],
    site_list: []
})

// 调用API查询详情
async function getMediaDetail() {
  if (mediaProps.mediaid && mediaProps.type) {
    mediaDetail.value = await api.get(`${mediaProps.source?.toLowerCase()}/detail`, {
      params: {
        cid: mediaProps.mediaid
      },
    })
    // 默认选中所有剧集
    // mediaDetail.value.episode_list?.forEach(episode => {
    //   episode.selected = true
    // })
    // 设置默认选中第一个清晰度
    if (mediaDetail.value.definition_list?.length > 0) {
      addForm.value.defn = mediaDetail.value.definition_list[0].name
    }
    // addForm 赋值
    addForm.value.cid = mediaProps.mediaid
    addForm.value.douban_id = mediaDetail.value.douban_id ?? ''
    addForm.value.cn_title = mediaDetail.value.title ?? ''
    addForm.value.year = mediaDetail.value.year ?? ''
    addForm.value.type = mediaProps.type ?? ''
    addForm.value.site = mediaProps.source ?? ''
    
    isRefreshed.value = true
  }
}

async function getSites() {
  try {
    siteList.value = await api.get('site/')
  } catch (error) {
    console.error(error)
  }
}

// 调用API添加采集任务
async function addCollect() {
  try {
    // 处理选中的剧集
    mediaDetail.value.episode_list?.forEach(episode => {
      if (episode.selected) {
        addForm.value.episode_list.push({
          cid: episode.cid,
          vid: episode.vid,
          episode: episode.title
        })
      }
    })
    // 提交前检查参数
    console.log(addForm.value)
    if (!validateForm()) return
    // 调用接口添加采集任务
    startNProgress()
    // 请求API
    const result: { [key: string]: any } = await api.post('collect/', addForm.value)
    // 添加采集任务状态
    if (result.success) {
      // 成功
      isSubscribed.value = true
    }

    // 提示
    showCollectAddToast(result.success, mediaDetail.value?.title ?? '', result.message)
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
  if (!mediaDetail.value.episode_list?.some(e => e.selected)) {
    errors.push('请至少选择一集！')
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
  const url = mediaDetail.value.new_pic_vt ?? ''
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
  const url = mediaDetail.value.new_pic_hz ?? ''
  // 使用图片缓存
  if (globalSettings.GLOBAL_IMAGE_CACHE)
    return `${import.meta.env.VITE_API_BASE_URL}system/cache/image?url=${encodeURIComponent(url)}`
  return url
})


// 计算订阅图标
const getSubscribeIcon = computed(() => {
  if (isSubscribed.value) return 'mdi-heart'
  else return 'mdi-heart-outline'
})

// 计算订阅按钮颜色
const getSubscribeColor = computed(() => {
  if (isSubscribed.value) return 'error'
  else return 'warning'
})

// 跳转播放页面
async function handlePlay() {
  // 获取播放链接地址
  try {
    const result: { [key: string]: any } = await api.get(`mediaserver/play/${existsItemId.value}`)
    if (result?.success) {
      // 打开链接地址
      setTimeout(() => {
        window.open(result.data.url, '_blank')
      }, 100)
    } else {
      $toast.error(`获取播放链接失败：${result.message}！`)
    }
  } catch (error) {
    console.error(error)
  }
}

onBeforeMount(() => {
  getMediaDetail()
  getSites()
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
          <VImg
            :src="getW500Image(getPosterUrl)"
            cover
            class="object-cover aspect-w-2 aspect-h-3 ring-1 ring-gray-500"
          >
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
              class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full whitespace-nowrap transition !no-underline bg-green-500 bg-opacity-80 border border-green-500 !text-green-100 hover:bg-green-500 hover:bg-opacity-100 false overflow-hidden"
            >
              <div class="relative z-20 flex items-center false"><span>已入库</span></div>
            </span>
          </div>
          <h1 class="d-flex flex-column flex-lg-row align-baseline justify-center justify-lg-start">
            <div class="align-self-center align-self-lg-end">
              {{ mediaDetail.title }}
            </div>
            <div v-if="mediaDetail.year" class="text-lg align-self-center align-self-lg-end">
              （{{ mediaDetail.year }}）
            </div>
          </h1>
          <span class="media-attributes">
            <span v-if="mediaDetail.areaName"
              >{{ mediaDetail.areaName }}</span
            >
            <span v-if="mediaDetail.douban_info && mediaDetail.douban_info.card_subtitle" class="mx-1">
              |
            </span>
            <span v-if="mediaDetail.douban_info && mediaDetail.douban_info.card_subtitle">{{ mediaDetail.douban_info.card_subtitle }}</span>
          </span>
        </div>
        <div class="media-actions">
          <VBtn         
            variant="tonal"
            color="info"
            class="mb-2"
            @click="addCollect"
          >
            <template #prepend>
              <VIcon icon="mdi-plus" />
            </template>
            {{ '添加' }}
          </VBtn>
          <VBtn
        
            class="ms-2 mb-2"
            :color="getSubscribeColor"
            variant="tonal"
          >
            <template #prepend>
              <VIcon :icon="getSubscribeIcon" />
            </template>
            {{ '订阅' }}
          </VBtn>
          <VBtn v-if="existsItemId" class="ms-2 mb-2" variant="tonal" @click="handlePlay()">
            <template #prepend>
              <VIcon icon="mdi-play" />
            </template>
            在线播放
          </VBtn>
        </div>
      </div>
      <div class="media-overview">
        <div class="media-overview-left">
          <div  class="tagline">
            tagline
          </div>
          <h2 v-if="mediaDetail.overview">简介</h2>
          <p>{{ mediaDetail.overview }}</p>
          
          <div class="mt-6">
            <VChipGroup column v-model="addForm.defn">
              <template v-for="definition in mediaDetail.definition_list" :key="definition.name">
                <VChip
                  v-if="definition.sname"
                  :color="addForm.defn === definition.name ? 'primary' : ''"
                  filter
                  variant="outlined"
                  :value="definition.name"
                >
                  {{ definition.sname }}
                </VChip>
              </template>
            </VChipGroup>
          </div>
          <div class="mt-6">
            <VChipGroup column v-model="addForm.site_list" multiple>
              <template v-for="site in siteList" :key="site.id">
                <VChip
                 
                  :color="addForm.site_list.includes(site.name) ? 'primary' : ''"
                  filter
                  variant="outlined"
                  :value="site.id"
                >
                  {{ site.name }}
                </VChip>
              </template>
            </VChipGroup>
          </div>
          
        </div>
        <div v-if="mediaDetail.douban_info" class="media-overview-right">
          <div class="media-facts">
            <div v-if="mediaDetail.douban_info.rating" class="media-ratings">
              <VRating v-model="mediaDetail.douban_info.rating" density="compact" length="10" class="ma-2" readonly />
            </div>
            <div v-if="mediaDetail.douban_info.id" class="media-fact">
              <span>ID</span>
              <span class="media-fact-value">{{ mediaDetail.douban_info.id }}</span>
            </div>
            <div v-if="mediaDetail.douban_info.title" class="media-fact">
              <span>原始标题</span>
              <span class="media-fact-value">{{ mediaDetail.douban_info.title }}</span>
            </div>
            
            <div v-if="mediaDetail.douban_info.year" class="media-fact">
              <span>上映日期</span>
              <span class="media-fact-value">
                <span class="flex items-center justify-end">
                  <svg
                    xmlns="http://www.w3.org/2000/svg"
                    fill="none"
                    viewBox="0 0 24 24"
                    stroke-width="1.5"
                    stroke="currentColor"
                    aria-hidden="true"
                    class="h-4 w-4"
                  >
                    <path
                      stroke-linecap="round"
                      stroke-linejoin="round"
                      d="M2.25 15a4.5 4.5 0 004.5 4.5H18a3.75 3.75 0 001.332-7.257 3 3 0 00-3.758-3.848 5.25 5.25 0 00-10.233 2.33A4.502 4.502 0 002.25 15z"
                    />
                  </svg>
                  <span class="ml-1.5">{{ mediaDetail.douban_info.year  }}</span>
                </span>
              </span>
            </div>
          </div>
        </div>
         
      </div>
      <div v-if="mediaDetail.episode_list">
        <SlideView>
          <template #content>
            <template v-for="data in mediaDetail.episode_list" :key="data.vid">
              <EpisodeCard :episode="data" height="9rem" width="16rem" />
            </template>
          </template>
        </SlideView>
      </div>
      
    </div>
  </div>
  <NoDataFound
    v-if="!mediaDetail.tmdb_id && !mediaDetail.douban_id && !mediaDetail.bangumi_id && isRefreshed"
    error-code="500"
    error-title="出错啦！"
    error-description="未识别到媒体信息。"
  />
</template>

<style lang="scss">
.vue-media-back {
  background-image: linear-gradient(
      180deg,
      rgba(var(--v-theme-background), 0) 50%,
      rgba(var(--v-theme-background), 1) 100%
    ),
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

@media (width >= 1280px) {
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

@media (width >= 1024px) {
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

@media (width >= 1280px) {
  .media-poster {
    inline-size: 13rem;
    margin-inline-end: 1rem;
  }
}

@media (width >= 768px) {
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

@media (width >= 1280px) {
  .media-title {
    margin-block-start: 0;
    margin-inline-end: 1rem;
    text-align: start;
  }
}

.media-title > h1 {
  font-size: 1.5rem;
  font-weight: 700;
  line-height: 2rem;
}

@media (width >= 1280px) {
  .media-title > h1 {
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

@media (width >= 640px) {
  ul.media-crew {
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }
}

ul.media-crew > li {
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

@media (width >= 1280px) {
  .media-attributes {
    justify-content: flex-start;
    font-size: 1rem;
    line-height: 1.5rem;
    margin-block-start: 0;
  }
}

@media (width >= 640px) {
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

@media (width >= 1280px) {
  .media-actions {
    margin-block-start: 0;
  }
}

@media (width >= 640px) {
  .media-actions {
    flex-wrap: nowrap;
    justify-content: flex-end;
  }
}

.media-overview-left {
  flex: 1 1 0%;
}

@media (width >= 1024px) {
  .media-overview-left {
    margin-inline-end: 2rem;
  }
}

.media-overview-right {
  inline-size: 100%;
  margin-block-start: 2rem;
}

@media (width >= 1024px) {
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

@media (width >= 640px) {
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

@media (width >= 1024px) {
  .tagline {
    font-size: 1.5rem;
    line-height: 2rem;
  }
}
</style>
