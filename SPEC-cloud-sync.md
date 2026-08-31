# 節奏小星星 — 雲端記錄與後台 規格書（v3）

> 交付對象：Claude Code（在本機 repo `rhythm-star/` 執行）
> 產出：改造後的 `index.html` ＋ 新增 `admin.html`
> 線上位置：https://mimicz.github.io/rhythm-star/
> v3：2026-08-31 安全強化（管理員改用 Firebase Authentication）。變更見文末「修訂紀錄」。
> v1 原稿備份於 `SPEC-cloud-sync_AI.md`。

---

## 0. 給 Claude Code 的起手 prompt（直接複製貼上）

```
請閱讀 SPEC-cloud-sync.md，照規格改造這個 repo。

重點要求：
1. 先完整讀過現有 index.html，理解它的音訊解鎖機制、混合時鐘（nowT/clkOff）、
   判定與計分流程。這些既有邏輯不可破壞，改動要以「疊加」為原則。
2. 雲端功能必須是「錦上添花」：DB_URL/API_KEY 未設定、沒網路、Firebase 回錯、
   匿名登入失敗，遊戲都要能照常玩，且畫面上要看得出目前同步狀態。
3. 不要引入任何前端框架或 Firebase SDK（Auth 也走 REST），用 fetch 即可，
   維持零相依、單檔可離線開啟的特性。
4. 每完成一個階段就用瀏覽器實測（規格書第 8 節有測試清單），
   包含「斷網」與「DB_URL 留空」兩種情境。
5. 改動前先把 index.html 備份成 index_AI.html（僅留本機，不 commit）。
```

---

## 1. 需求背景

| 項目 | 決定 |
|---|---|
| 使用情境 | 親戚之間偶爾玩，人數個位數 |
| 玩家身分 | **不需要登入**，輸入暱稱即為身分（背景自動匿名登入） |
| 管理員身分 | **Firebase Email/Password 登入**，密碼不進原始碼（見 §2.1） |
| 防偽造 | 不需要（可竄改，接受） |
| 防破壞 | **需要** — 刪除權以 `auth.uid` 由 Rules 強制（見 §2.3） |
| 跨裝置 | **需要** — 同一個暱稱在不同裝置上要能接續紀錄 |
| 個資 | 保守處理：引導使用暱稱、後台可清空 |
| 託管環境 | GitHub Pages（純靜態，**無法跑後端程式**） |

---

## 2. 技術選型與理由

**採用 Firebase Realtime Database（Spark 免費方案）+ REST API + 匿名登入（REST）。**

選它而不是 Supabase 的關鍵原因：
Supabase 免費專案在**連續 7 天低活動後會自動暫停**，暫停後 API 即失效，
需要有人登入後台手動喚醒。本案是「想到才玩一次」的家庭情境，極易觸發。
Firebase Spark 方案無此閒置暫停機制，且不需信用卡、1 GB 儲存、
10 GB/月下載、100 個同時連線，對本案綽綽有餘。

同時 RTDB 提供純 REST 端點，可用原生 `fetch` 存取，
**不需要載入任何 SDK**，維持現有單檔零相依架構。

### 2.1 兩種身分：玩家（匿名）與管理員（Email/Password）

| 身分 | 取得方式 | 讀 | 新增／更新 | 刪除 |
|---|---|---|---|---|
| 玩家 | 匿名登入（自動） | ✅ | ✅ | ❌ |
| 管理員 | Email/Password 登入（人工輸入密碼） | ✅ | ✅ | ✅ |
| 無 token | — | ❌ | ❌ | ❌ |

**管理員密碼不在原始碼裡**，存於 Firebase Authentication（scrypt 雜湊、伺服器端驗證）。
刪除權限由規則以 `auth.uid === '<管理員 UID>'` 強制，前端改不動。

### 2.2 API_KEY / DB_URL 的正確認知

`API_KEY` 與 `DB_URL` **本來就是公開識別碼，不是密鑰** —— Firebase 官方明示 Web API Key
可公開；純靜態站也不可能藏住（一定會進到瀏覽器）。它們本身不授予任何資料存取權。

防護分三層，唯一真正的防線是第一層：

