# NIGHTLINE 前端 ↔ 私人後端 API

公開前端只呼叫以下端點；後端只回傳「角色要傳的簡訊」，不回傳人格、Prompt、原作語料、規則或內部狀態。
後端網址與存取碼由使用者在 `#debug` 面板輸入，只存在使用者自己的瀏覽器。

驗證：`Authorization: Bearer <存取碼>`；後端只接受允許清單內的來源（CORS）。

## GET /api/health
`{ ok, characters: ["judy", …], provider, devMode }`

## POST /api/reply
```json
{ "contactId": "judy", "text": "V 的訊息",
  "settings": { "relationship": "friend|lover", "timeline": "working_at_lizzie", "proactive": true },
  "client": { "tz": "Asia/Taipei" },
  "recent": [{ "role": "user|assistant", "text": "…", "t": 0 }] }
```
回應：`{ "messages": ["…", "…"], "choices": null | ["…", "…", "…"], "meta"?: { "provider", "model", "characterVersion", "requestId", "elapsedMs", "usage" } }`

- `messages` 是 NPC 本輪的完整回覆；`choices` 是 **V 對這些剛生成訊息的回應**。建議同一次生成使用同一份近期對話產出兩者。分兩次生成時，第二次必須加入第一步的全部 NPC 訊息。
- 不足三個貼切選項時回傳 `choices: null` 或省略，前端保留自由輸入，不補通用選項。
- 點選選項會以 V 新訊息走同一個 `/api/reply` 流程；自由輸入相同。`recent` 通常已包含本次 `text`，後端不要重複附加最後一則。
- `meta` 選用，只含模型／用量／耗時等資訊，不可包含 prompt、原作語料或憑證。前端目前不將 `meta` 寫入聊天匯出。
- 後端錯誤或逾時不降級成 Mock；前端保留已送出的 V 訊息並顯示訊號中斷。選項可以為空，不代表回覆失敗。
- 本機最小後端預設使用明確標記的 Mock；實際 AI、人格與憑證留在公開 repo 外，不能從前端介面存在推斷後端已完成。

## POST /api/sync
重新開啟網站（`phase: "on_open"`）或網站開著時（`"while_open"`）呼叫，取得可能的主動訊息。
```json
{ "contactId": "judy", "settings": { … }, "client": { "tz": "Asia/Taipei" }, "phase": "on_open" }
```
回應：`{ "messages": [{ "text", "src": "proactive", "trigger", "phase", "genAt", "eventAt", "t" }] }`

## 開發模式（後端 `NIGHTLINE_DEV_MODE=1` 才開放）
- `GET /api/debug/state?contactId=judy&tz=…` → 時鐘、Dynamic Context、近期生活事件、主動訊息紀錄
- `POST /api/debug/clock { "offsetMs": 0 }` → 模擬時間

資料結構見 `schemas/`。
