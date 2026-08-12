<script setup lang="ts">
import { ref, onMounted, onUnmounted, nextTick } from 'vue'

// リアクティブ変数の定義
const isDarkMode = ref<boolean>(false)
const isLoaded = ref<boolean>(false)
const isScriptLoaded = ref<boolean>(false)
const statusMessage = ref<string>('wasm_exec.js を読み込み中...')
const fileName = ref<string>('')
const wasmType = ref<'canvas' | 'function'>('canvas')
const isFullscreen = ref<boolean>(false)

// 数値計算型WASM用
const num1 = ref<number>(10)
const num2 = ref<number>(25)
const result = ref<number | string | null>(null)
const executionTime = ref<string>('')

interface ExportedFunc {
  name: string
  kind: string
}
const exportedFunctions = ref<ExportedFunc[]>([])
const selectedFunc = ref<string>('')

// WebAssembly インスタンス
let wasmInstance: any = null
let wasmModule: WebAssembly.Module | null = null

// Canvas 参照
const wasmCanvas = ref<HTMLCanvasElement | null>(null)

// public/wasm_exec.js を動的に読み込む関数
const loadWasmExecScript = (): Promise<void> => {
  return new Promise((resolve, reject) => {
    if ((window as any).Go) {
      resolve()
      return
    }
    const script = document.createElement('script')
    script.src = '/wasm_exec.js'
    script.onload = () => resolve()
    script.onerror = () => reject(new Error('wasm_exec.js の読み込みに失敗しました'))
    document.head.appendChild(script)
  })
}

// テーマ切り替え
const toggleTheme = () => {
  isDarkMode.value = !isDarkMode.value
  if (isDarkMode.value) {
    document.documentElement.classList.add('dark')
  } else {
    document.documentElement.classList.remove('dark')
  }
}

// 全画面切り替え
const toggleFullscreen = () => {
  const element = document.getElementById('wasm-container')
  if (!element) return

  if (!document.fullscreenElement) {
    element.requestFullscreen().then(() => {
      isFullscreen.value = true
    }).catch(err => {
      console.error('全画面表示エラー:', err)
    })
  } else {
    document.exitFullscreen().then(() => {
      isFullscreen.value = false
    })
  }
}

// WASM ArrayBuffer からの読み込み処理
const loadWasmBuffer = async (buffer: ArrayBuffer, name: string) => {
  try {
    statusMessage.value = 'WebAssembly モジュールを解析中...'
    
    // Go (wasm_exec.js) のインスタンス化
    const go = (window as any).Go ? new (window as any).Go() : null
    const importObject = go ? go.importObject : {}

    const instantiated = await WebAssembly.instantiate(buffer, importObject)
    wasmInstance = instantiated.instance
    wasmModule = instantiated.module

    // GoのWASMエントリーポイント実行 (Gio UI等)
    if (go && typeof go.run === 'function') {
      go.run(wasmInstance)
      wasmType.value = 'canvas'
    }

    // エクスポート関数の検出
    const exportsInfo = WebAssembly.Module.exports(wasmModule)
    exportedFunctions.value = exportsInfo.filter(exp => exp.kind === 'function')

    if (exportedFunctions.value.length > 0) {
      selectedFunc.value = exportedFunctions.value[0].name
      const hasMainOrRun = exportedFunctions.value.some(f => ['main', 'run', '_main', 'step'].includes(f.name))
      if (hasMainOrRun && !go) {
        wasmType.value = 'canvas'
      }
    }

    fileName.value = name
    isLoaded.value = true
    statusMessage.value = `「${name}」を正常に読み込みました`

    await nextTick()
    if (wasmType.value === 'function') {
      executeWasm()
    } else {
      initCanvasDraw()
    }
  } catch (error) {
    console.error('WASM 読み込みエラー:', error)
    isLoaded.value = false
    statusMessage.value = 'WASM ファイルの解析に失敗しました。'
  }
}

// Canvas 描画の初期化
const initCanvasDraw = () => {
  if (!wasmCanvas.value) return
  const ctx = wasmCanvas.value.getContext('2d')
  if (!ctx) return

  wasmCanvas.value.width = wasmCanvas.value.parentElement?.clientWidth || 600
  wasmCanvas.value.height = 400

  ctx.fillStyle = isDarkMode.value ? '#111827' : '#f3f4f6'
  ctx.fillRect(0, 0, wasmCanvas.value.width, wasmCanvas.value.height)
  
  ctx.fillStyle = isDarkMode.value ? '#ffffff' : '#000000'
  ctx.font = 'bold 20px sans-serif'
  ctx.textAlign = 'center'
  ctx.fillText('WASM GUI', wasmCanvas.value.width / 2, wasmCanvas.value.height / 2)
}

