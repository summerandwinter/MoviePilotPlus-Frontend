<script lang="ts" setup>
import type { PropType } from 'vue'
import { useConfirm } from '@/composables/useConfirm'
import router from '@/router'
import type { Collect, SiteSeed, Site } from '@/api/types'
import api from '@/api'
import { seedStatus, collectStatus, tagOptions, categoryOptions } from '@/api/constants'
import { formatFileSize } from '@/@core/utils/formatters'
import { useToast } from 'vue-toastification'
import { useGlobalSettingsStore } from '@/stores'
import SiteSearchDialog from '@/components/dialog/SiteSearchDialog.vue'
import AddSiteSeedDialog from '@/components/dialog/AddSiteSeedDialog.vue'
import SiteSeedInfoDialog from '@/components/dialog/SiteSeedInfoDialog.vue'
import VideoDescInfoDialog from '@/components/dialog/VideoDescInfoDialog.vue'
import CollectOperationDialog from '@/components/dialog/CollectOperationDialog.vue'
const $toast = useToast()
// 确认框
const createConfirm = useConfirm()
// 定义触发的自定义事件
const emit = defineEmits(['remove'])
const showAddSiteSedd = ref(false)
const showSiteSeedInfo = ref(false)
const showDescInfo = ref(false)
const seedInfo = ref<SiteSeed>({} as SiteSeed)
// 从 provide 中获取全局设置
// 全局设置
const globalSettingsStore = useGlobalSettingsStore()
const globalSettings = globalSettingsStore.globalSettings
const showCollectOperation = ref(false)
const operationType = ref('')
// 所有站点
// 资源浏览弹窗
const resourceDialog = ref(false)
// 所有站点
const allSites = ref<Site[]>([])
// 输入参数
const props = defineProps({
  task: Object as PropType<Collect>,
})


// 任务信息
const task = ref(props.task)
const siteSeedList = ref<SiteSeed[]>(props.task?.seeds?.filter(seed => !seed.deleted) || [])
// 计算Poster地址
const getCoverUrl: Ref<string> = computed(() => {
  const url = props.task?.cover ?? ''
  // 使用图片缓存
  if (globalSettings.GLOBAL_IMAGE_CACHE)
    return `${import.meta.env.VITE_API_BASE_URL}system/cache/image?url=${encodeURIComponent(url)}`
  // 如果地址中包含douban则使用中转代理
  if (url.includes('doubanio.com'))
    return `${import.meta.env.VITE_API_BASE_URL}system/img/0?imgurl=${encodeURIComponent(url)}`
  return url
})
// 选中的站点
const selectedSites = ref<number>(0)
// 资源浏览弹窗关闭后的回调
function onSiteResourceDone() {
  resourceDialog.value = false
}
function getSelectedSite() {
  const selected_list = allSites.value.filter(item => selectedSites.value === item.id)
  if (selected_list.length > 0) return selected_list[0]
}
// 查询所有站点
async function querySites() {
  try {
    const data: Site[] = await api.get('site/')
    // 过滤站点，只有启用的站点才显示
    allSites.value = data.filter(item => item.is_active)
    if (allSites.value.length > 0) {
      selectedSites.value = allSites.value[0].id
    }
  } catch (error) {
    console.log(error)
  }
}
// 点击搜索
async function clickSearch() {
  if (allSites.value?.length > 0) return
  querySites()
}
// 开始搜索
function handleSearch() {
  // TODO 显示搜索弹框
  resourceDialog.value = true
}
function goDetail() {
  // 跳转到媒体详情页（新标签页）
  const route = router.resolve({
    path: '/cdetail',
    query: {
      id: props.task?.id
    },
  })
  window.open(route.href, '_blank')
  // router.push({
  //   path: '/cdetail',
  //   query: {
  //     id: props.task?.id
  //   },
  // })
}
// 删除任务成功
function deleteSiteSeedSuccess() {
  showSiteSeedInfo.value = false
  // 重新加载做种列表
  getSiteSeedList()
}
function showSiteSeedInfoDialog(seed: SiteSeed) {
  console.log('SiteSeedInfoDialog')
  seedInfo.value = seed
  showSiteSeedInfo.value = true
}
function addSiteSeedSuccess() {
  showAddSiteSedd.value = false
  // 重新加载做种列表
  getSiteSeedList()
}
function showAddSiteSeddoDialog() {
  console.log('AddSiteSeedDialog')
  showAddSiteSedd.value = true
}
function showDescInfoDialog() {
  showDescInfo.value = true
}
function showCollectOperationDialog(operation: string) {
  console.log('showCollectOperationDialog')
  operationType.value = operation
  showCollectOperation.value = true
}
function getTags() {
  if (!props.task?.tags)
    return []
  const tags = JSON.parse(props.task?.tags)
  if (!tags)
    return []
  // 明确指定 tagList 的类型为 string 数组，避免隐式的 any[] 类型
  let tagList: string[] = []
  tags.forEach((t: string) => {
    const tag = tagOptions[t as keyof typeof tagOptions]
    tagList.push(tag)
  })
  return tagList
}
// 添加做种任务失败
function addSiteSeedError(error: string) {
  showAddSiteSedd.value = false
}
function getSeedStatus(status: string) {
  return seedStatus[status as keyof typeof seedStatus]
}
function showSeedStatus(status: string) {
  return 'TorrentPublished' === status
}
function getCollectStatus(status: string | undefined) {
  return collectStatus[status as keyof typeof collectStatus]
}
function getCategory(status: string | undefined) {
  return categoryOptions[status as keyof typeof categoryOptions]
}
async function getSiteSeedList() {
  try {
    siteSeedList.value = await api.get(`collect/seed/${props.task?.id}`)
  } catch (error) {
    console.error(error)
  }
}
async function deleteCollect(collect_id: number | undefined) {
  try {
    if (!collect_id)
      return
    const isConfirmed = await createConfirm({
      title: '确认',
      content: '确认删除采集任务？',
    })

    if (!isConfirmed) return
    await api.delete(`collect/${collect_id}`, {
      params: {
        'delete_file': true,
        'remove_seed': false
      }
    })
    // 通知父组件刷新
    emit('remove', collect_id)
    $toast.success(`删除成功`)
  } catch (error) {
    console.error(error)
  }
}

