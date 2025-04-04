<script lang="ts" setup>
import { useToast } from 'vue-toast-notification'
import { useConfirm } from 'vuetify-use-dialog'
import SubscribeEditDialog from '../dialog/SubscribeEditDialog.vue'
import SubscribeFilesDialog from '../dialog/SubscribeFilesDialog.vue'
import SubscribeShareDialog from '../dialog/SubscribeShareDialog.vue'
import { formatDateDifference, formatSeason } from '@/@core/utils/formatters'
import api from '@/api'
import type { Subscribe } from '@/api/types'
import router from '@/router'

// 输入参数
const props = defineProps({
  media: Object as PropType<Subscribe>,
})

// 从 provide 中获取全局设置
const globalSettings: any = inject('globalSettings')

// 定义触发的自定义事件
const emit = defineEmits(['remove', 'save'])

// 确认框
const createConfirm = useConfirm()

// 提示框
const $toast = useToast()

// 图片是否加载完成
const imageLoaded = ref(false)

// 订阅弹窗
const subscribeEditDialog = ref(false)

// 订阅文件信息弹窗
const subscribeFilesDialog = ref(false)

// 分享订阅弹窗
const subscribeShareDialog = ref(false)

// 当前的订阅状态
const subscribeState = ref<string>(props.media?.state ?? 'P')

// 上一次更新时间
const lastUpdateText = computed(() => (props.media?.last_update ? formatDateDifference(props.media.last_update) : ''))

// 图片加载完成响应
function imageLoadHandler() {
  imageLoaded.value = true
}

// 计算百分比
function getPercentage() {
  if (props.media?.total_episode === 0) return 0

  return Math.round(
    (((props.media?.total_episode ?? 0) - (props.media?.lack_episode ?? 0)) / (props.media?.total_episode ?? 1)) * 100,
  )
}

// 删除订阅
async function removeSubscribe() {
  try {
    const result: { [key: string]: any } = await api.delete(`subscribe/${props.media?.id}`)

    if (result.success) {
      // 通知父组件刷新
      emit('remove')
    }
  } catch (e) {
    console.log(e)
  }
}

// 搜索订阅
async function searchSubscribe() {
  try {
    const result: { [key: string]: any } = await api.get(`subscribe/search/${props.media?.id}`)

    // 提示
    if (result.success) $toast.success(`${props.media?.name} 提交搜索请求成功！`)
  } catch (e) {
    console.log(e)
  }
}

// 切换订阅状态
async function toggleSubscribeStatus(state: 'R' | 'S') {
  try {
    // 根据传入的 state 判断对应的操作文字
    const action = state === 'S' ? '暂停' : '启用'
    // 弹出确认框
    const isConfirmed = await createConfirm({
      title: `确认${action}`,
      content: `是否${action}订阅 ${props.media?.name}？`,
    })
    if (!isConfirmed) return
    // 调用 API 更新订阅状态
    const result: { [key: string]: any } = await api.put(`subscribe/status/${props.media?.id}?state=${state}`)
    // 提示
    if (result.success) {
      $toast.success(`${props.media?.name} 已${action}！`)
      subscribeState.value = state
      emit('save')
    } else {
      $toast.error(`${action}失败：${result.message}`)
    }
  } catch (e) {
    console.log(e)
  }
}

// 重置订阅
async function resetSubscribe() {
  // 确认
  try {
    const isConfirmed = await createConfirm({
      title: '确认',
      content: `重置后 ${props.media?.name} 将恢复初始状态，已下载记录将被清除，未入库的内容将会重新下载，是否确认？`,
    })
    if (!isConfirmed) return
    // 重置
    const result: { [key: string]: any } = await api.get(`subscribe/reset/${props.media?.id}`)
    // 提示
    if (result.success) {
      $toast.success(`${props.media?.name} 重置成功！`)
      subscribeState.value = 'R'
      emit('save')
    } else $toast.error(`${props.media?.name} 重置失败：${result.message}`)
  } catch (e) {
    console.log(e)
  }
}

//  分享订阅
async function shareSubscribe() {
  subscribeShareDialog.value = true
}

// 编辑订阅响应
async function editSubscribeDialog() {
  subscribeEditDialog.value = true
}

