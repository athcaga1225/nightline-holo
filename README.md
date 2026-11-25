# NIGHTLINE // HOLO

非官方《電馭叛客 2077》NPC 私訊網站前端（Mock Mode）。以 V 的身分和茱蒂、帕娜、強尼傳訊息，回覆為預寫的模擬文字，尚未串接 AI。

- 前端為 `index.html`，角色資料與生活規則放在 `characters/`，用 GitHub Pages 部署
- 聊天紀錄存在瀏覽器的 IndexedDB，不限數量、不會自動刪除
- 本機測試請用 HTTP 伺服器開啟（例如 `python3 -m http.server`）；直接開 HTML 檔時瀏覽器會擋下角色規則檔，生活情境會停用，聊天仍可使用

本專案為粉絲作品，與 CD PROJEKT RED 無關。角色名稱權利屬原權利人，對話內容皆為原創。

## 角色資料（characters/judy/）

AI 用的角色資料拆成多個檔案，不寫成單一巨大 System Prompt。

| 檔案 | 用途 |
|---|---|
| `profile.md` | 人物背景（只收有依據的內容；待補充項目不得自行捏造） |
| `personality.md` | 固定人格（不得被關係、時間線、暫時狀態或記憶覆蓋） |
| `values.md` | 價值觀與處事原則 |
| `speech-style.md` | 清醒日常的簡訊語言習慣 |
| `relationships.json` | 普通朋友／信任的夥伴／戀人 |
| `emotional-response.md` | 不同情緒的表現方式 |
| `temporary-states.json` | 清醒、醉酒、宿醉（醉酒宿醉只能由事件觸發） |
| `canon-dialogues.json` | 原作簡訊的標記與索引（**不含原文**，見下方） |
| `dialogue-analysis.json` | 每組原作簡訊的人格分析 |
| `dialogue-examples.json` | 原創示範對話 |
| `negative-examples.json` | 錯誤範例與修正方向 |
| `terminology-zh-TW.json` | 官方繁中譯名與台灣用語規範 |
| `system-prompt.md` | 實際送給 AI 的角色指令 |
| `test-cases.json` | 人格驗收與回歸測試（尚未串接 AI，狀態為 `not_run`） |
| `sources.json` | 資料來源與校驗狀態 |
| `daily-routine.json` `location-rules.json` `activity-rules.json` `state-transitions.json` `proactive-message-rules.json` | 動態生活規則（本專案虛構延伸，非官方設定） |

### 原作簡訊原文

原作簡訊是官方遊戲文字，**不放進這個公開 repo**。`canon-dialogues.json` 只保留 id、關係、情境、情緒、暫時狀態、風格標記與是否作為預設風格等資訊。
原文請放在本機的 `characters/judy/private/canon-dialogues.local.json`（已列入 `.gitignore`，格式見 `sources.json`），之後由後端讀取；前端送給 AI 的只有相關語料的 id（`canon_refs`）。

## 即時時間與動態生活情境

- 使用裝置當地時間與時區（24 小時制）；故事年份與原作時間線另外管理，不會因現實時間推進原作劇情。
- 茱蒂的生活狀態（地點、活動、情緒、暫時身體狀態、計畫、進行中的工作）與生活事件存在 IndexedDB，依「角色＋故事時間線」分開；重新整理或關閉後沿用。
- 推進全由程式規則處理，不呼叫 AI；換地點一定經過移動；完成的事件不會重新開始。
- 關閉網站超過 12 小時後重新開啟：依目前時段重新對齊，不補算每一分鐘。
- 醉酒、宿醉只能由事件觸發：先有喝酒計畫 → 去喝酒（醉酒）→ 睡醒後才可能宿醉 → 一段時間後恢復。
- V 在茱蒂睡覺時傳訊息會把她叫醒，不需要配合她的作息。
- 主動訊息：依事件觸發，有冷卻（5 小時）與每日上限（2 則），可在人物資料開關。
  - `on_open`：重新開啟網站時產生，只送最新一則，時間戳記為實際產生時間
  - `while_open`：網站開著時依實際時間差檢查
  - 網站關閉時的背景訊息需要後端排程、雲端資料庫與 Web Push，GitHub Pages 無法執行，**尚未實作**
- 模擬模式的主動訊息與近況回覆是原創範本；接上真實 AI 後改由模型依 Dynamic Context 產生。

### 開發者 Debug Panel

網址加上 `#debug`（例如 `…/nightline-holo/#debug`）會出現 Debug Panel：可調整模擬時間（+1 小時、+6 小時、+1 天、+7 天、指定時間、回到實際時間）、切換故事時間線，並檢查目前地點、活動、情緒、暫時狀態、今日事件、主動訊息觸發紀錄，以及實際送入 AI 的 Dynamic Context。正式使用時不會顯示。

## 程式分層

`index.html` 的 script 依序分成七段，介面只透過下層提供的方法存取資料：

1. **角色人格資料（Persona）**：`personaOf(id, mode)`，整理成 AI 與介面共用的固定格式；模擬台詞不在這裡。
2. **儲存層（ChatStore）**：`IndexedDBStore()`，以及資料庫無法使用時的 `MemoryStore()` 備援。
3. **舊版 localStorage 遷移**：`migrateLegacy(store)`。
4. **動態生活情境**：`advanceLife()`、`dynamicContext()`、主動訊息 `handleTriggers()`。
4b. **AI Context**：`buildContext(id, text)`，與完整聊天紀錄分開。
5. **回覆來源**：`replyProvider`（目前是 `MockProvider`）。
6. **介面狀態與畫面**：分頁載入、送出、重設、匯出／匯入、錯誤提示。
7. **啟動流程**：`boot()`。

## 儲存架構（IndexedDB）

資料庫 `nightline-holo`，版本 2（舊版本開啟時自動升級，原有資料保留）：

| Object store | 主鍵 | 內容 |
|---|---|---|
| `messages` | `[contactId, seq]` | 全部聊天紀錄 `{ contactId, seq, from, text, t }` |
| `settings` | `id` | 每位 NPC 的設定：草稿、未讀、關係、待選回覆 |
| `meta` | `key` | 目前聊天室、遷移紀錄、舊資料原文備份 |
| `summaries` | `contactId` | 歷史摘要（預留給 AI，目前不寫入） |
| `memories` | `[contactId, id]` | 長期記憶（預留給 AI，目前不寫入） |
| `life` | `[contactId, timeline]` | 動態生活狀態 |
| `events` | `[contactId, timeline, id]` | 生活事件（進行中、待處理、已完成） |

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
  dynamic,    // 即時動態資料：時間、時區、距離上次互動、地點、活動、情緒、暫時狀態、今日事件、計畫、未完成工作
  canon_refs, // 與話題、狀態相關的原作語料 id（原文只在後端私人檔）
  recent,     // 近期對話：最新 AI_CONTEXT.recent（20）則，[{ role: "user" | "assistant", text, t }]
  summary,    // 歷史摘要：之後由真實 AI 產生後寫入 summaries；目前為 null
  memories    // 長期記憶：之後由真實 AI 擷取後寫入 memories；目前為 []
}
```

未串接真實 AI API 前，不產生、不寫入任何摘要或記憶，也不用假資料模擬。

## 之後接 AI 的位置

```js
// 約定：回傳 Promise<string[]>，或 Promise<{ messages: string[], choices?: string[] }>；失敗就丟出錯誤
replyProvider.proactive({ contactId, trigger, context })   // 主動訊息（選用），回傳 string[]
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
