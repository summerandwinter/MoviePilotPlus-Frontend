<script setup lang="ts">
import VideoDetailView from '@/views/collect/VideoDetailView.vue'

// 路由参数
const route = useRoute()

// TMDBID
const mediaid = route.query?.mediaid?.toString()

// 类型
const type = route.query?.type?.toString()

// 标题
const title = route.query?.title?.toString()

// 年份
const year = route.query?.year?.toString()
// 来源
const source = route.query?.source?.toString()
</script>

<template>
  <div>
    <VideoDetailView :mediaid="mediaid" :source="source" :type="type" :title="title" :year="year" />
  </div>
</template>
