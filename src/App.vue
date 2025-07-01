<template>
  <div class="container">
    <h1 class="title">留言抽籤工具</h1>
    <p class="announcement">
      🎉 全新 <strong>更快速、準確、自動化</strong> 的抽籤工具 Chrome 擴充套件上線了！<br>
      👉 <a href="https://github.com/artale-tool/discord-lottery-extension" target="_blank">點我前往 GitHub 下載</a>
    </p>
    <p class="desc">
      請將 DC 留言複製並貼入以下輸入框。由於 DC 限制一次可複製的留言數量，建議每次複製約 80 則留言。每次複製時，請務必確保與上一次複製的內容之間有換行分隔，以避免資料混淆或遺漏。
    </p>

    <div class="main">
      <!-- 左欄 -->
      <div class="left-panel">
        <textarea v-model="rawText" rows="8" placeholder="請貼上留言"></textarea>

        <div class="options">
          <div class="option-group">
            <label>推文關鍵字：
              <input v-model="keyword" />
            </label>
          </div>

          <div class="option-group">
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

        <!-- 重複帳號名單 -->
        <div v-if="filteredDuplicates.length" class="duplicates-list">
          <h2 class="duplicates-title">重複留言帳號（{{ filteredDuplicates.length }} 位）</h2>
          <p class="note-text">※ 部分使用者 ID 可能相同但實際為不同帳號，可能會誤判為重複帳號，請特別留意是否誤排除有效留言。</p>
          <ul class="duplicates-ul">
            <li v-for="(item, idx) in filteredDuplicates" :key="idx" class="duplicate-item">
              <span>
                {{ item.account }} - 留言 {{ item.count }} 次（{{ item.numbers.map(n => '推' + n).join(', ') }}）
              </span>
              <button @click="includeAccount(item.account)" class="btn-restore">還原參加</button>
            </li>
          </ul>
        </div>

        <!-- 推號重複名單 -->
        <div v-if="duplicateNumbers.length" class="duplicates-list">
          <h2 class="duplicates-title">推文號碼重複（{{ duplicateNumbers.length }} 筆已移除）</h2>
          <p class="note-text">※ 同一個推號只能保留第一個留言，其餘將自動略過。</p>
          <ul class="duplicates-ul">
            <li v-for="(item, idx) in duplicateNumbers" :key="idx">
              {{ item.account }} - 推{{ item.number }}
            </li>
          </ul>
        </div>

        <!-- 參加名單 -->
        <div v-if="parsed.length" class="participants-list">
          <h2 class="participants-title">參加名單（{{ parsed.length }}人）：</h2>
          <ul class="participants-ul">
            <li v-for="(entry, i) in parsed" :key="i">
              {{ entry.account }} - 推{{ entry.number }}
            </li>
          </ul>
        </div>
      </div>

      <!-- 右欄 -->
      <div class="right-panel">
        <label>抽幾人：
          <input type="number" v-model.number="drawCount" min="1" :max="parsed.length || 1" />
        </label>

        <button @click="drawWinners" :disabled="parsed.length === 0">抽籤</button>

        <div v-if="winners.length" class="winners-list">
          <h2>🎉 中獎名單（{{ winners.length }}人）：</h2>
          <ul class="winners-ul">
            <li v-for="(winner, i) in winners" :key="i">
              {{ winner.account }} - 推{{ winner.number }}
            </li>
          </ul>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      rawText: '',
      keyword: '推',
      duplicateMode: 'exclude',
      parsed: [],
      drawCount: 1,
      winners: [],
      duplicates: [],
      duplicateNumbers: [],
      entries: [],
      manualIncluded: [],
    }
  },
  computed: {
    filteredDuplicates() {
      return this.duplicates.filter(d => !this.manualIncluded.includes(d.account))
    }
  },
  methods: {
    parseText() {
      const lines = this.rawText.split('\n').map(l => l.trim())
      const entries = []

      const isPureMetaLine = (line) => !line || /^\[.*\]$/.test(line) || /^—/.test(line) || /^(昨天|上午|下午|晚上|凌晨).*/.test(line)
      const escapeRegExp = (str) => str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
      const escapedKeyword = escapeRegExp(this.keyword)
      const isPushLine = (line) => new RegExp(`^${escapedKeyword}\\d+$`).test(line)

      for (let i = 0; i < lines.length; i++) {
        const line = lines[i]
        const match = line.match(new RegExp(`^${escapedKeyword}(\\d+)$`))
        if (match) {
          const number = parseInt(match[1])
          let account = null
          for (let j = i - 1; j >= 0 && i - j <= 5; j--) {
            const candidate = lines[j]
            if (!candidate || isPureMetaLine(candidate) || isPushLine(candidate)) continue
            account = candidate.includes('—') ? candidate.split('—')[0].trim() : candidate.trim()
            break
          }
          if (account) entries.push({ account, number })
        }
      }

      // 移除推號重複，只保留第一筆
      const seenNumbers = new Set()
      const uniqueEntries = []
      const duplicateNumbers = []
      for (const entry of entries) {
        if (!seenNumbers.has(entry.number)) {
          seenNumbers.add(entry.number)
          uniqueEntries.push(entry)
        } else {
          duplicateNumbers.push(entry)
        }
      }

      this.entries = uniqueEntries
      this.duplicateNumbers = duplicateNumbers

      // 重複帳號統計
      const counts = {}
      this.entries.forEach(e => {
        counts[e.account] = counts[e.account] ? [...counts[e.account], e.number] : [e.number]
      })

      this.duplicates = Object.entries(counts)
        .filter(([, nums]) => nums.length > 1)
        .map(([acc, nums]) => ({ account: acc, count: nums.length, numbers: nums }))

      this.manualIncluded = []
      this.updateParsedList()
      this.winners = []
    },

    updateParsedList() {
      const counts = {}
      this.entries.forEach(e => {
        counts[e.account] = counts[e.account] ? [...counts[e.account], e.number] : [e.number]
      })

      if (this.duplicateMode === 'exclude') {
        this.parsed = this.entries.filter(
          (e, i, self) =>
            (self.findIndex(x => x.account === e.account) === i && counts[e.account].length === 1) ||
            this.manualIncluded.includes(e.account)
        )
      } else if (this.duplicateMode === 'first') {
        this.parsed = this.entries.filter(
          (e, i, self) =>
            self.findIndex(x => x.account === e.account) === i ||
            this.manualIncluded.includes(e.account)
        )
      } else {
        this.parsed = this.entries
      }
    },

    includeAccount(account) {
      if (!this.manualIncluded.includes(account)) {
        this.manualIncluded.push(account)
        this.updateParsedList()
      }
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

<style scoped>
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 24px;
  display: flex;
  flex-direction: column;
  font-family: Arial, sans-serif;
  color: #333;
}

.title {
  font-size: 24px;
  font-weight: bold;
  margin-bottom: 8px;
}

.announcement {
  background-color: #fff8dc;
  border-left: 4px solid #ffa500;
  padding: 10px;
  margin-bottom: 1em;
  font-size: 1.1em;
}

.desc {
  font-size: 14px;
  color: #666;
  margin-bottom: 24px;
}

.main {
  display: flex;
  gap: 24px;
}

.left-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 16px;
  border: 1px solid #ccc;
  border-radius: 6px;
  padding: 16px;
  box-sizing: border-box;
}