// 控制是否展开所有chip
const showAll = ref(false)

// 计算实际可见数量
const visibleCount = computed(() => showAll.value ? siteSeedList.value.length : defaultVisible.value)

// 添加容器ref
const chipContainer = ref<HTMLDivElement | null>(null)
// 单个chip的实际宽度（初始设为100px作为备用值）
const chipActualWidth = ref(100)

// 重新计算可见数量的方法
const recalculateVisible = () => {
  if (!chipContainer.value) return

  // 获取容器实际宽度（减去内边距）
  const containerWidth = chipContainer.value.getBoundingClientRect().width - 24 // .p-3的内边距是12px*2
  // 获取第一个chip的实际宽度（如果有chip的话）
  const firstChip = chipContainer.value.querySelector('.v-chip') as HTMLElement | null
  if (firstChip) {
    chipActualWidth.value = firstChip.getBoundingClientRect().width + 16 // 加上margin-right: 12px和margin-bottom: 4px的总和
  }

  // 计算实际可见数量（至少显示3个）
  defaultVisible.value = Math.max(2, Math.floor(containerWidth / chipActualWidth.value))
}

// 移除重复的computed声明，仅保留ref版本
// 计算默认显示数量（改为ref响应式变量）
const defaultVisible = ref(3)
// 更新监听逻辑
const handleResize = () => {
  recalculateVisible()
}

function getIcon(operation: string) {
  switch (operation) {
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
    case 'auto_update':
      return 'mdi-refresh'
    case 'remake_torrent':
      return 'mdi-transfer'
    default:
      return 'mdi-arrow-down-bold-circle'
  }
}
onMounted(() => {
  window.addEventListener('resize', handleResize)
  // 初始加载时计算一次
  setTimeout(() => recalculateVisible(), 100) // 等待DOM渲染完成
  //getSiteSeedList()
})

onUpdated(() => {
  // 内容变化后重新计算
  recalculateVisible()
})

onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
})
</script>

