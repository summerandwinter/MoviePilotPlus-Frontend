<!-- eslint-disable sonarjs/no-duplicate-string -->
<script lang="ts" setup>
import { useToast } from 'vue-toastification'
import { VRow, VSelect } from 'vuetify/lib/components/index.mjs'
import draggable from 'vuedraggable'
import api from '@/api'
import { MediaServerConf, Site } from '@/api/types'
import SiteSchemaCard from '@/components/cards/SiteSchemaCard.vue'
import ProgressDialog from '@/components/dialog/ProgressDialog.vue'
import { useI18n } from 'vue-i18n'
import { mediaServerOptions } from '@/api/constants'


// 国际化
const { t } = useI18n()

// 采集器设置项
const CollectSettings = ref<any>({
  // 基础设置
  Basic: {
    MEDIA_DIR: '',
    DOWNLOAD_DIR: '',
    PTGEN_URL: '',
    SECOND_PTGEN_URL: '',
    DOWNLOADER_SLEEP_TIME: 60,
    DOWNLOADER_THREAD_COUNT: 1,
    DOWNLOADER_SPEED: '10M',
    DOWNLOAD_TASK_MAX_WORKERS: 1,
    RAISE_EXCEPTION: false,
    API_DEBUG: false,
    DOWNLOADER_DELETE_AFTER_DONE: true,
    TV_FILE_FORMAT: '',
    MOVIE_FILE_FORMAT: '',
    TV_TITLE_FORMAT: '',
    MOVIE_TITLE_FORMAT: '',
    TV_FOLDER_FORMAT: '',
    MOVIE_FOLDER_FORMAT: '',
  },
  ImageHosting: {
    ipic: {
      active: true
    },
    smms: {
      apikey: '',
      active: true
    },
    imgbb: {
      apikey: '',
      active: true
    },
    panda: {
      apikey: '',
      active: true
    },
    imgbox: {
      username: '',
      password: '',
      active: true
    }
  }
})

// 是否发送请求的总开关
const isRequest = ref(true)

// 所有站点
const allSites = ref<Site[]>([])
// 选中的媒体服务器
const mediaServers = ref<MediaServerConf[]>([])


// 提示框
const $toast = useToast()

// 进度框
const progressDialog = ref(false)


// 腾讯视频Cookie
const tencentCookie = ref('')
const mgTvTicket = ref('')
const mgAppTicket = ref('')

// 查询已设置的腾讯视频Cookie
async function queryTencentCookie() {
  try {
    const result: { [key: string]: any } = await api.get('system/setting/TencentCookie')
    if (result && result.data && result.data.value) tencentCookie.value = result.data.value
  } catch (error) {
    console.log(error)
  }
}
async function queryTvAppTicket() {
  try {
    const result: { [key: string]: any } = await api.get('system/setting/MgTvTicket')
    if (result && result.data && result.data.value) mgTvTicket.value = result.data.value
  } catch (error) {
    console.log(error)
  }
}
// 查询已设置的腾讯视频Cookie
async function queryMgAppTicket() {
  try {
    const result: { [key: string]: any } = await api.get('system/setting/MgAppTicket')
    if (result && result.data && result.data.value) mgAppTicket.value = result.data.value
  } catch (error) {
    console.log(error)
  }
}
// 重载系统生效配置
async function reloadSystem() {
  try {
    const result: { [key: string]: any } = await api.get('system/reload')
    if (result.success) $toast.success('系统配置已生效')
    else $toast.error('重载系统失败！')
  } catch (error) {
    console.log(error)
  }
}

// 保存用户设置的腾讯视频Cookie
async function saveTencentCookie() {
  try {
    // 用户名密码
    const result: { [key: string]: any } = await api.post(
      'system/setting/TencentCookie',
      tencentCookie.value,
    )

    if (result.success) {
      $toast.success('腾讯视频Cookie保存成功')
      await reloadSystem()
    }
    else $toast.error('腾讯视频Cookie保存失败！')
  } catch (error) {
    console.log(error)
  }
}
async function saveMgTvTicket() {
  try {
    // 用户名密码
    const result: { [key: string]: any } = await api.post(
      'system/setting/MgTvTicket',
      mgTvTicket.value,
    )

    if (result.success) {
      $toast.success('芒果TV 电视端Ticket保存成功')
      await reloadSystem()
    }
    else $toast.error('芒果TV 电视端Ticket保存失败！')
  } catch (error) {
    console.log(error)
  }
}

