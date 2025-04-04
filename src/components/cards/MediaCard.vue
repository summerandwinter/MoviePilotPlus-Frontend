<script lang="ts" setup>
import type { PropType, Ref } from 'vue'
import { useToast } from 'vue-toast-notification'
import SubscribeEditDialog from '../dialog/SubscribeEditDialog.vue'
import { formatSeason, formatRating } from '@/@core/utils/formatters'
import api from '@/api'
import { doneNProgress, startNProgress } from '@/api/nprogress'
import type { MediaInfo, NotExistMediaInfo, Subscribe, MediaSeason, Site } from '@/api/types'
import router, { registerAbortController } from '@/router'
import noImage from '@images/no-image.jpeg'
import tmdbImage from '@images/logos/tmdb.png'
import doubanImage from '@images/logos/douban-black.png'
import bangumiImage from '@images/logos/bangumi.png'
import { useUserStore } from '@/stores'

// 输入参数
const props = defineProps({
  media: Object as PropType<MediaInfo>,
  width: String,
  height: String,
})

// 从 provide 中获取全局设置
const globalSettings: any = inject('globalSettings')

// 用户 Store
const userStore = useUserStore()

// 提示框
const $toast = useToast()

// 图片加载状态
const isImageLoaded = ref(false)

// 图片加载失败
const imageLoadError = ref(false)

// TMDB识别标志
const tmdbFlag = ref(true)

// 当前订阅状态
const isSubscribed = ref(false)

// 本地存在状态
const isExists = ref(false)

// 各季缺失状态：0-已入库 1-部分缺失 2-全部缺失，没有数据也是已入库
const seasonsNotExisted = ref<{ [key: number]: number }>({})

// 订阅季弹窗
const subscribeSeasonDialog = ref(false)

// 订阅编辑弹窗
const subscribeEditDialog = ref(false)

// 订阅ID
const subscribeId = ref<number>()

// 季详情
const seasonInfos = ref<MediaSeason[]>([])

// 选中的订阅季
const seasonsSelected = ref<MediaSeason[]>([])

// 来源角标字典
const sourceIconDict: { [key: string]: any } = {
  themoviedb: tmdbImage,
  douban: doubanImage,
  bangumi: bangumiImage,
}

// 绑定MediaCard元素
const mediaCardRef = ref<HTMLElement | null>(null)

// 创建Intersection Observer实例
const observer = ref<IntersectionObserver | null>(null)

// 所有站点
const allSites = ref<Site[]>([])

// 选中的站点
const selectedSites = ref<number[]>([])

// 搜索菜单显示状态
const searchMenuShow = ref(false)

// 查询所有站点
async function querySites() {
  try {
    const data: Site[] = await api.get('site/')

    // 过滤站点，只有启用的站点才显示
    allSites.value = data.filter(item => item.is_active)
  } catch (error) {
    console.log(error)
  }
}

// 查询用户选中的站点
async function querySelectedSites() {
  try {
    const result: { [key: string]: any } = await api.get('system/setting/IndexerSites')

    selectedSites.value = result.data?.value ?? []
  } catch (error) {
    console.log(error)
  }
}

// 获得mediaid
function getMediaId() {
  if (props.media?.tmdb_id) return `tmdb:${props.media?.tmdb_id}`
  else if (props.media?.douban_id) return `douban:${props.media?.douban_id}`
  else if (props.media?.bangumi_id) return `bangumi:${props.media?.bangumi_id}`
  else return `${props.media?.mediaid_prefix}:${props.media?.media_id}`
}

// 订阅弹窗选择的多季
function subscribeSeasons() {
  subscribeSeasonDialog.value = false
  seasonsSelected.value.forEach(season => {
    addSubscribe(season.season_number)
  })
}

// 角标颜色
function getChipColor(type: string) {
  if (type === '电影') return 'border-blue-500 bg-blue-600'
  else if (type === '电视剧') return ' bg-indigo-500 border-indigo-600'
  else return 'border-purple-600 bg-purple-600'
}

