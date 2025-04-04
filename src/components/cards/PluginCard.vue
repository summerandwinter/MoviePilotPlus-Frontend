<script lang="ts" setup>
import { useToast } from 'vue-toast-notification'
import { useConfirm } from 'vuetify-use-dialog'
import api from '@/api'
import type { Plugin } from '@/api/types'
import { isNullOrEmptyObject } from '@core/utils'
import noImage from '@images/logos/plugin.png'
import { getDominantColor } from '@/@core/utils/image'
import VersionHistory from '@/components/misc/VersionHistory.vue'
import ProgressDialog from '../dialog/ProgressDialog.vue'
import PluginConfigDialog from '../dialog/PluginConfigDialog.vue'
import PluginDataDialog from '../dialog/PluginDataDialog.vue'

// 输入参数
const props = defineProps({
  plugin: Object as PropType<Plugin>,
  count: Number, // 下载次数
  action: Boolean, // 动作标识
  width: String,
  height: String,
})

// 定义触发的自定义事件
const emit = defineEmits(['remove', 'save', 'actionDone'])

// 背景颜色
const backgroundColor = ref('#28A9E1')

// 图片对象
const imageRef = ref<any>()

// 提示框
const $toast = useToast()

// 确认框
const createConfirm = useConfirm()

// 本身是否可见
const isVisible = ref(true)

// 插件配置页面
const pluginConfigDialog = ref(false)

// 菜单显示状态
const menuVisible = ref(false)

// 进度框
const progressDialog = ref(false)

// 插件数据页面
const pluginInfoDialog = ref(false)

// 进度框文本
const progressText = ref('正在更新插件...')

// 用户头像是否加载完成
const isAvatarLoaded = ref(false)

// 图片是否加载完成
const isImageLoaded = ref(false)

// 图片是否加载失败
const imageLoadError = ref(false)

// 更新日志弹窗
const releaseDialog = ref(false)

// 监听动作标识，如为true则打开详情
watch(
  () => props.action,
  (newAction, oldAction) => {
    if (newAction && !oldAction) {
      openPluginDetail()
      emit('actionDone')
    }
  },
)

// 图片加载完成
async function imageLoaded() {
  isImageLoaded.value = true
  const imageElement = imageRef.value?.$el.querySelector('img') as HTMLImageElement
  // 从图片中提取背景色
  backgroundColor.value = await getDominantColor(imageElement)
}

// 显示更新日志
function showUpdateHistory() {
  // 检查当前版本是否有更新日志
  if (isNullOrEmptyObject(props.plugin?.history)) {
    updatePlugin()
  } else {
    releaseDialog.value = true
  }
}

// 调用API卸载插件
async function uninstallPlugin() {
  const isConfirmed = await createConfirm({
    title: '确认',
    content: `是否确认卸载插件 ${props.plugin?.plugin_name} ?`,
  })

  if (!isConfirmed) return

  try {
    // 显示等待提示框
    progressDialog.value = true
    progressText.value = `正在卸载 ${props.plugin?.plugin_name} ...`
    const result: { [key: string]: any } = await api.delete(`plugin/${props.plugin?.id}`)
    // 隐藏等待提示框
    progressDialog.value = false
    if (result.success) {
      $toast.success(`插件 ${props.plugin?.plugin_name} 已卸载`)

      // 通知父组件刷新
      emit('remove')
    } else {
      $toast.error(`插件 ${props.plugin?.plugin_name} 卸载失败：${result.message}}`)
    }
  } catch (error) {
    console.error(error)
  }
}

// 显示插件数据
async function showPluginInfo() {
  pluginConfigDialog.value = false
  pluginInfoDialog.value = true
}

// 显示插件配置
async function showPluginConfig() {
  // 显示对话框
  pluginInfoDialog.value = false
  pluginConfigDialog.value = true
}

// 计算图标路径
const iconPath: Ref<string> = computed(() => {
  if (imageLoadError.value) return noImage
  // 如果是网络图片则使用代理后返回
  if (props.plugin?.plugin_icon?.startsWith('http'))
    return `${import.meta.env.VITE_API_BASE_URL}system/img/1?imgurl=${encodeURIComponent(props.plugin?.plugin_icon)}`

  return `./plugin_icon/${props.plugin?.plugin_icon}`
})

// 插件作者头像路径
const authorPath: Ref<string> = computed(() => {
  // 网络图片则使用代理后返回
  return `${import.meta.env.VITE_API_BASE_URL}system/img/1?imgurl=${encodeURIComponent(
    props.plugin?.author_url + '.png',
  )}`
})

