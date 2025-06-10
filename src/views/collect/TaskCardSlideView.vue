<script lang="ts" setup>
import type { DownloadTask, Progress } from '@/api/types'
import TaskCard from '@/components/cards/TaskCard.vue'
import SlideView from '@/components/slide/SlideView.vue'
import { useI18n } from 'vue-i18n'

const { t } = useI18n()

// 输入参数
const props = defineProps({
  title: String,
  taskList: {
    type: Array as PropType<DownloadTask[]>,
    default: () => []
  },
  progress: {
    type: Array as PropType<Progress[]>,
    default: () => []
  },
  width: String,
  height: String,
})

// 提供给子组件的属性
provide('rankingPropsKey', reactive({ ...props }))

// 组件加载完成
const componentLoaded = ref(false)

// 数据列表
const dataList = ref<DownloadTask[]>([])
function removeTask(id: number) {
  const index = dataList.value.findIndex(item => item.id === id)
  if (index !== -1) {
    dataList.value.splice(index, 1)
  }
}
// 加载时获取数据
onMounted(() => {
  dataList.value = props.taskList
  componentLoaded.value = true
})
onActivated(() => {
  dataList.value = props.taskList
  componentLoaded.value = true
})
</script>

<template>
  <SlideView v-if="componentLoaded">
    <template #content>
      <template v-for="data in props.taskList" :key="data.id">
        <TaskCard :info="data" :progress="progress" height="11rem" width="20rem" @remove="removeTask" />
      </template>
    </template>
  </SlideView>
</template>
