<script lang="ts" setup>
import { cloneDeepWith } from 'lodash-es'
import type { Context } from '@/api/types'
import TorrentCard from '@/components/cards/TorrentCard.vue'

interface SearchTorrent extends Context {
  more?: Array<Context>
}

// 定义输入参数
const props = defineProps({
  // 数据列表
  items: Array as PropType<SearchTorrent[]>,
})

// 过滤表单
const filterForm: Record<string, string[]> = reactive({
  // 站点
  site: [] as string[],
  // 季
  season: [] as string[],
  // 制作组
  releaseGroup: [] as string[],
  // 视频编码
  videoCode: [] as string[],
  // 促销状态
  freeState: [] as string[],
  // 质量
  edition: [] as string[],
  // 分辨率
  resolution: [] as string[],
})

// 过滤项映射（保持中文标题）
const filterTitles: Record<string, string> = {
  site: '站点',
  season: '季集',
  freeState: '促销状态',
  videoCode: '视频编码',
  edition: '质量',
  resolution: '分辨率',
  releaseGroup: '制作组',
}

// 统一存储过滤选项
const filterOptions: Record<string, string[]> = reactive({
  site: [] as string[],
  season: [] as string[],
  freeState: [] as string[],
  edition: [] as string[],
  resolution: [] as string[],
  videoCode: [] as string[],
  releaseGroup: [] as string[],
})

// 非空值的过滤选项
const filterOptionsNotEmpty = computed(() => {
  const options: Record<string, string[]> = {}
  for (const key in filterOptions) {
    if (filterOptions[key].length > 0) options[key] = filterOptions[key]
  }
  return options
})

// 完整的数据列表
let dataList: SearchTorrent[]
// 显示用的数据列表
const displayDataList = ref<Array<SearchTorrent>>([])

// 分组后的数据列表
const groupedDataList = ref<Map<string, Context[]>>()

// 初始化过滤选项
function initOptions(data: Context) {
  const { torrent_info, meta_info } = data
  const optionValue = (options: Array<string>, value: string | undefined) => {
    value && !options.includes(value) && options.push(value)
  }
  optionValue(filterOptions.site, torrent_info?.site_name)
  optionValue(filterOptions.season, meta_info?.season_episode)
  optionValue(filterOptions.releaseGroup, meta_info?.resource_team)
  optionValue(filterOptions.videoCode, meta_info?.video_encode)
  optionValue(filterOptions.freeState, torrent_info?.volume_factor)
  optionValue(filterOptions.edition, meta_info?.edition)
  optionValue(filterOptions.resolution, meta_info?.resource_pix)
}

// 对季过滤选项进行排序
const sortSeasonFilterOptions = computed(() => {
  // 预解析所有选项
  const parsedOptions = filterOptions.season.map((option, index) => {
    const parseSeasonEpisode = (str: string) => {
      const match = str.match(/^S(\d+)(?:-S(\d+))?(?:\s*E(\d+)(?:-E(\d+))?)?$/)

      if (!match) {
        // 如果字符串格式不正确，返回默认值
        return {
          original: str,
          seasonStart: 0,
          seasonEnd: 0,
          episodeStart: 0,
          episodeEnd: 0,
          maxSeason: 0,
          maxEpisode: 0,
          index,
        }
      }

      const seasonStart = match[1] ? parseInt(match[1], 10) : 0
      const seasonEnd = match[2] ? parseInt(match[2], 10) : 0
      const episodeStart = match[3] ? parseInt(match[3], 10) : 0
      const episodeEnd = match[4] ? parseInt(match[4], 10) : 0
      const maxSeason = seasonEnd > 0 ? seasonEnd : seasonStart
      const maxEpisode = episodeEnd > 0 ? episodeEnd : episodeStart

      return {
        original: str,
        seasonStart,
        seasonEnd,
        episodeStart,
        episodeEnd,
        maxSeason,
        maxEpisode,
        index,
      }
    }

    return parseSeasonEpisode(option)
  })

  // 定义判断是否为整季或季范围的函数
  const isWholeSeason = (parsed: (typeof parsedOptions)[0]) =>
    parsed.seasonStart > 0 &&
    (parsed.seasonEnd === 0 || parsed.seasonEnd > parsed.seasonStart) &&
    parsed.episodeStart === 0 &&
    parsed.episodeEnd === 0

  // 定义判断是否包含集数的函数
  const hasEpisodes = (parsed: (typeof parsedOptions)[0]) => parsed.episodeStart > 0 || parsed.episodeEnd > 0

  // 排序逻辑
  parsedOptions.sort((a, b) => {
    const aIsWhole = isWholeSeason(a)
    const bIsWhole = isWholeSeason(b)
    const aHasEpisodes = hasEpisodes(a)
    const bHasEpisodes = hasEpisodes(b)

    // 优先级1：整季和季范围选项优先于带有集数的选项
    if (aIsWhole && !bIsWhole) return -1
    if (!aIsWhole && bIsWhole) return 1

    // 优先级2：如果都是整季或季范围选项，按 maxSeason 降序排列
    if (aIsWhole && bIsWhole) {
      if (b.maxSeason !== a.maxSeason) {
        return b.maxSeason - a.maxSeason
      }
      // 如果 maxSeason 相同，则按原始索引
      return a.index - b.index
    }

    // 优先级3：如果都是带有集数的选项，先按 maxSeason 降序，再按 maxEpisode 降序
    if (aHasEpisodes && bHasEpisodes) {
      if (b.maxSeason !== a.maxSeason) {
        return b.maxSeason - a.maxSeason
      }
      if (b.maxEpisode !== a.maxEpisode) {
        return b.maxEpisode - a.maxEpisode
      }
      // 如果 maxSeason 和 maxEpisode 相同，则按原始索引
      return a.index - b.index
    }

    // 优先级4：如果一个有集数，一个没有，优先有集数的选项
    if (aHasEpisodes && !bHasEpisodes) return -1
    if (!aHasEpisodes && bHasEpisodes) return 1

    // 优先级5：对于没有集数且不是整季的选项，按 seasonStart 和 seasonEnd 降序排序
    if (b.seasonStart !== a.seasonStart) {
      return b.seasonStart - a.seasonStart
    }
    if (b.seasonEnd !== a.seasonEnd) {
      return b.seasonEnd - a.seasonEnd
    }

    // 优先级6：按 episodeStart 和 episodeEnd 降序排序
    if (b.episodeStart !== a.episodeStart) {
      return b.episodeStart - a.episodeStart
    }
    if (b.episodeEnd !== a.episodeEnd) {
      return b.episodeEnd - a.episodeEnd
    }

    // 优先级7：兜底按字母降序排列
    if (a.original !== b.original) {
      return b.original.localeCompare(a.original)
    }

    // 优先级8：如果所有条件都相同，则按原始索引
    return a.index - b.index
  })

  // 返回排序后的原始字符串数组
  return parsedOptions.map(option => option.original)
})