async function saveMgAppTicket() {
  try {
    // 用户名密码
    const result: { [key: string]: any } = await api.post(
      'system/setting/MgAppTicket',
      mgAppTicket.value,
    )

    if (result.success) {
      $toast.success('芒果TV App端Ticket保存成功')
      await reloadSystem()
    }
    else $toast.error('芒果TV App端Ticket保存失败！')
  } catch (error) {
    console.log(error)
  }
}
// 调用API查询下载器设置
async function loadImageHostingSetting() {
  try {
    const result: { [key: string]: any } = await api.get('system/setting/ImageHostingParams')
    CollectSettings.value.ImageHosting = result.data?.value ?? {}
  } catch (error) {
    console.log(error)
  }
}
async function loadSiteList() {
  try {
    const data: Site[] = await api.get('site/')
    allSites.value = data
  } catch (error) {
    console.log(error)
  }
}
// 调用API保存下载器设置
async function saveImageHostingSetting() {
  try {
    const imageHostingParam = CollectSettings.value.ImageHosting
    console.warn('imageHostingParam', imageHostingParam)
    const result: { [key: string]: any } = await api.post('system/setting/ImageHostingParams', imageHostingParam)
    if (result.success) $toast.success(t('setting.collect.imageHostingSaveSuccess'))
    else $toast.error(t('setting.collect.imageHostingSaveFailed'))
    await loadImageHostingSetting()
  } catch (error) {
    console.log(error)
  }
}

// 处理默认下载器状态
function handleDefaultImageHostings(enabledImageHostings: any[], imageHostings: any[]) {
  const enabledDefaultImageHosting = enabledImageHostings.find(item => item.default)
  if (enabledImageHostings.length > 0 && !enabledDefaultImageHosting) {
    imageHostings = imageHostings.map(item => {
      if (item === enabledImageHostings[0]) {
        $toast.info(t('setting.collect.defaultImageHostingNotice', { name: item.name }))
        return { ...item, default: true }
      }
      // 清除其他下载器的默认下载器状态
      return { ...item, default: false }
    })
  }
  return imageHostings
}

// 调用API查询媒体服务器设置
async function loadMediaServerSetting() {
  try {
    const result: { [key: string]: any } = await api.get('system/setting/MediaServers')
    mediaServers.value = result.data?.value ?? []
  } catch (error) {
    console.log(error)
  }
}

// 调用API保存媒体服务器设置
async function saveMediaServerSetting() {
  try {
    const result: { [key: string]: any } = await api.post('system/setting/MediaServers', mediaServers.value)
    if (result.success) $toast.success(t('setting.collect.mediaServerSaveSuccess'))
    else $toast.error(t('setting.collect.mediaServerSaveFailed'))

    await loadMediaServerSetting()
  } catch (error) {
    console.log(error)
  }
}

// 加载系统设置
async function loadSystemSettings() {
  try {
    const result: { [key: string]: any } = await api.get('system/env')
    if (result.success) {
      // 将API返回的值赋值给SystemSettings
      for (const sectionKey of Object.keys(CollectSettings.value) as Array<keyof typeof CollectSettings.value>) {
        Object.keys(CollectSettings.value[sectionKey]).forEach((key: string) => {
          if (result.data.hasOwnProperty(key)) (CollectSettings.value[sectionKey] as any)[key] = result.data[key]
        })
      }
    }
  } catch (error) {
    console.log(error)
  }
}

// 调用API保存设置
async function saveSystemSetting(value: { [key: string]: any }) {
  try {
    const result: { [key: string]: any } = await api.post('system/env', value)
    if (result.success) {
      return true
    } else {
      $toast.error(result?.message || t('setting.collect.basicSaveFailed'))
      return false
    }
  } catch (error) {
    console.log(error)
  }
  return false
}

