<script setup lang="ts">

import { useToast } from 'vue-toast-notification'
import api from '@/api'
import { CollectProgress, Collect } from '@/api/types'
const progress = ref<Array<CollectProgress>>([])
// 输入参数
const props = defineProps({
  operation: String,
  collect: {
    type: Object as () => Collect,
    default: () => ({})
  }
})
// 提示框
const $toast = useToast()
// 注册事件
const emit = defineEmits(['close'])

function getIcon() {
  switch (props.operation) {
    case 'start_download_by_collect':
      return 'mdi-arrow-down-bold-circle'
    case 'metadata_by_collect':
      return 'mdi-camcorder-box'
    case 'screenshot_by_collect':
      return 'mdi-camera'
    case 'collect_desc_by_collect':
      return 'mdi-format-text'
    case 'collect_move':
      return 'mdi-rename-box'
    case 'torrent_create':
      return 'mdi-chevron-double-right'
    default:
      return 'mdi-arrow-down-bold-circle'
  }
}
function getTitle() {
  switch (props.operation) {
    case 'start_download_by_collect':
      return '下载媒体文件'
    case 'metadata_by_collect':
      return '采集视频元数据'
    case 'screenshot_by_collect':
      return '采集视频截图'
    case 'collect_desc_by_collect':
      return '采集简介信息'
    case 'collect_move':
      return '重命名媒体文件'
    case 'torrent_create':
      return '创建种子文件'
    default:
      return '操作'
  }
}
function getText() {
  switch (props.operation) {
    case 'start_download_by_collect':
      return '推送下载媒体文件事件，稍后可以在下载任务中查看下载进度'
    case 'metadata_by_collect':
      return '推送采集视频元数据事件，稍后可以在详情中查看采集结果'
    case 'screenshot_by_collect':
      return '推送采集视频截图事件，稍后可以在详情中查看采集结果'
    case 'collect_desc_by_collect':
      return '推送采集简介信息事件，稍后可以在详情中查看采集结果'
    case 'collect_move':
      return '推送重命名媒体文件事件，稍后可以在详情中查看采集结果'
    case 'torrent_create':
      return '推送创建种子文件事件，稍后可以在详情中查看采集结果'
    default:
      return '异步操作，请稍后查看结果'
  }
}
async function handleSubmit() {
  let result: { [key: string]: any }
   result = await api.get(`collect/${props?.operation}/${props?.collect.id}`)
   const title = getTitle()
 if (result && result.success) {
      // 添加下载成功
      $toast.success(`${title}事件发送成功！`)
      // 下载成功，返回链接
      emit('close')
    } else {
      // 添加下载失败
      $toast.error(`${title}事件发送失败：${result?.message}！`)
      // 下载失败，返回错误原因
      emit('close')
    }
}
onMounted(() => {

})
</script>
<template>
  <VDialog width="auto" transition="dialog-bottom-transition">
    <VCard max-width="400"
        :prepend-icon="getIcon()"
        :text="getText()"
        :title="getTitle()">
        <template v-slot:actions>
          <v-spacer></v-spacer>
          <VBtn
            class="ms-auto"
            text="取消"
            @click="emit('close')"
          ></VBtn>
          <VBtn
            class="ms-auto"
            text="确定"
            @click="handleSubmit"
          ></VBtn>
        </template>
    </VCard>
  </VDialog>
</template>

<style lang="scss" scoped>
.v-table th {
  white-space: nowrap;
}
</style>