// 计算分组后的列表
onMounted(() => {
  // 数据分组
  const groupMap = new Map<string, Context[]>()
  // 遍历数据
  props.items?.forEach(item => {
    const { torrent_info, meta_info } = item
    // init options
    initOptions(item)
    // group data
    const key = `${meta_info.name}_${meta_info.resource_pix}_${meta_info.edition}_${meta_info.resource_team}_${meta_info.season_episode}_${torrent_info.size}`
    if (groupMap.has(key)) {
      // 已入库相同标题和大小的分组，将当前上下文信息添加到分组中
      const group = groupMap.get(key)
      group?.push(item)
    } else {
      // 创建新的分组，并将当前上下文信息添加到分组中
      groupMap.set(key, [item])
    }
  })
  groupedDataList.value = groupMap
})

// 只监听filterForm和groupedDataList的变化。因为displayDataList的变化不需要清空列表
watch([filterForm, groupedDataList], filterData)
function filterData() {
  // 清空列表
  dataList = []
  displayDataList.value = []
  // 匹配过滤函数，filter中有任一值包含value则返回true
  const match = (filter: Array<string>, value: string | undefined) =>
    filter.length === 0 || (value && filter.includes(value))

  groupedDataList.value?.forEach(value => {
    if (value.length > 0) {
      const matchData = value.filter(data => {
        const { meta_info, torrent_info } = data
        // 季、制作组、视频编码
        return (
          // 站点过滤
          match(filterForm.site, torrent_info.site_name) &&
          // 促销状态过滤
          match(filterForm.freeState, torrent_info.volume_factor) &&
          // 季过滤
          match(filterForm.season, meta_info.season_episode) &&
          // 制作组过滤
          match(filterForm.releaseGroup, meta_info.resource_team) &&
          // 视频编码过滤
          match(filterForm.videoCode, meta_info.video_encode) &&
          // 分辨率过滤
          match(filterForm.resolution, meta_info.resource_pix) &&
          // 质量过滤
          match(filterForm.edition, meta_info.edition)
        )
      })
      if (matchData.length > 0) {
        const firstData = cloneDeepWith(matchData[0]) as SearchTorrent
        if (matchData.length > 1) firstData.more = matchData.slice(1)

        // 显示前20个，4行左右。
        if (displayDataList.value.length < 20) {
          displayDataList.value.push(firstData)
        } else {
          // 后续内容不显示，存在list里。loadMore的时候再加载。
          dataList.push(firstData)
        }
      }
    }
  })
}

function loadMore({ done }: { done: any }) {
  const itemsToMove = dataList.splice(0, 20) // 从 dataList 中获取最前面的 20 个元素
  displayDataList.value.push(...itemsToMove)
  done('ok')
}
</script>

<template>
  <VCard class="bg-transparent mb-3 pt-2 shadow-none">
    <VRow>
      <VCol v-for="(options, key) in filterOptionsNotEmpty" :key="key" cols="6" md="">
        <VSelect
          v-if="key === 'season'"
          v-model="filterForm[key]"
          :items="sortSeasonFilterOptions"
          size="small"
          density="compact"
          chips
          :label="filterTitles[key]"
          multiple
          clearable
        />
        <VSelect
          v-else
          v-model="filterForm[key]"
          :items="options"
          size="small"
          density="compact"
          chips
          :label="filterTitles[key]"
          multiple
          clearable
        />
      </VCol>
    </VRow>
  </VCard>
  <VInfiniteScroll mode="intersect" side="end" :items="displayDataList" class="overflow-hidden" @load="loadMore">
    <template #loading />
    <template #empty />
    <div class="grid gap-3 grid-torrent-card items-start">
      <TorrentCard
        v-for="item in displayDataList"
        :key="`${item.torrent_info.page_url}`"
        :torrent="item"
        :more="item.more"
      />
    </div>
  </VInfiniteScroll>
</template>
