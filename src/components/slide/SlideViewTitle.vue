<script lang="ts" setup>
// 输入参数
const props: any = inject('rankingPropsKey')
console.log('props: ', props)
</script>

<template>
  <div class="ms-1">
    <span class="slider-title" v-if="props?.link_to === 'disabled'">{{ props?.title }}</span>
    <RouterLink v-else :to="props?.linkurl ? props?.linkurl : ''" class="slider-title">
      <span>{{ props?.title }}</span>
      <VIcon icon="mdi-arrow-right-circle-outline" class="ms-1" />
    </RouterLink>
  </div>
</template>