| 層 | 機制 | 強度 |
|---|---|---|
| 1 | **Security Rules**（`auth.uid` 判斷刪除權） | 伺服器端強制，繞不過 |
| 2 | **API key 限 HTTP referrer**（`mimicz.github.io`、`localhost:8123`） | 減速丘：`curl` 可偽造 Referer |
| 3 | App Check（reCAPTCHA） | 公開網頁 app 的正解 —— **本案未做** |

**仍存在的已知風險**（家庭用途接受）：持有 API_KEY 者可取得匿名 token，
因而能**讀取**全部成績，也能**竄改**（例如把分數改成別的合法數值）。
擋掉的是**破壞**（刪除），沒擋掉的是**偽造**，這與 §1「不需防偽造」的前提一致。

### 2.3 ⚠️ RTDB 規則的關鍵陷阱（實作時踩過）

**`.write` 權限由上往下繼承，且下層無法收回。**
若把 `".write": "auth != null && (newData.exists() || auth.uid === ADMIN)"` 放在
`players/$pid`，刪除 `players/$pid/rec/xxx` 時 `$pid` 節點依然存在 →
`newData.exists()` 為真 → **匿名使用者仍可刪掉個別成績**。

正確寫法是把兩種權限拆到不同層級：

- `players/$pid`（父）：`".write": "auth.uid === '<管理員UID>'"` — 只有管理員能刪整個節點
- `nick`／`unlocked`／`plays`／`updated`／`rec/$k`（葉）：`".write": "auth != null && newData.exists()"`
  — 玩家能新增／更新，但 `newData` 不存在（＝刪除）時一律拒絕

遊戲的多路徑 PATCH 會逐葉節點評估，因此正常寫入不受影響。

### 2.4 使用者需先完成的設定（人工，約 10 分鐘）

1. 前往 <https://console.firebase.google.com> → **建立專案**（可關閉 Google Analytics）
2. 左側 **建構 → Realtime Database → 建立資料庫**
   - 位置建議 `asia-southeast1`（新加坡，離台灣最近）
   - 安全性規則選「**鎖定模式**」，稍後貼上第 5 節正式規則覆蓋
3. 左側 **建構 → Authentication → 開始使用 → Sign-in method** →
   啟用「**匿名**」（給玩家）與「**電子郵件/密碼**」（給管理員）；
   再到 **Users 分頁 → Add user** 建立管理員帳號（密碼自訂，不會出現在任何原始碼中），
   並記下該使用者的 **UID**，填入第 5 節規則的 `<管理員UID>`
4. 左上齒輪 → **專案設定 → 一般** → 「你的應用程式」→ **新增網頁應用程式（`</>` 圖示）**
   （專案剛建立時沒有任何 app，**要先註冊 Web app 才會產生 API 金鑰**；
   不勾 Firebase Hosting）→ 從顯示的 `firebaseConfig` 複製 `apiKey` → 貼進兩個 HTML 的 `API_KEY`
5. 複製資料庫網址（格式類似
   `https://你的專案-default-rtdb.asia-southeast1.firebasedatabase.app`）
   → 貼進兩個 HTML 的 `DB_URL`
6. 回到 **Realtime Database → 規則** 分頁，貼上第 5 節的規則 JSON 後**發布**

---

## 3. 資料模型

Firebase key 不得含 `. $ # [ ] /`，中文可用。

```
/v1
  /players
    /{pid}
      nick      : string   顯示用原始暱稱（保留大小寫與空白）
      unlocked  : number   已解鎖到第幾首（1..5）
      plays     : number   累計完成局數
      updated   : number   伺服器時間戳
      /rec
        /{songId}_{diff}   例：twinkle_1
          score : number
          rate  : string   'S'|'A'|'B'|'C'|'D'
          combo : number
          acc   : number   0..100
          at    : number   達成時間戳
  /board
    /{songId}
      /{pid}
        nick : string
        s    : number  該玩家此曲最佳分數
        r    : string  評價
        d    : string  難度名稱
        at   : number
```

**設計要點**：`board` 每位玩家每首歌只留一筆最佳成績，
資料量不會隨遊玩次數成長，也不需要分頁或清理。

### 3.1 pid（玩家鍵值）產生規則

```js
// 暱稱即身分。同暱稱視為同一人，紀錄以「取最高分」合併。
function nickKey(nick){
  return String(nick || '')
    .trim()
    .replace(/[.$#\[\]\/\x00-\x1F\x7F]/g, '_')  // Firebase 非法字元
    .replace(/\s+/g, '_')
    .slice(0, 24)
    .toLowerCase() || 'guest';
}
```

