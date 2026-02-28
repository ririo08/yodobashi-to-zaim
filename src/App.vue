<script setup lang="ts">
import { computed, onBeforeUnmount, reactive, ref, watch } from 'vue'

type Entry = {
  date: string
  store: string
  amount: number
  source: string
}

const dropActive = ref(false)
const files = ref<File[]>([])
const entries = ref<Entry[]>([])
const errors = ref<string[]>([])
const isProcessing = ref(false)
const fileInput = ref<HTMLInputElement | null>(null)

const settings = reactive({
  method: 'payment',
  category: '未分類',
  subcategory: '未分類',
  paymentSource: 'ヨドバシゴールドポイントカードプラス',
  incomeTarget: '-',
  currency: 'JPY',
  aggregation: '常に集計に含める',
  itemPrefix: '',
  memoPrefix: 'ヨドバシGPC+',
  includeFilenameInMemo: true,
})

const summary = computed(() => {
  const total = entries.value.reduce((sum, entry) => sum + entry.amount, 0)
  return {
    count: entries.value.length,
    total,
  }
})

const previewEntries = computed(() => entries.value.slice(0, 20))
const sortedEntries = computed(() =>
  [...entries.value].sort((a, b) => a.date.localeCompare(b.date)),
)

const csvText = computed(() => buildZaimCsv(sortedEntries.value, settings))
const downloadUrl = ref('')

watch(
  csvText,
  (value) => {
    if (downloadUrl.value) {
      URL.revokeObjectURL(downloadUrl.value)
    }
    if (!value) {
      downloadUrl.value = ''
      return
    }
    const blob = new Blob([value], { type: 'text/csv' })
    downloadUrl.value = URL.createObjectURL(blob)
  },
  { immediate: true },
)

onBeforeUnmount(() => {
  if (downloadUrl.value) {
    URL.revokeObjectURL(downloadUrl.value)
  }
})

const handleFiles = async (fileList: FileList | File[]) => {
  const incoming = Array.from(fileList)
  if (incoming.length === 0) return

  files.value = incoming
  entries.value = []
  errors.value = []
  isProcessing.value = true

  try {
    const parsedEntries: Entry[] = []

    for (const file of incoming) {
      const buffer = await file.arrayBuffer()
      const text = decodeCsv(buffer)
      const rows = parseCsv(text)
      parsedEntries.push(...convertToEntries(rows, file.name))
    }

    entries.value = parsedEntries
  } catch (error) {
    errors.value.push('CSVの読み込みに失敗しました。ファイル形式をご確認ください。')
    console.error(error)
  } finally {
    isProcessing.value = false
  }
}

const handleDrop = async (event: DragEvent) => {
  event.preventDefault()
  dropActive.value = false
  if (!event.dataTransfer?.files?.length) return
  await handleFiles(event.dataTransfer.files)
}

const handleDragOver = (event: DragEvent) => {
  event.preventDefault()
  dropActive.value = true
}

const handleDragLeave = () => {
  dropActive.value = false
}

const handleFilePick = () => {
  fileInput.value?.click()
}

const handleFileChange = async (event: Event) => {
  const target = event.target as HTMLInputElement
  if (!target.files?.length) return
  await handleFiles(target.files)
  target.value = ''
}

const handleLoadSample = async () => {
  isProcessing.value = true
  errors.value = []

  try {
    const response = await fetch(`${import.meta.env.BASE_URL}yodobashi-sample.csv`)
    if (!response.ok) {
      throw new Error('Sample fetch failed')
    }
    const buffer = await response.arrayBuffer()
    const text = decodeCsv(buffer)
    const rows = parseCsv(text)
    const sampleEntries = convertToEntries(rows, 'yodobashi-sample.csv')
    files.value = [new File([], 'yodobashi-sample.csv')]
    entries.value = sampleEntries
  } catch (error) {
    errors.value.push('サンプルの読み込みに失敗しました。')
    console.error(error)
  } finally {
    isProcessing.value = false
  }
}

