<script setup lang="ts">
import { debounce } from 'lodash-es'
import { useToast } from 'vue-toast-notification'
import api from '@/api'
import type { TransferHistory } from '@/api/types'
import ReorganizeDialog from '@/components/dialog/ReorganizeDialog.vue'
import TransferQueueDialog from '@/components/dialog/TransferQueueDialog.vue'
import ProgressDialog from '@/components/dialog/ProgressDialog.vue'
import { useRoute } from 'vue-router'
import router from '@/router'
import { useDisplay } from 'vuetify'
import { storageDict } from '@/api/constants'

// APP
const display = useDisplay()
const appMode = inject('pwaMode') && display.mdAndDown.value

// 提示框
const $toast = useToast()

// 路由
const route = useRoute()

// 组合式输入法状态
const isComposing = ref(false)

// 重新整理对话框
const redoDialog = ref(false)

// 整理队列对话框
const transferQueueDialog = ref(false)

// 当前操作记录
const currentHistory = ref<TransferHistory>()

// 重新整理IDS
const redoIds = ref<number[]>([])
const redoTargetStorage = ref<string>()

// 已选中的数据
const selected = ref<TransferHistory[]>([])

// 表头
const headers = [
  {
    title: '标题',
    key: 'title',
    sortable: true,
  },
  {
    title: '路径',
    key: 'src',
    sortable: true,
  },
  {
    title: '转移方式',
    key: 'mode',
    sortable: true,
  },
  {
    title: '时间',
    key: 'date',
    sortable: true,
  },
  {
    title: '状态',
    key: 'status',
    sortable: true,
  },
  {
    title: '',
    key: 'actions',
    sortable: false,
  },
]

// 分组表头
const groupHeaders = [
  {
    title: '季集/类别',
    key: 'title',
    sortable: true,
  },
  {
    title: '路径',
    key: 'src',
    sortable: true,
  },
  {
    title: '转移方式',
    key: 'mode',
    sortable: true,
  },
  {
    title: '时间',
    key: 'date',
    sortable: true,
  },
  {
    title: '状态',
    key: 'status',
    sortable: true,
  },
  {
    title: '',
    key: 'actions',
    sortable: false,
  },
]

const pageRange = [
  { title: '25', value: 25 },
  { title: '50', value: 50 },
  { title: '100', value: 100 },
  { title: '500', value: 500 },
  { title: '1000', value: 1000 },
  { title: 'All', value: -1 },
]

// 数据列表
const dataList = ref<TransferHistory[]>([])

// 搜索
const search = ref(route.query.search as string)

// 搜索提示词列表
const searchHintList = ref<string[]>([])

// 加载状态
const loading = ref(false)

// 总条数
const totalItems = ref(0)

// 是否要分组
const group = ref(false)

// 分组条件
const groupBy = ref<any>([
  {
    key: 'title',
  },
])

// 每页条数
const itemsPerPage = ref<number>(ensureNumber(route.query.itemsPerPage, 50))

// 当前页码
const currentPage = ref<number>(ensureNumber(route.query.currentPage, 1))

// 进度条
const progressDialog = ref(false)

// 进度文本
const progressText = ref('请稍候 ...')

// 进度值
const progressValue = ref(0)

// 是否已刷新
const isRefreshed = ref(false)

// 删除确认对话框
const deleteConfirmDialog = ref(false)

// 确认框标题
const confirmTitle = ref('')

// 转移方式字典
const TransferDict: { [key: string]: string } = {
  copy: '复制',
  move: '移动',
  link: '硬链接',
  softlink: '软链接',
  rclone_copy: 'Rclone复制',
  rclone_move: 'Rclone移动',
}

const tableStyle = computed(() => {
  return appMode
    ? 'height: calc(100vh - 15.5rem - env(safe-area-inset-bottom) - 3.5rem)'
    : 'height: calc(100vh - 14.5rem - env(safe-area-inset-bottom)'
})