// 添加订阅处理
async function handleAddSubscribe() {
  if (props.media?.type === '电视剧') {
    // 查询所有季信息
    await getMediaSeasons()
    if (!seasonInfos.value || seasonInfos.value.length === 0) {
      $toast.error(`${props.media?.title} 查询剧集信息失败！`)
      return
    }
    // 检查各季的缺失状态
    await checkSeasonsNotExists()
    if (!tmdbFlag.value) return

    if (seasonInfos.value.length === 1) {
      // 添加订阅
      addSubscribe(1)
    } else {
      // 弹出季选择列表，支持多选
      seasonsSelected.value = []
      subscribeSeasonDialog.value = true
    }
  } else {
    // 电影
    addSubscribe()
  }
}

// 调用API添加订阅，电视剧的话需要指定季
async function addSubscribe(season = 0) {
  // 开始处理
  startNProgress()
  try {
    // 是否洗版
    let best_version = isExists.value ? 1 : 0
    if (season && props.media?.tmdb_id)
      // 全部存在时洗版
      best_version = !seasonsNotExisted.value[season] ? 1 : 0
    // 请求API
    const result: { [key: string]: any } = await api.post('subscribe/', {
      name: props.media?.title,
      type: props.media?.type,
      year: props.media?.year,
      tmdbid: props.media?.tmdb_id,
      doubanid: props.media?.douban_id,
      bangumiid: props.media?.bangumi_id,
      mediaid: props.media?.media_id ? `${props.media?.mediaid_prefix}:${props.media?.media_id}` : '',
      season,
      best_version,
    })

    // 订阅状态
    if (result.success) {
      // 订阅成功
      isSubscribed.value = true
    }

    // 提示
    showSubscribeAddToast(result.success, props.media?.title ?? '', season, result.message, best_version)

    // 弹出订阅编辑弹窗
    if (result.success && seasonsSelected.value.length <= 1) {
      const show_edit_dialog = await queryDefaultSubscribeConfig()
      if (show_edit_dialog) {
        subscribeId.value = result.data.id
        subscribeEditDialog.value = true
      }
    }
  } catch (error) {
    console.error(error)
  } finally {
    doneNProgress()
  }
}

// 弹出添加订阅提示
function showSubscribeAddToast(result: boolean, title: string, season: number, message: string, best_version: number) {
  if (season) title = `${title} ${formatSeason(season.toString())}`

  let subname = '订阅'
  if (best_version > 0) subname = '洗版订阅'

  if (result) $toast.success(`${title} 添加${subname}成功！`)
  else if (!result) $toast.error(`${title} 添加${subname}失败：${message}！`)
}

// 调用API取消订阅
async function removeSubscribe() {
  // 开始处理
  startNProgress()
  try {
    const mediaid = getMediaId()

    const result: { [key: string]: any } = await api.delete(`subscribe/media/${mediaid}`, {
      params: {
        season: props.media?.season,
      },
    })

    if (result.success) {
      isSubscribed.value = false
      $toast.success(`${props.media?.title} 已取消订阅！`)
    } else {
      $toast.error(`${props.media?.title} 取消订阅失败：${result.message}！`)
    }
  } catch (error) {
    console.error(error)
  } finally {
    doneNProgress()
  }
}

// 查询当前媒体是否已订阅
async function handleCheckSubscribe() {
  try {
    const result = await checkSubscribe(props.media?.season)
    if (result) isSubscribed.value = true
  } catch (error) {
    console.error(error)
  }
}

// 查询当前媒体是否已入库
async function handleCheckExists() {
  try {
    const abortController = new AbortController()
    registerAbortController(abortController)
    const { signal } = abortController
    const result: { [key: string]: any } = await api.get('mediaserver/exists', {
      params: {
        tmdbid: props.media?.tmdb_id,
        title: props.media?.title,
        year: props.media?.year,
        season: props.media?.season,
        mtype: props.media?.type,
      },
      signal,
    })

    if (result.success) isExists.value = true
  } catch (error) {
    console.error(error)
  }
}

// 调用API检查是否已订阅，电视剧需要指定季
async function checkSubscribe(season = 0) {
  try {
    const abortController = new AbortController()
    registerAbortController(abortController)
    const { signal } = abortController
    const mediaid = getMediaId()
    const result: Subscribe = await api.get(`subscribe/media/${mediaid}`, {
      params: {
        season,
        title: props.media?.title,
      },
      signal,
    })

    return result.id || null
  } catch (error) {
    console.error(error)
  }

  return null
}