// ファイル選択ハンドラー
const handleFileChange = (event: Event) => {
  const target = event.target as HTMLInputElement
  if (target.files && target.files[0]) {
    const file = target.files[0]
    const reader = new FileReader()
    reader.onload = (e) => {
      if (e.target?.result instanceof ArrayBuffer) {
        loadWasmBuffer(e.target.result, file.name)
      }
    }
    reader.readAsArrayBuffer(file)
  }
}

// ドラッグ＆ドロップ処理
const handleDrop = (event: DragEvent) => {
  event.preventDefault()
  if (event.dataTransfer?.files && event.dataTransfer.files[0]) {
    const file = event.dataTransfer.files[0]
    if (!file.name.endsWith('.wasm')) {
      statusMessage.value = '.wasm 拡張子のファイルを選択してください。'
      return
    }
    const reader = new FileReader()
    reader.onload = (e) => {
      if (e.target?.result instanceof ArrayBuffer) {
        loadWasmBuffer(e.target.result, file.name)
      }
    }
    reader.readAsArrayBuffer(file)
  }
}

const handleDragOver = (event: DragEvent) => {
  event.preventDefault()
}

// 選択された WASM 関数を実行
const executeWasm = () => {
  if (!wasmInstance || !selectedFunc.value) return

  const func = wasmInstance.exports[selectedFunc.value]
  if (typeof func !== 'function') {
    result.value = '実行可能な関数ではありません'
    return
  }

  try {
    const startTime = performance.now()
    const res = func(num1.value, num2.value)
    const endTime = performance.now()
    
    executionTime.value = (endTime - startTime).toFixed(4)
    result.value = res !== undefined ? res : '実行完了 (戻り値なし)'
  } catch (err: any) {
    console.error('実行エラー:', err)
    result.value = `エラー: ${err.message || '実行中にエラーが発生しました'}`
  }
}

const onFullscreenChange = () => {
  isFullscreen.value = !!document.fullscreenElement
}

onMounted(async () => {
  document.addEventListener('fullscreenchange', onFullscreenChange)
  
  // wasm_exec.js の事前読み込み
  try {
    await loadWasmExecScript()
    isScriptLoaded.value = true
    statusMessage.value = 'WASMファイルを選択してください'
  } catch (err) {
    console.error(err)
    statusMessage.value = 'wasm_exec.js の読み込みに失敗しました (publicフォルダを確認してください)'
  }
})

onUnmounted(() => {
  document.removeEventListener('fullscreenchange', onFullscreenChange)
})
</script>

