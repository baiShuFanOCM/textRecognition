<template>
  <div class="image-ocr">
    <div class="upload-area" v-if="!uploadedImage">
      <input 
        type="file" 
        accept="image/*" 
        @change="handleImageUpload" 
        ref="fileInput"
        class="file-input"
      >
      <div class="upload-content">
        <svg class="upload-icon" viewBox="0 0 24 24">
          <path d="M21 19V5c0-1.1-.9-2-2-2H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2zM8.5 13.5l2.5 3.01L14.5 12l4.5 6H5l3.5-4.5z" fill="currentColor"/>
        </svg>
        <h3 class="upload-title">拖拽或点击上传图片</h3>
        <p class="upload-hint">支持 jpg、png、gif 等常见格式</p>
        <button class="primary-button" @click="$refs.fileInput.click()">
          <svg class="icon" viewBox="0 0 24 24">
            <path d="M9 16h6v-6h4l-7-7-7 7h4v6zm-4 2h14v2H5v-2z" fill="currentColor"/>
          </svg>
          <span class="text">选择图片</span>
        </button>
      </div>
    </div>

    <div class="content-container" v-if="uploadedImage">
      <div class="image-section">
        <div class="toolbar">
          <button class="primary-button" @click="startSelection">
            <svg class="icon" viewBox="0 0 24 24">
              <path d="M3 3h18v18H3V3zm2 2v14h14V5H5zm2 2h10v10H7V7zm2 2v6h6V9H9z" fill="currentColor"/>
            </svg>
            <span class="text">框选识别</span>
          </button>
          <button class="secondary-button" @click="resetImage">
            <svg class="icon" viewBox="0 0 24 24">
              <path d="M17.65 6.35A7.958 7.958 0 0012 4c-4.42 0-7.99 3.58-7.99 8s3.57 8 7.99 8c3.73 0 6.84-2.55 7.73-6h-2.08A5.99 5.99 0 0112 18c-3.31 0-6-2.69-6-6s2.69-6 6-6c1.66 0 3.14.69 4.22 1.78L13 11h7V4l-2.35 2.35z" fill="currentColor"/>
            </svg>
            <span class="text">重新上传</span>
          </button>
        </div>

        <div class="image-wrapper" ref="imageWrapper">
          <img :src="uploadedImage" ref="targetImage" alt="上传的图片">
          <div v-if="isSelecting" 
               class="selection-overlay"
               @mousedown="startDrag"
               @mousemove="updateDrag"
               @mouseup="endDrag">
            <div class="selection" 
                 :style="selectionStyle"
                 v-if="isDragging">
              <div class="selection-size">
                {{ Math.round(Math.abs(endX - startX)) }} × {{ Math.round(Math.abs(endY - startY)) }}
              </div>
            </div>
          </div>
        </div>
      </div>

      <div class="result-section" v-if="recognizedText">
        <div class="result">
          <div class="result-header">
            <div class="header-left">
              <svg class="icon" viewBox="0 0 24 24">
                <path d="M14 2H6c-1.1 0-2 .9-2 2v16c0 1.1.9 2 2 2h12c1.1 0 2-.9 2-2V8l-6-6zM6 20V4h7v5h5v11H6zm4-9h4v2h-4v-2zm0 3h4v2h-4v-2zm0-6h1v2h-1V8z" fill="currentColor"/>
              </svg>
              <h3>识别结果</h3>
            </div>
            <div class="header-actions">
              <button class="icon-button" @click="copyText" title="复制文本">
                <svg class="icon" viewBox="0 0 24 24">
                  <path d="M9 16h6v-6h4l-7-7-7 7h4v6zm-4 2h14v2H5v-2z" fill="currentColor"/>
                </svg>
              </button>
              <button class="icon-button" @click="clearText" title="清空结果">
                <svg class="icon" viewBox="0 0 24 24">
                  <path d="M6 19c0 1.1.9 2 2 2h8c1.1 0 2-.9 2-2V7H6v12zM8 9h8v10H8V9zm7.5-5l-1-1h-5l-1 1H5v2h14V4h-3.5z" 
                        fill="currentColor"/>
                </svg>
              </button>
            </div>
          </div>
          <pre>{{ recognizedText }}</pre>
        </div>
      </div>
    </div>

    <div v-if="isLoading" class="loading">
      <div class="loading-spinner"></div>
      <span>正在识别中...</span>
    </div>
  </div>