撞名的處理方式是**合併而非覆蓋**，所以最壞情況只是兩人共用一份紀錄，
不會出現「誰的成績被洗掉」。這符合使用者「不用防偽造」的要求。

---

## 4. `index.html` 改造規格

### 4.1 設定區塊（放在 `<script>` 最頂端，要顯眼）

```js
/* ===================== 雲端設定 =====================
   貼上 Firebase 專案的兩個值即可啟用雲端記錄。
   任一留空 '' → 自動退回純本機模式，遊戲功能完全不受影響。
   ==================================================== */
const DB_URL  = '';  // 例：'https://xxx-default-rtdb.asia-southeast1.firebasedatabase.app'
const API_KEY = '';  // Firebase 專案設定 → 一般 → Web API 金鑰
const DB_ROOT = 'v1';
const CLOUD_ON = /^https:\/\/.+/.test(DB_URL) && API_KEY.length > 10;
```

### 4.2 匿名登入層（新增，約 50 行）

```js
/* 匿名登入（REST，不載 SDK）
   localStorage key: rhythmStar_auth = { idToken, refreshToken, exp }
   - 首次取得：POST https://identitytoolkit.googleapis.com/v1/accounts:signUp?key=API_KEY
               body: {"returnSecureToken": true}
   - 換發：    POST https://securetoken.googleapis.com/v1/token?key=API_KEY
               body: grant_type=refresh_token&refresh_token=...
               （Content-Type: application/x-www-form-urlencoded）
   - idToken 有效 1 小時；剩不到 5 分鐘就主動換發
   - RTDB 回 401/403 → 換發一次並重試 → 仍失敗 → 視同離線（遊戲照玩）
   - refreshToken 失效（帳號被停用等）→ 重新 signUp 取新匿名帳號
*/
async function getToken()  // 回傳有效 idToken；必要時自動 signUp/refresh；失敗回 null
```

### 4.3 網路層（新增，約 60 行）

- 一律用 `fetch`，包 `AbortController` **6 秒逾時**
- 所有呼叫用 `try/catch` 包住，**任何失敗都不得往上拋進遊戲迴圈**
- 每個請求先 `getToken()`，URL 加上 `?auth=<idToken>`；拿不到 token 視同離線
- 提供四個函式：

```js
async function dbGet(path)            // GET    /v1/{path}.json?auth=...
async function dbPatch(path, obj)     // PATCH  合併寫入
async function dbPut(path, obj)       // PUT    覆蓋寫入
async function dbDel(path)            // DELETE
```

- 時間戳一律用 Firebase 伺服器值：`{ ".sv": "timestamp" }`
- 失敗時更新同步狀態並把「遊玩結果事件」丟進重試佇列（見 4.6）

### 4.4 合併邏輯（核心，務必正確）

```js
// 回傳合併後的 store。規則：分數高者勝；解鎖進度取大值。
function mergeStore(localS, cloudP){
  const out = { ...localS, rec: { ...localS.rec } };
  if (!cloudP) return out;
  out.unlocked = Math.max(out.unlocked || 1, cloudP.unlocked || 1);
  out.plays    = Math.max(out.plays || 0,    cloudP.plays || 0);
  for (const k in (cloudP.rec || {})){
    const c = cloudP.rec[k], l = out.rec[k];
    if (!l || (c.score || 0) > (l.score || 0)) out.rec[k] = c;
  }
  return out;
}
```

⚠️ **注意**：現有本機 `ST.rec` 的 key 已經是 `songId + '_' + diff`，
與雲端格式一致，不需轉換。`ST.board` 是本機排行榜，**保留不動**，
雲端排行榜另外存一份，兩者在 UI 上可切換。

已知取捨：`plays` 用 `Math.max` 合併，跨裝置各自的局數不會相加，
累計值會偏低。家庭用途接受，不做加總語意。

### 4.5 同步觸發時機與 push 流程

| 時機 | 動作 |
|---|---|
| 頁面載入且暱稱已存在 | `pullPlayer()` → 合併 → 更新選單顯示 |
| 使用者改完暱稱（`blur` 或按「開始遊戲」） | `pullPlayer()` → 合併 |
| `finish()` 結算完成且**非練習模式** | `pushResult()` 背景上傳 |
| 進入排行榜畫面 | `pullBoard(songId)`，失敗則顯示本機榜並提示 |
| 重試佇列有資料且網路恢復 | 自動 flush（重跑完整 push 流程） |