// 保存基础设置
async function saveBasicSettings() {
  if (await saveSystemSetting(CollectSettings.value.Basic)) {
    $toast.success(t('setting.collect.basicSaveSuccess'))
  }
}



// 添加媒体服务器
function addMediaServer(mediaserver: string) {
  let name = `服务器${mediaServers.value.length + 1}`
  while (mediaServers.value.some(item => item.name === name)) {
    name = `服务器${parseInt(name.split('服务器')[1]) + 1}`
  }
  mediaServers.value.push({
    name: name,
    type: mediaserver,
    enabled: false,
    config: {},
  })
}

// 删除媒体服务器
function removeMediaServer(ele: MediaServerConf) {
  const index = mediaServers.value.indexOf(ele)
  if (index !== -1) mediaServers.value.splice(index, 1)
}

// 变更媒体服务器
function onMediaServerChange(mediaserver: MediaServerConf, name: string) {
  const index = mediaServers.value.findIndex(item => item.name === name)
  if (index !== -1) mediaServers.value[index] = mediaserver
}


// 加载数据
onMounted(() => {
  queryTencentCookie()
  queryTvAppTicket()
  queryMgAppTicket()
  loadImageHostingSetting()
  loadMediaServerSetting()
  loadSystemSettings()
  loadSiteList()
})

onActivated(async () => {
  isRequest.value = true
})

onDeactivated(() => {
  isRequest.value = false
})
</script>

