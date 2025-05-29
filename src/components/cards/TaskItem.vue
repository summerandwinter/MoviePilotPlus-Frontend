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

// 更多来源界面
const showMoreTorrents = ref(false)

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
onMounted(() => {
  getSiteSeedList()
})
</script>

<template>
  <div>

    <VListItem @click.stop="goDetail()" class="mb-2" variant="flat">
      <VChip variant="outlined" size="small"
        class="bg-green-600 border-green-600 absolute left-2 top-2 bg-opacity-90 shadow-md text-white font-bold">
        {{ getCollectStatus(task?.status) }}
      </VChip>

      <template v-if="!showMoreTorrents" #prepend>
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
        <!-- <VChipGroup class="p-3" column>
              <VChip v-for="(item, index) in siteSeedList" :key="index">
                <template #append>
                  <VBadge color="primary" :content="getSeedStatus(item.status)" inline size="x-small" />
                </template>
               
                {{ item.site_name }}
              </VChip>
            </VChipGroup> -->
        <div class="p-3">
          <template v-for="(item, index) in siteSeedList">
            <template v-if="showSeedStatus(item.status)">
              <VBadge color="primary" class="mr-5" :content="getSeedStatus(item.status)" size="x-small">
                <VChip>
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
