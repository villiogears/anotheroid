<script setup lang="ts">
import { ref, onMounted, onUnmounted, nextTick } from 'vue'

// --- 画面遷移状態管理 ---
const currentView = ref<'runner' | 'library'>('runner')

// --- UI / アプリ状態 ---
const isDarkMode = ref<boolean>(false)
const isLoaded = ref<boolean>(false)
const isScriptLoaded = ref<boolean>(false)
const statusMessage = ref<string>('wasm_exec.js を読み込み中...')
const fileName = ref<string>('')
const wasmType = ref<'canvas' | 'function'>('canvas')
const isFullscreen = ref<boolean>(false)

// --- ライブラリ画面用状態 ---
interface StoredWasmFile {
  name: string
  size: number
  updatedAt: Date
}
const savedFiles = ref<StoredWasmFile[]>([])
const isLoadingList = ref<boolean>(false)

// --- 数値計算型WASM用 ---
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

// --- WebAssembly / Canvas 参照 ---
let wasmInstance: any = null
let wasmModule: WebAssembly.Module | null = null
const wasmCanvas = ref<HTMLCanvasElement | null>(null)
let resizeObserver: ResizeObserver | null = null

// Cordovaが有効か判定
const isCordova = (): boolean => {
  return typeof (window as any).cordova !== 'undefined'
}

// public/wasm_exec.js の自動読み込み
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

// Cordova 内部ストレージへ書き込み
const saveWasmToCordovaStorage = (fileName: string, buffer: ArrayBuffer): Promise<string> => {
  return new Promise((resolve, reject) => {
    const windowFile = (window as any).resolveLocalFileSystemURL
    if (!windowFile) {
      reject(new Error('Cordova File Plugin が利用できません'))
      return
    }

    const storagePath = (window as any).cordova.file.dataDirectory

    windowFile(storagePath, (dirEntry: any) => {
      dirEntry.getFile(fileName, { create: true, exclusive: false }, (fileEntry: any) => {
        fileEntry.createWriter((fileWriter: any) => {
          fileWriter.onwriteend = () => {
            resolve(fileEntry.toURL())
          }
          fileWriter.onerror = (e: any) => reject(e)
          const blob = new Blob([buffer], { type: 'application/wasm' })
          fileWriter.write(blob)
        }, reject)
      }, reject)
    }, reject)
  })
}

// Cordova 内部ストレージから特定ファイルを読み込み
const loadWasmFromCordovaStorage = (fileName: string): Promise<ArrayBuffer> => {
  return new Promise((resolve, reject) => {
    const windowFile = (window as any).resolveLocalFileSystemURL
    const storagePath = (window as any).cordova.file.dataDirectory + fileName

    windowFile(storagePath, (fileEntry: any) => {
      fileEntry.file((file: File) => {
        const reader = new FileReader()
        reader.onloadend = () => {
          if (reader.result instanceof ArrayBuffer) {
            resolve(reader.result)
          } else {
            reject(new Error('ArrayBufferの取得に失敗しました'))
          }
        }
        reader.onerror = reject
        reader.readAsArrayBuffer(file)
      }, reject)
    }, reject)
  })
}

// 保存済み WASM ファイル一覧を取得する関数
const fetchStoredWasmFiles = (): Promise<StoredWasmFile[]> => {
  return new Promise((resolve, reject) => {
    if (!isCordova()) {
      resolve([])
      return
    }

    const windowFile = (window as any).resolveLocalFileSystemURL
    const storagePath = (window as any).cordova.file.dataDirectory

    windowFile(storagePath, (dirEntry: any) => {
      const directoryReader = dirEntry.createReader()
      directoryReader.readEntries((entries: any[]) => {
        const wasmEntries = entries.filter((e) => e.isFile && e.name.endsWith('.wasm'))
        
        const filePromises = wasmEntries.map((entry) => {
          return new Promise<StoredWasmFile>((res) => {
            entry.file((file: File) => {
              res({
                name: file.name,
                size: file.size,
                updatedAt: new Date(file.lastModified)
              })
            }, () => {
              res({ name: entry.name, size: 0, updatedAt: new Date() })
            })
          })
        })

        Promise.all(filePromises).then((files) => resolve(files))
      }, reject)
    }, reject)
  })
}