<template>
  <div class="w-100">

    <VListItem @click.stop="goDetail()"
      class="pa-3 mb-2 rounded torrent-item transition-all duration-300 hover:-translate-y-1 overflow-hidden"
      :class="'border-start border-success border-3 opacity-85'">
      <div
        class="discount-banner text-white px-2 py-1 text-sm font-weight-bold rounded-bl-lg bg-green-600 border-green-600">
        {{ getCollectStatus(task?.status) }}
      </div>
      <template v-slot:prepend>
        <div class="d-none d-sm-flex flex-column align-center pr-3">
          <VImg v-if="getCoverUrl" :src="getCoverUrl" :alt="task?.name" class="rounded mb-1" width="110"
            aspect-ratio="9/16" cover />
        </div>
      </template>
      <VListItemTitle>
        <div class="d-flex flex-row flex-wrap align-center mb-2">
          <span class="text-h6 font-weight-bold me-2">{{ task?.cn_title }}</span>
          <VChip class="chip-season rounded-sm font-weight-bold" variant="elevated" size="small">
            ↓{{ task?.episodes_downloaded }}/{{ task?.episodes_total }}
          </VChip>

        </div>

        <div class="text-subtitle-2 font-weight-bold text-black mb-2 font-title" :title="task?.title || task?.name">
          {{ task?.title || task?.name }}
        </div>

        <div class="text-body-2 text-medium-emphasis mb-2" :title="task?.sub_title || '暂无描述'">
          {{ task?.sub_title || '暂无描述' }}
        </div>
        <div class="d-flex flex-wrap gap-1 mb-2">
          <VChip v-if="task?.type" label class="ml-1" variant="outlined" size="x-small" color="primary">
            {{ task?.type }}
          </VChip>
          <VChip v-if="task?.cate" label class="ml-1" variant="outlined" size="x-small" color="primary">
            {{ getCategory(task?.cate) }}
          </VChip>
          <VChip v-if="task?.resolution" label class="ml-1" variant="outlined" size="x-small" color="primary">
            {{ task?.resolution }}
          </VChip>
          <VChip v-if="task?.file_size" label class="ml-1" variant="outlined" size="x-small" color="primary">
            {{ formatFileSize(task?.file_size) }}
          </VChip>
          <VChip v-for="tag in getTags()" :key="tag" label class="ml-1" variant="outlined" size="x-small"
            color="primary">
            {{ tag }}
          </VChip>
        </div>

      </VListItemTitle>

      <div class="pt-2">
        <div ref="chipContainer">
          <!-- 显示可见范围内的chip -->
          <VChip class="mr-1 mb-1" color="success" variant="outlined" size="small"
            @click.stop="showAddSiteSeddoDialog()">
            添加
          </VChip>
          <template v-for="(item, index) in siteSeedList.slice(0, visibleCount)" :key="index">
            <template v-if="showSeedStatus(item.status)">
              <VBadge color="primary" class="mr-5" :content="getSeedStatus(item.status)" size="x-small">
                <VChip class="mr-1 mb-1" @click.stop="showSiteSeedInfoDialog(item)" size="small">
                  {{ item.site_name }}
                </VChip>
              </VBadge>
            </template>
            <template v-else>
              <VChip class="mr-1 mb-1" @click.stop="showSiteSeedInfoDialog(item)" size="small">
                {{ item.site_name }}
              </VChip>
            </template>
          </template>

          <!-- 调整按钮显示条件：总数量 > 默认显示数（5） -->
          <!-- 修复：使用defaultVisible.value判断 -->
          <template v-if="siteSeedList.length > defaultVisible">
            <VBtn variant="text" color="primary" size="x-small" @click.stop="showAll = !showAll" class="ml-0">
              <template #prepend>
                <VIcon :icon="showAll ? 'mdi-chevron-up' : 'mdi-chevron-down'" />
              </template>
              {{ showAll ? '收起' : '展开' }}
            </VBtn>
          </template>
        </div>
      </div>

      <template #append>
        <div class="me-n3">
          <IconBtn>
            <VIcon icon="mdi-dots-vertical" />
            <VMenu activator="parent" close-on-content-click>
              <VList>
                <VListItem variant="plain" @click="showDescInfoDialog()">
                  <template #prepend>
                    <VIcon icon="mdi-information" />
                  </template>
                  <VListItemTitle>查看详情</VListItemTitle>
                </VListItem>

                <VMenu close-on-content-click max-width="450">
                  <template v-slot:activator="{ props }">
                    <VListItem v-bind="props" variant="plain" @click="clickSearch()">
                      <template #prepend>
                        <VIcon icon="mdi-magnify" />
                      </template>
                      <VListItemTitle>搜索站点</VListItemTitle>
                    </VListItem>


                  </template>
                  <VList>
                    <VListItem>
                      <VChipGroup v-model="selectedSites" column @click.stop>
                        <VChip v-for="site in allSites" :key="site.id"
                          :color="selectedSites === site.id ? 'primary' : ''" filter variant="outlined" :value="site.id"
                          size="small">
                          {{ site.name }}
                        </VChip>
                      </VChipGroup>
                    </VListItem>
                    <VListItem>
                      <VBtn @click="handleSearch" block>搜索</VBtn>
                    </VListItem>
                  </VList>
                </VMenu>


                <VListItem variant="plain" @click="showCollectOperationDialog('start_download_by_collect')">
                  <template #prepend>
                    <VIcon :icon="getIcon('start_download_by_collect')" />
                  </template>
                  <VListItemTitle>下载</VListItemTitle>
                </VListItem>
                <VListItem variant="plain" @click="showCollectOperationDialog('auto_update')">
                  <template #prepend>
                    <VIcon :icon="getIcon('auto_update')" />
                  </template>
                  <VListItemTitle>更新信息</VListItemTitle>
                </VListItem>
                <VListItem variant="plain" @click="showCollectOperationDialog('metadata_by_collect')">
                  <template #prepend>
                    <VIcon :icon="getIcon('metadata_by_collect')" />
                  </template>
                  <VListItemTitle>采集媒体信息</VListItemTitle>
                </VListItem>
                <VListItem variant="plain" @click="showCollectOperationDialog('screenshot_by_collect')">
                  <template #prepend>
                    <VIcon :icon="getIcon('screenshot_by_collect')" />
                  </template>
                  <VListItemTitle>截图</VListItemTitle>
                </VListItem>
                <VListItem variant="plain" @click="showCollectOperationDialog('collect_desc_by_collect')">
                  <template #prepend>
                    <VIcon :icon="getIcon('collect_desc_by_collect')" />
                  </template>
                  <VListItemTitle>采集简介</VListItemTitle>
                </VListItem>
                <VListItem variant="plain" @click="showCollectOperationDialog('collect_move')">
                  <template #prepend>
                    <VIcon :icon="getIcon('collect_move')" />
                  </template>
                  <VListItemTitle>重命名</VListItemTitle>
                </VListItem>
                <VListItem variant="plain" @click="showCollectOperationDialog('torrent_create')">
                  <template #prepend>
                    <VIcon :icon="getIcon('torrent_create')" />
                  </template>
                  <VListItemTitle>制种</VListItemTitle>
                </VListItem>
                <VListItem variant="plain" @click="showCollectOperationDialog('remake_torrent')">
                  <template #prepend>
                    <VIcon :icon="getIcon('remake_torrent')" />
                  </template>
                  <VListItemTitle>转种</VListItemTitle>
                </VListItem>

                <VListItem variant="plain" @click="deleteCollect(task?.id)" class="bg-error-container">
                  <template #prepend>
                    <VIcon icon="mdi-delete" color="error" />
                  </template>
                  <VListItemTitle>删除任务</VListItemTitle>
                </VListItem>
              </VList>
            </VMenu>
          </IconBtn>
        </div>
      </template>
    </VListItem>
  </div>
  <SiteSeedInfoDialog v-if="showSiteSeedInfo" v-model="showSiteSeedInfo" :seed="seedInfo" @close="deleteSiteSeedSuccess"
    @remove="deleteSiteSeedSuccess" />
  <AddSiteSeedDialog v-if="showAddSiteSedd" v-model="showAddSiteSedd" :collect="task" :siteSeedList="siteSeedList"
    @done="addSiteSeedSuccess" @error="addSiteSeedError" @close="showAddSiteSedd = false" />
  <VideoDescInfoDialog v-if="showDescInfo" v-model="showDescInfo" :collect="task" @close="showDescInfo = false" />
  <CollectOperationDialog v-if="showCollectOperation" v-model="showCollectOperation" :collect_id="task?.id"
    :operation="operationType" @close="showCollectOperation = false" />
  <!-- 站点资源弹窗 -->
  <SiteSearchDialog v-if="resourceDialog" v-model="resourceDialog" :site="getSelectedSite()" :keyword="task?.cn_title"
    @close="onSiteResourceDone" />
