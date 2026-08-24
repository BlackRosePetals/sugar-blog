<template>
  <div class="wechat-cover-maker">
    <div class="preview-container" ref="containerRef">
      <div class="preview-wrapper" :style="{ transform: `scale(${previewScale})` }">
        <iframe
          ref="iframeRef"
          src="/wechat-cover-frame.html"
          width="900"
          height="383"
          frameborder="0"
          scrolling="no"
          @load="onIframeLoad"
        ></iframe>
      </div>
    </div>
    
    <div class="settings-container">
      <div class="settings-section">
        <h3 class="section-title">Logo 设置</h3>
        <div class="controls-grid">
          <div class="control-group">
            <label>前缀文本 (Prefix)</label>
            <input type="text" v-model="config.prefixText" @input="updateIframe" />
          </div>
          
          <div class="control-group">
            <label>后缀文本 (Suffix)</label>
            <input type="text" v-model="config.suffixText" @input="updateIframe" />
          </div>
          
          <div class="control-group">
            <label>前缀文字颜色</label>
            <input type="color" v-model="config.prefixColor" @input="updateIframe" />
          </div>
          
          <div class="control-group">
            <label>后缀文字颜色</label>
            <input type="color" v-model="config.suffixColor" @input="updateIframe" />
          </div>
          
          <div class="control-group">
            <label>后缀背景颜色</label>
            <input type="color" v-model="config.suffixBgColor" @input="updateIframe" />
          </div>
          
          <div class="control-group">
            <label>背景颜色</label>
            <input type="color" v-model="config.bgColor" @input="updateIframe" />
          </div>
          
          <div class="control-group">
            <label>字体大小 (px)</label>
            <input type="range" min="30" max="150" v-model="config.fontSize" @input="updateIframe" />
            <span>{{ config.fontSize }}px</span>
          </div>
          
          <div class="control-group">
            <label>圆角大小 (px)</label>
            <input type="range" min="0" max="50" v-model="config.borderRadius" @input="updateIframe" />
            <span>{{ config.borderRadius }}px</span>
          </div>
        </div>
      </div>
      
      <div class="settings-section">
        <div class="section-header">
          <h3 class="section-title">光影设置 (Aura)</h3>
          <div class="toggle-wrapper">
            <label class="toggle-label">开启光影</label>
            <input type="checkbox" v-model="config.auraEnabled" @change="updateIframe" />
          </div>
        </div>
        
        <div class="controls-grid" v-show="config.auraEnabled">
          <div class="control-group full-width">
            <label>配色 (Palette)</label>
            <div class="button-group">
              <button 
                v-for="p in ['opal', 'aurora', 'sunset', 'ocean', 'sakura', 'ember', 'ultraviolet']" 
                :key="p"
                :class="['preset-btn', { active: config.aura.palette === p }]"
                @click="config.aura.palette = p; updateIframe()"
              >{{ p }}</button>
            </div>
          </div>
          
          <div class="control-group full-width">
            <label>预设 (Preset)</label>
            <div class="button-group">
              <button 
                v-for="p in ['default', 'subtle', 'vivid', 'calm', 'thin']" 
                :key="p"
                :class="['preset-btn', { active: config.aura.preset === p }]"
                @click="applyPreset(p)"
              >{{ p }}</button>
            </div>
          </div>
          
          <div class="control-group">
            <label>厚度 (Thickness) - {{ config.aura.band }}px</label>
            <input type="range" min="8" max="150" v-model="config.aura.band" @input="updateIframe" />
          </div>
          
          <div class="control-group">
            <label>圆角 (Corner radius) - {{ config.aura.cornerRadius }}px</label>
            <input type="range" min="0" max="50" v-model="config.aura.cornerRadius" @input="updateIframe" />
          </div>
          
          <div class="control-group">
            <label>内边距 (Inset) - {{ config.aura.inset }}px</label>
            <input type="range" min="0" max="50" v-model="config.aura.inset" @input="updateIframe" />
          </div>
          
          <div class="control-group">
            <label>不透明度 (Opacity) - {{ config.aura.ringAlpha }}</label>
            <input type="range" min="0" max="1" step="0.01" v-model="config.aura.ringAlpha" @input="updateIframe" />
          </div>
          
          <div class="control-group">
            <label>柔和度 (Pastel) - {{ config.aura.pastel }}</label>
            <input type="range" min="0" max="1" step="0.01" v-model="config.aura.pastel" @input="updateIframe" />
          </div>
          
          <div class="control-group">
            <label>空闲速度 (Idle speed) - {{ config.aura.rotateIdleS }} s/turn</label>
            <input type="range" min="1" max="20" step="0.5" v-model="config.aura.rotateIdleS" @input="updateIframe" />
          </div>
        </div>
      </div>
    </div>
    
    <div class="actions">
      <p class="size-hint">注：导出图片尺寸为 900x383，符合公众号头图 2.35:1 的标准规范</p>
      <button class="export-btn" @click="exportImage" :disabled="isExporting">
        {{ isExporting ? '导出中...' : '导出图片 (Export)' }}
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted, watch } from 'vue'

