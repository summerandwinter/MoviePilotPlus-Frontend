<script lang="ts" setup>
import { useToast } from 'vue-toastification'
import api from '@/api'

// 提示框
const $toast = useToast()

// 腾讯视频Cookie
const tencentCookie = ref('')

// 查询已设置的腾讯视频Cookie
async function queryTencentCookie() {
  try {
    const result: { [key: string]: any } = await api.get('system/setting/TencentCookie')
    if (result && result.data && result.data.value) tencentCookie.value = result.data.value
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


onMounted(() => {
  queryTencentCookie()
})
</script>

<template>
  <VRow>
    <VCol cols="12">
      <VCard>
        <VCardItem>
          <VCardTitle>腾讯视频Cookie</VCardTitle>
          <VCardSubtitle> 添加腾讯视频Cookie </VCardSubtitle>
        </VCardItem>
        <VCardText>
          <VTextarea v-model="tencentCookie" auto-grow placeholder="腾讯视频Cookie" hint="腾讯视频Cookie" persistent-hint />
        </VCardText>
        <VCardText>
          <VAlert type="info" variant="tonal" title="获取方式：">
            <span v-html="`
              访问https://film.qq.com/<br>
              打开控制台<br>
              复制请求头中完整的Cookie<br>
              复制到下面的输入框中<br>
              `
              " />
          </VAlert>
        </VCardText>
        <VCardText>
          <VForm @submit.prevent="() => { }">
            <div class="d-flex flex-wrap gap-4 mt-4">
              <VBtn type="submit" @click="saveTencentCookie"> 保存 </VBtn>
            </div>
          </VForm>
        </VCardText>
      </VCard>
    </VCol>
  </VRow>
</template>