// 重置插件
async function resetPlugin() {
  const isConfirmed = await createConfirm({
    title: '确认',
    content: `此操作将恢复插件 ${props.plugin?.plugin_name} 的默认设置，并清除所有相关数据，确定要继续吗？`,
  })

  if (!isConfirmed) return

  try {
    const result: { [key: string]: any } = await api.get(`plugin/reset/${props.plugin?.id}`)
    if (result.success) {
      $toast.success(`插件 ${props.plugin?.plugin_name} 数据已重置`)
      // 通知父组件刷新
      emit('save')
    } else {
      $toast.error(`插件 ${props.plugin?.plugin_name} 重置失败：${result.message}}`)
    }
  } catch (error) {
    console.error(error)
  }
}

// 更新插件
async function updatePlugin() {
  try {
    releaseDialog.value = false
    // 显示等待提示框
    progressDialog.value = true
    progressText.value = `正在更新 ${props.plugin?.plugin_name} ...`

    const result: { [key: string]: any } = await api.get(`plugin/install/${props.plugin?.id}`, {
      params: {
        repo_url: props.plugin?.repo_url,
        force: true,
      },
    })

    // 隐藏等待提示框
    progressDialog.value = false

    if (result.success) {
      $toast.success(`插件 ${props.plugin?.plugin_name} 更新成功！`)

      // 通知父组件刷新
      emit('save')
    } else {
      $toast.error(`插件 ${props.plugin?.plugin_name} 更新失败：${result.message}`)
    }
  } catch (error) {
    console.error(error)
  }
}

// 访问作者主页
function visitAuthorPage() {
  window.open(props.plugin?.author_url, '_blank')
}

// 查看日志URL
function openLoggerWindow() {
  const url = `${
    import.meta.env.VITE_API_BASE_URL
  }system/logging?length=-1&logfile=plugins/${props.plugin?.id?.toLowerCase()}.log`
  window.open(url, '_blank')
}

// 打开插件详情
function openPluginDetail() {
  if (props.plugin?.has_page) showPluginInfo()
  else showPluginConfig()
}

// 配置完成
function configDone() {
  pluginConfigDialog.value = false
  emit('save')
}

// 弹出菜单
const dropdownItems = ref([
  {
    title: '查看数据',
    value: 1,
    show: props.plugin?.has_page,
    props: {
      prependIcon: 'mdi-information-outline',
      click: showPluginInfo,
    },
  },
  {
    title: '设置',
    value: 2,
    show: true,
    props: {
      prependIcon: 'mdi-cog-outline',
      click: showPluginConfig,
    },
  },
  {
    title: '更新',
    value: 3,
    show: props.plugin?.has_update,
    props: {
      prependIcon: 'mdi-arrow-up-circle-outline',
      color: 'success',
      click: showUpdateHistory,
    },
  },
  {
    title: '重置',
    value: 4,
    show: true,
    props: {
      prependIcon: 'mdi-cancel',
      color: 'warning',
      click: resetPlugin,
    },
  },
  {
    title: '卸载',
    value: 5,
    show: true,
    props: {
      prependIcon: 'mdi-trash-can-outline',
      color: 'error',
      click: uninstallPlugin,
    },
  },
  {
    title: '查看日志',
    value: 6,
    show: true,
    props: {
      prependIcon: 'mdi-file-document-outline',
      click: () => {
        openLoggerWindow()
      },
    },
  },
  {
    title: '作者主页',
    value: 7,
    show: true,
    props: {
      prependIcon: 'mdi-home-circle-outline',
      click: visitAuthorPage,
    },
  },
])

// 监听插件状态变化
watch(
  () => props.plugin?.has_update,
  (newHasUpdate, _) => {
    const updateItemIndex = dropdownItems.value.findIndex(item => item.value === 3)
    if (updateItemIndex !== -1) dropdownItems.value[updateItemIndex].show = newHasUpdate
  },
)

// 监听插件窗口状态变化
watch(
  () => props.plugin?.page_open,
  (newOpenState, _) => {
    if (newOpenState) openPluginDetail()
  },
)
</script>

