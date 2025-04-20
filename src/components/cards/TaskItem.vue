<script lang="ts" setup>
import type { PropType } from 'vue'
import router from '@/router'
import type { Collect } from '@/api/types'


// 输入参数
const props = defineProps({
  task: Object as PropType<Collect>,
})

// 更多来源界面
const showMoreTorrents = ref(false)

// 任务信息
const task = ref(props.task)

// 站点图标
const siteIcon = ref('')

function goDetail() {

    // 跳转到媒体详情页
    router.push({
      path: '/cdetail',
      query: {
        id: props.task?.id
      },
    })

}
// 促销Chip类
function getVolumeFactorClass(downloadVolume: number, uploadVolume: number) {
  if (downloadVolume === 0) return 'text-white bg-lime-500'
  else if (downloadVolume < 1) return 'text-white bg-green-500'
  else if (uploadVolume !== 1) return 'text-white bg-sky-500'
  else return 'text-white bg-gray-500'
}

onMounted(() => {
})
</script>

<template>
  <div>
    
    <VListItem
    @click.stop="goDetail()"  class="mb-2"
      variant="flat"
    >
    <VChip
              variant="outlined"
              size="small"
              class="bg-green-600 border-green-600 absolute left-2 top-2 bg-opacity-90 shadow-md text-white font-bold"
            >
              {{task?.status}}
            </VChip>

      <template v-if="!showMoreTorrents" #prepend>
        <VListItemMedia class="pr-2"><VImg :src="siteIcon" :width="60" 
  aspect-ratio="9/16"
  cover/></VListItemMedia>
        
      </template>
      <VListItemTitle class="break-words overflow-visible whitespace-break-spaces">
        {{ task?.name }}
        <span class="text-green-700 ms-2 text-sm">↑{{ task?.episodes_downloaded }}</span>
        <span class="text-orange-700 ms-2 text-sm">↓{{ task?.episodes_total }}</span>
      </VListItemTitle>
      <VListItemSubtitle> 【{{ task?.type }}】{{ task?.sub_title }} </VListItemSubtitle>
      <div class="pt-2">
        
        <VChip v-if="task?.team" variant="elevated" size="small" class="me-1 mb-1 text-white bg-red-500">
          {{ task?.team }}
        </VChip>
        <VChip v-if="task?.copyright" variant="elevated" size="small" class="me-1 mb-1 text-white bg-red-500">
          {{ task?.copyright }}
        </VChip>
        
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
                <VListItem
                  variant="plain"
                
                >
                  <template #prepend>
                    <VIcon icon="mdi-download" />
                  </template>
                  <VListItemTitle>下载种子文件</VListItemTitle>
                </VListItem>
              </VList>
            </VMenu>
          </IconBtn>
        </div>
      </template>
    </VListItem>
  </div>
</template>
