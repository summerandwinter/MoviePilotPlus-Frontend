<script lang="ts" setup>
import VerticalNavSectionTitle from '@/@layouts/components/VerticalNavSectionTitle.vue'
import VerticalNavLayout from '@layouts/components/VerticalNavLayout.vue'
import VerticalNavLink from '@layouts/components/VerticalNavLink.vue'
import Footer from '@/layouts/components/Footer.vue'
import UserNofification from '@/layouts/components/UserNotification.vue'
import SearchBar from '@/layouts/components/SearchBar.vue'
import ShortcutBar from '@/layouts/components/ShortcutBar.vue'
import UserProfile from '@/layouts/components/UserProfile.vue'
import { useUserStore } from '@/stores'
import { getNavMenus } from '@/router/i18n-menu'
import { NavMenu } from '@/@layouts/types'
import { useDisplay } from 'vuetify'
import { useI18n } from 'vue-i18n'

const display = useDisplay()
const appMode = inject('pwaMode')
const { t } = useI18n()

// 用户 Store
const userStore = useUserStore()

// 是否超级用户
let superUser = userStore.superUser

// 开始菜单项
const startMenus = ref<NavMenu[]>([])

// 发现菜单项
const discoveryMenus = ref<NavMenu[]>([])

// 订阅菜单项
const subscribeMenus = ref<NavMenu[]>([])

// 整理菜单项
const organizeMenus = ref<NavMenu[]>([])

// 系统菜单项
const systemMenus = ref<NavMenu[]>([])

// 采集菜单项
const collectMenus = ref<NavMenu[]>([])

// 根据分类获取菜单列表
const getMenuList = (header: string) => {
  // 使用国际化菜单
  const menus = getNavMenus()
  return menus.filter((item: NavMenu) => item.header === header && (superUser || !item.admin))
}

// 返回上一页
function goBack() {
  history.back()
}

onMounted(() => {
  // 获取菜单列表
  startMenus.value = getMenuList(t('menu.start'))
  collectMenus.value = getMenuList(t('menu.collect'))
  discoveryMenus.value = getMenuList(t('menu.discovery'))
  subscribeMenus.value = getMenuList(t('menu.subscribe'))
  organizeMenus.value = getMenuList(t('menu.organize'))
  systemMenus.value = getMenuList(t('menu.system'))
})
</script>

<template>
  <VerticalNavLayout>
    <!-- 👉 Navbar -->
    <template #navbar="{ toggleVerticalOverlayNavActive }">
      <div class="d-flex h-100 align-center mx-1">
        <!-- 👉 Vertical Nav Toggle -->
        <IconBtn v-if="!appMode && display.mdAndDown.value" class="ms-n2" @click="toggleVerticalOverlayNavActive(true)">
          <VIcon icon="mdi-menu" />
        </IconBtn>
        <!-- 👉 Back Button -->
        <IconBtn v-if="appMode" class="ms-n2" @click="goBack">
          <VIcon icon="mdi-arrow-left" size="32" />
        </IconBtn>
        <!-- 👉 Search Bar -->
        <SearchBar />
        <!-- 👉 Spacer -->
        <VSpacer />
        <!-- 👉 Shortcuts -->
        <ShortcutBar v-if="superUser" />
        <!-- 👉 Notification -->
        <UserNofification />
        <!-- 👉 UserProfile -->
        <UserProfile />
      </div>
    </template>

    <template #vertical-nav-content>
      <VerticalNavLink v-for="item in startMenus" :item="item" />
      <!-- 👉 采集 -->
      <VerticalNavSectionTitle
        v-if="collectMenus.length > 0"
        :item="{
          heading: t('menu.collect'),
        }"
      />
      <VerticalNavLink v-for="item in collectMenus" :item="item" />
      <!-- 👉 发现 -->
      <VerticalNavSectionTitle
        v-if="discoveryMenus.length > 0"
        :item="{
          heading: t('menu.discovery'),
        }"
      />
      <VerticalNavLink v-for="item in discoveryMenus" :item="item" />
      <!-- 👉 订阅 -->
      <VerticalNavSectionTitle
        v-if="subscribeMenus.length > 0"
        :item="{
          heading: t('menu.subscribe'),
        }"
      />
      <VerticalNavLink v-for="item in subscribeMenus" :item="item" />
      <!-- 👉 整理 -->
      <VerticalNavSectionTitle
        v-if="organizeMenus.length > 0"
        :item="{
          heading: t('menu.organize'),
        }"
      />
      <VerticalNavLink v-for="item in organizeMenus" :item="item" />
      <!-- 👉 系统 -->
      <VerticalNavSectionTitle
        v-if="systemMenus.length > 0"
        :item="{
          heading: t('menu.system'),
        }"
      />
      <VerticalNavLink v-for="item in systemMenus" :item="item" />
    </template>

    <template #after-vertical-nav-items />
    <!-- 👉 Pages -->
    <slot />
    <!-- 👉 Footer -->
    <template #footer>
      <Footer />
    </template>
  </VerticalNavLayout>
</template>

<style lang="scss" scoped>
.meta-key {
  border: thin solid rgba(var(--v-border-color), var(--v-border-opacity));
  border-radius: 6px;
  block-size: 1.5625rem;
  line-height: 1.3125rem;
  padding-block: 0.125rem;
  padding-inline: 0.25rem;
}
</style>
