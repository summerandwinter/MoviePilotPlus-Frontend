<script setup lang="ts">
import { defineProps, PropType, ref } from 'vue'
import { useToast } from 'vue-toast-notification'
import { formatRating } from '@/@core/utils/formatters'
import { VideoInfo } from '@/api/types'
import router from '@/router'
import noImage from '@images/no-image.jpeg'
import tmdbImage from '@images/logos/tmdb.png'
import doubanImage from '@images/logos/douban-black.png'
import bangumiImage from '@images/logos/bangumi.png'
import { useUserStore } from '@/stores'

// 输入参数
const props = defineProps({
  media: Object as PropType<VideoInfo>,
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


// 本地存在状态
const isExists = ref(false)



// 来源角标字典
const sourceIconDict: { [key: string]: any } = {
  themoviedb: tmdbImage,
  douban: doubanImage,
  bangumi: bangumiImage,
}

// 绑定MediaCard元素
const videoCardRef = ref<HTMLElement | null>(null)


// 搜索菜单显示状态
const searchMenuShow = ref(false)



// 获得mediaid
function getMediaId() {
  if (props.media?.tmdb_id) return `tmdb:${props.media?.tmdb_id}`
  else if (props.media?.douban_id) return `douban:${props.media?.douban_id}`
  else if (props.media?.bangumi_id) return `bangumi:${props.media?.bangumi_id}`
  else return `${props.media?.source}:${props.media?.vid}`
}


// 角标颜色
function getChipColor(type: string) {
  if (type === '电影') return 'border-blue-500 bg-blue-600'
  else if (type === '电视剧') return ' bg-indigo-500 border-indigo-600'
  else return 'border-purple-600 bg-purple-600'
}

// 打开详情页
function goMediaDetail(isHovering = false) {
  if (isHovering) {
    if (props.media?.vid) {
      // 跳转到合集列表
      router.push({
        path: `/collection/detail/${props.media?.vid}`,
        query: {
          title: props.media?.title,
        },
      })
    } else {
      // 跳转到媒体详情页
      router.push({
        path: '/video',
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


onMounted(() => {

})

onBeforeUnmount(() => {
})

// 计算图片地址
const getImgUrl: Ref<string> = computed(() => {
  if (imageLoadError.value) return noImage
  const url = props.media?.new_pic_vt ?? noImage
  // 使用图片缓存
  if (globalSettings.GLOBAL_IMAGE_CACHE)
    return `${import.meta.env.VITE_API_BASE_URL}system/cache/image?url=${encodeURIComponent(url)}`
  // 如果地址中包含douban则使用中转代理
  if (url.includes('doubanio.com'))
    return `${import.meta.env.VITE_API_BASE_URL}system/img/0?imgurl=${encodeURIComponent(url)}`
  return url
})

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

</script>

<template>
  <VHover>
    <template #default="hover">
      <div ref="videoCardRef">
        <VCard v-bind="hover.props" :height="props.height" :width="props.width"
          class="outline-none shadow ring-gray-500 rounded-lg" :class="{
      'transition transform-cpu duration-300 scale-105 shadow-lg': hover.isHovering,
      'ring-1': isImageLoaded,
    }" @click.stop="goMediaDetail(hover.isHovering ?? false)">
          <VImg aspect-ratio="2/3" :src="getImgUrl" class="object-cover aspect-w-2 aspect-h-3" cover
            @load="isImageLoaded = true" @error="imageLoadError = true">
            <template #placeholder>
              <div class="w-full h-full">
                <VSkeletonLoader class="object-cover aspect-w-2 aspect-h-3" />
              </div>
            </template>
          </VImg>
          <!-- 详情 -->
          <VCardText v-show="hover.isHovering || imageLoadError || searchMenuShow"
            class="w-full h-full flex flex-col flex-wrap justify-end align-left text-white absolute bottom-0 cursor-pointer pa-2"
            style="background: linear-gradient(rgba(45, 55, 72, 40%) 0%, rgba(45, 55, 72, 90%) 100%)">
            <span class="font-bold">{{ props.media?.year }}</span>
            <h1 class="mb-1 text-white font-extrabold text-xl line-clamp-2 overflow-hidden text-ellipsis ...">
              {{ props.media?.title }}
            </h1>
            <p class="leading-4 line-clamp-4 overflow-hidden text-ellipsis ...">
              {{ props.media?.overview }}
            </p>
            <div v-if="props.media?.vid" class="mb-3" @click.stop=""></div>
          </VCardText>
          <!-- 类型角标 -->
          <VChip v-show="isImageLoaded" variant="elevated" size="small" :class="getChipColor(props.media?.type || '')"
            class="absolute left-2 top-2 bg-opacity-80 shadow-md text-white font-bold">
            {{ props.media?.type }}
          </VChip>
          <!-- 本地存在标识 -->
          <ExistIcon v-if="isExists && !hover.isHovering" />
          <!-- 评分角标 -->
          <VChip v-if="isImageLoaded && props.media?.vote_average && !(isExists && !hover.isHovering)"
            variant="elevated" size="small" :class="getChipColor('rating')"
            class="absolute right-2 top-2 bg-opacity-80 shadow-md text-white font-bold">
            {{ formatRating(props.media?.vote_average) }}
          </VChip>
          <!--来源图标-->
          <VAvatar size="24" density="compact" class="absolute bottom-1 right-1" tile
            v-if="!hover.isHovering && isImageLoaded && props.media?.source && !imageLoadError">
            <VImg cover :src="sourceIconDict[props.media?.source]" class="shadow-lg" />
          </VAvatar>
        </VCard>
      </div>
    </template>
  </VHover>
</template>