</template>

<script>
import { createWorker } from 'tesseract.js';

export default {
  name: 'ImageOCR',
  data() {
    return {
      uploadedImage: null,
      recognizedText: '',
      isLoading: false,
      worker: null,
      isSelecting: false,
      isDragging: false,
      startX: 0,
      startY: 0,
      endX: 0,
      endY: 0,
    }
  },
  computed: {
    selectionStyle() {
      const left = Math.min(this.startX, this.endX);
      const top = Math.min(this.startY, this.endY);
      const width = Math.abs(this.endX - this.startX);
      const height = Math.abs(this.endY - this.startY);
      
      return {
        left: `${left}px`,
        top: `${top}px`,
        width: `${width}px`,
        height: `${height}px`
      }
    }
  },
  async created() {
    try {
      console.log('开始初始化 Tesseract worker...');
      this.worker = await createWorker({
        logger: progress => {
          console.log('Tesseract 进度:', progress);
        },
        langPath: 'https://raw.githubusercontent.com/naptha/tessdata_best/master',
      });
      
      console.log('加载语言包...');
      await this.worker.loadLanguage('chi_sim');
      
      console.log('初始化语言包...');
      await this.worker.initialize('chi_sim');
      
      console.log('设置识别参数...');
      await this.worker.setParameters({
        tessedit_pageseg_mode: '6',
        preserve_interword_spaces: '0',
        tessedit_char_blacklist: '|[]{}«»""~',
        tessedit_do_invert: '0',
        tessedit_enable_dict_correction: '1',
        segment_nonalphabetic_script: '1',
        language_model_ngram_on: '1',
        language_model_ngram_space_delimited: '0',
      });
      
      console.log('Tesseract 初始化完成');
    } catch (error) {
      console.error('Tesseract 初始化失败:', error);
    }
  },
  beforeDestroy() {
    if (this.worker) {
      this.worker.terminate();
    }
  },
  methods: {
    handleImageUpload(event) {
      const file = event.target.files[0];
      if (file) {
        this.uploadedImage = URL.createObjectURL(file);
      }
    },

    resetImage() {
      this.uploadedImage = null;
      this.recognizedText = '';
      this.isSelecting = false;
    },

    startSelection() {
      this.isSelecting = true;
    },

    startDrag(e) {
      const rect = this.$refs.imageWrapper.getBoundingClientRect();
      const scrollLeft = this.$refs.imageWrapper.scrollLeft;
      const scrollTop = this.$refs.imageWrapper.scrollTop;
      
      this.isDragging = true;
      this.startX = e.clientX - rect.left + scrollLeft;
      this.startY = e.clientY - rect.top + scrollTop;
      this.endX = this.startX;
      this.endY = this.startY;
    },

    updateDrag(e) {
      if (!this.isDragging) return;
      const rect = this.$refs.imageWrapper.getBoundingClientRect();
      const scrollLeft = this.$refs.imageWrapper.scrollLeft;
      const scrollTop = this.$refs.imageWrapper.scrollTop;
      
      this.endX = e.clientX - rect.left + scrollLeft;
      this.endY = e.clientY - rect.top + scrollTop;

      const margin = 50;
      const scrollSpeed = 15;

      if (e.clientY - rect.top < margin) {
        this.$refs.imageWrapper.scrollTop -= scrollSpeed;
      } else if (rect.bottom - e.clientY < margin) {
        this.$refs.imageWrapper.scrollTop += scrollSpeed;
      }

      if (e.clientX - rect.left < margin) {
        this.$refs.imageWrapper.scrollLeft -= scrollSpeed;
      } else if (rect.right - e.clientX < margin) {
        this.$refs.imageWrapper.scrollLeft += scrollSpeed;
      }
    },

    async endDrag(e) {
      if (!this.isDragging) return;
      this.isDragging = false;
      this.isSelecting = false;

      const width = Math.abs(this.endX - this.startX);
      const height = Math.abs(this.endY - this.startY);
      
      if (width < 10 || height < 10) return;

      this.isLoading = true;
      try {
        console.log('开始处理选中区域...');
        const img = this.$refs.targetImage;
        const sx = Math.min(this.startX, this.endX);
        const sy = Math.min(this.startY, this.endY);

        // 单次识别，简化流程
        const result = await this.recognizeWithParams(img, sx, sy, width, height, 1.3, 1.1);
        
        if (result.text) {
          this.recognizedText = result.text.trim();
        } else {
          this.recognizedText = '未能识别出文字，请重试';
        }
      } catch (error) {
        console.error('识别失败:', error);
        this.recognizedText = '识别失败，请重试。错误信息：' + error.message;
      } finally {
        this.isLoading = false;
      }
    },

    async recognizeWithParams(img, sx, sy, width, height) {
      try {
        console.log('开始图片预处理...');
        const canvas = document.createElement('canvas');
        const scale = 2;
        canvas.width = width * scale;
        canvas.height = height * scale;
        const ctx = canvas.getContext('2d');

        ctx.imageSmoothingEnabled = true;
        ctx.imageSmoothingQuality = 'high';

        ctx.drawImage(img, sx, sy, width, height, 0, 0, width * scale, height * scale);

        const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
        const data = imageData.data;

        const processImage = () => {
          return new Promise(resolve => {
            const worker = new Worker(URL.createObjectURL(new Blob([`
              self.onmessage = function(e) {
                const data = e.data;
                for (let i = 0; i < data.length; i += 4) {
                  const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
                  
                  const contrast = 1.5;
                  const factor = (259 * (contrast + 255)) / (255 * (259 - contrast));
                  let value = factor * (avg - 128) + 128;
                  
                  value += 10;
                  
                  value = Math.max(0, Math.min(255, value));
                  
                  data[i] = data[i + 1] = data[i + 2] = value;
                }
                self.postMessage(data);
              }
            `], { type: 'text/javascript' })));

            worker.onmessage = function(e) {
              imageData.data.set(e.data);
              worker.terminate();
              resolve();
            };

            worker.postMessage(data);
          });
        };

        await processImage();
        ctx.putImageData(imageData, 0, 0);

        ctx.filter = 'contrast(1.2) brightness(1.1) saturate(1.2)';
        ctx.drawImage(canvas, 0, 0);

        const blob = await new Promise(resolve => {
          canvas.toBlob(resolve, 'image/png', 1.0);
        });

        if (!blob) {
          throw new Error('Blob 创建失败');
        }

        console.log('开始文字识别...');
        const result = await this.worker.recognize(blob, {
          rectangle: {
            left: 0,
            top: 0,
            width: canvas.width,
            height: canvas.height
          }
        });

        return {
          text: result.data.text,
          confidence: result.data.confidence
        };
      } catch (error) {
        console.error('识别过程出错:', error);
        throw error;
      }
    },

    getOptimalThreshold(data) {
      const histogram = new Array(256).fill(0);
      for (let i = 0; i < data.length; i += 4) {
        const avg = Math.round((data[i] + data[i + 1] + data[i + 2]) / 3);
        histogram[avg]++;
      }

      let sum = 0;
      for (let i = 0; i < 256; i++) {
        sum += i * histogram[i];
      }

      let sumB = 0;
      let wB = 0;
      let wF = 0;
      let maxVariance = 0;
      let threshold = 0;
      const total = data.length / 4;

      for (let i = 0; i < 256; i++) {
        wB += histogram[i];
        if (wB === 0) continue;
        wF = total - wB;
        if (wF === 0) break;

        sumB += i * histogram[i];
        const mB = sumB / wB;
        const mF = (sum - sumB) / wF;
        const variance = wB * wF * Math.pow(mB - mF, 2);

        if (variance > maxVariance) {
          maxVariance = variance;
          threshold = i;
        }
      }

      return threshold;
    },

    async copyText() {
      try {
        await navigator.clipboard.writeText(this.recognizedText);
      } catch (err) {
        console.error('复制失败:', err);
      }
    },

    clearText() {
      this.recognizedText = '';
    }
  }
}
</script>

