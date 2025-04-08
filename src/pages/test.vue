<template>
    <v-app>
      <v-main class="main-content">
        <!-- 直接使用 v-card 实现毛玻璃效果 -->
        <v-card
          class="glass-card"
          :elevation="12"
          width="600"
          height="400"
        >
          <!-- 卡片背景图 -->
          <v-img
            src="https://picsum.photos/1200/800"
            class="card-bg"
            cover
          ></v-img>
  
          <!-- 卡片内容 -->
          <v-card-item class="card-content">
            <v-card-title class="text-h4 mb-4">
              Vuetify 卡片毛玻璃
            </v-card-title>
            <v-card-text>
              <v-btn
                color="white"
                variant="outlined"
                @click="showAlert"
              >
                <template v-slot:prepend>
                  <v-icon icon="mdi-sparkles"></v-icon>
                </template>
                点击交互
              </v-btn>
            </v-card-text>
          </v-card-item>
        </v-card>
      </v-main>
    </v-app>
  </template>
  
  <script setup>
  const showAlert = () => {
    alert('直接在 v-card 上实现的毛玻璃效果!')
  }
  </script>
  
  <style scoped>
  /* 主内容区域定位 */
  .main-content {
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    background: linear-gradient(45deg, #4a5568, #718096); /* 备用背景 */
  }
  
  /* 卡片整体样式 */
  .glass-card {
    position: relative;
    overflow: hidden;
    border-radius: 16px;
    
    /* 创建伪元素实现玻璃效果 */
    &::before {
      content: '';
      position: absolute;
      inset: 0;
      background: rgba(255, 255, 255, 0.1);
      backdrop-filter: blur(12px);
      -webkit-backdrop-filter: blur(12px);
      border: 1px solid rgba(255, 255, 255, 0.2);
    }
  }
  
  /* 卡片背景图处理 */
  .card-bg {
    position: absolute;
    width: 100%;
    height: 100%;
    filter: blur(8px) brightness(0.8);
    transform: scale(1.05); /* 防止模糊边缘漏出 */
    
    /* 覆盖 Vuetify 图片容器样式 */
    :deep(.v-img__img) {
      object-fit: cover;
    }
  }
  
  /* 卡片内容定位 */
  .card-content {
    position: relative;
    z-index: 2;
    height: 100%;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    color: white;
    text-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  }
  
  /* 按钮样式增强 */
  :deep(.v-btn) {
    backdrop-filter: blur(4px);
    background: rgba(255, 255, 255, 0.1) !important;
    border: 1px solid rgba(255, 255, 255, 0.3) !important;
    
    &:hover {
      background: rgba(255, 255, 255, 0.2) !important;
    }
  }
  
  /* 响应式调整 */
  @media (max-width: 600px) {
    .glass-card {
      width: 90% !important;
      height: 300px;
    }
    
    .card-content {
      padding: 1rem;
    }
  }
  </style>