// 分页提示
const pageTip = computed(() => {
  const begin = itemsPerPage.value * (currentPage.value - 1) + 1
  const end = itemsPerPage.value * currentPage.value === -1 ? 'ALL' : itemsPerPage.value * currentPage.value
  return {
    begin,
    end,
  }
})

// 分页总数
const totalPage = computed(() => {
  const total = Math.ceil(totalItems.value / itemsPerPage.value)
  return total
})

// 切换页签
watch(
  [() => currentPage.value, () => itemsPerPage.value],
  debounce(async () => {
    reloadPage()
  }, 1000),
)

// 搜索监听
watch(
  [() => search.value, () => isComposing.value],
  debounce(async () => {
    if (!isComposing.value) {
      console.log('search: ' + search.value)
      reloadPage(true)
    }
  }, 1000),
)

// 获取订阅列表数据
async function fetchData(page = currentPage.value, count = itemsPerPage.value) {
  loading.value = true

  try {
    const result: { [key: string]: any } = await api.get('history/transfer', {
      params: {
        page,
        count,
        title: search.value,
      },
    })
    isRefreshed.value = true
    dataList.value = result.data?.list
    totalItems.value = result.data?.total
    searchHintList.value = ['失败', '成功', ...new Set(dataList.value.map(item => item.title || ''))].filter(
      title => title !== '',
    )
  } catch (error) {
    console.error(error)
  }
  loading.value = false
}

// 根据 type 返回不同的图标
function getIcon(type: string) {
  if (type === '电影') return 'mdi-movie'
  else if (type === '电视剧') return 'mdi-television-classic'
  else return 'mdi-help-circle'
}

// 删除历史记录
async function removeHistory(item: TransferHistory) {
  currentHistory.value = item
  confirmTitle.value = `确认删除 ${item.title} ${item.seasons}${item.episodes} ?`
  deleteConfirmDialog.value = true
}

// 调用API删除记录
async function remove(item: TransferHistory, deleteSrc: boolean, deleteDest: boolean) {
  try {
    // 调用删除API
    const result: {
      [key: string]: any
    } = await api.delete(`history/transfer?deletesrc=${deleteSrc}&deletedest=${deleteDest}`, {
      data: item,
    })

    if (!result.success) $toast.error(`删除失败: ${result.message}`)
  } catch (error) {
    console.error(error)
  }
}

// 删除单条记录
async function removeSingle(deleteSrc: boolean, deleteDest: boolean) {
  // 关闭弹窗
  deleteConfirmDialog.value = false
  if (!currentHistory.value) return

  // 删除
  await remove(currentHistory.value, deleteSrc, deleteDest)
  // 刷新
  fetchData()
}

// 批量删除记录
async function removeBatch(deleteSrc: boolean, deleteDest: boolean) {
  // 关闭弹窗
  deleteConfirmDialog.value = false
  // 总条数
  const total = selected.value.length
  if (total === 0) return

  // 已处理条数
  let handled = 0
  // 显示进度条
  progressDialog.value = true
  // 循环调用removeHistory
  for (const item of selected.value) {
    // 开始删除
    progressText.value = `正在删除 ${item.title} ${item.seasons}${item.episodes} ...`
    await remove(item, deleteSrc, deleteDest)
    // 删除完成
    handled++
    progressValue.value = (handled / total) * 100
  }
  // 清空选中项
  selected.value = []
  // 隐藏进度条
  progressDialog.value = false
  // 重新获取数据
  fetchData()
}

// 响应删除操作
async function deleteConfirmHandler(deleteSrc: boolean, deleteDest: boolean) {
  if (currentHistory.value) await removeSingle(deleteSrc, deleteDest)
  else await removeBatch(deleteSrc, deleteDest)
}

// 批量删除历史记录
async function removeHistoryBatch() {
  if (selected.value.length === 0) return

  // 清空当前操作记录
  currentHistory.value = undefined
  confirmTitle.value = `确认删除 ${selected.value.length} 条记录 ?`
  // 打开确认弹窗
  deleteConfirmDialog.value = true
}