const clearAll = () => {
  files.value = []
  entries.value = []
  errors.value = []
}

const formatNumber = (value: number) => value.toLocaleString('ja-JP')

const decodeCsv = (buffer: ArrayBuffer) => {
  const shiftJisText = new TextDecoder('shift-jis').decode(buffer)
  const utf8Text = new TextDecoder('utf-8').decode(buffer)
  const shiftBad = (shiftJisText.match(/\uFFFD/g) || []).length
  const utf8Bad = (utf8Text.match(/\uFFFD/g) || []).length
  return shiftBad <= utf8Bad ? shiftJisText : utf8Text
}

const parseCsv = (text: string): string[][] => {
  const rows: string[][] = []
  let row: string[] = []
  let field = ''
  let inQuotes = false

  for (let i = 0; i < text.length; i += 1) {
    const char = text[i]

    if (inQuotes) {
      if (char === '"') {
        const next = text[i + 1]
        if (next === '"') {
          field += '"'
          i += 1
        } else {
          inQuotes = false
        }
      } else {
        field += char
      }
      continue
    }

    if (char === '"') {
      inQuotes = true
      continue
    }

    if (char === ',') {
      row.push(field.trim())
      field = ''
      continue
    }

    if (char === '\n') {
      row.push(field.trim())
      if (row.length > 1 || row[0]) {
        rows.push(row)
      }
      row = []
      field = ''
      continue
    }

    if (char === '\r') {
      continue
    }

    field += char
  }

  if (field || row.length > 0) {
    row.push(field.trim())
    rows.push(row)
  }

  return rows
}