**`pushResult()` 鐵律：push 前必先 pull 成功。**
拉不到雲端現狀就不寫（改進佇列），避免用過期的本機快照倒退雲端紀錄。

流程：
1. `pullPlayer()` 拿雲端現狀 → `mergeStore()` 合併
2. **多路徑 PATCH**（body 的 key 用斜線路徑，Firebase 逐路徑寫入，
   不會像 `rec: {...}` 那樣把整個 `rec` 節點換掉）：

```js
const body = { nick, unlocked: m.unlocked, plays: m.plays, updated: {'.sv':'timestamp'} };
for (const k in m.rec) body['rec/' + k] = m.rec[k];
await dbPatch('players/' + pid, body);
```

3. `board` 寫入判斷**不需額外讀雲端**：`board/{songId}/{pid}` 是每人每曲一筆，
   合併後 `m.rec` 中該曲各難度的最高分就是自己的雲端最佳。
   若本次分數 = 該最高分（即本次創新高）→
   `dbPatch('board/' + songId + '/' + pid, { nick, s, r, d, at: SV })`

### 4.6 離線重試佇列（存事件，不存原始請求）

```js
// localStorage key: rhythmStar_pending
// 結構：[{ song:'twinkle', diff:1, score:1234, rate:'A', combo:56, acc:87 }, ...]
// 上限 50 筆，超過丟棄最舊的。
```

- 佇列存的是「遊玩結果事件」；flush 時把事件先套進本機 `ST`（其實結算當下已套過），
  然後**重跑一次完整的 `pushResult()` 流程（pull → merge → 多路徑 push）**，
  成功後才移除該批事件。
- **絕不重播原始 HTTP 請求** — 離線當下的快照不含其他裝置的紀錄，
  直接重播會把別台的高分洗掉。
- 觸發 flush：頁面載入、`window 'online'` 事件、每次成功的網路呼叫之後。

### 4.7 UI 變更

**首頁**
- 暱稱欄位 placeholder 改為 `輸入暱稱（建議用綽號，不要用真實全名）`
- 暱稱欄下方新增說明小字：`同一個暱稱在不同裝置上會共用紀錄`
- 新增雲端狀態列（沿用現有 `.badge` 樣式）：

| 狀態 | 顯示文字 |
|---|---|
| `CLOUD_ON === false` | `📱 本機模式（成績只存在這台裝置）` |
| 同步中 | `☁️ 同步中…` |
| 成功 | `☁️ 已同步（最後更新 HH:MM）` |
| 失敗／離線／登入失敗 | `📴 離線中，成績已暫存，連線後自動上傳` |

**排行榜畫面**
- 頂端加兩顆切換鈕：`🌏 雲端排行` / `📱 這台裝置`
- 雲端榜每首歌顯示前 10 名（客戶端排序，資料量小不需 `orderBy`）
- 雲端榜載入失敗時自動退回本機榜並顯示提示列

**結算畫面**
- 若為新的雲端最佳成績，加一列 `☁️ 已上傳雲端排行`

### 4.8 不可破壞的既有邏輯（回歸重點）

改動時請特別保護以下三處，它們是先前修 bug 的成果：

1. **混合時鐘** `nowT()` / `clkOff` / `clkSrc`
   音訊被擋住時 `AudioContext.currentTime` 會凍結，必須自動切換到
   `performance.now()` 並做偏移補償。排程時 `at = G.t0 + e.t - clkOff` 的
   時間軸轉換不可漏掉。
2. **音訊解鎖機制** `ensureAudio()` / 全域 pointerdown/keydown 解鎖 /
   無聲 buffer 解鎖 / `audioOverlay` 救援畫面。
3. **判定窗** `W_PERFECT = 0.125` / `W_GOOD = 0.235` — 刻意放寬給兒童用，不要改。

---

## 5. Firebase 安全規則（貼到「規則」分頁）

策略：**玩家（匿名）可讀可寫但不可刪；管理員（Email/Password）才有刪除權。**
關鍵是把兩種權限拆到不同層級 —— 原因見 §2.3 的繼承陷阱。

把 `<ADMIN_UID>` 換成 Authentication → Users 中管理員帳號的 UID。

