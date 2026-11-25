# NIGHTLINE // HOLO

非官方《電馭叛客 2077》NPC 私訊網站前端（Mock Mode）。以 V 的身分和茱蒂、帕娜、強尼傳訊息，回覆為預寫的模擬文字，尚未串接 AI。

- 單檔 `index.html`，直接開啟或用 GitHub Pages 部署即可
- 聊天紀錄存在瀏覽器的 IndexedDB，不限數量、不會自動刪除

本專案為粉絲作品，與 CD PROJEKT RED 無關。角色名稱權利屬原權利人，對話內容皆為原創。

## 程式分層

`index.html` 的 script 依序分成七段，介面只透過下層提供的方法存取資料：

1. **角色人格資料（Persona）**：`personaOf(id, mode)`，整理成 AI 與介面共用的固定格式；模擬台詞不在這裡。
2. **儲存層（ChatStore）**：`IndexedDBStore()`，以及資料庫無法使用時的 `MemoryStore()` 備援。
3. **舊版 localStorage 遷移**：`migrateLegacy(store)`。
4. **AI Context**：`buildContext(id)`，與完整聊天紀錄分開。
5. **回覆來源**：`replyProvider`（目前是 `MockProvider`）。
6. **介面狀態與畫面**：分頁載入、送出、重設、匯出／匯入、錯誤提示。
7. **啟動流程**：`boot()`。

## 儲存架構（IndexedDB）

資料庫 `nightline-holo`，版本 1：

| Object store | 主鍵 | 內容 |
|---|---|---|
| `messages` | `[contactId, seq]` | 全部聊天紀錄 `{ contactId, seq, from, text, t }` |
| `settings` | `id` | 每位 NPC 的設定：草稿、未讀、關係、待選回覆 |
| `meta` | `key` | 目前聊天室、遷移紀錄、舊資料原文備份 |
| `summaries` | `contactId` | 歷史摘要（預留給 AI，目前不寫入） |
| `memories` | `[contactId, id]` | 長期記憶（預留給 AI，目前不寫入） |

- **每位 NPC 獨立**：訊息以 `[contactId, seq]` 為主鍵，每位 NPC 各占一段鍵範圍，`seq` 在該 NPC 內遞增。讀取、清除、匯出都只作用在自己的範圍。
- **不限數量**：不設上限、不自動刪除。
- **分頁載入**：開啟聊天室先讀最新 50 則（`PAGE`），往上捲動每次再讀 50 則，畫面停在原本的位置。
- **錯誤不忽略**：所有寫入都經過同一個出口，失敗時在輸入框上方顯示原因（例如儲存空間不足），提供「重試」與「匯出紀錄」，處理完才會消失；沒存成功的訊息會標示「未儲存」。IndexedDB 完全打不開時改用記憶體暫存，並明確提示這次的對話不會保存。

### ChatStore 介面（之後接 Supabase 等雲端資料庫）

換成雲端資料庫時，提供同名、同回傳格式的實作即可，介面不用改：

```
open()
getMeta(key) / putMeta(key, value)
getSettings(id) / putSettings(record)
addMessages(id, messages)            → 附上 seq 的訊息
latest(id, n) / before(id, seq, n)   → 由舊到新排序
count(id) / dumpContact(id)
clearContact(id, settings)
mergeContacts(plan)                  → 單一交易合併，重複略過，失敗整筆不寫入
getSummary(id) / putSummary(id, summary)
listMemories(id) / putMemory(id, memory)
```

### 舊版 localStorage 遷移

第一次開啟新版時自動執行：

1. 讀取舊的 `nightline-holo-v1`，**只讀不改、不刪**，並在 IndexedDB 另存一份原文（`meta` 的 `legacy.localStorage.v1.raw`）。
2. 用單一交易寫入全部訊息與設定。
3. 逐一核對每位 NPC 的筆數，全部吻合才標記完成。
4. 任何一步失敗都不會標記完成，下次開啟會重試；重複的訊息會被略過，不會寫兩次。

### 匯出／匯入

在「人物資料」最下方。

- 匯出：下載 `nightline-holo-YYYYMMDD-HHMM.json`，包含三位 NPC 的全部訊息與設定。
- 匯入：先確認再執行；與現有紀錄合併並依時間排序，重複略過，不會刪除現有紀錄。格式錯誤的檔案不會寫入任何資料。

```json
{
  "format": "nightline-holo-chat",
  "version": 1,
  "exportedAt": "2026-11-25T09:47:00.000Z",
  "active": "judy",
  "contacts": {
    "judy": { "name": "…", "settings": { "mode": "lover", "draft": "" },
              "messages": [{ "from": "n", "text": "…", "t": 1764000000000 }] }
  }
}
```

## AI Context（與完整紀錄分開）

完整聊天紀錄全部留在儲存層，送給 AI 的只有 `buildContext(id)` 組出來的內容：

```js
{
  persona,    // 角色人格（固定資料）
  recent,     // 近期對話：最新 AI_CONTEXT.recent（20）則，[{ role: "user" | "assistant", text, t }]
  summary,    // 歷史摘要：之後由真實 AI 產生後寫入 summaries；目前為 null
  memories    // 長期記憶：之後由真實 AI 擷取後寫入 memories；目前為 []
}
```

未串接真實 AI API 前，不產生、不寫入任何摘要或記憶，也不用假資料模擬。

## 之後接 AI 的位置

```js
// 約定：回傳 Promise<string[]>，或 Promise<{ messages: string[], choices?: string[] }>；失敗就丟出錯誤
replyProvider.reply({
  contactId,            // "judy" | "panam" | "johnny"
  text,                 // V 這次送出的訊息
  context               // buildContext(id) 的結果
})
```

- 目前使用 `MockProvider`，關鍵字與預設台詞集中在 `mockCompose()`；回覆選項由 `mockChoices()` 產生（目前只有茱蒂）。
- `choices` 最多三個，會顯示成可點選的回覆選項；使用者點選或自行輸入後就收起。
- 每個聊天室各自排隊，回覆一定寫回發問的聊天室；中途重設會讓舊回覆作廢。
- provider 丟出錯誤時，聊天室會顯示「訊號中斷」，不會寫入角色台詞。