</template>
<style scoped>
.discount-banner {
  position: absolute;
  z-index: 3;
  inset-block-start: 0;
  inset-inline-end: 0;
}

.torrent-item {
  border: 1px solid transparent;
}

.torrent-item:hover {
  border-color: rgba(var(--v-theme-primary), 0.3);
}

.chip-season {
  background-color: #3f51b5;
  color: white;
}

.chip-edition {
  background-color: #f44336;
  color: white;
}

.chip-resolution {
  background-color: #7b1fa2;
  color: white;
}

.chip-codec {
  background-color: #ff9800;
  color: white;
}

.chip-team {
  background-color: #00897b;
  color: white;
}

.chip-label {
  background-color: #5c6bc0;
  color: white;
}

.chip-hr {
  background-color: #212121;
  color: white;
}

.chip-expire {
  background-color: #7e57c2;
  color: white;
}

/* 优惠标签样式 */
.bg-success {
  background-color: #4caf50;
}

.bg-orange {
  background-color: #ff5722;
}

.bg-purple {
  background-color: #9c27b0;
}

.chip-free {
  background-color: #4caf50;
  color: white;
}

.chip-discount {
  background-color: #ff5722;
  color: white;
}

.chip-bonus {
  background-color: #9c27b0;
  color: white;
}
</style>
