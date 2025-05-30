<script lang="ts" setup>
import type { PropType } from 'vue'
import router from '@/router'
import type { Collect, SiteSeed } from '@/api/types'
import api from '@/api'
import { seedStatus, collectStatus } from '@/api/constants'
import { useToast } from 'vue-toast-notification'

const $toast = useToast()

// 定义触发的自定义事件
const emit = defineEmits(['remove'])

// 从 provide 中获取全局设置
const globalSettings: any = inject('globalSettings')
// 输入参数
const props = defineProps({
  task: Object as PropType<Collect>,
})


// 任务信息
const task = ref(props.task)
const siteSeedList = ref<SiteSeed[]>([])
// 计算Poster地址
const getCoverUrl: Ref<string> = computed(() => {
  const url = props.task?.cover ?? ''
  // 使用图片缓存
  if (globalSettings.GLOBAL_IMAGE_CACHE)
    return `${import.meta.env.VITE_API_BASE_URL}system/cache/image?url=${encodeURIComponent(url)}`
  // 如果地址中包含douban则使用中转代理
  if (url.includes('doubanio.com'))
    return `${import.meta.env.VITE_API_BASE_URL}system/img/0?imgurl=${encodeURIComponent(url)}`
  return url
})

function goDetail() {
  // 跳转到媒体详情页（新标签页）
  const route = router.resolve({
    path: '/cdetail',
    query: {
      id: props.task?.id
    },
  })
  window.open(route.href, '_blank')
}
function getSeedStatus(status: string) {
  return seedStatus[status as keyof typeof seedStatus]
}
function showSeedStatus(status: string) {
  return 'TorrentPublished' === status
}
function getCollectStatus(status: string | undefined) {
  return collectStatus[status as keyof typeof collectStatus]
}
async function getSiteSeedList() {
  try {
    siteSeedList.value = await api.get(`collect/seed/${props.task?.id}`)
  } catch (error) {
    console.error(error)
  }
}
async function deleteCollect(collect_id: number | undefined) {
  try {
    if (!collect_id)
      return
    await api.delete(`collect/${collect_id}`, {
      params: {
        'delete_file': true,
        'remove_seed': false
      }
    })
    // 通知父组件刷新
    emit('remove', collect_id)
    $toast.success(`删除成功`)
  } catch (error) {
    console.error(error)
  }
}
// 控制是否展开所有chip
const showAll = ref(false)

// 计算实际可见数量
const visibleCount = computed(() => showAll.value ? siteSeedList.value.length : defaultVisible.value)

// 添加容器ref
const chipContainer = ref<HTMLDivElement | null>(null)
// 单个chip的实际宽度（初始设为100px作为备用值）
const chipActualWidth = ref(100)

// 重新计算可见数量的方法
const recalculateVisible = () => {
  if (!chipContainer.value) return
  
  // 获取容器实际宽度（减去内边距）
  const containerWidth = chipContainer.value.getBoundingClientRect().width - 24 // .p-3的内边距是12px*2
  // 获取第一个chip的实际宽度（如果有chip的话）
  const firstChip = chipContainer.value.querySelector('.v-chip') as HTMLElement | null
  if (firstChip) {
    chipActualWidth.value = firstChip.getBoundingClientRect().width + 16 // 加上margin-right: 12px和margin-bottom: 4px的总和
  }
  
  // 计算实际可见数量（至少显示3个）
  defaultVisible.value = Math.max(2, Math.floor(containerWidth / chipActualWidth.value))
}

// 移除重复的computed声明，仅保留ref版本
// 计算默认显示数量（改为ref响应式变量）
const defaultVisible = ref(3)
// 更新监听逻辑
const handleResize = () => {
  recalculateVisible()
}

onMounted(() => {
  window.addEventListener('resize', handleResize)
  // 初始加载时计算一次
  setTimeout(() => recalculateVisible(), 100) // 等待DOM渲染完成
  getSiteSeedList()
})

onUpdated(() => {
  // 内容变化后重新计算
  recalculateVisible()
})

onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
})
</script>

<template>
  <div>

    <VListItem @click.stop="goDetail()" class="mb-2" variant="flat">
      <VChip variant="outlined" size="small"
        class="bg-green-600 border-green-600 absolute left-2 top-2 bg-opacity-90 shadow-md text-white font-bold">
        {{ getCollectStatus(task?.status) }}
      </VChip>

      <template #prepend>
        <VListItemMedia class="pr-2">
          <VImg :src="getCoverUrl" :width="60" aspect-ratio="9/16" cover />
        </VListItemMedia>

      </template>
      <VListItemTitle class="break-words overflow-visible whitespace-break-spaces">
        {{ task?.name }}
        <span class="text-green-700 ms-2 text-sm">↑{{ task?.episodes_downloaded }}</span>
        <span class="text-orange-700 ms-2 text-sm">↓{{ task?.episodes_total }}</span>
      </VListItemTitle>
      <VListItemSubtitle> 【{{ task?.type }}】{{ task?.sub_title }} </VListItemSubtitle>
      <div class="pt-2">
        <div class="p-3" ref="chipContainer">
          <!-- 显示可见范围内的chip -->
          <template v-for="(item, index) in siteSeedList.slice(0, visibleCount)" :key="index">
            <template v-if="showSeedStatus(item.status)">
              <VBadge color="primary" class="mr-5" :content="getSeedStatus(item.status)" size="x-small">
                <VChip class="mr-1 mb-1">
                  {{ item.site_name }}
                </VChip>
              </VBadge>
            </template>
            <template v-else>
              <VChip class="mr-1 mb-1">
                {{ item.site_name }}
              </VChip>
            </template>
          </template>
          
          <!-- 调整按钮显示条件：总数量 > 默认显示数（5） -->
          <!-- 修复：使用defaultVisible.value判断 -->
          <template v-if="siteSeedList.length > defaultVisible">
            <VBtn 
              variant="text" 
              color="primary" 
              size="x-small" 
              @click.stop="showAll = !showAll"
              class="ml-0"
            >
              <template #prepend>
                <VIcon :icon="showAll ? 'mdi-chevron-up' : 'mdi-chevron-down'" />
              </template>
              {{ showAll ? '收起' : '展开' }}
            </VBtn>
          </template>
        </div>
      </div>
        


      <template #append>
        <div class="me-n3">
          <IconBtn>
            <VIcon icon="mdi-dots-vertical" />
            <VMenu activator="parent" close-on-content-click>
              <VList>
                <VListItem variant="plain">
                  <template #prepend>
                    <VIcon icon="mdi-information" />
                  </template>
                  <VListItemTitle>查看详情</VListItemTitle>
                </VListItem>
                <VListItem variant="plain" @click="deleteCollect(task?.id)">
                  <template #prepend>
                    <VIcon icon="mdi-delete" />
                  </template>
                  <VListItemTitle>删除任务</VListItemTitle>
                </VListItem>
              </VList>
            </VMenu>
          </IconBtn>
        </div>
      </template>
    </VListItem>
  </div>
</template>
