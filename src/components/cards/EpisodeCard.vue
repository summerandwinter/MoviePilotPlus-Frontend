<script setup lang="ts">
import { defineProps, PropType, ref } from 'vue'
import { useToast } from 'vue-toast-notification'
import { formatRating } from '@/@core/utils/formatters'
import { VideoEpisode } from '@/api/types'
import router from '@/router'
import noImage from '@images/no-image.jpeg'
import tmdbImage from '@images/logos/tmdb.png'
import doubanImage from '@images/logos/douban-black.png'
import bangumiImage from '@images/logos/bangumi.png'
import { useUserStore } from '@/stores'

// 输入参数
const props = defineProps({
  episode: Object as PropType<VideoEpisode>,
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

const isSelected = ref(false)

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



// 角标颜色
function getChipColor(type: string) {
  if (type === '电影') return 'border-blue-500 bg-blue-600'
  else if (type === '电视剧') return ' bg-indigo-500 border-indigo-600'
  else return 'border-purple-600 bg-purple-600'
}

function selectEsipode() {
  if (!props.episode) return
  props.episode.selected = !props.episode.selected
}

// 打开详情页
function goMediaDetail(isHovering = false) {
  if (isHovering) {
    // 跳转到媒体详情页
    router.push({
      path: '/video',
      query: {
        source: props.episode?.source,
        mediaid: props.episode?.cid,
        title: props.episode?.title,
        year: props.episode?.year,
        type: props.episode?.type,
      },
    })
  }
}


onMounted(() => {

})

onBeforeUnmount(() => {
})

// 计算图片地址
const getImgUrl: Ref<string> = computed(() => {
  if (imageLoadError.value) return noImage
  const url = props.episode?.image_url ?? noImage
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
      'ring-1': isImageLoaded,  }"
      @click.stop="selectEsipode">
          <VImg  :src="getImgUrl" class="align-end" cover
            @load="isImageLoaded = true" @error="imageLoadError = true">
            
            <VCardTitle class="text-white" v-text="props.episode.play_title || props.episode?.title"></VCardTitle>
          </VImg>
          
  
          
          <!--来源图标-->
          <VIcon v-if="props.episode.selected" icon="mdi-check"  color="success" class="absolute top-1 right-1"
          />
        </VCard>
      </div>
    </template>
  </VHover>
</template>
