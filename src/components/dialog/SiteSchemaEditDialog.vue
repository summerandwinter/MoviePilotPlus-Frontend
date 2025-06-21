<script lang="ts" setup>
import { useToast } from 'vue-toast-notification'
import type { SiteSchema, Site } from '@/api/types'
import { doneNProgress, startNProgress } from '@/api/nprogress'
import { numberValidator, requiredValidator } from '@/@validators'
import api from '@/api'
import { useDisplay } from 'vuetify'
import { useI18n } from 'vue-i18n'

// 国际化
const { t } = useI18n()

// 显示器宽度
const display = useDisplay()

// 输入参数
const props = defineProps({
  site: {
    type: Object as () => Site,
    required: true
  }
})
const oper = computed(() => siteForm.value?.id ? 'edit' : 'add')

// 注册事件
const emit = defineEmits(['save', 'remove', 'close'])

// 站点编辑表单数据
const siteForm = ref<SiteSchema>({
  id: 0,
  name: '',
  domain: '',
  download_page: '',
  upload_page: '',
  upload_api: '',
  update_api: '',
  tracker_api: '',
  detail_page: '',
  is_https: true,
  cookie_required: true,
  template: '{}',
})

// 提示框
const $toast = useToast()
// 查询站点信息
async function fetchSiteInfo() {
  try {
    siteForm.value = await api.get(`siteschema/${props.site.domain}`)
    siteForm.value.template = JSON.stringify(siteForm.value.template)
  } catch (error) {
    console.error(error)
  }
}

// 调用API 新增站点
async function addSite() {
  if (!siteForm.value?.template) return
  startNProgress()
  try {
    siteForm.value.template = JSON.parse(siteForm.value.template)
    const result: { [key: string]: string } = await api.post('siteschema/', siteForm.value)
    if (result.success) {
      $toast.success(t('siteshema.messages.addSuccess'))
      emit('save')
    } else {
      $toast.error(`${t('siteshema.messages.addFailed')}：${result.message}`)
    }
  } catch (error) {
    console.error(error)
  }
  doneNProgress()
}

// 调用API更新站点信息
async function updateSiteInfo() {
  startNProgress()
  try {
    siteForm.value.template = JSON.parse(siteForm.value.template)
    const result: { [key: string]: any } = await api.put('siteschema/', siteForm.value)
    if (result.success) {
      $toast.success(`${siteForm.value?.name} ${t('siteshema.messages.updateSuccess')}`)
      emit('save')
    } else {
      $toast.error(`${siteForm.value?.name} ${t('siteshema.messages.updateFailed')}：${result.message}`)
    }
  } catch (error) {
    $toast.error(`${siteForm.value?.name} ${t('siteshema.messages.updateFailed')}！`)
    console.error(error)
  }
  doneNProgress()
}

onMounted(async () => {
  await fetchSiteInfo()

})
</script>

<template>
  <VDialog scrollable :close-on-back="false" eager max-width="45rem" :fullscreen="!display.mdAndUp.value">
    <VCard>
      <VCardItem class="py-2">
        <template #prepend>
          <VIcon :icon="oper == 'add' ? 'mdi-web-plus' : 'mdi-web'" class="me-2" />
        </template>
        <VCardTitle>{{ `${oper === 'add' ? t('siteshema.actions.add') : t('siteshema.actions.edit')}` }}</VCardTitle>
        <VCardSubtitle>{{ siteForm.name }}</VCardSubtitle>
      </VCardItem>
      <VDialogCloseBtn @click="emit('close')" />
      <VDivider />
      <VCardText>
        <VForm @submit.prevent="() => { }">
          <VRow>
            <VCol cols="12" md="6">
              <VTextField v-model="siteForm.name" :label="t('siteshema.fields.name')" :rules="[requiredValidator]"
                :hint="t('siteshema.hints.name')" persistent-hint prepend-inner-icon="mdi-account" />
            </VCol>
            <VCol cols="6" md="3">
              <VSwitch v-model="siteForm.cookie_required" :label="t('siteshema.fields.cookie_required')" />
            </VCol>
            <VCol cols="6" md="3">
              <VSwitch v-model="siteForm.is_https" :label="t('siteshema.fields.is_https')" />
            </VCol>
          </VRow>
          <VRow>
            <VCol cols="12" md="6">
              <VTextField v-model="siteForm.domain" :label="t('siteshema.fields.domain')"
                :hint="t('siteshema.hints.domain')" persistent-hint prepend-inner-icon="mdi-web" />
            </VCol>
            <VCol cols="12" md="6">
              <VTextField v-model="siteForm.upload_api" :label="t('siteshema.fields.upload_api')"
                :hint="t('siteshema.hints.upload_api')" persistent-hint prepend-inner-icon="mdi-upload" />
            </VCol>

          </VRow>
          <VRow>
            <VCol cols="12" md="6">
              <VTextField v-model="siteForm.update_api" :label="t('siteshema.fields.update_api')"
                :hint="t('siteshema.hints.update_api')" persistent-hint prepend-inner-icon="mdi-pencil-box-outline" />
            </VCol>
            <VCol cols="12" md="6">
              <VTextField v-model="siteForm.download_page" :label="t('siteshema.fields.download_page')"
                :hint="t('siteshema.hints.download_page')" persistent-hint prepend-inner-icon="mdi-download" />
            </VCol>

          </VRow>
          <VRow>
            <VCol cols="12" md="6">
              <VTextField v-model="siteForm.detail_page" :label="t('siteshema.fields.detail_page')"
                :hint="t('siteshema.hints.detail_page')" persistent-hint prepend-inner-icon="mdi-information-variant" />
            </VCol>
            <VCol cols="12" md="6">
              <VTextField v-model="siteForm.tracker_api" :label="t('siteshema.fields.tracker_api')"
                :hint="t('siteshema.hints.tracker_api')" persistent-hint prepend-inner-icon="mdi-incognito" />
            </VCol>

          </VRow>
          <VRow>
            <VCol cols="12">
              <VTextarea v-model="siteForm.template" :label="t('siteshema.fields.template')" mdi-format-text
                :hint="t('siteshema.hints.template')" persistent-hint prepend-inner-icon="mdi-format-text" />
            </VCol>

          </VRow>
        </VForm>
      </VCardText>
      <VCardActions class="pt-3">
        <VSpacer />
        <VBtn v-if="oper === 'add'" color="primary" @click="addSite" prepend-icon="mdi-plus" class="px-5">
          {{ t('siteshema.actions.add') }}
        </VBtn>
        <VBtn v-else color="primary" @click="updateSiteInfo" prepend-icon="mdi-content-save" class="px-5">
          {{ t('common.save') }}
        </VBtn>
      </VCardActions>
    </VCard>
  </VDialog>
</template>
