<template>
  <div class="container">
    <h1 class="title">留言抽籤工具</h1>
    <p class="desc">
      請將DC留言複製並貼入以下輸入框。由於DC限制一次複製的留言數量，建議每次複製約80則留言。每次複製時，請確保與上一次複製的內容之間有換行分隔，以避免資料混淆。
    </p>

    <div class="main">
      <!-- 左欄 -->
      <div class="left-panel">
        <textarea
          v-model="rawText"
          rows="8"
          placeholder="請貼上留言"
        ></textarea>

        <div class="options">
          <div>
            <label>推文關鍵字：
              <input v-model="keyword" />
            </label>
          </div>

          <div>
            <label>重複帳號處理：
              <select v-model="duplicateMode">
                <option value="exclude">排除重複帳號</option>
                <option value="first">僅保留第一筆</option>
                <option value="allow">允許重複參加</option>
              </select>
            </label>
          </div>

          <button @click="parseText">整理留言</button>
        </div>

        <!-- 重複留言名單 -->
        <div v-if="duplicates.length" class="duplicates-list">
          <h2 class="font-semibold mb-2 text-red-600">
            重複留言帳號（{{ duplicates.length }} 位）
          </h2>
          <ul class="list-disc pl-5 text-sm text-gray-700">
            <li v-for="(item, idx) in duplicates" :key="idx">
              {{ item.account }} - 留言 {{ item.count }} 次（
              {{ item.numbers.map(n => '推' + n).join(', ') }}
              ）
            </li>
          </ul>
        </div>

        <!-- 參加名單 -->
        <div v-if="parsed.length" class="participants-list">
          <h2 class="font-semibold mb-2">參加名單（{{ parsed.length }}人）：</h2>
          <ul class="list-disc pl-5">
            <li v-for="(entry, i) in parsed" :key="i">
              {{ entry.account }} - 推{{ entry.number }}
            </li>
          </ul>
        </div>
      </div>

      <!-- 右欄 -->
      <div class="right-panel">
        <label>抽幾人：
          <input
            type="number"
            v-model.number="drawCount"
            min="1"
            :max="parsed.length || 1"
          />
        </label>

        <button
          @click="drawWinners"
          :disabled="parsed.length === 0"
        >
          抽籤
        </button>

        <div v-if="winners.length" class="winners-list">
          <h2>🎉 中獎名單（{{ winners.length }}人）：</h2>
          <ul>
            <li v-for="(winner, i) in winners" :key="i">
              {{ winner.account }} - 推{{ winner.number }}
            </li>
          </ul>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
  /* 外層容器 */
  .container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 24px;
    box-sizing: border-box;
    display: flex;
    flex-direction: column;
  }

  .title {
    font-size: 1.5rem;
    font-weight: bold;
    margin-bottom: 8px;
  }

  .desc {
    font-size: 0.875rem;
    color: #666;
    margin-bottom: 24px;
  }

  /* 主要區塊，左右並排 */
  .main {
    flex: 1;
    display: flex;
    gap: 24px;
    min-height: 0; /* 讓子元素可正確overflow */
  }

  /* 左欄 */
  .left-panel {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 16px;
    border: 1px solid #ccc;
    border-radius: 6px;
    padding: 16px;
    min-height: 0;
  }

  /* 左欄輸入框固定高度 */
  .left-panel textarea {
    width: 100%;
    height: 200px;
    resize: none;
    padding: 8px;
    border: 1px solid #ccc;
    border-radius: 4px;
    font-size: 1rem;
    box-sizing: border-box;
    flex-shrink: 0;
  }

  /* 選項 */
  .options {
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .options label {
    font-size: 0.9rem;
  }

  .options input,
  .options select {
    margin-left: 8px;
    padding: 4px 8px;
    border: 1px solid #ccc;
    border-radius: 4px;
  }

  .options button {
    margin-top: 8px;
    background-color: #3b82f6;
    color: white;
    border: none;
    padding: 8px;
    border-radius: 6px;
    cursor: pointer;
    width: 100px;
  }

  /* 重複留言名單 */
  .duplicates-list {
    max-height: 300px;
    overflow-y: auto;
    border: 1px solid #ccc;
    border-radius: 6px;
    padding: 12px;
    background-color: #fafafa;
  }

  /* 參加名單 */
  .participants-list {
    max-height: 300px;
    overflow-y: auto;
    border: 1px solid #ccc;
    border-radius: 6px;
    padding: 12px;
    background-color: #fafafa;
    margin-top: 16px;
  }

  /* 右欄 */
  .right-panel {
    width: 320px;
    max-height: 400px;
    display: flex;
    flex-direction: column;
    gap: 16px;
    border: 1px solid #ccc;
    border-radius: 6px;
    padding: 16px;
    min-height: 0;
  }

  .right-panel label {
    font-size: 0.9rem;
  }

  .right-panel input[type="number"] {
    margin-left: 8px;
    padding: 4px 8px;
    width: 60px;
    border: 1px solid #ccc;
    border-radius: 4px;
  }

  .right-panel button {
    background-color: #22c55e;
    color: white;
    border: none;
    padding: 8px;
    border-radius: 6px;
    cursor: pointer;
    width: 100px;
  }

  .right-panel button:disabled {
    background-color: #94a3b8;
    cursor: not-allowed;
  }

  /* 中獎名單 */
  .winners-list {
    overflow-y: auto;
    border-top: 1px solid #ddd;
    padding-top: 12px;
    margin-top: 16px;
  }

  .winners-list h2 {
    font-weight: 600;
    margin-bottom: 8px;
  }

  .winners-list ul {
    list-style: decimal inside;
    padding-left: 16px;
    margin: 0;
  }

  .winners-list li {
    margin-bottom: 4px;
  }
</style>


<script>
export default {
  data() {
    return {
      rawText: '',
      keyword: '推',
      duplicateMode: 'exclude', // exclude, first, allow
      parsed: [],
      drawCount: 1,
      winners: [],
      duplicates: [],
    }
  },
  methods: {
    parseText() {
      const lines = this.rawText.split('\n').map((l) => l.trim())
      const entries = []

      const isPureMetaLine = (line) => {
        if (!line) return true
        if (/^\[.*\]$/.test(line)) return true
        if (/^—/.test(line)) return true
        if (/^(昨天|上午|下午|晚上|凌晨).*/.test(line)) return true
        return false
      }

      const escapeRegExp = (string) =>
        string.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')

      const escapedKeyword = escapeRegExp(this.keyword)

      const isPushLine = (line) =>
        new RegExp(`^${escapedKeyword}\\d+$`).test(line)

      for (let i = 0; i < lines.length; i++) {
        const line = lines[i]
        const match = line.match(new RegExp(`^${escapedKeyword}(\\d+)$`))
        if (match) {
          const number = parseInt(match[1])

          let account = null

          for (let j = i - 1; j >= 0 && i - j <= 5; j--) {
            const candidate = lines[j]
            if (!candidate) continue
            if (isPureMetaLine(candidate)) continue
            if (isPushLine(candidate)) break

            if (candidate.includes('—')) {
              account = candidate.split('—')[0].trim()
            } else {
              account = candidate.trim()
            }
            break
          }

          if (account) {
            entries.push({ account, number })
          }
        }
      }

      // 找重複帳號（出現超過1次）
      const counts = {}
      entries.forEach((e) => {
        counts[e.account] = counts[e.account] ? [...counts[e.account], e.number] : [e.number]
      })
      this.duplicates = Object.entries(counts)
        .filter(([, nums]) => nums.length > 1)
        .map(([acc, nums]) => ({ account: acc, count: nums.length, numbers: nums }))

      // 處理重複帳號邏輯
      if (this.duplicateMode === 'exclude') {
        this.parsed = entries.filter(
          (e, i, self) => self.findIndex((x) => x.account === e.account) === i && counts[e.account].length === 1
        )
      } else if (this.duplicateMode === 'first') {
        this.parsed = entries.filter(
          (e, i, self) => self.findIndex((x) => x.account === e.account) === i
        )
      } else if (this.duplicateMode === 'allow') {
        this.parsed = entries
      } else {
        this.parsed = entries
      }

      this.winners = []
    },
    drawWinners() {
      const maxDraw = this.parsed.length
      if (this.drawCount < 1) this.drawCount = 1
      if (this.drawCount > maxDraw) this.drawCount = maxDraw

      const shuffled = [...this.parsed].sort(() => 0.5 - Math.random())
      this.winners = shuffled.slice(0, this.drawCount)
    },
  },
}
</script>