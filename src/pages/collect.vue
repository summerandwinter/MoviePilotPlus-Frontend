<script setup lang="ts">
import { CollectTabs } from '@/router/menu'
import router from '@/router'
import { default as TencentView } from '@/views/collect/TencentView.vue'
import { default as YoukuView } from '@/views/collect/YoukuView.vue'
import { default as IQiyiView } from '@/views/collect/IQiyiView.vue'
import { default as ExtraSourceView } from '@/views/collect/ExtraSourceView.vue'
import { DiscoverSource } from '@/api/types'
import api from '@/api'

const route = useRoute()
const activeTab = ref(route.query.tab)

function jumpTab(tab: string) {
  router.push('/subscribe/discover?tab=' + tab)
}

// 额外的数据源
const extraDiscoverSources = ref<DiscoverSource[]>([])

// 加载额外的发现数据源
async function loadExtraDiscoverSources() {
  try {
    extraDiscoverSources.value = await api.get('discover/source')
  } catch (error) {
    console.log(error)
  }
}

onMounted(async () => {
  await loadExtraDiscoverSources()
})

onActivated(async () => {
  loadExtraDiscoverSources()
})
</script>

<template>
  <div>
    <VTabs v-model="activeTab" show-arrows>
      <VTab v-for="item in CollectTabs" :key="item.tab" :value="item.tab" @to="jumpTab(item.tab)">
        <div class="min-w-24">
          {{ item.title }}
        </div>
      </VTab>

      <VTab
        v-for="item in extraDiscoverSources"
        :key="item.mediaid_prefix"
        :value="item.mediaid_prefix"
        @to="jumpTab(item.mediaid_prefix)"
      >
        <div class="min-w-24">
          {{ item.name }}
        </div>
      </VTab>
    </VTabs>

    <VWindow v-model="activeTab" class="mt-5 disable-tab-transition" :touch="false">
      <VWindowItem value="tencent">
        <transition name="fade-slide" appear>
          <div>
            <TencentView />
          </div>
        </transition>
      </VWindowItem>
      <VWindowItem value="iqiyi">
        <transition name="fade-slide" appear>
          <div>
            <IQiyiView />
          </div>
        </transition>
      </VWindowItem>
      <VWindowItem value="youku">
        <transition name="fade-slide" appear>
          <div>
            <YoukuView />
          </div>
        </transition>
      </VWindowItem>
      <VWindowItem v-for="item in extraDiscoverSources" :key="item.mediaid_prefix" :value="item.mediaid_prefix">
        <transition name="fade-slide" appear>
          <div>
            <ExtraSourceView :source="item" />
          </div>
        </transition>
      </VWindowItem>
    </VWindow>
  </div>
</template>