// ライブラリ一覧の更新処理
const refreshLibrary = async () => {
  isLoadingList.value = true
  try {
    savedFiles.value = await fetchStoredWasmFiles()
  } catch (err) {
    console.error('ファイル一覧取得エラー:', err)
  } finally {
    isLoadingList.value = false
  }
}

// ファイル一覧からファイルを選択して即座にロード
const selectAndLoadFromLibrary = async (fileItem: StoredWasmFile) => {
  try {
    statusMessage.value = `「${fileItem.name}」を読み込み中...`
    currentView.value = 'runner'
    const buffer = await loadWasmFromCordovaStorage(fileItem.name)
    await loadWasmBuffer(buffer, fileItem.name)
    localStorage.setItem('last_loaded_wasm', fileItem.name)
  } catch (err) {
    console.error('ライブラリからの復元エラー:', err)
    statusMessage.value = 'ファイルの読み込みに失敗しました。'
  }
}

// Cordova ストレージからファイルを削除
const deleteWasmFromStorage = (fileNameToDelete: string) => {
  if (!confirm(`「${fileNameToDelete}」を削除してもよろしいですか？`)) return

  const windowFile = (window as any).resolveLocalFileSystemURL
  const storagePath = (window as any).cordova.file.dataDirectory + fileNameToDelete

  windowFile(storagePath, (fileEntry: any) => {
    fileEntry.remove(() => {
      if (localStorage.getItem('last_loaded_wasm') === fileNameToDelete) {
        localStorage.removeItem('last_loaded_wasm')
      }
      refreshLibrary()
    }, (err: any) => {
      console.error('削除失敗:', err)
    })
  }, (err: any) => console.error('ファイルアクセス失敗:', err))
}

// GioUI 用 Canvas 初期化処理 (修正点: IDとサイズ計算を調整)
const setupGioCanvas = () => {
  const canvas = wasmCanvas.value
  if (!canvas) return

  // ID を Gio UI が標準参照する 'gioui-canvas' に設定
  canvas.id = 'gioui-canvas'
  canvas.tabIndex = 1
  canvas.focus()

  const updateCanvasSize = () => {
    if (!canvas.parentElement) return
    const rect = canvas.parentElement.getBoundingClientRect()
    const dpr = window.devicePixelRatio || 1

    canvas.style.width = `${rect.width}px`
    canvas.style.height = `${rect.height}px`

    canvas.width = Math.floor(rect.width * dpr)
    canvas.height = Math.floor(rect.height * dpr)

    // リサイズイベントを人工的に発行して Gio UI に伝達
    window.dispatchEvent(new Event('resize'))
  }

  updateCanvasSize()

  if (resizeObserver) resizeObserver.disconnect()
  resizeObserver = new ResizeObserver(() => {
    updateCanvasSize()
  })
  if (canvas.parentElement) {
    resizeObserver.observe(canvas.parentElement)
  }
}

// WASMの解析・インスタンス化
const loadWasmBuffer = async (buffer: ArrayBuffer, name: string) => {
  try {
    statusMessage.value = 'WebAssembly モジュールを解析中...'

    wasmType.value = 'canvas'
    await nextTick()

    // WASM 起動前に Canvas の準備を完了させる
    setupGioCanvas()

    const go = (window as any).Go ? new (window as any).Go() : null
    const importObject = go ? go.importObject : {}

    const instantiated = await WebAssembly.instantiate(buffer, importObject)
    wasmInstance = instantiated.instance
    wasmModule = instantiated.module

    fileName.value = name
    isLoaded.value = true
    statusMessage.value = `「${name}」を実行中`

    if (go && typeof go.run === 'function') {
      go.run(wasmInstance).catch((err: any) => {
        console.error('Go run error:', err)
      })
    }

    const exportsInfo = WebAssembly.Module.exports(wasmModule)
    exportedFunctions.value = exportsInfo.filter(exp => exp.kind === 'function')

    if (exportedFunctions.value.length > 0) {
      selectedFunc.value = exportedFunctions.value[0].name
    }
  } catch (error) {
    console.error('WASM 読み込みエラー:', error)
    isLoaded.value = false
    statusMessage.value = 'WASM ファイルの解析・実行に失敗しました。'
  }
}

