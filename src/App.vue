<script setup lang="ts">
import { ref, onMounted } from 'vue'

// リアクティブ変数の定義
const num1 = ref<number>(10)
const num2 = ref<number>(25)
const result = ref<number | string | null>(null)
const isLoaded = ref<boolean>(false)
const statusMessage = ref<string>('ファイルを選択するか、ドラッグ＆ドロップしてください')
const executionTime = ref<string>('')
const fileName = ref<string>('')

// エクスポート関数の情報
interface ExportedFunc {
  name: string
  kind: string
}
const exportedFunctions = ref<ExportedFunc[]>([])
const selectedFunc = ref<string>('add')

// WebAssembly インスタンス
let wasmInstance: any = null
let wasmModule: WebAssembly.Module | null = null

/**
 * 組み込みのテスト用 WebAssembly バイナリ（i32加算関数 add(a, b)）
 */
const defaultWasmCode = new Uint8Array([
  0x00, 0x61, 0x73, 0x6d, 0x01, 0x00, 0x00, 0x00,
  0x01, 0x07, 0x01, 0x60, 0x02, 0x7f, 0x7f, 0x01, 0x7f,
  0x03, 0x02, 0x01, 0x00,
  0x07, 0x07, 0x01, 0x03, 0x61, 0x64, 0x64, 0x00, 0x00,
  0x0a, 0x09, 0x01, 0x07, 0x00, 0x20, 0x00, 0x20, 0x01, 0x6a, 0x0b
])

// WASM ArrayBuffer からインスタンス化する処理
const loadWasmBuffer = async (buffer: ArrayBuffer, name: string) => {
  try {
    statusMessage.value = 'WebAssembly モジュールを解析中...'
    
    // モジュールのコンパイルとインスタンス化
    wasmModule = await WebAssembly.compile(buffer)
    const instance = await WebAssembly.instantiate(wasmModule)
    wasmInstance = instance

    // エクスポートされている関数群を検出
    const exportsInfo = WebAssembly.Module.exports(wasmModule)
    exportedFunctions.value = exportsInfo.filter(exp => exp.kind === 'function')

    if (exportedFunctions.value.length > 0) {
      selectedFunc.value = exportedFunctions.value[0].name
    }

    fileName.value = name
    isLoaded.value = true
    statusMessage.value = `「${name}」を正常に読み込みました！`
    executeWasm()
  } catch (error) {
    console.error('WASM 読み込みエラー:', error)
    isLoaded.value = false
    statusMessage.value = 'WASM ファイルの解析に失敗しました。正しいフォーマットか確認してください。'
  }
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
    
    // 入力値に応じて呼び出し
    const res = func(num1.value, num2.value)
    
    const endTime = performance.now()
    executionTime.value = (endTime - startTime).toFixed(4)
    result.value = res !== undefined ? res : '実行完了 (戻り値なし)'
  } catch (err: any) {
    console.error('実行エラー:', err)
    result.value = `エラー: ${err.message || '実行中にエラーが発生しました'}`
  }
}

// デフォルトのサンプルWASMを読み込む
const loadDefaultWasm = () => {
  loadWasmBuffer(defaultWasmCode.buffer, 'sample_add.wasm')
}

onMounted(() => {
  loadDefaultWasm()
})
</script>

<template>
  <main class="container">
    <header class="header">
      <h1>Vue.js + WebAssembly</h1>
      <p class="subtitle">ローカル `.wasm` ファイルのアップロード & 実行環境</p>
    </header>

    <!-- ファイル選択・ドロップエリア -->
    <section 
      class="upload-card"
      @drop="handleDrop"
      @dragover="handleDragOver"
    >
      <div class="upload-icon">📁</div>
      <h3>WASM ファイルをアップロード</h3>
      <p>ローカルデバイスから `.wasm` ファイルを選択またはドロップしてください</p>

      <div class="button-group">
        <label class="file-label">
          ファイルを選択
          <input 
            type="file" 
            accept=".wasm" 
            @change="handleFileChange" 
            class="file-input"
          />
        </label>
        <button class="sample-btn" @click="loadDefaultWasm">
          サンプルWASMを試す
        </button>
      </div>
    </section>

    <!-- 実行・結果カード -->
    <section class="card">
      <div class="status-bar" :class="{ success: isLoaded }">
        <span class="dot"></span>
        <span>{{ statusMessage }}</span>
      </div>

      <div v-if="isLoaded" class="content">
        <!-- エクスポート関数選択 -->
        <div class="form-group" v-if="exportedFunctions.length > 0">
          <label>実行する関数を選択:</label>
          <select v-model="selectedFunc" @change="executeWasm" class="select-box">
            <option v-for="fn in exportedFunctions" :key="fn.name" :value="fn.name">
              {{ fn.name }} ()
            </option>
          </select>
        </div>

        <!-- 引数入力フォーム -->
        <div class="calculator">
          <div class="input-group">
            <label>引数 1 (a)</label>
            <input 
              v-model.number="num1" 
              type="number" 
              @input="executeWasm" 
              placeholder="1" 
            />
          </div>

          <div class="input-group">
            <label>引数 2 (b)</label>
            <input 
              v-model.number="num2" 
              type="number" 
              @input="executeWasm" 
              placeholder="2" 
            />
          </div>
        </div>

        <!-- 実行結果領域 -->
        <div class="result-container">
          <span class="result-label">実行結果:</span>
          <div class="result-box">
            {{ result }}
          </div>
        </div>

        <div class="metrics" v-if="executionTime">
          実行時間: <strong>{{ executionTime }} ms</strong>
        </div>
      </div>
    </section>
  </main>