```json
{
  "rules": {
    ".read": false,
    ".write": false,
    "v1": {
      "players": {
        ".read": "auth != null",
        "$pid": {
          ".write": "auth.uid === '<ADMIN_UID>'",
          ".validate": "$pid.length <= 48 && newData.hasChildren(['nick'])",
          "nick":     { ".write": "auth != null && newData.exists()", ".validate": "newData.isString() && newData.val().length <= 16" },
          "unlocked": { ".write": "auth != null && newData.exists()", ".validate": "newData.isNumber() && newData.val() >= 1 && newData.val() <= 50" },
          "plays":    { ".write": "auth != null && newData.exists()", ".validate": "newData.isNumber() && newData.val() >= 0 && newData.val() <= 100000" },
          "updated":  { ".write": "auth != null && newData.exists()", ".validate": "newData.isNumber()" },
          "rec": {
            "$k": {
              ".write": "auth != null && newData.exists()",
              ".validate": "newData.hasChildren(['score'])",
              "score": { ".validate": "newData.isNumber() && newData.val() >= 0 && newData.val() <= 1000000" },
              "rate":  { ".validate": "newData.isString() && newData.val().length <= 2" },
              "combo": { ".validate": "newData.isNumber() && newData.val() >= 0 && newData.val() <= 100000" },
              "acc":   { ".validate": "newData.isNumber() && newData.val() >= 0 && newData.val() <= 100" },
              "at":    { ".validate": "newData.isNumber()" },
              "$other": { ".validate": false }
            }
          },
          "$other": { ".validate": false }
        }
      },
      "board": {
        ".read": "auth != null",
        "$song": {
          "$pid": {
            ".write": "auth.uid === '<ADMIN_UID>'",
            ".validate": "newData.hasChildren(['nick','s'])",
            "nick": { ".write": "auth != null && newData.exists()", ".validate": "newData.isString() && newData.val().length <= 16" },
            "s":    { ".write": "auth != null && newData.exists()", ".validate": "newData.isNumber() && newData.val() >= 0 && newData.val() <= 1000000" },
            "r":    { ".write": "auth != null && newData.exists()", ".validate": "newData.isString() && newData.val().length <= 2" },
            "d":    { ".write": "auth != null && newData.exists()", ".validate": "newData.isString() && newData.val().length <= 8" },
            "at":   { ".write": "auth != null && newData.exists()", ".validate": "newData.isNumber()" },
            "$other": { ".validate": false }
          }
        }
      }
    }
  }
}
```

> **後台實作必讀**：讀寫權限只開在 `players`／`board` 以下，
> **`GET /v1.json` 與 `DELETE /v1.json` 都會被拒絕**（權限不會往上層生效）。
> 後台的「重新整理」與「清空」必須逐節點操作，見 §6.3。

## 6. `admin.html` 後台規格（新檔案）

### 6.1 存取控制（Firebase Authentication）

- **不使用寫死的通行碼**。進站顯示 Email／密碼登入表單，
  送 `accounts:signInWithPassword` 給 Firebase 驗證，換回 idToken。
- token 存 `localStorage`，鍵名 **`rhythmStar_adminAuth`** —— 必須與遊戲頁的
  `rhythmStar_auth` 分開，否則會蓋掉玩家的匿名 token。
- 密碼用完立即從 DOM 清除，不寫入任何儲存。
- 後台的 `getToken()` **不做匿名註冊**：沒有可用 refreshToken 就退回登入畫面。
- 登入後才顯示資料；`reload()` 開頭若取不到 token 即自動登出。
- 若 `DB_URL`／`API_KEY` 為空，顯示設定指引而非空白畫面。

### 6.2 版面（單頁，三個分頁籤）

**① 玩家總覽**

表格欄位：暱稱 / 累計局數 / 解鎖進度 / 各曲最佳分數（5 欄）/ 總分 / 最後遊玩時間

- 可依任一欄排序
- 上方搜尋框過濾暱稱
- 每列尾端有「🗑 刪除此玩家」

**② 各曲排行榜**

依曲目分區，每區列出所有玩家的最佳成績（名次、暱稱、分數、評價、難度、日期）

**③ 統計**

統計卡片：總玩家數 / 總遊玩局數 / 最熱門曲目 / 最高分紀錄保持者 /
各難度使用比例（簡單長條圖即可，不要引入圖表庫，用 CSS div 寬度就好）