const convertToEntries = (rows: string[][], source: string): Entry[] => {
  const output: Entry[] = []
  const datePattern = /^\d{4}[/-]\d{2}[/-]\d{2}$/

  for (const row of rows) {
    const date = (row[0] ?? '').trim()
    if (!date || !datePattern.test(date)) continue

    const normalizedDate = date.replace(/\//g, '-')
    const store = (row[1] ?? '').trim()
    const amountCandidate = (row[5] ?? row[2] ?? '').trim()
    const amount = parseAmount(amountCandidate)
    if (!amount) continue

    output.push({
      date: normalizedDate,
      store,
      amount,
      source,
    })
  }

  return output
}

const parseAmount = (value: string) => {
  const cleaned = value.replace(/[^\d.-]/g, '')
  const amount = Number.parseFloat(cleaned)
  return Number.isFinite(amount) ? Math.abs(amount) : 0
}

function buildZaimCsv(
  list: Entry[],
  config: {
    method: string
    category: string
    subcategory: string
    paymentSource: string
    incomeTarget: string
    currency: string
    aggregation: string
    itemPrefix: string
    memoPrefix: string
    includeFilenameInMemo: boolean
  },
) {
  if (list.length === 0) return ''

  const header = [
    '日付',
    '方法',
    'カテゴリ',
    'カテゴリの内訳',
    '支払元',
    '入金先',
    '品目',
    'メモ',
    'お店',
    '通貨',
    '収入',
    '支出',
    '振替',
    '残高調整',
    '通貨変換前の金額',
    '集計の設定',
  ]
  const rows = list.map((entry) => {
    const item =
      config.itemPrefix.trim().length > 0
        ? `${config.itemPrefix.trim()} ${entry.store}`.trim()
        : entry.store || 'ヨドバシカード'
    const memoParts = []
    if (config.memoPrefix.trim().length > 0) {
      memoParts.push(config.memoPrefix.trim())
    }
    if (config.includeFilenameInMemo && entry.source) {
      memoParts.push(entry.source)
    }
    const memo = memoParts.join(' / ')

    return [
      entry.date,
      config.method.trim(),
      config.category.trim(),
      config.subcategory.trim(),
      config.paymentSource.trim(),
      config.incomeTarget.trim(),
      item,
      memo,
      entry.store,
      config.currency.trim(),
      '0',
      entry.amount.toString(),
      '0',
      '0',
      entry.amount.toString(),
      config.aggregation.trim(),
    ]
  })

  return [header, ...rows].map((row) => row.map(escapeCsv).join(',')).join('\r\n')
}

function escapeCsv(value: string) {
  if (value == null) return ''
  const needsQuotes = /[",\n]/.test(value)
  if (!needsQuotes) return value
  return `"${value.replace(/"/g, '""')}"`
}
</script>

<template>
  <main class="app">
    <header class="hero">
      <p class="eyebrow">Yodobashi Gold Point Card+ → Zaim</p>
      <h1>ヨドバシ明細をまとめてZaimに取り込む</h1>
      <p class="lead">
        複数月分のCSVをドラッグ&ドロップで読み込み、Zaimのインポート用CSVを1つにまとめます。
      </p>
    </header>

    <section class="panel">
      <div
        class="dropzone"
        :class="{ active: dropActive }"
        @drop="handleDrop"
        @dragover="handleDragOver"
        @dragleave="handleDragLeave"
      >
        <input
          ref="fileInput"
          type="file"
          class="file-input"
          multiple
          accept=".csv"
          @change="handleFileChange"
        />
        <div class="dropzone-inner">
          <p class="dropzone-title">CSVをここにドロップ</p>
          <p class="dropzone-sub">またはクリックしてファイルを選択</p>
          <div class="dropzone-actions">
            <button type="button" class="btn ghost" @click="handleFilePick">
              ファイルを選択
            </button>
            <button type="button" class="btn ghost" @click="handleLoadSample">
              サンプルを読み込む
            </button>
          </div>
          <p class="dropzone-note">
            {{ isProcessing ? '読み込み中...' : 'Shift-JIS/UTF-8のCSVに対応' }}
          </p>
        </div>
      </div>

      <div class="status">
        <div class="status-card">
          <p class="status-label">変換件数</p>
          <p class="status-value">{{ summary.count }} 件</p>
        </div>
        <div class="status-card">
          <p class="status-label">合計金額</p>
          <p class="status-value">¥ {{ formatNumber(summary.total) }}</p>
        </div>
        <div class="status-card">
          <p class="status-label">読み込みファイル</p>
          <p class="status-value">{{ files.length }} 件</p>
        </div>
      </div>

      <div v-if="errors.length" class="error">
        <p v-for="(error, index) in errors" :key="index">{{ error }}</p>
      </div>

      <div class="controls">
        <div class="control-group">
          <label>
            方法
            <input v-model="settings.method" type="text" />
          </label>
          <label>
            カテゴリ
            <input v-model="settings.category" type="text" />
          </label>
          <label>
            カテゴリの内訳
            <input v-model="settings.subcategory" type="text" />
          </label>
          <label>
            支払元
            <input v-model="settings.paymentSource" type="text" />
          </label>
          <label>
            入金先
            <input v-model="settings.incomeTarget" type="text" />
          </label>
          <label>
            通貨
            <input v-model="settings.currency" type="text" />
          </label>
          <label>
            集計の設定
            <input v-model="settings.aggregation" type="text" />
          </label>
          <label>
            内容プレフィックス
            <input v-model="settings.itemPrefix" type="text" placeholder="例: ヨドバシ" />
          </label>
        </div>
        <div class="control-group">
          <label>
            メモプレフィックス
            <input v-model="settings.memoPrefix" type="text" placeholder="例: ヨドバシGPC+" />
          </label>
          <label class="checkbox">
            <input v-model="settings.includeFilenameInMemo" type="checkbox" />
            メモにファイル名を含める
          </label>
        </div>
        <div class="control-actions">
          <a
            class="btn primary"
            :class="{ disabled: !downloadUrl || summary.count === 0 }"
            :href="downloadUrl"
            download="zaim-import.csv"
          >
            Zaim CSVをダウンロード
          </a>
          <a
            class="btn ghost"
            href="https://content.zaim.net/home/money"
            target="_blank"
            rel="noopener noreferrer"
          >
            Zaimにインポート
          </a>
          <a
            class="btn ghost"
            href="https://secure.goldpoint.co.jp/memx/web_meisai/top/index.html"
            target="_blank"
            rel="noopener noreferrer"
          >
            ヨドバシ明細を開く
          </a>
          <button type="button" class="btn ghost" @click="clearAll">クリア</button>
        </div>
        <p class="hint">
          Zaimのインポート形式: 日付 / 方法 / カテゴリ / カテゴリの内訳 / 支払元 / 入金先 / 品目 /
          メモ / お店 / 通貨 / 収入 / 支出 / 振替 / 残高調整 / 通貨変換前の金額 / 集計の設定
        </p>
      </div>
    </section>

    <section class="panel">
      <div class="panel-header">
        <h2>プレビュー</h2>
        <p>最初の20件を表示します。</p>
      </div>
      <div class="table">
        <div class="table-row table-head">
          <span>日付</span>
          <span>店名</span>
          <span>金額</span>
          <span>元ファイル</span>
        </div>
        <div
          v-for="(entry, index) in previewEntries"
          :key="`${entry.date}-${entry.store}-${index}`"
          class="table-row"
        >
          <span>{{ entry.date }}</span>
          <span>{{ entry.store }}</span>
          <span>¥ {{ formatNumber(entry.amount) }}</span>
          <span>{{ entry.source }}</span>
        </div>
        <p v-if="previewEntries.length === 0" class="empty">ここにプレビューが表示されます。</p>
      </div>
    </section>

    <section class="panel note">
      <h2>使い方のヒント</h2>
      <ul>
        <li>複数月分のCSVをまとめてドロップすると1つのCSVに統合されます。</li>
        <li>変換件数と合計金額を確認してからダウンロードしてください。</li>
        <li>Zaim側でカテゴリを調整したい場合はここで空欄にしてもOKです。</li>
      </ul>
    </section>
  </main>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=Fraunces:wght@400;600&family=Space+Grotesk:wght@400;500;600&display=swap');

:root {
  color-scheme: light;
  font-family: 'Space Grotesk', 'Helvetica Neue', sans-serif;
  background-color: #f6f1e7;
  color: #1c1914;
}

body {
  margin: 0;
  min-height: 100vh;
  background:
    radial-gradient(circle at top left, #f8e7d0 0%, rgba(248, 231, 208, 0) 45%),
    radial-gradient(circle at 80% 10%, #f4d6b0 0%, rgba(244, 214, 176, 0) 45%),
    linear-gradient(120deg, #f6f1e7 0%, #f3eadc 100%);
}

#app {
  min-height: 100vh;
}

.app {
  max-width: 1080px;
  margin: 0 auto;
  padding: 48px 24px 80px;
  display: flex;
  flex-direction: column;
  gap: 32px;
  animation: fadeIn 0.8s ease-out both;
}

.hero {
  display: grid;
  gap: 12px;
}

.eyebrow {
  text-transform: uppercase;
  letter-spacing: 0.2em;
  font-size: 12px;
  color: #5b4a35;
}

h1 {
  font-family: 'Fraunces', 'Times New Roman', serif;
  font-size: clamp(32px, 5vw, 52px);
  margin: 0;
}

.lead {
  font-size: 18px;
  max-width: 680px;
  margin: 0;
  color: #4a3b2b;
}

.panel {
  background: rgba(255, 255, 255, 0.85);
  border-radius: 24px;
  padding: 28px;
  box-shadow: 0 18px 40px rgba(35, 28, 18, 0.12);
  display: grid;
  gap: 24px;
}

.dropzone {
  border: 2px dashed rgba(92, 74, 52, 0.4);
  border-radius: 22px;
  padding: 32px;
  text-align: center;
  background: linear-gradient(140deg, rgba(255, 255, 255, 0.9), rgba(242, 232, 216, 0.6));
  transition: border 0.3s ease, transform 0.3s ease;
}

.dropzone.active {
  border-color: #b86b38;
  transform: translateY(-2px);
}

.dropzone-inner {
  display: grid;
  gap: 10px;
}

.dropzone-title {
  font-size: 20px;
  font-weight: 600;
  margin: 0;
}

.dropzone-sub {
  margin: 0;
  color: #6b5640;
}

.dropzone-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  justify-content: center;
}

.dropzone-note {
  font-size: 12px;
  color: #8b755c;
  margin: 0;
}

.file-input {
  display: none;
}

.btn {
  border: 1px solid transparent;
  border-radius: 999px;
  padding: 10px 22px;
  font-weight: 600;
  font-family: inherit;
  cursor: pointer;
  transition: transform 0.2s ease, box-shadow 0.2s ease, background 0.2s ease;
}

.btn.primary {
  background: #b86b38;
  color: #fff;
  box-shadow: 0 12px 20px rgba(184, 107, 56, 0.3);
}

.btn.primary:hover {
  transform: translateY(-1px);
}

.btn.primary.disabled {
  pointer-events: none;
  opacity: 0.5;
}

.btn.ghost {
  background: rgba(255, 255, 255, 0.6);
  border-color: rgba(92, 74, 52, 0.3);
  color: #553f2b;
}

.status {
  display: grid;
  gap: 16px;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
}

.status-card {
  background: #fdf9f3;
  border-radius: 16px;
  padding: 16px 18px;
  border: 1px solid rgba(92, 74, 52, 0.2);
  display: grid;
  gap: 6px;
  animation: rise 0.5s ease both;
}

.status-label {
  font-size: 12px;
  text-transform: uppercase;
  letter-spacing: 0.14em;
  color: #7a644c;
  margin: 0;
}

.status-value {
  font-size: 22px;
  font-weight: 600;
  margin: 0;
}

.error {
  background: #fff0ec;
  color: #a63b2d;
  padding: 12px 16px;
  border-radius: 12px;
  border: 1px solid rgba(166, 59, 45, 0.3);
}

.controls {
  display: grid;
  gap: 20px;
}

.control-group {
  display: grid;
  gap: 12px;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
}

label {
  display: grid;
  gap: 6px;
  font-size: 14px;
  color: #3f2f20;
}

input[type='text'] {
  padding: 10px 12px;
  border-radius: 10px;
  border: 1px solid rgba(92, 74, 52, 0.3);
  font-family: inherit;
  background: #fff;
}

.checkbox {
  align-items: center;
  grid-template-columns: auto 1fr;
  gap: 10px;
}

.control-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  align-items: center;
}

.hint {
  font-size: 13px;
  color: #7b6750;
  margin: 0;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
  gap: 12px;
}

.panel-header h2 {
  margin: 0;
  font-size: 22px;
}

.panel-header p {
  margin: 0;
  color: #6f5a44;
}

.table {
  display: grid;
  gap: 8px;
}

.table-row {
  display: grid;
  grid-template-columns: 1fr 2fr 1fr 1fr;
  gap: 8px;
  padding: 10px 12px;
  border-radius: 12px;
  background: #fdf8f1;
  border: 1px solid rgba(92, 74, 52, 0.1);
  font-size: 14px;
}

.table-head {
  background: #f6e5cf;
  font-weight: 600;
}

.empty {
  text-align: center;
  color: #8b755c;
  margin: 16px 0 0;
}

.note {
  background: rgba(255, 255, 255, 0.7);
}

.note ul {
  margin: 0;
  padding-left: 18px;
  display: grid;
  gap: 8px;
  color: #5d4a36;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(12px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes rise {
  from {
    opacity: 0;
    transform: translateY(6px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@media (max-width: 720px) {
  .app {
    padding: 32px 18px 60px;
  }

  .panel {
    padding: 20px;
  }

  .panel-header {
    flex-direction: column;
    align-items: flex-start;
  }

  .table-row {
    grid-template-columns: 1fr;
  }
}
</style>