</template>

<style scoped>
.container {
  max-width: 600px;
  margin: 0 auto;
  padding: 24px;
  font-family: system-ui, -apple-system, sans-serif;
  color: #2c3e50;
}

.header {
  text-align: center;
  margin-bottom: 24px;
}

.header h1 {
  margin: 0;
  font-size: 1.8rem;
  color: #42b883;
}

.subtitle {
  margin-top: 8px;
  color: #666;
  font-size: 0.9rem;
}

.upload-card {
  background: #f0fdf4;
  border: 2px dashed #42b883;
  border-radius: 12px;
  padding: 24px;
  text-align: center;
  margin-bottom: 20px;
  transition: background-color 0.2s;
}

.upload-card:hover {
  background: #e6f9ed;
}

.upload-icon {
  font-size: 2.5rem;
  margin-bottom: 8px;
}

.upload-card h3 {
  margin: 0 0 4px 0;
  color: #2c3e50;
}

.upload-card p {
  margin: 0 0 16px 0;
  font-size: 0.85rem;
  color: #666;
}

.button-group {
  display: flex;
  justify-content: center;
  gap: 12px;
}

.file-label {
  background-color: #42b883;
  color: white;
  padding: 10px 18px;
  border-radius: 8px;
  font-size: 0.9rem;
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.2s;
}

.file-label:hover {
  background-color: #33a06f;
}

.file-input {
  display: none;
}

.sample-btn {
  background-color: #ffffff;
  color: #2c3e50;
  border: 1px solid #ccc;
  padding: 10px 18px;
  border-radius: 8px;
  font-size: 0.9rem;
  cursor: pointer;
  transition: border-color 0.2s;
}

.sample-btn:hover {
  border-color: #42b883;
}

.card {
  background: #ffffff;
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
  border: 1px solid #eef2f5;
}

.status-bar {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 14px;
  border-radius: 6px;
  background-color: #fff3cd;
  color: #856404;
  font-size: 0.85rem;
  margin-bottom: 20px;
}

.status-bar.success {
  background-color: #d4edda;
  color: #155724;
}

.dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background-color: currentColor;
}

.form-group {
  margin-bottom: 16px;
}

.form-group label {
  display: block;
  font-size: 0.85rem;
  font-weight: bold;
  margin-bottom: 6px;
  color: #444;
}

.select-box {
  width: 100%;
  padding: 10px;
  border: 2px solid #ddd;
  border-radius: 8px;
  font-size: 1rem;
  outline: none;
  background-color: #fff;
}

.calculator {
  display: flex;
  gap: 12px;
  margin-bottom: 16px;
}

.input-group {
  flex: 1;
}

.input-group label {
  display: block;
  font-size: 0.8rem;
  color: #666;
  margin-bottom: 4px;
}

input[type="number"] {
  width: 100%;
  padding: 10px;
  font-size: 1.1rem;
  border: 2px solid #ddd;
  border-radius: 8px;
  text-align: center;
  outline: none;
  box-sizing: border-box;
}

input[type="number"]:focus {
  border-color: #42b883;
}

.result-container {
  margin-top: 16px;
}

.result-label {
  font-size: 0.85rem;
  font-weight: bold;
  color: #444;
}

.result-box {
  margin-top: 6px;
  padding: 14px;
  font-size: 1.3rem;
  font-weight: bold;
  background-color: #f8f9fa;
  border: 2px solid #42b883;
  border-radius: 8px;
  text-align: center;
  color: #42b883;
  word-break: break-all;
}

.metrics {
  font-size: 0.8rem;
  color: #888;
  text-align: right;
  margin-top: 12px;
}
</style>