.left-panel textarea {
  width: 100%;
  height: 200px;
  padding: 8px;
  font-size: 16px;
  border: 1px solid #ccc;
  border-radius: 4px;
  resize: none;
  box-sizing: border-box;
}

.options {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.options input,
.options select {
  margin-left: 8px;
  padding: 4px 8px;
  font-size: 14px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.options button {
  margin-top: 8px;
  width: 100px;
  padding: 8px;
  font-size: 14px;
  background-color: #3b82f6;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}

.duplicates-list {
  max-height: 300px;
  overflow-y: auto;
  border: 1px solid #ccc;
  border-radius: 6px;
  padding: 12px;
  background-color: #fafafa;
  margin-top: 16px;
}

.duplicates-title {
  font-weight: 600;
  margin-bottom: 4px;
  color: #333;
}

.note-text {
  font-size: 12px;
  color: #555;
  margin-bottom: 8px;
}

.duplicates-ul,
.participants-ul {
  list-style: disc inside;
  font-size: 14px;
  color: #333;
  padding-left: 12px;
  margin: 0;
}

.duplicate-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 6px;
}

.btn-restore {
  background-color: white;
  color: #333;
  border: 1px solid #333;
  padding: 4px 10px;
  font-size: 12px;
  border-radius: 6px;
  cursor: pointer;
  transition: background-color 0.2s ease;
}

.btn-restore:hover {
  background-color: #f0f0f0;
}

.participants-list {
  max-height: 300px;
  overflow-y: auto;
  border: 1px solid #ccc;
  border-radius: 6px;
  padding: 12px;
  background-color: #fafafa;
}

.participants-title {
  font-weight: 600;
  margin-bottom: 4px;
}

.right-panel {
  width: 320px;
  max-height: 400px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  border: 1px solid #ccc;
  border-radius: 6px;
  padding: 16px;
  box-sizing: border-box;
  font-size: 14px;
}

.right-panel input[type='number'] {
  margin-left: 8px;
  padding: 4px 8px;
  width: 60px;
  font-size: 14px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.right-panel button {
  width: 100px;
  padding: 8px;
  font-size: 14px;
  background-color: #22c55e;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}

.right-panel button:disabled {
  background-color: #94a3b8;
  cursor: not-allowed;
}

.winners-list {
  overflow-y: auto;
  border-top: 1px solid #ddd;
  padding-top: 12px;
  margin-top: 16px;
}

.winners-ul {
  list-style: decimal inside;
  padding-left: 16px;
  margin: 0;
}

.winners-ul li {
  margin-bottom: 4px;
}
</style>