<template>
  <div class="app-shell" :class="{ 'dark-theme': isDarkMode }">
    <header class="md-top-app-bar">
      <div class="app-bar-brand">
        <span class="material-icon"></span>
        <h1>Anotheroid</h1>
      </div>
      
      <div class="app-bar-actions">
        <button class="icon-btn theme-svg-btn" @click="toggleTheme" :title="isDarkMode ? 'ライトモードに切替' : 'ダークモードに切替'">
          <img :src="isDarkMode ? '/2.svg' : '/1.svg'" :alt="isDarkMode ? 'ライトモードに切り替え' : 'ダークモードに切り替え'" class="theme-icon-img" />
        </button>
      </div>
    </header>

    <main class="main-content">
      <section 
        class="md-card upload-card"
        @drop="handleDrop"
        @dragover="handleDragOver"
      >
        <div class="upload-icon">
          <img src="/3.svg" alt="アップロード" class="custom-icon-img" />
        </div>
        <h2>WASM モジュールをロード</h2>
        <p>Golang Gio UIのwasmファイルに対応</p>

        <div class="button-row">
          <label class="md-button md-button-filled" :class="{ disabled: !isScriptLoaded }">
            <img src="/3.svg" alt="追加" class="btn-icon-img" />
            ファイルを選択
            <input 
              type="file" 
              accept=".wasm" 
              @change="handleFileChange" 
              class="hidden-input"
              :disabled="!isScriptLoaded"
            />
          </label>
        </div>
      </section>

      <section class="md-card display-card">
        <div class="card-header">
          <div class="status-indicator" :class="{ active: isLoaded }">
            <span class="pulse-dot"></span>
            <span>{{ statusMessage }}</span>
          </div>

          <div class="control-actions" v-if="isLoaded">
            <div class="segmented-button">
              <button 
                :class="{ active: wasmType === 'canvas' }" 
                @click="wasmType = 'canvas'; nextTick(initCanvasDraw)"
              >
                GUI / Canvas
              </button>
              <button 
                :class="{ active: wasmType === 'function' }" 
                @click="wasmType = 'function'"
              >
                関数実行
              </button>
            </div>

            <button 
              v-if="wasmType === 'canvas'"
              class="icon-btn" 
              @click="toggleFullscreen" 
              title="全画面表示"
            >
              <img src="/4.svg" alt="全画面表示" class="fullscreen-icon-img" />
            </button>
          </div>
        </div>

        <!-- 1. GUI / Canvas モード -->
        <div 
          v-show="isLoaded && wasmType === 'canvas'" 
          id="wasm-container" 
          class="canvas-container"
          :class="{ 'is-fullscreen': isFullscreen }"
        >
          <canvas id="wasm-canvas" ref="wasmCanvas"></canvas>
          
          <button 
            v-if="isFullscreen" 
            class="exit-fullscreen-btn md-button md-button-filled"
            @click="toggleFullscreen"
          >
            <img src="/4.svg" alt="全画面表示解除" class="btn-icon-img" />
            全画面表示を解除
          </button>
        </div>

        <!-- 2. 関数呼び出しモード -->
        <div v-show="isLoaded && wasmType === 'function'" class="function-container">
          <div class="md-text-field" v-if="exportedFunctions.length > 0">
            <label>実行関数:</label>
            <select v-model="selectedFunc" @change="executeWasm" class="md-select">
              <option v-for="fn in exportedFunctions" :key="fn.name" :value="fn.name">
                {{ fn.name }} ()
              </option>
            </select>
          </div>

          <div class="inputs-grid">
            <div class="input-item">
              <label>引数 1 (a)</label>
              <input v-model.number="num1" type="number" @input="executeWasm" class="md-input" />
            </div>
            <div class="input-item">
              <label>引数 2 (b)</label>
              <input v-model.number="num2" type="number" @input="executeWasm" class="md-input" />
            </div>
          </div>

          <div class="result-card">
            <span class="result-title">出力結果</span>
            <div class="result-value">{{ result }}</div>
            <span v-if="executionTime" class="execution-time">実行時間: {{ executionTime }} ms</span>
          </div>
        </div>
      </section>
    </main>
  </div>
</template>

<style>
/* Material Icons の読み込み */
@import url('https://fonts.googleapis.com/icon?family=Material+Icons');

/* グローバルカラー定義 */
:root {
  --md-sys-color-bg: #f8fafc;
  --md-sys-color-surface: #ffffff;
  --md-sys-color-primary: #2563eb;
  --md-sys-color-primary-hover: #1d4ed8;
  --md-sys-color-on-primary: #ffffff;
  --md-sys-color-text: #000000;
  --md-sys-color-text-secondary: #000000;
  --md-sys-color-border: #e2e8f0;
  --md-sys-color-card-bg: #ffffff;
  --md-sys-color-container: #eff6ff;
  --md-sys-color-on-container: #000000;
}

.dark {
  --md-sys-color-bg: #0f172a;
  --md-sys-color-surface: #1e293b;
  --md-sys-color-primary: #3b82f6;
  --md-sys-color-primary-hover: #60a5fa;
  --md-sys-color-on-primary: #ffffff;
  --md-sys-color-text: #ffffff;
  --md-sys-color-text-secondary: #ffffff;
  --md-sys-color-border: #334155;
  --md-sys-color-card-bg: #1e293b;
  --md-sys-color-container: #1e3a8a;
  --md-sys-color-on-container: #ffffff;
}

body {
  margin: 0;
  padding: 0;
  background-color: var(--md-sys-color-bg);
  color: var(--md-sys-color-text);
  font-family: 'Roboto', system-ui, -apple-system, sans-serif;
  transition: background-color 0.3s, color 0.3s;
}

.app-shell {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* アプリバー */
.md-top-app-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 24px;
  background-color: var(--md-sys-color-surface);
  border-bottom: 1px solid var(--md-sys-color-border);
}

.app-bar-brand {
  display: flex;
  align-items: center;
  gap: 12px;
  color: var(--md-sys-color-primary);
}

.app-bar-brand h1 {
  font-size: 1.25rem;
  margin: 0;
  font-weight: 600;
  color: var(--md-sys-color-text);
}

.main-content {
  max-width: 800px;
  width: 100%;
  margin: 0 auto;
  padding: 24px 16px;
  box-sizing: border-box;
}

/* Material Card */
.md-card {
  background-color: var(--md-sys-color-card-bg);
  border: 1px solid var(--md-sys-color-border);
  border-radius: 16px;
  padding: 24px;
  margin-bottom: 24px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.04);
}

/* アップロード領域 */
.upload-card {
  text-align: center;
  border: 2px dashed var(--md-sys-color-primary);
  background-color: var(--md-sys-color-surface);
}