// 获得mediaid
function getMediaId() {
  if (props.media?.tmdbid) return `tmdb:${props.media?.tmdbid}`
  else if (props.media?.doubanid) return `douban:${props.media?.doubanid}`
  else if (props.media?.bangumiid) return `bangumi:${props.media?.bangumiid}`
  else return props.media?.mediaid
}

// 查看媒体详情
async function viewMediaDetail() {
  router.push({
    path: '/media',
    query: {
      mediaid: getMediaId(),
      title: props.media?.name,
      year: props.media?.year,
      type: props.media?.type,
    },
  })
}

// 查看文件详情
async function viewSubscribeFiles() {
  subscribeFilesDialog.value = true
}

// 弹出菜单
const dropdownItems = computed(() => [
  {
    title: '编辑',
    value: 1,
    props: {
      prependIcon: 'mdi-file-edit-outline',
      click: editSubscribeDialog,
    },
  },
  {
    title: '搜索',
    value: 2,
    props: {
      prependIcon: 'mdi-magnify',
      click: searchSubscribe,
    },
  },
  {
    title: '详情',
    value: 3,
    props: {
      prependIcon: 'mdi-information-outline',
      click: viewMediaDetail,
    },
  },
  {
    title: '文件',
    value: 4,
    props: {
      prependIcon: 'mdi-file-document-outline',
      click: viewSubscribeFiles,
    },
  },
  {
    title: subscribeState.value === 'S' ? '启用' : '暂停',
    value: 5,
    props: {
      prependIcon: subscribeState.value === 'S' ? 'mdi-play' : 'mdi-pause',
      click: () => toggleSubscribeStatus(subscribeState.value === 'S' ? 'R' : 'S'),
      color: subscribeState.value === 'S' ? 'success' : 'info',
    },
  },
  {
    title: '重置',
    value: 6,
    props: {
      prependIcon: 'mdi-restore-alert',
      click: resetSubscribe,
      color: 'warning',
    },
  },
  {
    title: '分享',
    value: 7,
    props: {
      prependIcon: 'mdi-share',
      click: shareSubscribe,
      color: 'success',
    },
    show: props.media?.type === '电视剧',
  },
  {
    title: '取消订阅',
    value: 8,
    props: {
      prependIcon: 'mdi-trash-can-outline',
      color: 'error',
      click: removeSubscribe,
    },
  },
])

// 监听插件窗口状态变化
watch(
  () => props.media?.page_open,
  (newOpenState, _) => {
    if (newOpenState) editSubscribeDialog()
  },
)

// 监听订阅状态
watch(
  () => props.media?.state,
  newState => {
    subscribeState.value = newState ?? 'P'
  },
)

// 计算backdrop图片地址
const backdropUrl = computed(() => {
  const url = props.media?.backdrop || props.media?.poster
  // 使用图片缓存
  if (globalSettings.GLOBAL_IMAGE_CACHE && url)
    return `${import.meta.env.VITE_API_BASE_URL}system/cache/image?url=${encodeURIComponent(url)}`
  return url
})

// 计算海报图片地址
const posterUrl = computed(() => {
  const url = props.media?.poster
  // 使用图片缓存
  if (globalSettings.GLOBAL_IMAGE_CACHE && url)
    return `${import.meta.env.VITE_API_BASE_URL}system/cache/image?url=${encodeURIComponent(url)}`
  return url
})

// 订阅编辑保存
function onSubscribeEditSave() {
  subscribeEditDialog.value = false
  emit('save')
}

// 订阅编辑取消
function onSubscribeEditRemove() {
  subscribeEditDialog.value = false
  emit('remove')
}
</script>