<style scoped>
.image-ocr {
  padding: 20px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  max-width: 1440px;
  margin: 0 auto;
}

.glass-effect {
  background: #FFFFFF;
}

.upload-area {
  min-height: 400px;
  border: 1px dashed #2196F3;
  border-radius: 8px;
  background: #FAFAFA;
  transition: all 0.2s ease;
  cursor: pointer;
}

.upload-area:hover {
  border-color: #1976D2;
  background: #F5F5F5;
}

.upload-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  padding: 40px;
}

.upload-icon {
  width: 64px;
  height: 64px;
  color: #2196F3;
}

.upload-title {
  font-size: 24px;
  color: #424242;
  margin: 0 0 8px 0;
}

.upload-hint {
  color: #757575;
  margin: 0 0 24px 0;
}

.primary-button, .secondary-button {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 12px 24px;
  border: none;
  border-radius: 4px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.2s ease;
}

.primary-button {
  background-color: #2196F3;
  color: white;
}

.primary-button:hover {
  background-color: #1976D2;
  transform: none;
}

.secondary-button {
  background-color: #E0E0E0;
  color: #424242;
}

.secondary-button:hover {
  background-color: #BDBDBD;
}

.icon-button {
  padding: 8px;
  border: none;
  border-radius: 4px;
  background: transparent;
  cursor: pointer;
  transition: background-color 0.2s ease;
}

