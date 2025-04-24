<script setup lang="ts">

import api from '@/api'
import { CollectProgress, SiteSeed } from '@/api/types'
import { useToast } from 'vue-toast-notification'
const progress = ref<Array<CollectProgress>>([])
// 输入参数
const props = defineProps({
  seed: {
    type: Object as () => SiteSeed,
    default: () => ({})
  }
})
// 提示框
const $toast = useToast()
// 加载中
const loading = ref(false)
async function getProgressInfo() {
  try {
    progress.value = await api.get(`collect/progress/seed/${props?.seed.id}`)
  } catch (error) {
    console.error(error)
  }
}

// 调用API添加采集任务
async function publish(id: number) {
  loading.value = true
  try {
    // 请求API
    const result: { [key: string]: any } = await api.get('collect/torrent_publish/' + props?.seed.id)
    // 添加采集任务状态
    if (result.success) {
      // 成功
      $toast.success(`发布事件提交成功！`)

    } else {
      $toast.error(`发布事件提交失败`)
    }
  } catch (error) {
    console.error(error)
  }
  loading.value = false
}

async function deleteSeed() {
  try {
    loading.value = true
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
  loading.value = false
}
// 注册事件
const emit = defineEmits(['close'])

onMounted(() => {
  getProgressInfo()
})
</script>
<template>
  <VDialog scrollable max-width="60rem" :scrim="false" transition="dialog-bottom-transition">
    <VCard>
      <!-- Toolbar -->
      <div>
        <VToolbar color="primary">
          <VToolbarTitle>{{ `做种信息 - ${props.seed?.site_name}` }}</VToolbarTitle>
          <VSpacer />
          <VToolbarItems>
            <VBtn icon variant="plain" @click="emit('close')" class="me-3">
              <VIcon size="large" color="white" icon="ri-close-line" />
            </VBtn>
          </VToolbarItems>
        </VToolbar>
      </div>
      <VCardText class="d-flex flex-row  justify-center ">
        <v-sheet
          class="d-flex align-center justify-center flex-wrap mx-auto px-4"
          elevation="0"
          > 
          <VTimeline align="start">
            <VTimelineItem
              :dot-color="item.success?'success':'error'"
              size="small"
              v-for="(item, index) in progress"
            >
              <div class="d-flex">
                <strong class="me-4">{{item.created_at}}</strong>
                <div>
                  <strong>{{item.name}}</strong>
                  <div class="text-caption" v-if="!item.success">
                    {{item.error_msg}}
                  </div>
                </div>
              </div>
            </VTimelineItem>
          </VTimeline>
        </v-sheet>
      </VCardText>
      <VCardItem class="text-center mt-10">
        <VBtn
          variant="elevated"
          @click="publish"
          :disabled="loading"
          color="success"
          prepend-icon="mdi-progress-upload"
          class="px-5"
          size="small"
        >
          发布
        </VBtn>
        <VBtn
          variant="elevated"
          @click="publish"
          :disabled="loading"
          color="error"
          prepend-icon="mdi-delete"
          class="px-5 ml-5"
          size="small"
        >
          删除
        </VBtn>
      </VCardItem>
    </VCard>
  </VDialog>
</template>

<style lang="scss" scoped>
.v-table th {
  white-space: nowrap;
}
</style>