.upload-icon {
  width: 56px;
  height: 56px;
  border-radius: 50%;
  background-color: var(--md-sys-color-container);
  color: var(--md-sys-color-primary);
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 12px;
}

.custom-icon-img {
  width: 28px;
  height: 28px;
  display: block;
}

.btn-icon-img {
  width: 20px;
  height: 20px;
  display: inline-block;
  vertical-align: middle;
}

.fullscreen-icon-img {
  width: 24px;
  height: 24px;
  display: block;
}

.upload-card h2 {
  margin: 0 0 6px 0;
  font-size: 1.2rem;
  color: var(--md-sys-color-text);
}

.upload-card p {
  color: var(--md-sys-color-text-secondary);
  font-size: 0.85rem;
  margin-bottom: 20px;
}

.button-row {
  display: flex;
  justify-content: center;
  gap: 12px;
}

/* Material Buttons */
.md-button {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 10px 20px;
  border-radius: 20px;
  font-size: 0.9rem;
  font-weight: 500;
  cursor: pointer;
  border: none;
  transition: all 0.2s;
}

.md-button.disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.md-button-filled {
  background-color: var(--md-sys-color-primary);
  color: var(--md-sys-color-on-primary);
}

.md-button-filled:hover:not(.disabled) {
  background-color: var(--md-sys-color-primary-hover);
}

.md-button-outlined {
  background-color: transparent;
  border: 1px solid var(--md-sys-color-border);
  color: var(--md-sys-color-text);
}

.md-button-outlined:hover {
  background-color: var(--md-sys-color-container);
}

.hidden-input {
  display: none;
}

.icon-btn {
  background: none;
  border: none;
  color: var(--md-sys-color-text);
  padding: 8px;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}

.icon-btn:hover {
  background-color: var(--md-sys-color-border);
}

.theme-icon-img {
  width: 24px;
  height: 24px;
  display: block;
}

/* ステータスバー & コントロール */
.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  flex-wrap: wrap;
  gap: 12px;
}

.status-indicator {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.85rem;
  color: var(--md-sys-color-text-secondary);
}

.status-indicator.active {
  color: var(--md-sys-color-text);
  font-weight: 500;
}

.pulse-dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background-color: var(--md-sys-color-primary);
}

.control-actions {
  display: flex;
  align-items: center;
  gap: 12px;
}

/* Segmented Button (タブ切り替え) */
.segmented-button {
  display: flex;
  border: 1px solid var(--md-sys-color-border);
  border-radius: 20px;
  overflow: hidden;
}

.segmented-button button {
  background: none;
  border: none;
  padding: 6px 16px;
  font-size: 0.85rem;
  color: var(--md-sys-color-text);
  cursor: pointer;
}

.segmented-button button.active {
  background-color: var(--md-sys-color-container);
  color: var(--md-sys-color-on-container);
  font-weight: bold;
}

/* Canvas / Fullscreen */
.canvas-container {
  position: relative;
  width: 100%;
  background-color: #000;
  border-radius: 12px;
  overflow: hidden;
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 400px;
}

canvas {
  max-width: 100%;
  height: auto;
  display: block;
}

.canvas-container.is-fullscreen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: 9999;
  border-radius: 0;
}

.exit-fullscreen-btn {
  position: absolute;
  top: 16px;
  right: 16px;
  z-index: 10000;
}

/* 関数実行フォーム */
.md-text-field {
  margin-bottom: 16px;
}

.md-text-field label {
  display: block;
  font-size: 0.8rem;
  color: var(--md-sys-color-text-secondary);
  margin-bottom: 6px;
}

.md-select, .md-input {
  width: 100%;
  padding: 10px;
  border-radius: 8px;
  border: 1px solid var(--md-sys-color-border);
  background-color: var(--md-sys-color-surface);
  color: var(--md-sys-color-text);
  box-sizing: border-box;
}

.inputs-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin-bottom: 16px;
}

.input-item label {
  color: var(--md-sys-color-text-secondary);
}

.result-card {
  background-color: var(--md-sys-color-container);
  color: var(--md-sys-color-on-container);
  padding: 16px;
  border-radius: 12px;
  text-align: center;
}

.result-title {
  font-size: 0.8rem;
  color: var(--md-sys-color-on-container);
  opacity: 0.9;
}

.result-value {
  font-size: 1.5rem;
  font-weight: bold;
  margin: 6px 0;
  color: var(--md-sys-color-on-container);
}

.execution-time {
  font-size: 0.75rem;
  color: var(--md-sys-color-on-container);
  opacity: 0.8;
  display: block;
}
</style>