.icon-button:hover {
  background-color: #F5F5F5;
}

.content-container {
  display: grid;
  grid-template-columns: minmax(0, 800px) 400px;
  gap: 24px;
  margin-top: 24px;
  justify-content: center;
}

.image-section {
  flex: 1;
  min-width: 0;
  max-width: 800px;
}

.toolbar {
  display: flex;
  gap: 12px;
  padding: 12px;
  border-radius: 4px;
  margin-bottom: 16px;
  background-color: #FAFAFA;
}

.image-wrapper {
  position: relative;
  border-radius: 4px;
  overflow: hidden;
  background: #FAFAFA;
  width: 800px;
  height: 600px;
  margin: 0 auto;
}

.image-wrapper img {
  display: block;
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.selection-overlay {
  position: absolute;
  inset: 0;
  background: rgba(0, 0, 0, 0.3);
  cursor: crosshair;
  width: 100%;
  height: 100%;
}

.selection {
  position: absolute;
  border: 2px solid #2196F3;
  background: rgba(33, 150, 243, 0.1);
}

.selection-size {
  position: absolute;
  top: -24px;
  left: 50%;
  transform: translateX(-50%);
  background: #000000;
  color: white;
  padding: 2px 8px;
  border-radius: 2px;
  font-size: 12px;
}

.result {
  border-radius: 4px;
  overflow: hidden;
  height: calc(100vh - 280px);
  display: flex;
  flex-direction: column;
  background: #FFFFFF;
}

.result-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  background-color: #FAFAFA;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 8px;
}

.header-left h3 {
  margin: 0;
  font-size: 16px;
  color: #424242;
}

.result pre {
  margin: 0;
  padding: 20px;
  font-size: 14px;
  line-height: 1.6;
  color: #424242;
  flex: 1;
  overflow-y: auto;
  background: transparent;
}

.loading {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  padding: 20px 40px;
  border-radius: 4px;
  display: flex;
  align-items: center;
  gap: 12px;
  z-index: 1000;
  background: #FFFFFF;
}

.loading-spinner {
  width: 24px;
  height: 24px;
  border: 3px solid #E3F2FD;
  border-top-color: #2196F3;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

@media (max-width: 1024px) {
  .content-container {
    grid-template-columns: 1fr;
    max-width: 800px;
    margin-left: auto;
    margin-right: auto;
  }
  
  .result-section {
    width: 100%;
  }
  
  .result {
    height: 300px;
  }
}

pre::-webkit-scrollbar {
  width: 8px;
}

pre::-webkit-scrollbar-track {
  background: #F5F5F5;
}

pre::-webkit-scrollbar-thumb {
  background: #BDBDBD;
  border-radius: 0;
}

pre::-webkit-scrollbar-thumb:hover {
  background: #9E9E9E;
}

.icon {
  width: 20px;
  height: 20px;
}

.icon-button .icon {
  width: 18px;
  height: 18px;
  color: #666;
}

.icon-button:hover .icon {
  color: #2196F3;
}

.primary-button .icon,
.secondary-button .icon {
  margin-right: 4px;
}

.primary-button .icon {
  color: white;
}

.secondary-button .icon {
  color: #424242;
}
</style> 