// ファイル選択ハンドラー
const handleFileChange = async (event: Event) => {
  const target = event.target as HTMLInputElement
  if (target.files && target.files[0]) {
    const file = target.files[0]
    const reader = new FileReader()
    reader.onload = async (e) => {
      if (e.target?.result instanceof ArrayBuffer) {
        const buffer = e.target.result
        
        if (isCordova()) {
          try {
            statusMessage.value = 'アプリ内部ストレージに保存中...'
            await saveWasmToCordovaStorage(file.name, buffer)
            localStorage.setItem('last_loaded_wasm', file.name)
            await refreshLibrary()
          } catch (err) {
            console.error('Cordova保存エラー:', err)
          }
        }
        
        loadWasmBuffer(buffer, file.name)
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
    reader.onload = async (e) => {
      if (e.target?.result instanceof ArrayBuffer) {
        const buffer = e.target.result
        if (isCordova()) {
          try {
            await saveWasmToCordovaStorage(file.name, buffer)
            localStorage.setItem('last_loaded_wasm', file.name)
            await refreshLibrary()
          } catch (err) {
            console.error('Cordova保存エラー:', err)
          }
        }
        loadWasmBuffer(buffer, file.name)
      }
    }
    reader.readAsArrayBuffer(file)
  }
}

const handleDragOver = (event: DragEvent) => {
  event.preventDefault()
}

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

const toggleTheme = () => {
  isDarkMode.value = !isDarkMode.value
  if (isDarkMode.value) {
    document.documentElement.classList.add('dark')
  } else {
    document.documentElement.classList.remove('dark')
  }
}

const toggleFullscreen = () => {
  const element = document.getElementById('wasm-container')
  if (!element) return

  if (!document.fullscreenElement) {
    element.requestFullscreen().then(() => {
      isFullscreen.value = true
      nextTick(setupGioCanvas)
    }).catch(err => {
      console.error('全画面表示エラー:', err)
    })
  } else {
    document.exitFullscreen().then(() => {
      isFullscreen.value = false
      nextTick(setupGioCanvas)
    })
  }
}

const onFullscreenChange = () => {
  isFullscreen.value = !!document.fullscreenElement
  nextTick(setupGioCanvas)
}

// アプリ初期化処理
const initApp = async () => {
  try {
    await loadWasmExecScript()
    isScriptLoaded.value = true
    statusMessage.value = 'WASMファイルを選択してください'

    if (isCordova()) {
      await refreshLibrary()
      const lastSavedFile = localStorage.getItem('last_loaded_wasm')
      if (lastSavedFile) {
        try {
          statusMessage.value = `「${lastSavedFile}」を復元中...`
          const buffer = await loadWasmFromCordovaStorage(lastSavedFile)
          await loadWasmBuffer(buffer, lastSavedFile)
        } catch (e) {
          statusMessage.value = 'WASMファイルを選択してください'
        }
      }
    }
  } catch (err) {
    console.error(err)
    statusMessage.value = 'wasm_exec.js の読み込みに失敗しました'
  }
}

onMounted(() => {
  document.addEventListener('fullscreenchange', onFullscreenChange)
  
  if (isCordova()) {
    document.addEventListener('deviceready', initApp, false)
  } else {
    initApp()
  }
})

onUnmounted(() => {
  document.removeEventListener('fullscreenchange', onFullscreenChange)
  if (resizeObserver) {
    resizeObserver.disconnect()
  }
})

// ファイルサイズをバイトから適切な単位にフォーマット
const formatSize = (bytes: number): string => {
  if (bytes === 0) return '0 B'
  const k = 1024
  const sizes = ['B', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}
</script>

<template>
  <div class="app-shell" :class="{ 'dark-theme': isDarkMode }">
    <!-- 上部 アプリバー -->
    <header class="md-top-app-bar">
      <div class="app-bar-brand">
        <span class="material-icons">extension</span>
        <h1>Anotheroid</h1>
      </div>
      
      <div class="app-bar-actions">
        <button class="icon-btn theme-svg-btn" @click="toggleTheme" :title="isDarkMode ? 'ライトモードに切替' : 'ダークモードに切替'">
          <img :src="isDarkMode ? '/2.svg' : '/1.svg'" :alt="isDarkMode ? 'ライトモードに切り替え' : 'ダークモードに切り替え'" class="theme-icon-img" />
        </button>
      </div>
    </header>

    <!-- メインコンテンツ（画面切り替え対応） -->
    <main class="main-content">
      
      <!-- ページ 1: WASM 実行画面 -->
      <template v-if="currentView === 'runner'">
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
                  @click="wasmType = 'canvas'; nextTick(setupGioCanvas)"
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
            <!-- Gio UI が参照できるよう id="gioui-canvas" を設定 -->
            <canvas 
              id="gioui-canvas" 
              ref="wasmCanvas"
              tabindex="1"
              @mousedown="wasmCanvas?.focus()"
            ></canvas>
            
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
      </template>

      <!-- ページ 2: 保存済みWASMアプリ一覧（ライブラリ）画面 -->
      <template v-if="currentView === 'library'">
        <section class="md-card library-card">
          <div class="library-header">
            <h2>保存済み WASM 一覧</h2>
            <button class="icon-btn" @click="refreshLibrary" title="更新">
              <span class="material-icons">refresh</span>
            </button>
          </div>

          <p class="library-desc">Androidアプリ内部ストレージに保存されているWASMモジュールです。タップして即座に起動できます。</p>

          <div v-if="isLoadingList" class="loading-state">
            <p>読み込み中...</p>
          </div>

          <div v-else-if="savedFiles.length === 0" class="empty-state">
            <span class="material-icons empty-icon">folder_open</span>
            <p>保存されているWASMファイルはありません。</p>
            <p class="sub-text">メイン画面でWASMファイルを読み込むと自動的に保存されます。</p>
          </div>

          <ul v-else class="wasm-list">
            <li v-for="file in savedFiles" :key="file.name" class="wasm-item">
              <div class="wasm-info" @click="selectAndLoadFromLibrary(file)">
                <div class="wasm-icon">
                  <span class="material-icons">sports_esports</span>
                </div>
                <div class="wasm-details">
                  <span class="wasm-name">{{ file.name }}</span>
                  <span class="wasm-meta">{{ formatSize(file.size) }} • {{ file.updatedAt.toLocaleDateString() }}</span>
                </div>
              </div>
              <div class="wasm-actions">
                <button class="md-button md-button-filled sm-btn" @click="selectAndLoadFromLibrary(file)">
                  起動
                </button>
                <button class="icon-btn delete-btn" @click="deleteWasmFromStorage(file.name)" title="削除">
                  <span class="material-icons">delete_outline</span>
                </button>
              </div>
            </li>
          </ul>
        </section>
      </template>

    </main>

    <!-- 下部 Android標準風 ボトムナビゲーションバー -->
    <nav class="bottom-nav">
      <button 
        class="nav-item" 
        :class="{ active: currentView === 'runner' }" 
        @click="currentView = 'runner'"
      >
        <span class="material-icons">play_circle_filled</span>
        <span class="nav-label">実行画面</span>
      </button>

      <button 
        class="nav-item" 
        :class="{ active: currentView === 'library' }" 
        @click="currentView = 'library'; refreshLibrary()"
      >
        <span class="material-icons">apps</span>
        <span class="nav-label">アプリ一覧</span>
      </button>
    </nav>
  </div>
</template>

<style>
@import url('https://fonts.googleapis.com/icon?family=Material+Icons');

:root {
  --md-sys-color-bg: #f8fafc;
  --md-sys-color-surface: #ffffff;
  --md-sys-color-primary: #2563eb;
  --md-sys-color-primary-hover: #1d4ed8;
  --md-sys-color-on-primary: #ffffff;
  --md-sys-color-text: #000000;
  --md-sys-color-text-secondary: #64748b;
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
  --md-sys-color-text-secondary: #94a3b8;
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
  padding-bottom: 70px;
  box-sizing: border-box;
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

.md-card {
  background-color: var(--md-sys-color-card-bg);
  border: 1px solid var(--md-sys-color-border);
  border-radius: 16px;
  padding: 24px;
  margin-bottom: 24px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.04);
}

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

.sm-btn {
  padding: 6px 14px;
  font-size: 0.8rem;
  border-radius: 12px;
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

.canvas-container {
  position: relative;
  width: 100%;
  height: 450px;
  background-color: #000;
  border-radius: 12px;
  overflow: hidden;
  display: flex;
  justify-content: center;
  align-items: center;
}

canvas#gioui-canvas {
  width: 100%;
  height: 100%;
  display: block;
  outline: none;
  touch-action: none;
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

/* --- 保存済みWASMライブラリ画面スタイル --- */
.library-card {
  padding: 20px;
}

.library-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.library-header h2 {
  margin: 0;
  font-size: 1.25rem;
}

.library-desc {
  font-size: 0.85rem;
  color: var(--md-sys-color-text-secondary);
  margin: 6px 0 20px 0;
}

.empty-state {
  text-align: center;
  padding: 40px 10px;
  color: var(--md-sys-color-text-secondary);
}

.empty-icon {
  font-size: 48px;
  margin-bottom: 8px;
  opacity: 0.5;
}

.sub-text {
  font-size: 0.75rem;
}

.wasm-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.wasm-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px;
  border: 1px solid var(--md-sys-color-border);
  border-radius: 12px;
  margin-bottom: 10px;
  background-color: var(--md-sys-color-surface);
  transition: background-color 0.2s;
}

.wasm-info {
  display: flex;
  align-items: center;
  gap: 12px;
  flex: 1;
  cursor: pointer;
}

.wasm-icon {
  width: 40px;
  height: 40px;
  border-radius: 10px;
  background-color: var(--md-sys-color-container);
  color: var(--md-sys-color-primary);
  display: flex;
  align-items: center;
  justify-content: center;
}

.wasm-details {
  display: flex;
  flex-direction: column;
}

.wasm-name {
  font-weight: 600;
  font-size: 0.95rem;
}

.wasm-meta {
  font-size: 0.75rem;
  color: var(--md-sys-color-text-secondary);
  margin-top: 2px;
}

.wasm-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.delete-btn {
  color: #ef4444;
}

/* --- Android風ボトムナビゲーションバー --- */
.bottom-nav {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  height: 64px;
  background-color: var(--md-sys-color-surface);
  border-top: 1px solid var(--md-sys-color-border);
  display: flex;
  justify-content: space-around;
  align-items: center;
  z-index: 1000;
}

.nav-item {
  background: none;
  border: none;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  color: var(--md-sys-color-text-secondary);
  cursor: pointer;
  flex: 1;
  padding: 8px 0;
}

.nav-item .material-icons {
  font-size: 24px;
}

.nav-label {
  font-size: 0.75rem;
  font-weight: 500;
}

.nav-item.active {
  color: var(--md-sys-color-primary);
}

.nav-item.active .material-icons {
  transform: scale(1.1);
  transition: transform 0.2s;
}
</style>