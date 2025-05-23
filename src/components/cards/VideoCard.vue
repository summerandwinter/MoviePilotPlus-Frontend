<script setup lang="ts">
import { defineProps, PropType, ref } from 'vue'
import { useToast } from 'vue-toast-notification'
import api from '@/api'
import { VideoInfo } from '@/api/types'
import router, { registerAbortController } from '@/router'
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

// 创建Intersection Observer实例
const observer = ref<IntersectionObserver | null>(null)

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


// 绑定MediaCard元素
const videoCardRef = ref<HTMLElement | null>(null)


// 搜索菜单显示状态
const searchMenuShow = ref(false)



// 获得mediaid
function getMediaId() {
  if (props.media?.tmdb_id) return `tmdb:${props.media?.tmdb_id}`
  else if (props.media?.douban_id) return `douban:${props.media?.douban_id}`
  else if (props.media?.bangumi_id) return `bangumi:${props.media?.bangumi_id}`
  else return `${props.media?.source}:${props.media?.cid}`
}



// 打开详情页
function goMediaDetail(isHovering = false) {
  if (isHovering) {
    // 跳转到媒体详情页
    router.push({
      path: '/video',
      query: {
        source: props.media?.source,
        mediaid: props.media?.cid,
        title: props.media?.title,
        year: props.media?.year,
        type: props.media?.type,
      },
    })
  }
}
// 查询当前媒体是否已入库
async function handleCheckExists() {
  try {
    const abortController = new AbortController()
    registerAbortController(abortController)
    const { signal } = abortController
    const result: { [key: string]: any } = await api.get('task/exist_cid/' + props.media?.cid, {
      params: {},
      signal,
    })

    if (result.success) isExists.value = true
  } catch (error) {
    console.error(error)
  }
}
// 懒加载检查
function handleCheckLazy() {
  console.log('handleCheckLazy', props.media?.cid)
  // if (props.media?.cid) {
  //   return
  // }
  handleCheckExists()
}
// 在元素进入视窗时触发懒加载函数
function setupIntersectionObserver() {
  if (videoCardRef.value) {
    observer.value = new IntersectionObserver(
      entries => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            // 只要 VideoCard 进入视窗，就调用懒加载的操作
            handleCheckLazy()
            // 加载后销毁观察者实例
            observer.value?.disconnect()
            observer.value = null
          }
        })
      },
      { threshold: 0.1 },
    )
    observer.value.observe(videoCardRef.value)
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
          <VChip v-show="isImageLoaded && props.media?.pay_type && !hover.isHovering" variant="elevated" size="small" 
            class="absolute left-2 top-2 bg-opacity-80 shadow-md text-white font-bold border-red-600 bg-red-600">
            {{ props.media?.pay_type }}
          </VChip>
          <!-- 本地存在标识 -->
          <ExistIcon v-if="isExists && !hover.isHovering" />
          <!-- 评分角标 -->
          <VChip v-if="isImageLoaded && !hover.isHovering && props.media?.rating"
            variant="elevated" size="small"
            class="absolute right-2 bottom-2 bg-opacity-80 shadow-md text-white font-bold border-purple-600 bg-purple-600">
            {{ props.media?.rating }}
          </VChip>
          
        </VCard>
      </div>
    </template>
  </VHover>
</template>