// 检查所有季的缺失状态（数据库）
async function checkSeasonsNotExists() {
  // 开始处理
  startNProgress()
  try {
    const result: NotExistMediaInfo[] = await api.post('mediaserver/notexists', props.media)
    if (result) {
      result.forEach(item => {
        // 0-已入库 1-部分缺失 2-全部缺失
        let state = 0
        if (item.episodes.length === 0) state = 2
        else if (item.episodes.length < item.total_episode) state = 1
        seasonsNotExisted.value[item.season] = state
      })
    }
  } catch (error) {
    $toast.error(`${props.media?.title}无法识别TMDB媒体信息！`)
    tmdbFlag.value = false
  } finally {
    // 处理完成
    doneNProgress()
  }
}

// 查询TMDB的所有季信息
async function getMediaSeasons() {
  startNProgress()
  try {
    seasonInfos.value = await api.get('media/seasons', {
      params: {
        mediaid: getMediaId(),
        title: props.media?.title,
        year: props.media?.year,
        season: props.media?.season,
      },
    })
  } catch (error) {
    console.error(error)
  }
  doneNProgress()
}

// 查询订阅弹窗规则
async function queryDefaultSubscribeConfig() {
  // 非管理员不显示
  if (!userStore.superUser) return false
  try {
    let subscribe_config_url = ''
    if (props.media?.type === '电影') subscribe_config_url = 'system/setting/DefaultMovieSubscribeConfig'
    else subscribe_config_url = 'system/setting/DefaultTvSubscribeConfig'

    const result: { [key: string]: any } = await api.get(subscribe_config_url)

    if (result.data?.value) return result.data.value.show_edit_dialog
  } catch (error) {
    console.log(error)
  }
  return false
}

// 爱心订阅按钮响应
function handleSubscribe() {
  if (isSubscribed.value) removeSubscribe()
  else handleAddSubscribe()
}

// 计算存在状态的颜色
function getExistColor(season: number) {
  const state = seasonsNotExisted.value[season]
  if (!state) return 'success'

  if (state === 1) return 'warning'
  else if (state === 2) return 'error'
  else return 'success'
}

// 计算存在状态的文本
function getExistText(season: number) {
  const state = seasonsNotExisted.value[season]
  if (!state) return '已入库'

  if (state === 1) return '部分缺失'
  else if (state === 2) return '缺失'
  else return '已入库'
}

// 打开详情页
function goMediaDetail(isHovering = false) {
  if (isHovering) {
    if (props.media?.collection_id) {
      // 跳转到合集列表
      router.push({
        path: `/browse/tmdb/collection/${props.media?.collection_id}`,
        query: {
          title: props.media?.title,
        },
      })
    } else {
      // 跳转到媒体详情页
      router.push({
        path: '/media',
        query: {
          mediaid: getMediaId(),
          title: props.media?.title,
          year: props.media?.year,
          type: props.media?.type,
        },
      })
    }
  }
}

// 点击搜索
async function clickSearch() {
  if (allSites.value?.length > 0) return
  querySites()
  querySelectedSites()
}

// 开始搜索
function handleSearch() {
  router.push({
    path: '/resource',
    query: {
      keyword: getMediaId(),
      type: props.media?.type,
      area: 'title',
      title: props.media?.title,
      year: props.media?.year,
      season: props.media?.season,
      sites: selectedSites.value.join(','),
    },
  })
}

// 懒加载检查
function handleCheckLazy() {
  if (props.media?.collection_id) {
    return
  }
  handleCheckSubscribe()
  handleCheckExists()
}

// 在元素进入视窗时触发懒加载函数
function setupIntersectionObserver() {
  if (mediaCardRef.value) {
    observer.value = new IntersectionObserver(
      entries => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            // 只要MediaCard进入视窗，就调用懒加载的操作
            handleCheckLazy()
            // 加载后销毁观察者实例
            observer.value?.disconnect()
            observer.value = null
          }
        })
      },
      { threshold: 0.1 },
    )
    observer.value.observe(mediaCardRef.value)
  }
}

onMounted(() => {
  setupIntersectionObserver()
})

onBeforeUnmount(() => {
  observer.value?.disconnect()
  observer.value = null
})