<template>
  <div>
    <VHover>
      <template #default="hover">
        <VCard
          v-bind="hover.props"
          :key="props.media?.id"
          class="flex flex-col rounded-lg h-full"
          :class="{
            'outline-dashed outline-1': props.media?.best_version && imageLoaded,
            'transition transform-cpu duration-300 scale-105 shadow-lg': hover.isHovering,
            'opacity-70': subscribeState === 'S',
          }"
          min-height="170"
          @click="editSubscribeDialog"
          :ripple="false"
        >
          <div class="me-n3 absolute top-1 right-2">
            <IconBtn>
              <VIcon icon="mdi-dots-vertical" color="white" />
              <VMenu activator="parent" close-on-content-click>
                <VList>
                  <template v-for="(item, i) in dropdownItems" :key="i">
                    <VListItem
                      v-if="item.show !== false"
                      variant="plain"
                      :base-color="item.props.color"
                      @click="item.props.click"
                    >
                      <template #prepend>
                        <VIcon :icon="item.props.prependIcon" />
                      </template>
                      <VListItemTitle v-text="item.title" />
                    </VListItem>
                  </template>
                </VList>
              </VMenu>
            </IconBtn>
          </div>
          <template #image>
            <VImg :src="backdropUrl || posterUrl" aspect-ratio="3/2" cover @load="imageLoadHandler" position="top">
              <template #placeholder>
                <div class="w-full h-full">
                  <VSkeletonLoader class="object-cover aspect-w-3 aspect-h-2" />
                </div>
              </template>
              <div class="absolute inset-0 subscribe-card-background"></div>
            </VImg>
            <div v-if="subscribeState === 'P'" class="absolute inset-0 bg-yellow-900 opacity-80 pointer-events-none" />
          </template>
          <div>
            <VCardText class="flex items-center">
              <div class="h-auto w-12 flex-shrink-0 overflow-hidden rounded-md shadow-lg" v-if="imageLoaded">
                <VImg :src="posterUrl" aspect-ratio="2/3" cover @click.stop="viewMediaDetail">
                  <template #placeholder>
                    <div class="w-full h-full">
                      <VSkeletonLoader class="object-cover aspect-w-2 aspect-h-3" />
                    </div>
                  </template>
                </VImg>
              </div>
              <div class="flex flex-col justify-center overflow-hidden pl-2 xl:pl-4">
                <div class="text-sm font-medium text-white sm:pt-1">{{ props.media?.year }}</div>
                <div class="mr-2 min-w-0 text-lg font-bold text-white">
                  {{ props.media?.name }}
                  {{ formatSeason(props.media?.season ? props.media?.season.toString() : '') }}
                </div>
              </div>
            </VCardText>
            <VCardText class="flex justify-space-between align-center flex-wrap">
              <div class="flex align-center">
                <IconBtn
                  v-if="props.media?.total_episode"
                  v-bind="props"
                  icon="mdi-progress-download"
                  color="white"
                  class="me-1"
                />
                <div v-if="props.media?.season" class="text-subtitle-2 me-4 text-white">
                  {{ (props.media?.total_episode || 0) - (props.media?.lack_episode || 0) }} /
                  {{ props.media?.total_episode }}
                </div>
                <IconBtn v-if="props.media?.username" icon="mdi-account" color="white" class="me-1" />
                <span v-if="props.media?.username" class="text-subtitle-2 me-4 text-white">
                  {{ props.media?.username }}
                </span>
              </div>
            </VCardText>
            <VCardText v-if="lastUpdateText" class="absolute right-0 bottom-0 d-flex align-center p-2 text-gray-300">
              <VIcon icon="mdi-download" class="me-1" />
              {{ lastUpdateText }}
            </VCardText>
            <div class="w-full absolute bottom-0">
              <VProgressLinear
                v-if="getPercentage() > 0"
                :model-value="getPercentage()"
                bg-color="success"
                color="success"
              />
            </div>
            <div v-if="hover.isHovering" class="me-n3 absolute top-1 right-10">
              <IconBtn><VIcon class="cursor-move text-white">mdi-drag</VIcon></IconBtn>
            </div>
          </div>
        </VCard>
      </template>
    </VHover>
    <!-- 订阅编辑弹窗 -->
    <SubscribeEditDialog
      v-if="subscribeEditDialog"
      v-model="subscribeEditDialog"
      :subid="props.media?.id"
      @remove="onSubscribeEditRemove"
      @save="onSubscribeEditSave"
      @close="subscribeEditDialog = false"
    />

    <!-- 订阅文件信息弹窗 -->
    <SubscribeFilesDialog
      v-if="subscribeFilesDialog"
      v-model="subscribeFilesDialog"
      :subid="props.media?.id"
      @close="subscribeFilesDialog = false"
    />
    <!-- 分享订阅弹窗 -->
    <SubscribeShareDialog
      v-if="subscribeShareDialog"
      v-model="subscribeShareDialog"
      :sub="props.media"
      @close="subscribeShareDialog = false"
    />
  </div>
</template>
<style lang="scss">
.subscribe-card-background {
  background-image: linear-gradient(90deg, rgba(31, 41, 55, 47%) 0%, rgb(31, 41, 55) 100%);
}
</style>