const iframeRef = ref<HTMLIFrameElement | null>(null)
const containerRef = ref<HTMLDivElement | null>(null)
const isExporting = ref(false)
const previewScale = ref(1)

const STORAGE_KEY = 'wechat-cover-maker-config'

let resizeObserver: ResizeObserver | null = null

onMounted(() => {
  if (containerRef.value) {
    resizeObserver = new ResizeObserver((entries) => {
      for (let entry of entries) {
        const { width } = entry.contentRect
        // If container width is less than 900, scale down. Otherwise scale is 1.
        previewScale.value = width < 900 ? width / 900 : 1
      }
    })
    resizeObserver.observe(containerRef.value)
  }

  // 从 localStorage 恢复配置
  try {
    const savedConfig = localStorage.getItem(STORAGE_KEY)
    if (savedConfig) {
      const parsed = JSON.parse(savedConfig)
      Object.keys(parsed).forEach(key => {
        if (key === 'aura' && parsed.aura) {
          Object.assign(config.aura, parsed.aura)
        } else {
          (config as any)[key] = parsed[key]
        }
      })
      updateIframe()
    }
  } catch (e) {
    console.error('Failed to parse saved config', e)
  }

  // 监听配置变化并保存到 localStorage
  watch(config, (newVal) => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(newVal))
  }, { deep: true })
})

onUnmounted(() => {
  if (resizeObserver) {
    resizeObserver.disconnect()
  }
})

const config = reactive({
  prefixText: 'Sugar',
  suffixText: 'Blog',
  prefixColor: '#ffffff',
  suffixColor: '#000000',
  suffixBgColor: '#ff9900',
  bgColor: '#000000',
  fontSize: 80,
  borderRadius: 15,
  auraEnabled: true,
  aura: {
    palette: 'opal',
    preset: 'vivid',
    band: 76,
    cornerRadius: 0,
    inset: 0,
    ringAlpha: 0.9,
    pastel: 0.35,
    rotateIdleS: 8
  }
})

const applyPreset = (presetName: string) => {
  config.aura.preset = presetName;
  
  // Apply preset default values to sliders so UI updates
  if (presetName === 'default') {
    config.aura.band = 76;
    config.aura.cornerRadius = 11;
    config.aura.inset = 3;
    config.aura.ringAlpha = 0.9;
    config.aura.pastel = 0.35;
    config.aura.rotateIdleS = 8;
  } else if (presetName === 'subtle') {
    config.aura.band = 76;
    config.aura.cornerRadius = 11;
    config.aura.inset = 3;
    config.aura.ringAlpha = 0.5;
    config.aura.pastel = 0.7;
    config.aura.rotateIdleS = 8;
  } else if (presetName === 'vivid') {
    config.aura.band = 120;
    config.aura.cornerRadius = 11;
    config.aura.inset = 3;
    config.aura.ringAlpha = 1.0;
    config.aura.pastel = 0;
    config.aura.rotateIdleS = 8;
  } else if (presetName === 'calm') {
    config.aura.band = 76;
    config.aura.cornerRadius = 11;
    config.aura.inset = 3;
    config.aura.ringAlpha = 0.9;
    config.aura.pastel = 0.35;
    config.aura.rotateIdleS = 25;
  } else if (presetName === 'thin') {
    config.aura.band = 24;
    config.aura.cornerRadius = 11;
    config.aura.inset = 3;
    config.aura.ringAlpha = 0.9;
    config.aura.pastel = 0.35;
    config.aura.rotateIdleS = 8;
  }
  
  updateIframe();
}