<template>
  <ProgressDialog v-if="progressDialog" v-model="progressDialog" :text="t('setting.collect.reloading')"
    :indeterminate="true" />

  <VRow>
    <VCol cols="12">
      <VCard>
        <VCardItem>
          <VCardTitle>{{ t('setting.collect.basicSettings') }}</VCardTitle>
          <VCardSubtitle>{{ t('setting.collect.basicSettingsDesc') }}</VCardSubtitle>
        </VCardItem>
        <VCardText>
          <VForm @submit.prevent="() => { }">
            <VRow>
              <VCol cols="12" md="6">
                <VTextField v-model="CollectSettings.Basic.MEDIA_DIR" :label="t('setting.collect.mediaDir')"
                  :hint="t('setting.collect.mediaDirHint')" placeholder="/mnt/media" persistent-hint
                  prepend-inner-icon="mdi-folder-download" />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField v-model="CollectSettings.Basic.DOWNLOAD_DIR" :label="t('setting.collect.downloadDir')"
                  :hint="t('setting.collect.downloadDirHint')" placeholder="/mnt/media" persistent-hint
                  prepend-inner-icon="mdi-folder-download" />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField v-model="CollectSettings.Basic.DOWNLOADER_THREAD_COUNT"
                  :label="t('setting.collect.downloaderThreadCount')"
                  :hint="t('setting.collect.downloaderThreadCountHint')" placeholder="10" persistent-hint
                  prepend-inner-icon="mdi-numeric" />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField v-model="CollectSettings.Basic.DOWNLOADER_SPEED"
                  :label="t('setting.collect.downloaderSpeed')" :hint="t('setting.collect.downloaderSpeedHint')"
                  placeholder="10M" persistent-hint prepend-inner-icon="mdi-speedometer" />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField v-model="CollectSettings.Basic.DOWNLOAD_TASK_MAX_WORKERS"
                  :label="t('setting.collect.downloadTaskMaxWorkers')"
                  :hint="t('setting.collect.downloadTaskMaxWorkersHint')" placeholder="1" persistent-hint
                  prepend-inner-icon="mdi-view-week" />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField v-model="CollectSettings.Basic.DOWNLOADER_SLEEP_TIME"
                  :label="t('setting.collect.downloaderSleepTime')" :hint="t('setting.collect.downloaderSleepTimeHint')"
                  placeholder="1" persistent-hint prepend-inner-icon="mdi-fan" />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField v-model="CollectSettings.Basic.PTGEN_URL" :label="t('setting.collect.ptgenUrl')"
                  :hint="t('setting.collect.ptgenUrlHint')" placeholder="1" persistent-hint
                  prepend-inner-icon="mdi-apple-safari" />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField v-model="CollectSettings.Basic.SECOND_PTGEN_URL"
                  :label="t('setting.collect.secondPtgenUrl')" :hint="t('setting.collect.secondPtgenUrlHint')"
                  placeholder="1" persistent-hint prepend-inner-icon="mdi-google-chrome" />
              </VCol>
              <VCol cols="12" md="6">
                <VSwitch v-model="CollectSettings.Basic.DOWNLOADER_DELETE_AFTER_DONE"
                  :label="t('setting.collect.downloaderDeleteAfterDone')"
                  :hint="t('setting.collect.downloaderDeleteAfterDoneHint')" persistent-hint />
              </VCol>
              <VCol cols="12" md="6">
                <VSwitch v-model="CollectSettings.Basic.RAISE_EXCEPTION" :label="t('setting.collect.raiseException')"
                  :hint="t('setting.collect.raiseExceptionHint')" persistent-hint />
              </VCol>
              <VCol cols="12" md="6">
                <VSwitch v-model="CollectSettings.Basic.API_DEBUG" :label="t('setting.collect.apiDebug')"
                  :hint="t('setting.collect.apiDebugHint')" persistent-hint />
              </VCol>
              <VCol cols="12" md="12">
                <VTextarea v-model="CollectSettings.Basic.TV_FILE_FORMAT" auto-grow
                  :placeholder="t('setting.collect.tvFileFormat')" :hint="t('setting.collect.tvFileFormatHint')"
                  rows="3" persistent-hint />
              </VCol>
              <VCol cols="12" md="12">
                <VTextarea v-model="CollectSettings.Basic.TV_FOLDER_FORMAT" auto-grow
                  :placeholder="t('setting.collect.tvFolderFormat')" :hint="t('setting.collect.tvFolderFormatHint')"
                  rows="3" persistent-hint />
              </VCol>
              <VCol cols="12" md="12">
                <VTextarea v-model="CollectSettings.Basic.TV_TITLE_FORMAT" auto-grow
                  :placeholder="t('setting.collect.tvTitleFormat')" :hint="t('setting.collect.tvTitleFormatHint')"
                  rows="3" persistent-hint />
              </VCol>
              <VCol cols="12" md="12">
                <VTextarea v-model="CollectSettings.Basic.MOVIE_FOLDER_FORMAT" auto-grow
                  :placeholder="t('setting.collect.movieFolderFormat')"
                  :hint="t('setting.collect.movieFolderFormatHint')" rows="3" persistent-hint />
              </VCol>
              <VCol cols="12" md="12">
                <VTextarea v-model="CollectSettings.Basic.MOVIE_FILE_FORMAT" auto-grow
                  :placeholder="t('setting.collect.movieFileFormat')" :hint="t('setting.collect.movieFileFormatHint')"
                  rows="3" persistent-hint />
              </VCol>
              <VCol cols="12" md="12">
                <VTextarea v-model="CollectSettings.Basic.MOVIE_TITLE_FORMAT" auto-grow
                  :placeholder="t('setting.collect.movieTitleFormat')" :hint="t('setting.collect.movieTitleFormatHint')"
                  rows="3" persistent-hint />
              </VCol>
            </VRow>
          </VForm>
        </VCardText>
        <VCardText>
          <VForm @submit.prevent="() => { }">
            <div class="d-flex flex-wrap gap-4 mt-4">
              <VBtn type="submit" @click="saveBasicSettings" prepend-icon="mdi-content-save">
                {{ t('common.save') }}
              </VBtn>
            </div>
          </VForm>
        </VCardText>
      </VCard>
    </VCol>
  </VRow>
  <VRow>
    <VCol cols="12">
      <VCard>
        <VCardItem>
          <VCardTitle>{{ t('setting.collect.imageHosting') }}</VCardTitle>
          <VCardSubtitle>{{ t('setting.collect.imageHostingDesc') }}</VCardSubtitle>
        </VCardItem>
        <VCardText>
          <VForm @submit.prevent="() => { }">
            <VRow>
              <!-- ipic -->
              <VCol cols="12" class="pb-2">
                <VListSubheader class="text-lg font-bold">{{ t('setting.collect.ipic') }}</VListSubheader>
              </VCol>
              <VCol cols="12" md="6">
                <VSwitch v-model="CollectSettings.ImageHosting.ipic.active" :label="t('setting.collect.active')"
                  persistent-hint />
              </VCol>
              <!-- smms -->
              <VCol cols="12" class="pb-2">
                <VListSubheader class="text-lg font-bold">{{ t('setting.collect.smms') }}</VListSubheader>
              </VCol>
              <VCol cols="12" md="6">
                <VTextField v-model="CollectSettings.ImageHosting.smms.apikey" :label="t('setting.collect.apikey')"
                  prepend-inner-icon="mdi-key" />
              </VCol>
              <VCol cols="12" md="6">
                <VSwitch v-model="CollectSettings.ImageHosting.smms.active" :label="t('setting.collect.active')" />
              </VCol>
              <!-- imgbb -->
              <VCol cols="12" class="pb-2">
                <VListSubheader class="text-lg font-bold">{{ t('setting.collect.imgbb') }}</VListSubheader>
              </VCol>
              <VCol cols="12" md="6">
                <VTextField v-model="CollectSettings.ImageHosting.imgbb.apikey" :label="t('setting.collect.apikey')"
                  prepend-inner-icon="mdi-key" />
              </VCol>
              <VCol cols="12" md="6">
                <VSwitch v-model="CollectSettings.ImageHosting.imgbb.active" :label="t('setting.collect.active')" />
              </VCol>

              <!-- panda -->
              <VCol cols="12" class="pb-2">
                <VListSubheader class="text-lg font-bold">{{ t('setting.collect.panda') }}</VListSubheader>
              </VCol>
              <VCol cols="12" md="6">
                <VTextField v-model="CollectSettings.ImageHosting.panda.apikey" :label="t('setting.collect.apikey')"
                  prepend-inner-icon="mdi-key" />
              </VCol>
              <VCol cols="12" md="6">
                <VSwitch v-model="CollectSettings.ImageHosting.panda.active" :label="t('setting.collect.active')" />
              </VCol>

              <!-- imgbox -->
              <VCol cols="12" class="pb-2">
                <VListSubheader class="text-lg font-bold">{{ t('setting.collect.imgbox') }}</VListSubheader>
              </VCol>
              <VCol cols="12" md="6">
                <VTextField v-model="CollectSettings.ImageHosting.imgbox.username"
                  :label="t('setting.collect.username')" prepend-inner-icon="mdi-account" />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField v-model="CollectSettings.ImageHosting.imgbox.password"
                  :label="t('setting.collect.password')" prepend-inner-icon="mdi-account-key" />
              </VCol>
              <VCol cols="12" md="6">
                <VSwitch v-model="CollectSettings.ImageHosting.imgbox.active" :label="t('setting.collect.active')" />
              </VCol>
            </VRow>
          </VForm>
        </VCardText>
        <VCardText>
          <VForm @submit.prevent="() => { }">
            <div class="d-flex flex-wrap gap-4 mt-4">
              <VBtn type="submit" @click="saveImageHostingSetting" prepend-icon="mdi-content-save">
                {{ t('common.save') }}
              </VBtn>

            </div>
          </VForm>
        </VCardText>
      </VCard>
    </VCol>
  </VRow>
  <VRow>
    <VCol cols="12">
      <VCard>
        <VCardItem>
          <VCardTitle>{{ t('setting.collect.siteSchema') }}</VCardTitle>
          <VCardSubtitle>{{ t('setting.collect.siteSchemaDesc') }}</VCardSubtitle>
        </VCardItem>
        <VCardText>
          <draggable v-model="allSites" handle=".cursor-move" item-key="id" tag="div"
            :component-data="{ 'class': 'grid gap-3 grid-app-card' }">
            <template #item="{ element }">
              <SiteSchemaCard :site="element" @close="removeMediaServer(element)" @change="onMediaServerChange" />
            </template>
          </draggable>
        </VCardText>
        <VCardText>
          <VForm @submit.prevent="() => { }">
            <div class="d-flex flex-wrap gap-4 mt-4">
              <VBtn type="submit" @click="saveMediaServerSetting" prepend-icon="mdi-content-save">
                {{ t('common.save') }}
              </VBtn>
              <VBtn color="success" variant="tonal">
                <VIcon icon="mdi-plus" />
                <VMenu activator="parent" close-on-content-click>
                  <VList>
                    <VListItem v-for="item in mediaServerOptions" @click="addMediaServer(item.value)">
                      <VListItemTitle>{{ item.title }}</VListItemTitle>
                    </VListItem>
                    <VListItem @click="addMediaServer('custom')">
                      <VListItemTitle>{{ t('setting.collect.custom') }}</VListItemTitle>
                    </VListItem>
                  </VList>
                </VMenu>
              </VBtn>
            </div>
          </VForm>
        </VCardText>
      </VCard>
    </VCol>
  </VRow>
  <VRow>
    <VCol cols="12">
      <VCard>
        <VCardItem>
          <VCardTitle> {{ t('setting.collect.tencentCookie') }}</VCardTitle>
          <VCardSubtitle>{{ t('setting.collect.tencentCookieHint') }} </VCardSubtitle>
        </VCardItem>
        <VCardText>
          <VTextarea v-model="tencentCookie" auto-grow :placeholder="t('setting.collect.tencentCookie')"
            :hint="t('setting.collect.tencentCookieHint')" rows="3" persistent-hint />
        </VCardText>
        <VCardText>
          <VAlert type="info" variant="tonal" :title="t('setting.collect.tencentCookieTipsTitle')">
            <span v-html="t('setting.collect.tencentCookieTips')" />
          </VAlert>
        </VCardText>
        <VCardText>
          <VForm @submit.prevent="() => { }">
            <div class="d-flex flex-wrap gap-4 mt-4">
              <VBtn type="submit" @click="saveTencentCookie"> {{ t('common.save') }} </VBtn>
            </div>
          </VForm>
        </VCardText>
      </VCard>
    </VCol>
  </VRow>
  <VRow>
    <VCol cols="12">
      <VCard>
        <VCardItem>
          <VCardTitle> {{ t('setting.collect.mgTvTicket') }}</VCardTitle>
          <VCardSubtitle>{{ t('setting.collect.mgTvTicketHint') }} </VCardSubtitle>
        </VCardItem>
        <VCardText>
          <VTextField v-model="mgTvTicket" auto-grow :placeholder="t('setting.collect.mgTvTicket')"
            :hint="t('setting.collect.mgTvTicketHint')" rows="3" persistent-hint />
        </VCardText>
        <VCardText>
          <VForm @submit.prevent="() => { }">
            <div class="d-flex flex-wrap gap-4 mt-4">
              <VBtn type="submit" @click="saveMgTvTicket"> {{ t('common.save') }} </VBtn>
            </div>
          </VForm>
        </VCardText>
      </VCard>
    </VCol>
  </VRow>
  <VRow>
    <VCol cols="12">
      <VCard>
        <VCardItem>
          <VCardTitle> {{ t('setting.collect.mgAppTicket') }}</VCardTitle>
          <VCardSubtitle>{{ t('setting.collect.mgAppTicketHint') }} </VCardSubtitle>
        </VCardItem>
        <VCardText>
          <VTextField v-model="mgAppTicket" auto-grow :placeholder="t('setting.collect.mgAppTicket')"
            :hint="t('setting.collect.mgAppTicketHint')" rows="3" persistent-hint />
        </VCardText>
        <VCardText>
          <VForm @submit.prevent="() => { }">
            <div class="d-flex flex-wrap gap-4 mt-4">
              <VBtn type="submit" @click="saveMgAppTicket"> {{ t('common.save') }} </VBtn>
            </div>
          </VForm>
        </VCardText>
      </VCard>
    </VCol>
  </VRow>
</template>
