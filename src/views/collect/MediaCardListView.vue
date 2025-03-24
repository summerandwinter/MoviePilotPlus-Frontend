<script lang="ts" setup>
import { ref, defineProps, PropType } from 'vue'
import api from '@/api'
import { VideoInfo } from '@/api/types'
import { default as VideoCard } from '@/components/cards/VideoCard.vue'
import { default as NoDataFound } from '@/components/NoDataFound.vue'

// 输入参数
const props = defineProps({
  apipath: String,
  params: Object as PropType<{ [key: string]: any }>,
})

// 判断是否有滚动条
function hasScroll() {
  return document.body.scrollHeight - (window.innerHeight || document.documentElement.clientHeight) > 2
}

// 当前页码
const page = ref(0)

// 是否加载中
const loading = ref(false)

// 是否加载完成
const isRefreshed = ref(false)

// 数据列表
const dataList = ref<VideoInfo[]>([])
const currData = ref<VideoInfo[]>([])

// 拼装参数
function getParams() {
  let params = {
    page: page.value,
  }
  console.log('page.value', page.value)
  if (props.params) params = { ...params, ...props.params }

  return params
}

// 获取列表数据
async function fetchData({ done }: { done: any }) {
  try {
    if (!props.apipath) return

    // 如果正在加载中，直接返回
    if (loading.value) {
      done('ok')
      return
    }

    // 加载到满屏或者加载出错
    if (!hasScroll()) {
      // 加载多次
      while (!hasScroll()) {
        // 设置加载中
        loading.value = true
        // 请求API
        currData.value = await api.get(props.apipath, {
          params: getParams(),
        })
        // 取消加载中
        loading.value = false
        // 标计为已请求完成
        isRefreshed.value = true
        if (currData.value.length === 0) {
          // 如果没有数据，跳出
          done('empty')
          return
        }
        // 合并数据
        dataList.value = [...dataList.value, ...currData.value]
        // 页码+1
        page.value++
        // 返回加载成功
        done('ok')
      }
    } else {
      // 加载一次
      // 设置加载中
      loading.value = true
      // 请求API
      currData.value = await api.get(props.apipath, {
        params: getParams(),
      })
      // 标计为已请求完成
      isRefreshed.value = true
      if (currData.value.length === 0) {
        // 如果没有数据，跳出
        done('empty')
      } else {
        // 合并数据
        dataList.value = [...dataList.value, ...currData.value]
        // 页码+1
        page.value++
        // 返回加载成功
        done('ok')
      }
    }
    // 取消加载中
    loading.value = false
  } catch (error) {
    console.error(error)
    // 返回加载失败
    done('error')
  }
}
</script>

<template>
  <div>
    <LoadingBanner v-if="!isRefreshed" class="mt-12" />
    <VInfiniteScroll mode="intersect" side="end" :items="dataList" class="overflow-hidden" @load="fetchData">
      <template #loading />
      <template #empty />
      <div v-if="dataList.length > 0" class="grid gap-4 grid-media-card mx-3" tabindex="0">
        <VideoCard v-for="data in dataList" :key="data.vid" :media="data" />
      </div>
      <NoDataFound v-if="dataList.length === 0 && isRefreshed" error-code="404" error-title="没有数据"
        error-description="无法获取到媒体信息。" />
    </VInfiniteScroll>
  </div>
</template>