// 计算图片地址
const getImgUrl: Ref<string> = computed(() => {
  if (imageLoadError.value) return noImage
  const url = props.media?.poster_path?.replace('original', 'w500') ?? noImage
  // 使用图片缓存
  if (globalSettings.GLOBAL_IMAGE_CACHE)
    return `${import.meta.env.VITE_API_BASE_URL}system/cache/image?url=${encodeURIComponent(url)}`
  // 如果地址中包含douban则使用中转代理
  if (url.includes('doubanio.com'))
    return `${import.meta.env.VITE_API_BASE_URL}system/img/0?imgurl=${encodeURIComponent(url)}`
  return url
})

// 拼装季图片地址
function getSeasonPoster(posterPath: string) {
  if (!posterPath) return ''
  return `https://${globalSettings.TMDB_IMAGE_DOMAIN}/t/p/w500${posterPath}`
}

// 将yyyy-mm-dd转换为yyyy年mm月dd日
function formatAirDate(airDate: string) {
  if (!airDate) return ''
  const date = new Date(airDate.replaceAll(/-/g, '/'))
  return `${date.getFullYear()}年${date.getMonth() + 1}月${date.getDate()}日`
}

// 从yyyy-mm-dd中提取年份
function getYear(airDate: string) {
  if (!airDate) return ''
  const date = new Date(airDate.replaceAll(/-/g, '/'))
  return date.getFullYear()
}

// 移除订阅
function onRemoveSubscribe() {
  subscribeEditDialog.value = false
}
</script>

