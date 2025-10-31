<script setup lang="ts">
import api from '@/api'
import { ref, reactive, watch, onMounted } from 'vue'
import type { CategoryInfo, CategoryItem } from '@/api/types'
import { default as MediaCardListView } from '@/views/collect/MediaCardListView.vue'
import { default as MediaSearchView } from '@/views/collect/MediaSearchView.vue'
import { VTextField } from 'vuetify/components'

// 排序 类型 资费 出品 地区 年份 状态 画风 年龄 全部 性别 语言  动画明星 剧场 奖项 其他-characteristic
// 电影或者电视剧 movies/tvs
const type = ref('1')
// 当前Key
const currentKey = ref(0)

const cates = ref<Record<string, CategoryInfo[]>>({})
// 搜索词
const searchWord = ref<string | null>(null)
const isSearch = ref(false)


// 过滤参数
const defaultType = '1'
const defaultSort = '2'
const cate = ref('TV')

const filterParams = reactive({
  'type': type.value,
  "season_version": "-1",
  "spoken_language_type": "-1",
  "area": "-1",
  "is_finish": "-1",
  "copyright": "-1",
  "season_status": "-1",
  "season_month": "-1",
  "year": "-1",
  "style_id": "-1",
  "release_date": "-1",
  "producer_id": "-1",
  "order": defaultSort,
})

// 1 番剧 https://www.bilibili.com/anime/index
// 2 电影 https://www.bilibili.com/movie/index
// 3 纪录片 https://www.bilibili.com/documentary/index
// 4 国创 https://www.bilibili.com/guochuang/index
// 5 电视剧 https://www.bilibili.com/tv/index
// 7 综艺 https://www.bilibili.com/variety/index
// 分类字典
const cateDictArray: CategoryItem[] = [
  { "value": "番剧", "key": "1", "cate": "Comic" },
  { "value": "电影", "key": "2", "cate": "Movie" },
  { "value": "纪录片", "key": "3", "cate": "Documentary" },
  { "value": "电视剧", "key": "5", "cate": "TV" },
  { "value": "国创", "key": "4", "cate": "Movie" },
  { "value": "综艺", "key": "7", "cate": "Show" },
]
// 分类信息
async function queryCate(type: string) {
  try {
    const data: CategoryInfo[] = await api.get('bilibili/category', {
      params: {
        type: type
      }
    })
    cateDictArray.forEach(item => {
      if (item.key == type) {
        cate.value = item.cate
      }
    })

    const groupedData: Record<string, CategoryInfo[]> = {};
    data.forEach((item: CategoryInfo) => {
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
function searchMedia() {
  isSearch.value = true
}
function searchClear() {
  searchWord.value = null
  isSearch.value = false
}
onMounted(() => {
  queryCate(defaultType)
})
// 类型变化
watch(type, () => {
  filterParams.type = type.value
  queryCate(type.value)
  currentKey.value++
})

// 过滤参数变化
watch(filterParams, () => {
  if (!filterParams.order) {
    filterParams.order = defaultSort
  }
  if (!filterParams.type) {
    filterParams.type = defaultType
  }
  currentKey.value++
})
</script>

<template>
  <div>
    <div class="px-3 flex justify-start align-center">
      <VCombobox ref="searchWordInput" v-model="searchWord" density="comfortable" variant="outlined"
        class="search-input" prepend-inner-icon="mdi-magnify" append-inner-icon="mdi-close"
        @click:append-inner="searchClear()" placeholder="搜索哔哩哔哩" @keydown.enter="searchMedia()" hide-details />
    </div>
    <div class="px-3" v-show="!isSearch">
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
        <VChipGroup v-model="filterParams[key as keyof typeof filterParams]" column mandatory>
          <VChip :color="filterParams[key as keyof typeof filterParams] == option.option_value ? 'primary' : ''" tile
            :value="option.option_value" v-for="option in item" :key="option.option_value" size="small">
            {{ option.option_name }}
          </VChip>
        </VChipGroup>
      </div>
    </div>


    <div class="pt-3">
      <MediaSearchView v-if="isSearch" :key="currentKey" :apipath="`bilibili/search`" :keyword="searchWord || ''"
        :cate="cate" />
      <MediaCardListView v-show="!isSearch" :key="currentKey" :apipath="`bilibili/page_data`" :params="filterParams"
        :cate="cate" :first-page="1" />
    </div>
  </div>
</template>