### 6.3 功能

| 功能 | 實作 |
|---|---|
| 重新整理 | `GET /v1/players.json` ＋ `GET /v1/board.json`（規則不允許 `GET /v1.json`） |
| 匯出 CSV | **UTF-8 with BOM**（`﻿` 開頭），Excel 開啟不亂碼 |
| 刪除單一玩家 | `DELETE /v1/players/{pid}.json` ＋ 逐曲 `DELETE /v1/board/{song}/{pid}.json` |
| 清空所有雲端資料 | 規則不允許 `DELETE /v1.json` → 先讀出玩家與 board 清單，**逐玩家、逐曲逐 pid 刪除**；執行前需輸入 `DELETE` 確認 |

CSV 匯出注意事項：
- 欄位含逗號或引號時要用雙引號包住並跳脫
- 檔名帶日期：`rhythm-star-成績-YYYYMMDD.csv`
- 用 `Blob` + `URL.createObjectURL` + `<a download>` 觸發下載

### 6.4 樣式

沿用 `index.html` 的配色變數與 `.btn` / `.panel` / `.badge` 樣式，
但版面改為**資訊密度較高的桌面取向**（表格為主），並保持 RWD 可在手機上看。

---

## 7. 檔案與部署

```
rhythm-star/
├── index.html          # 遊戲（改造）
├── admin.html          # 後台（新增）
├── SPEC-cloud-sync.md
└── .gitignore          # 新增：*_AI.* 備份檔不 commit
```

- 備份檔（`index_AI.html`、`SPEC-cloud-sync_AI.md`）**僅留本機**，
  不 commit — 否則會跟著發布到 Pages 網站上
- 後台網址：`https://mimicz.github.io/rhythm-star/admin.html`
- **不要**從遊戲畫面連到後台，避免小孩點進去
- 兩個檔案的 `DB_URL`／`API_KEY` 必須一致；`admin.html` 載入時若偵測到
  未設定，要顯示提示

---

## 8. 驗收測試清單（用內建瀏覽器實測）

### 必測情境

| # | 情境 | 預期結果 |
|---|---|---|
| 1 | `DB_URL = ''` | 遊戲完全正常，狀態列顯示「本機模式」，無任何 console error |
| 2 | 設定正確、網路正常 | 完成一局後，Firebase 出現 `players` 與 `board` 資料 |
| 3 | 攔截 fetch 使其全部失敗（含 auth 端點） | 遊戲照常可玩可結算，狀態列顯示離線，`pending` 佇列有資料 |
| 4 | 情境 3 後恢復網路並重載 | 佇列自動 flush（重跑 pull→merge→push），資料上傳成功 |
| 5 | 裝置 A 玩完 → 裝置 B 輸入同暱稱 | B 讀到 A 的最高分與解鎖進度 |
| 6 | B 的分數較低（含 B 離線暫存後才恢復連線） | 合併後**保留 A 的高分**，不被覆蓋 |
| 7 | 練習模式完成一局 | **不上傳**任何雲端資料 |
| 8 | 暱稱含空白、`.`、`/`、emoji | `nickKey()` 正確轉換，Firebase 不報錯 |
| 9 | 後台通行碼錯誤 | 不顯示任何資料 |
| 10 | 後台匯出 CSV | 用 Excel 開啟中文不亂碼 |
| 11 | 音訊被擋住（stub `AudioContext.state = 'suspended'`） | 遊戲仍能跑完並結算（回歸測試） |
| 12 | 暫停 → 等 2 秒 → 繼續 | 時間軸不跳動（漂移 < 0.4 秒，回歸測試） |
| 13 | `API_KEY` 錯誤或匿名登入未啟用 | 降級為離線表現，狀態列有提示，無未捕捉錯誤 |
| 14 | `idToken` 過期（模擬首次請求回 401） | 自動換發 token 後重試成功 |

### 測試技巧（在頁面 console 執行）

```js
// 情境 3：模擬全部網路失敗（測完把 rf 還原）
const rf = window.fetch;
window.fetch = () => Promise.reject(new TypeError('offline'));
// 恢復：window.fetch = rf;

// 情境 11：模擬音訊被擋（要在第一次點擊畫面「之前」執行，
// 因為 actx 是第一次使用者操作時才建立）
const Real = window.AudioContext;
window.AudioContext = function(){
  const c = new Real();
  Object.defineProperty(c, 'state', { get: () => 'suspended' });
  c.resume = () => Promise.resolve();
  return c;
};
```