// 批量重新整理
async function retransferBatch() {
  if (selected.value.length === 0) return

  // 清空当前操作记录
  currentHistory.value = undefined
  // 重新整理IDS
  redoIds.value = selected.value.map(item => item.id)
  // 打开识别弹窗
  redoDialog.value = true
}

// 整理完成
function transferDone() {
  redoDialog.value = false
  // 清空当前操作记录
  currentHistory.value = undefined
  selected.value = []
  // 刷新
  fetchData()
}

// 弹出菜单
const dropdownItems = ref([
  {
    title: '重新整理',
    value: 1,
    props: {
      prependIcon: 'mdi-redo-variant',
      click: (item: TransferHistory) => {
        redoIds.value = [item.id]
        redoTargetStorage.value = item.dest_storage
        redoDialog.value = true
      },
    },
  },
  {
    title: '删除',
    value: 2,
    props: {
      prependIcon: 'mdi-trash-can-outline',
      color: 'error',
      click: (item: TransferHistory) => {
        removeHistory(item)
      },
    },
  },
])

// 添加url参数
function addUrlQuery(url: string, name: string, value: any) {
  if (!url || !name || !value) return url
  const separator = url.includes('?') ? '&' : '?'
  return url + separator + name + '=' + encodeURIComponent(value)
}

// 重载页面
function reloadPage(resetPage = false) {
  let url = '/history'
  if (search.value) {
    url = addUrlQuery(url, 'search', search.value)
  }
  if (itemsPerPage.value) {
    url = addUrlQuery(url, 'itemsPerPage', itemsPerPage.value)
  }
  if (currentPage.value) {
    url = addUrlQuery(url, 'currentPage', resetPage ? 1 : currentPage.value)
  }
  router.push(url)
}

// 确保值为number类型
function ensureNumber(value: any, defaultValue: number = 0) {
  value = Number(value)
  // 如果不是数字
  if (Number.isNaN(value)) {
    value = defaultValue
  }
  return value
}

// 初始加载数据
onMounted(fetchData)
</script>

