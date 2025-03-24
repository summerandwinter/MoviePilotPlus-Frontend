<script setup lang="ts">
import api from '@/api'
import { ref, reactive, watch, onMounted } from 'vue'
import type { TencentCategoryInfo, CategoryItem } from '@/api/types'
import { default as MediaCardListView } from '@/views/collect/MediaCardListView.vue'

// 排序 类型 资费 出品 地区 年份 状态 画风 年龄 全部 性别 语言  动画明星 剧场 奖项 其他-characteristic
// 电影或者电视剧 movies/tvs
const type = ref('100113')
// 当前Key
const currentKey = ref(0)

const cates = ref<Record<string, TencentCategoryInfo[]>>({})



// 过滤参数
const defaultType = '100113'
const defaultSort = '75'

const filterParams = reactive({
  'type': defaultType,
  'sort': defaultSort,
  'itype': '-1',
  'iyear': '-1',
  'pay': '-1',
  'iarea': '-1',
  'iregion': '0',
  'itrailer': '-1',
  'producer': '-1',
  'award': '-1',
  'theater': '-1',
  'attr': '-1',
  'anime_status': '-1',
  'item': '1',
  'iage': '-1',
  'gender': '-1',
  'language': '-1',
  'child_ip': '-1',
  'characteristic': '-1',
  'three': '-1',
  'ipay': '-1',
  'exclusive': '-1',
  'all': '-1',
  'prefer': '-1',
})

// 分类字典
const cateDictArray: CategoryItem[] = [
  { "key": "100113", "value": "电视剧" },
  { "key": "100173", "value": "电影" },
  { "key": "100105", "value": "纪录片" },
  { "key": "100109", "value": "综艺" },
  { "key": "100119", "value": "动漫" },
  { "key": "100150", "value": "少儿" },
  { "key": "110755", "value": "短剧" }
]
// 分类信息
async function queryCate(type: string) {
  try {
    const data: TencentCategoryInfo[] = await api.get('tencent/category', {
      params: {
        type: type
      }
    })
    const groupedData: Record<string, TencentCategoryInfo[]> = {};
    data.forEach((item: TencentCategoryInfo) => {
      const filter_key = item.filter_key;
      if (!groupedData[filter_key]) {
        groupedData[filter_key] = [];
      }
      groupedData[filter_key].push(item);
    });

    cates.value = groupedData;
  } catch (error) {
    console.log(error)
  }
}

onMounted(() => {
  queryCate(defaultType)
})
// 类型变化
watch(type, () => {
  console.log(type.value)
  filterParams.type = type.value
  queryCate(type.value)
  currentKey.value++
})

// 过滤参数变化
watch(filterParams, () => {
  if (!filterParams.sort) {
    filterParams.sort = '75'
  }
  if (!filterParams.type) {
    filterParams.type = '100113'
  }
  currentKey.value++
})
</script>

<template>
  <div>
    <div class="px-3">
      <div class="flex justify-start align-center">
        <VChipGroup v-model="type" column mandatory>
          <!-- 遍历数组 -->
          <VChip :color="type == item.key ? 'primary' : ''" tile :value="item.key" size="small"
            v-for="item in cateDictArray" :key="item.key">
            {{ item.value }}
          </VChip>
        </VChipGroup>
      </div>
      <div class="flex justify-start align-center" v-for="(item, key) in cates" :key="key">
        <VChipGroup v-model="filterParams[key]" column mandatory>
          <VChip :color="filterParams[option.filter_key] == option.option_value ? 'primary' : ''" tile
            :value="option.option_value" v-for="option in item" :key="option.option_value" size="small">
            {{ option.option_name }}
          </VChip>
        </VChipGroup>
      </div>
    </div>


    <div>
      <MediaCardListView :key="currentKey" :apipath="`tencent/page_data`" :params="filterParams" />
    </div>
  </div>
</template>