const onIframeLoad = () => {
  updateIframe()
}

const updateIframe = () => {
  if (iframeRef.value && iframeRef.value.contentWindow) {
    const win = iframeRef.value.contentWindow as any
    if (typeof win.updateConfig === 'function') {
      // Need to map palette names to the actual edge-aura palettes
      // wait, the iframe imports EDGE_AURA_PALETTES, so we can just pass the string
      // and in the iframe we can resolve it, or we can just pass the string and the iframe handles it.
      // edge-aura's setPalette accepts preset name string directly!
      win.updateConfig(config)
    }
  }
}

const exportImage = async () => {
  if (iframeRef.value && iframeRef.value.contentWindow) {
    const win = iframeRef.value.contentWindow as any
    if (typeof win.exportImage === 'function') {
      try {
        isExporting.value = true
        const dataUrl = await win.exportImage()
        const a = document.createElement('a')
        a.href = dataUrl
        a.download = `wechat-cover-${Date.now()}.png`
        a.click()
      } catch (e) {
        console.error('Export failed:', e)
        alert('导出失败，请重试')
      } finally {
        isExporting.value = false
      }
    }
  }
}
</script>

<style scoped>
.wechat-cover-maker {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
  padding: 20px;
  background-color: var(--vp-c-bg);
  border-radius: 8px;
}

.preview-container {
  width: 100%;
  max-width: 900px;
  aspect-ratio: 900 / 383;
  display: flex;
  justify-content: center;
  position: relative;
}

.preview-wrapper {
  position: absolute;
  top: 0;
  left: 0;
  width: 900px;
  height: 383px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  border-radius: 8px;
  overflow: hidden;
  transform-origin: top left;
}

.settings-container {
  display: flex;
  flex-direction: column;
  gap: 20px;
  width: 100%;
  max-width: 900px;
}

.settings-section {
  background-color: var(--vp-c-bg-soft);
  border-radius: 8px;
  padding: 20px;
  border: 1px solid var(--vp-c-divider);
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
}

.section-title {
  font-size: 18px;
  font-weight: 600;
  margin: 0 0 15px 0;
  color: var(--vp-c-text-1);
}

.section-header .section-title {
  margin-bottom: 0;
}

.controls-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 15px;
}

.full-width {
  grid-column: 1 / -1;
}

.button-group {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.preset-btn {
  padding: 6px 12px;
  border: 1px solid var(--vp-c-divider);
  border-radius: 20px;
  background: var(--vp-c-bg);
  color: var(--vp-c-text-2);
  font-size: 13px;
  cursor: pointer;
  transition: all 0.2s;
}

.preset-btn:hover {
  border-color: var(--vp-c-brand);
  color: var(--vp-c-brand);
}

.preset-btn.active {
  background: var(--vp-c-brand);
  border-color: var(--vp-c-brand);
  color: white;
}

.toggle-wrapper {
  display: flex;
  align-items: center;
  gap: 8px;
}

.toggle-label {
  font-size: 14px;
  font-weight: 500;
  color: var(--vp-c-text-1);
}

.control-group {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.control-group label {
  font-size: 14px;
  font-weight: 500;
  color: var(--vp-c-text-1);
}

.control-group input[type="text"],
.control-group select {
  padding: 6px 10px;
  border: 1px solid var(--vp-c-divider);
  border-radius: 4px;
  background-color: var(--vp-c-bg);
  color: var(--vp-c-text-1);
}

.control-group input[type="color"] {
  width: 100%;
  height: 36px;
  padding: 0;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.control-group input[type="range"] {
  width: 100%;
}

.toggle-group {
  flex-direction: row;
  align-items: center;
  gap: 10px;
}

.toggle-group input[type="checkbox"] {
  width: 18px;
  height: 18px;
  cursor: pointer;
}

.actions {
  margin-top: 10px;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.size-hint {
  font-size: 13px;
  color: var(--vp-c-text-2);
  margin-bottom: 12px;
}

.export-btn {
  padding: 10px 24px;
  background-color: var(--vp-c-brand);
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.2s;
}

.export-btn:hover {
  background-color: var(--vp-c-brand-dark);
}

.export-btn:disabled {
  background-color: var(--vp-c-gray-light-2);
  cursor: not-allowed;
}
</style>