<template>
  <VCard>
    <VCardItem>
      <VCardTitle>
        <VRow>
          <VCol cols="8" md="6" class="flex">
            <VCombobox
              key="search_navbar"
              v-model="search"
              :items="searchHintList"
              @compositionstart="isComposing = true"
              @compositionend="isComposing = false"
              class="text-disabled"
              density="compact"
              label="搜索整理记录"
              prepend-inner-icon="mdi-magnify"
              variant="solo-filled"
              single-line
              hide-details
              flat
              rounded
              clearable
            />
          </VCol>
          <VCol cols="4" md="6" class="text-end">
            <VBtn
              color="primary"
              prepend-icon="mdi-tray-full"
              append-icon="mdi-dots-horizontal"
              @click="transferQueueDialog = true"
            >
              <span v-if="display.mdAndUp.value" class="ms-2">整理队列</span>
            </VBtn>
          </VCol>
        </VRow>
      </VCardTitle>
    </VCardItem>
    <!-- 分组模式 -->
    <VDataTableVirtual
      v-if="group"
      v-model="selected"
      :groupBy="groupBy"
      :headers="groupHeaders"
      :items="dataList"
      :loading="loading"
      density="compact"
      return-object
      fixed-header
      show-select
      loading-text="加载中..."
      hover
      :style="tableStyle"
    >
      <template #header.data-table-group>
        <span>标题</span>
      </template>
      <template v-slot:group-header="{ item, columns, toggleGroup, isGroupOpen }">
        <tr>
          <td :colspan="columns.length">
            <VBtn
              :icon="isGroupOpen(item) ? '$expand' : '$next'"
              size="small"
              variant="text"
              @click="toggleGroup(item)"
            />
            {{ item.value }}
          </td>
        </tr>
      </template>
      <template #item.title="{ item }">
        <div class="d-flex align-center">
          <VAvatar>
            <VIcon :icon="getIcon(item.type || '')" />
          </VAvatar>
          <div class="d-flex flex-column ms-1">
            <span v-if="item.type === '电视剧'" class="d-block text-high-emphasis min-w-20">
              {{ item?.seasons }}{{ item?.episodes }}
            </span>
            <small>{{ item?.category }}</small>
          </div>
        </div>
      </template>
      <template #item.src="{ item }">
        <div>
          <span>
            <VChip variant="tonal" size="small" label class="my-1"> {{ storageDict[item?.src_storage || ''] }}</VChip>
            <small>{{ item?.src }}</small>
          </span>
          <span class="text-high-emphasis text-bold"> => </span>
          <br />
          <span v-if="item?.dest">
            <VChip variant="tonal" size="small" label class="my-1"> {{ storageDict[item?.dest_storage || ''] }}</VChip>
            <small>{{ item?.dest }}</small>
          </span>
        </div>
      </template>
      <template #item.mode="{ item }">
        <VChip variant="outlined" color="primary" size="small">
          {{ TransferDict[item?.mode ?? ''] || '未知' }}
        </VChip>
      </template>
      <template #item.status="{ item }">
        <VChip v-if="item?.status" color="success" size="small"> 成功 </VChip>
        <VTooltip v-else :text="item?.errmsg">
          <template #activator="{ props }">
            <VChip v-bind="props" color="error" size="small"> 失败 </VChip>
          </template>
        </VTooltip>
      </template>
      <template #item.date="{ item }">
        <small>{{ item?.date }}</small>
      </template>
      <template #item.actions="{ item }">
        <IconBtn>
          <VIcon icon="mdi-dots-vertical" />
          <VMenu activator="parent" close-on-content-click>
            <VList>
              <VListItem
                v-for="(menu, i) in dropdownItems"
                :key="i"
                variant="plain"
                :base-color="menu.props.color"
                @click="menu.props.click(item)"
              >
                <template #prepend>
                  <VIcon :icon="menu.props.prependIcon" />
                </template>
                <VListItemTitle v-text="menu.title" />
              </VListItem>
            </VList>
          </VMenu>
        </IconBtn>
      </template>
      <template #no-data> 没有数据 </template>
    </VDataTableVirtual>
    <!-- 列表模式 -->
    <VDataTableVirtual
      v-else
      v-model="selected"
      :headers="headers"
      :items="dataList"
      :loading="loading"
      density="compact"
      return-object
      fixed-header
      show-select
      loading-text="加载中..."
      hover
      :style="tableStyle"
    >
      <template #item.title="{ item }">
        <div class="d-flex align-center">
          <VAvatar>
            <VIcon :icon="getIcon(item.type || '')" />
          </VAvatar>
          <div class="d-flex flex-column ms-1">
            <span v-if="item.type === '电视剧'" class="d-block text-high-emphasis min-w-20">
              {{ item?.title }} {{ item?.seasons }}{{ item?.episodes }}
            </span>
            <span v-else class="d-block text-high-emphasis min-w-20">
              {{ item?.title }}
            </span>
            <small>{{ item?.category }}</small>
          </div>
        </div>
      </template>
      <template #item.src="{ item }">
        <div>
          <span>
            <VChip variant="tonal" size="small" label class="my-1"> {{ storageDict[item?.src_storage || ''] }}</VChip>
            <small>{{ item?.src }}</small>
          </span>
          <span class="text-high-emphasis text-bold"> => </span>
          <br />
          <span v-if="item?.dest">
            <VChip variant="tonal" size="small" label class="my-1"> {{ storageDict[item?.dest_storage || ''] }}</VChip>
            <small>{{ item?.dest }}</small>
          </span>
        </div>
      </template>
      <template #item.mode="{ item }">
        <VChip variant="outlined" color="primary" size="small">
          {{ TransferDict[item?.mode ?? ''] || '未知' }}
        </VChip>
      </template>
      <template #item.status="{ item }">
        <VChip v-if="item?.status" color="success" size="small"> 成功 </VChip>
        <VTooltip v-else :text="item?.errmsg">
          <template #activator="{ props }">
            <VChip v-bind="props" color="error" size="small"> 失败 </VChip>
          </template>
        </VTooltip>
      </template>
      <template #item.date="{ item }">
        <small>{{ item?.date }}</small>
      </template>
      <template #item.actions="{ item }">
        <IconBtn>
          <VIcon icon="mdi-dots-vertical" />
          <VMenu activator="parent" close-on-content-click>
            <VList>
              <VListItem
                v-for="(menu, i) in dropdownItems"
                :key="i"
                variant="plain"
                :base-color="menu.props.color"
                @click="menu.props.click(item)"
              >
                <template #prepend>
                  <VIcon :icon="menu.props.prependIcon" />
                </template>
                <VListItemTitle v-text="menu.title" />
              </VListItem>
            </VList>
          </VMenu>
        </IconBtn>
      </template>
      <template #no-data> 没有数据 </template>
    </VDataTableVirtual>
    <!-- 分页 -->
    <VDivider />
    <div class="flex items-center justify-between">
      <div class="w-auto">
        <VSelect v-model="itemsPerPage" :items="pageRange" density="compact" variant="solo" flat />
      </div>
      <div class="w-auto text-sm">{{ pageTip.begin }} - {{ pageTip.end }} / {{ totalItems }}</div>
      <VPagination
        v-model="currentPage"
        show-first-last-page
        :length="totalPage"
        :total-visible="display.mdAndUp.value ? 7 : 0"
        @next="currentPage + 1"
        @prev="currentPage - 1"
      >
      </VPagination>
    </div>
  </VCard>

  <!-- 底部操作按钮 -->
  <div v-if="isRefreshed && selected.length > 0">
    <VFab
      icon="mdi-trash-can-outline"
      color="error"
      location="bottom"
      size="x-large"
      fixed
      app
      appear
      @click="removeHistoryBatch"
      :class="{ 'mb-12': appMode }"
    />
    <VFab
      :class="appMode ? 'mb-28' : 'mb-16'"
      icon="mdi-redo-variant"
      location="bottom"
      size="x-large"
      fixed
      app
      appear
      @click="retransferBatch"
    />
  </div>
  <div v-else-if="isRefreshed">
    <VFab
      :icon="group ? 'mdi-format-list-bulleted' : 'mdi-format-list-group'"
      color="primary"
      location="bottom"
      size="x-large"
      fixed
      app
      appear
      @click="group = !group"
      :class="{ 'mb-12': appMode }"
    />
  </div>
  <!-- 底部弹窗 -->
  <VBottomSheet v-model="deleteConfirmDialog" inset>
    <VCard class="text-center rounded-t">
      <DialogCloseBtn @click="deleteConfirmDialog = false" />
      <VCardTitle class="pe-10">
        {{ confirmTitle }}
      </VCardTitle>
      <div class="d-flex flex-column flex-lg-row justify-center my-3">
        <VBtn color="primary" class="mb-2 mx-2" @click="deleteConfirmHandler(false, false)"> 仅删除整理记录 </VBtn>
        <VBtn color="warning" class="mb-2 mx-2" @click="deleteConfirmHandler(true, false)"> 删除整理记录和源文件 </VBtn>
        <VBtn color="info" class="mb-2 mx-2" @click="deleteConfirmHandler(false, true)">
          删除整理记录和媒体库文件
        </VBtn>
        <VBtn color="error" class="mb-2 mx-2" @click="deleteConfirmHandler(true, true)">
          删除整理记录、源文件和媒体库文件
        </VBtn>
      </div>
    </VCard>
  </VBottomSheet>
  <!-- 进度框 -->
  <ProgressDialog v-if="progressDialog" v-model="progressDialog" :text="progressText" :value="progressValue" />
  <!-- 文件整理弹窗 -->
  <ReorganizeDialog
    v-if="redoDialog"
    v-model="redoDialog"
    :logids="redoIds"
    :target_storage="redoTargetStorage"
    @done="transferDone"
    @close="redoDialog = false"
  />
  <!-- 整理队列进度弹窗 -->
  <TransferQueueDialog v-if="transferQueueDialog" v-model="transferQueueDialog" @close="transferQueueDialog = false" />
</template>

<style lang="scss">
.v-table th {
  white-space: nowrap;
}
</style>