---

## 9. 實作順序建議

1. 備份 `index.html` → `index_AI.html`（本機），新增 `.gitignore`（`*_AI.*`）
2. 加入設定區塊 + 匿名登入層 + 網路層，**先不接 UI**，用 console 驗證
   「取 token → 讀寫 RTDB」全通（此時使用者需已完成 §2.2 設定並發布規則）
3. 加入 `mergeStore()` 與 `pullPlayer()` / `pushResult()`（含「push 前必先 pull 成功」鐵律）
4. 接上 UI：狀態列、排行榜切換、結算畫面提示
5. 加入離線重試佇列（事件制）
6. 跑測試 1、3、4、7、11、12、13（遊戲端全數通過再往下）
7. 寫 `admin.html`
8. 跑測試 2、5、6、8、9、10、14
9. 更新 `README.md`，補上 Firebase 設定章節

---

## 10. 明確不要做的事

- ❌ 不要引入 Firebase SDK（**Auth 也走 REST**）、React、Vue 或任何前端框架
- ❌ 不要把 `ST.board`（本機排行榜）拿掉，它是離線時的後備
- ❌ 不要為了雲端功能而讓遊戲在無網路時無法開始
- ❌ 不要改動判定窗、音訊解鎖、混合時鐘這三塊既有邏輯
- ❌ 不要在遊戲畫面放後台入口連結
- ❌ 不要把 `*_AI.*` 備份檔 commit 上 repo
- ❌ 不要未經確認就 `git push`

---

## 修訂紀錄

**v3（2026-08-31，安全強化）**

1. **管理員改用 Firebase Authentication（Email/Password）**，移除寫死的 `ADMIN_CODE`。
   起因：使用者發現通行碼會隨公開原始碼外洩。釐清後確認 —— 任何純前端檢查都只是裝飾
   （攻擊者連密碼都不用知道，直接在主控台呼叫 `boot()` 即可跳過）。
2. **Rules 加入 `auth.uid` 刪除保護**：玩家可寫不可刪，只有管理員 UID 能刪。
3. **記錄 RTDB 權限繼承陷阱（§2.3）** —— 第一版規則實測時發現匿名仍可刪個別成績，
   必須把父／葉節點權限拆開才真正堵住。
4. **釐清 API_KEY／DB_URL 的定位（§2.2）**：公開識別碼而非密鑰，藏不住也不需藏；
   防護靠 Rules，並加上 API key referrer 限制作為第二層。
5. 後台 token 改用獨立儲存鍵 `rhythmStar_adminAuth`，避免蓋掉遊戲頁的匿名 token。

**v2（2026-08-31，Claude Code design review 後修訂）**

1. **新增匿名登入**（使用者決定）：Auth 走 REST 不載 SDK，規則改 `auth != null`；
   並於 §2.1 誠實註記防護界線 — API key 公開，此為門檻非安全機制。
2. **修正規則與後台 API 的矛盾**：v1 規則只在 `players`／`board` 層開權限，
   但後台卻要 `GET /v1.json`、`DELETE /v1.json`，會被規則直接拒絕。
   改為逐節點讀取／刪除（§5 註記、§6.3）。
3. **修正離線佇列的資料倒退 bug**：v1 佇列重播原始 HTTP 請求，且 `pushResult`
   PATCH 整包 `rec`（PATCH 只合併頂層 key，`rec` 會被整個換掉）——
   裝置 B 離線快照 flush 時會把裝置 A 的紀錄洗掉，違反驗收 #6。
   改為：佇列存遊玩結果事件、flush 重跑 pull→merge→push、
   push 改多路徑 PATCH、「push 前必先 pull 成功」鐵律（§4.5、§4.6）。
4. **簡化 board 寫入判斷**：不需額外讀雲端，由合併後 `rec` 即可得知自己是否創新高。
5. 小修正：`上架說明.md` → `README.md`（§9）；備份檔加入 `.gitignore` 不 commit（§7）；
   驗收測試改用內建瀏覽器實測，並新增 #13、#14 auth 情境；
   明示 `plays` 用 max 合併的低估取捨（§4.4）。

**v1（cowork 原稿）**：見 `SPEC-cloud-sync_AI.md`。
