<template>
  <div id="app">
    <div class="title-container">
      <div class="title-content">
        <h1 class="title">
          <i class="iconfont icon-OCR"></i>
          图片文字识别
          <div class="subtitle">OCR Image Recognition</div>
        </h1>
      </div>
    </div>
    <div class="content-wrapper">
      <ImageOCR />
    </div>
  </div>
</template>

<script>
import ImageOCR from './components/ImageOCR.vue'

export default {
  name: 'App',
  components: {
    ImageOCR
  }
}
</script>

<style>
body {
  margin: 0;
  padding: 0;
}

#app {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  margin: 0;
  padding: 0;
  min-height: 100vh;
}

.title-container {
  width: 100vw;
  background: linear-gradient(135deg, #2196F3 0%, #1976D2 100%);
  margin: 0;
  padding: 24px 0;
  position: relative;
  box-shadow: 0 2px 12px rgba(33, 150, 243, 0.2);
  left: 50%;
  right: 50%;
  margin-left: -50vw;
  margin-right: -50vw;
}

.title-content {
  max-width: 1440px;
  margin: 0 auto;
  padding: 0 20px;
}

.title {
  margin: 0;
  color: white;
  font-size: 32px;
  font-weight: 600;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
}

.title .icon {
  font-size: 36px;
}

.subtitle {
  font-size: 14px;
  opacity: 0.8;
  font-weight: normal;
  margin-top: 8px;
  letter-spacing: 1px;
  text-transform: uppercase;
}

.content-wrapper {
  padding: 20px;
  max-width: 1440px;
  margin: 0 auto;
}
</style> 