<template>
  <VHover>
    <template #default="hover">
      <div ref="mediaCardRef">
        <VCard
          v-bind="hover.props"
          :height="props.height"
          :width="props.width"
          class="outline-none shadow ring-gray-500 rounded-lg"
          :class="{
            'transition transform-cpu duration-300 scale-105 shadow-lg': hover.isHovering,
            'ring-1': isImageLoaded,
          }"
          @click.stop="goMediaDetail(hover.isHovering ?? false)"
        >
          <VImg
            aspect-ratio="2/3"
            :src="getImgUrl"
            class="object-cover aspect-w-2 aspect-h-3"
            cover
            @load="isImageLoaded = true"
            @error="imageLoadError = true"
          >
            <template #placeholder>
              <div class="w-full h-full">
                <VSkeletonLoader class="object-cover aspect-w-2 aspect-h-3" />
              </div>
            </template>
          </VImg>
          <!-- 详情 -->
          <VCardText
            v-show="hover.isHovering || imageLoadError || searchMenuShow"
            class="w-full h-full flex flex-col flex-wrap justify-end align-left text-white absolute bottom-0 cursor-pointer pa-2"
            style="background: linear-gradient(rgba(45, 55, 72, 40%) 0%, rgba(45, 55, 72, 90%) 100%)"
          >
            <span class="font-bold">{{ props.media?.year }}</span>
            <h1 class="mb-1 text-white font-extrabold text-xl line-clamp-2 overflow-hidden text-ellipsis ...">
              {{ props.media?.title }}
            </h1>
            <p class="leading-4 line-clamp-4 overflow-hidden text-ellipsis ...">
              {{ props.media?.overview }}
            </p>
            <div v-if="props.media?.collection_id" class="mb-3" @click.stop=""></div>
            <div v-else class="flex align-center justify-between">
              <VMenu close-on-content-click v-model="searchMenuShow" max-width="450">
                <template v-slot:activator="{ props }">
                  <IconBtn v-bind="props" icon="mdi-magnify" color="white" @click.stop="clickSearch" />
                </template>
                <VList>
                  <VListItem>
                    <VChipGroup v-model="selectedSites" column multiple @click.stop>
                      <VChip
                        v-for="site in allSites"
                        :key="site.id"
                        :color="selectedSites.includes(site.id) ? 'primary' : ''"
                        filter
                        variant="outlined"
                        :value="site.id"
                        size="small"
                      >
                        {{ site.name }}
                      </VChip>
                    </VChipGroup>
                  </VListItem>
                  <VListItem>
                    <VBtn @click="handleSearch" block>搜索</VBtn>
                  </VListItem>
                </VList>
              </VMenu>
              <IconBtn icon="mdi-heart" :color="isSubscribed ? 'error' : 'white'" @click.stop="handleSubscribe" />
            </div>
          </VCardText>
          <!-- 类型角标 -->
          <VChip
            v-show="isImageLoaded"
            variant="elevated"
            size="small"
            :class="getChipColor(props.media?.type || '')"
            class="absolute left-2 top-2 bg-opacity-80 shadow-md text-white font-bold"
          >
            {{ props.media?.type }}
          </VChip>
          <!-- 本地存在标识 -->
          <ExistIcon v-if="isExists && !hover.isHovering" />
          <!-- 评分角标 -->
          <VChip
            v-if="isImageLoaded && props.media?.vote_average && !(isExists && !hover.isHovering)"
            variant="elevated"
            size="small"
            :class="getChipColor('rating')"
            class="absolute right-2 top-2 bg-opacity-80 shadow-md text-white font-bold"
          >
            {{ formatRating(props.media?.vote_average) }}
          </VChip>
          <!--来源图标-->
          <VAvatar
            size="24"
            density="compact"
            class="absolute bottom-1 right-1"
            tile
            v-if="!hover.isHovering && isImageLoaded && props.media?.source && !imageLoadError"
          >
            <VImg cover :src="sourceIconDict[props.media?.source]" class="shadow-lg" />
          </VAvatar>
        </VCard>
      </div>
    </template>
  </VHover>
  <!-- 订阅季弹窗 -->
  <VBottomSheet v-if="subscribeSeasonDialog" v-model="subscribeSeasonDialog" inset scrollable>
    <VCard class="rounded-t">
      <DialogCloseBtn @click="subscribeSeasonDialog = false" />
      <VCardItem>
        <VCardTitle class="pe-10"> 订阅 - {{ props.media?.title }} </VCardTitle>
      </VCardItem>
      <VDivider />
      <VCardText>
        <VList v-model:selected="seasonsSelected" lines="three" select-strategy="classic">
          <VListItem v-for="(item, i) in seasonInfos" :key="i" :value="item">
            <template #prepend>
              <VImg
                height="90"
                width="60"
                :src="getSeasonPoster(item.poster_path || '')"
                aspect-ratio="2/3"
                class="object-cover rounded shadow ring-gray-500 me-3"
                cover
              >
                <template #placeholder>
                  <div class="w-full h-full">
                    <VSkeletonLoader class="object-cover aspect-w-2 aspect-h-3" />
                  </div>
                </template>
              </VImg>
            </template>
            <VListItemTitle> 第 {{ item.season_number }} 季 </VListItemTitle>
            <VListItemSubtitle class="mt-1 me-2">
              <VChip v-if="item.vote_average" color="primary" size="small" class="mb-1">
                <VIcon icon="mdi-star" /> {{ item.vote_average }}
              </VChip>
              {{ getYear(item.air_date || '') }} • {{ item.episode_count }} 集
            </VListItemSubtitle>
            <VListItemSubtitle>
              《{{ media?.title }}》第 {{ item.season_number }} 季于 {{ formatAirDate(item.air_date || '') }} 首播。
            </VListItemSubtitle>
            <VListItemSubtitle>
              <VChip v-if="seasonsNotExisted" class="mt-2" size="small" :color="getExistColor(item.season_number || 0)">
                {{ getExistText(item.season_number || 0) }}
              </VChip>
            </VListItemSubtitle>
            <template #append="{ isSelected }">
              <VListItemAction start>
                <VSwitch :model-value="isSelected" />
              </VListItemAction>
            </template>
          </VListItem>
        </VList>
      </VCardText>
      <div class="my-2 text-center">
        <VBtn size="large" :disabled="seasonsSelected.length === 0" width="30%" @click="subscribeSeasons">
          {{ seasonsSelected.length === 0 ? '请选择订阅季' : '提交订阅' }}
        </VBtn>
      </div>
    </VCard>
  </VBottomSheet>
  <!-- 订阅编辑弹窗 -->
  <SubscribeEditDialog
    v-if="subscribeEditDialog"
    v-model="subscribeEditDialog"
    :subid="subscribeId"
    @close="subscribeEditDialog = false"
    @save="subscribeEditDialog = false"
    @remove="onRemoveSubscribe"
  />
</template>