<template>
  <div>
    <!-- 插件卡片 -->
    <VHover>
      <template #default="hover">
        <VCard
          v-if="isVisible"
          v-bind="hover.props"
          :width="props.width"
          :height="props.height"
          @click="openPluginDetail"
          class="flex flex-col h-full"
        >
          <div
            class="relative flex flex-row items-start pa-3 justify-between grow"
            :style="{ background: `${backgroundColor}` }"
          >
            <div
              class="absolute inset-0 bg-cover bg-center"
              :style="{ background: `${backgroundColor}`, filter: 'brightness(0.5)' }"
            />
            <div class="relative flex-1 min-w-0">
              <VCardTitle class="text-white text-lg px-2 text-shadow whitespace-nowrap overflow-hidden text-ellipsis">
                <VBadge v-if="props.plugin?.state" dot inline color="success" />
                {{ props.plugin?.plugin_name }}
                <span class="text-sm mt-1 text-gray-200"> v{{ props.plugin?.plugin_version }} </span>
              </VCardTitle>
              <VCardText class="px-2 py-0 text-white text-sm text-shadow overflow-hidden line-clamp-3 ...">
                {{ props.plugin?.plugin_desc }}
              </VCardText>
            </div>
            <div class="relative flex-shrink-0 self-center">
              <VAvatar size="64">
                <VImg
                  ref="imageRef"
                  :src="iconPath"
                  aspect-ratio="4/3"
                  cover
                  :class="{ shadow: isImageLoaded }"
                  @load="imageLoaded"
                  @error="imageLoadError = true"
                />
              </VAvatar>
            </div>
          </div>
          <VCardText class="flex flex-none align-self-baseline py-3 w-full align-end">
            <span class="author-info">
              <VImg :src="authorPath" class="author-avatar" @load="isAvatarLoaded = true">
                <VIcon v-if="!isAvatarLoaded" icon="mdi-github" class="me-1" />
              </VImg>
              <a :href="props.plugin?.author_url" target="_blank" @click.stop>
                {{ props.plugin?.plugin_author }}
              </a>
            </span>
            <span v-if="props.count" class="ms-3">
              <VIcon icon="mdi-download" />
              <span class="text-sm ms-1 mt-1">{{ props.count?.toLocaleString() }}</span>
            </span>
            <div class="me-n3 absolute bottom-1 right-3">
              <IconBtn>
                <VIcon icon="mdi-dots-vertical" />
                <VMenu v-model="menuVisible" activator="parent" close-on-content-click>
                  <VList>
                    <VListItem
                      v-for="(item, i) in dropdownItems"
                      v-show="item.show"
                      :key="i"
                      variant="plain"
                      :base-color="item.props.color"
                      @click="item.props.click"
                    >
                      <template #prepend>
                        <VIcon :icon="item.props.prependIcon" />
                      </template>
                      <VListItemTitle v-text="item.title" />
                    </VListItem>
                  </VList>
                </VMenu>
              </IconBtn>
            </div>
          </VCardText>
          <div v-if="hover.isHovering" class="me-n3 absolute top-0 right-5">
            <VIcon class="cursor-move text-white">mdi-drag</VIcon>
          </div>
          <div v-else-if="props.plugin?.has_update" class="me-n3 absolute top-0 right-5">
            <VIcon icon="mdi-new-box" class="text-white" />
          </div>
        </VCard>
      </template>
    </VHover>

    <!-- 插件配置页面 -->
    <PluginConfigDialog
      v-if="pluginConfigDialog"
      v-model="pluginConfigDialog"
      :plugin="props.plugin"
      @save="configDone"
      @close="pluginConfigDialog = false"
      @switch="showPluginInfo"
    />

    <!-- 插件数据页面 -->
    <PluginDataDialog
      v-if="pluginInfoDialog"
      v-model="pluginInfoDialog"
      :plugin="props.plugin"
      @close="pluginInfoDialog = false"
      @switch="showPluginConfig"
    />

    <!-- 进度框 -->
    <ProgressDialog v-if="progressDialog" v-model="progressDialog" :text="progressText" />

    <!-- 更新日志 -->
    <VDialog v-if="releaseDialog" v-model="releaseDialog" width="600" scrollable>
      <VCard :title="`${props.plugin?.plugin_name} 更新说明`">
        <DialogCloseBtn @click="releaseDialog = false" />
        <VDivider />
        <VersionHistory :history="props.plugin?.history" />
        <VDivider />
        <VCardItem>
          <VBtn @click="updatePlugin" block>
            <template #prepend>
              <VIcon icon="mdi-arrow-up-circle-outline" />
            </template>
            更新到最新版本
          </VBtn>
        </VCardItem>
      </VCard>
    </VDialog>
  </div>
</template>

<style lang="scss" scoped>
.card-cover-blurred::before {
  position: absolute;
  /* stylelint-disable-next-line property-no-vendor-prefix */
  -webkit-backdrop-filter: blur(2px);
  backdrop-filter: blur(2px);
  background: rgba(29, 39, 59, 48%);
  content: '';
  inset: 0;
}

.author-info {
  display: flex;
  align-items: center;
}

.author-avatar {
  border-radius: 50%;
  block-size: 24px;
  inline-size: 24px;
  margin-inline-end: 8px;
  object-fit: cover;
}
</style>
