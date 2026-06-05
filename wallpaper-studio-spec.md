# 壁紙工作室 — 完整規格文件

> 單一 HTML 檔案，無框架，vanilla HTML/CSS/JS

---

## 技術架構

- **單檔**：一個 `wallpaper-studio.html`，CSS / JS 全部 inline
- **無框架**：純 vanilla JS，不依賴任何 library
- **字體**：Google Fonts CDN（Noto Serif TC、Noto Sans TC、Jf Open Huninn、DM Serif Display、DM Mono、Playfair Display、Cormorant Garamond、Lora、Raleway、Josefin Sans、Cinzel）
- **儲存**：localStorage（key prefix: `wps4_`）
- **Canvas**：preview canvas 乘以 DPR 保持清晰，export canvas 用實際像素

---

## Layout

### 桌機版（視窗寬度 > 700px）

```
┌─────────────────┬──────────────────────────────┐
│  Sidebar 320px  │        Preview Area           │
│  ┌───────────┐  │   [↩] [↪] [⟲橫向]            │
│  │ Tab Nav   │  │                               │
│  ├───────────┤  │      ┌──────────────┐         │
│  │           │  │      │  Phone Frame │         │
│  │  Panel    │  │      │   Canvas     │         │
│  │ (scroll)  │  │      └──────────────┘         │
│  │           │  │      preview meta text        │
│  └───────────┘  │                               │
│                 │   [⬇ 下載桌布]                 │
└─────────────────┴──────────────────────────────┘
```

- Sidebar 固定 320px，內部 panel 可獨立滾動
- 預覽區 flex:1，下載按鈕在預覽正下方
- undo bar 右上角：↩ ↪ ⟲橫向（無下載按鈕）

### 手機版（視窗寬度 ≤ 700px）

```
┌─────────────────────────┐
│       Preview Area      │
│  [↩][↪][⟲橫向][⬇下載]  │
│   ┌──────────────────┐  │
│   │   Phone Frame    │  │
│   └──────────────────┘  │
│     preview meta text   │
├─────────────────────────┤
│  ▔▔▔▔ (handle bar)      │
│  [文字][背景][排版][匯出][歷史] │
│                         │
│   Panel content (scroll)│
│                         │
└─────────────────────────┘
```

- 底部抽屜預設收起（高度 50px，只顯示 handle + tab nav）
- 抽屜開啟高度 = `window.innerHeight * 0.46`
- 抽屜動畫結束後重新計算預覽尺寸
- undo bar 含下載按鈕（桌機版隱藏）

---

## 預覽尺寸計算

```js
function computeSize() {
  // 根據選定的 model 計算正確比例
  const m = getSelModel();
  const AR = S.land ? m.w / m.h : m.w / m.h; // 寬/高

  // 可用空間
  // 桌機：previewArea.getBoundingClientRect()
  // 手機：扣掉抽屜高度

  // 在可用空間內最大化，保持比例
  // 限制：最小 100×160，最大 380×800
}
```

- canvas.width = PW \* DPR（高清）
- canvas.style.width = PW + ‘px’（CSS 尺寸不變）
- 繪製前 ctx.scale(DPR, DPR)
- 視窗 resize 時 debounce 150ms 重新計算
- `document.fonts.ready` + `requestAnimationFrame` 確保初始尺寸正確

---

## State 結構

```js
const S = {
  // 文字
  text: '每一天，都是重新開始的機會。',
  font: "'Noto Serif TC',serif",
  fw: '400',           // fontWeight: 300 / 400 / 700
  fs: 32,              // fontSize px
  tc: '#f2ead8',       // textColor
  to: 95,              // textOpacity 20–100
  lh: 1.65,            // lineHeight 1.0–3.0
  ta: 'center',        // textAlign: left / center / right
  shOn: true,          // shadow on
  shAmt: 15,           // shadowBlur 0–40
  bgOn: false,         // text background block on
  bgCol: '#000000',    // text bg color
  bgOp: 40,            // text bg opacity 5–95

  // 背景
  bgs: 'solid',        // solid / gradient / mesh / noise / photo
  sc: '#1c1814',       // solid color
  g1: '#1c1814',       // gradient start
  g2: '#3d2b1a',       // gradient end
  gd: 'to bottom',     // gradient direction
  mc: [...],           // mesh colors [4 hex]
  nc: '#1c1814',       // noise base color
  na: 20,              // noise amount 5–60

  // 圖片
  blur: 0,             // blur 0–20
  dim: 30,             // dim 0–80
  ix: 0,               // image x offset (ratio)
  iy: 0,               // image y offset (ratio)
  iscale: 1,           // image scale 0.3–5

  // 排版
  tx: 0.5,             // text x position (ratio)
  ty: 0.65,            // text y position (ratio)
  smart: true,         // smart layout on
  land: false,         // landscape mode

  // 其他
  isComp: false,       // using compressed image
  selModel: 0,         // selected export model index
}
```

---

## 5 個 Tab

### 1. 文字 Tab

- textarea（150 字上限，顯示剩餘字數，近限 120 字變色，超限紅色）
- 自訂字體 Dropdown（見字體章節）
- 字重：Light / Regular / Bold
- 大小滑桿 14–80
- 行距滑桿 1.0–3.0
- 字色：7 個色票 + 自訂 color picker
  - 暖白 `#f2ead8`、純白 `#ffffff`、深黑 `#1c1814`、純黑 `#000000`、暖金 `#d4a96a`、淡綠 `#a8d4a9`、淡藍 `#a9c4d4`
- 透明度滑桿 20–100
- 對齊：左 / 中 / 右
- 陰影 toggle + 強度滑桿 0–40
- 文字背景色塊 toggle + 顏色 picker + 透明度滑桿 5–95

### 2. 背景 Tab

風格切換：單色 / 漸層 / 色塊 / 噪點 / 圖片

**單色**

- 16 個預設色票（深色暖調 8 個 + 淺色暖調 8 個）
- 骰子（真隨機 hex）
- 顏色歷史（見顏色歷史章節）
- 自訂 color picker

**漸層**

- 8 個預設漸層色票
- 骰子（兩色各自隨機 hex）
- 漸層歷史
- 起色 / 終色 picker
- 方向：↓ 由上至下 / → 由左至右 / ↘ 對角 / ↗ 反對角

**色塊漸層（Mesh）**

- 6 個預設組合
- 骰子（四色各自隨機 hex）
- 色塊歷史
- 四角各一個 radial gradient 疊加

**噪點**

- 8 個預設底色
- 骰子（隨機 hex）
- 顏色歷史
- 自訂 color picker
- 強度滑桿 5–60
- **噪點快取**：只在底色或強度改變時重新生成，其他操作用快取 canvas

**圖片**

- 壓縮版警告（isComp = true 時顯示）
- 上傳區（點擊或拖曳，JPG / PNG / WEBP）
- 模糊滑桿 0–20
- 暗化滑桿 0–80
- 圖片浮動控制列（上傳後在 phone frame 上疊加顯示）：
  - 智能符合（重置 ix / iy / iscale）
  - ＋ / － 縮放（step 0.1）
  - ↑ ↓ ← → 微調位置（step 0.05）
  - 手勢：拖曳移位、雙指捏放縮放
  - 邊界：圖片至少 20% 面積要在畫面內

### 3. 排版 Tab

- 智能排版 toggle（開啟時文字自動放在 H\*0.62 安全區，鎖住手動拖動）
- 手動拖動文字說明文字
- 輸出尺寸列表（見模型清單）
- 裝置資訊：螢幕尺寸、DPR、實際解析度
- 重置設定按鈕（重置 S 回預設值，保留 selModel，清除 photo，噪點快取清除）

### 4. 匯出 Tab

- 目前尺寸說明文字（model name + W×H px）
- 下載按鈕（按 selModel 尺寸輸出）
- 分享按鈕（Web Share API，不支援時降級為下載）
- 複製設定連結按鈕（URL 帶 base64 編碼 state，不含圖片）

### 5. 歷史 Tab

- 清除無星號紀錄按鈕
- 2 欄網格
- 空白時：`hist-empty-wrap` 垂直置中顯示提示文字
- 每筆：縮圖（160×346 JPEG 0.7）、壓縮版標籤、星號（右上）、刪除（左上）、文字前20字、時間
- 點擊還原：載入 state，還原 photo（如有），切回文字 Tab

---

## 字體 Dropdown

自訂元件，非原生 select：

```
┌─────────────────────────────────┐
│ 思源宋體                         ▾│
│ — 優雅 · 適合勵志句               │
└─────────────────────────────────┘
展開後：
┌─────────────────────────────────┐
│ 中文                             │  ← group label
│ 思源宋體  — 優雅 · 適合勵志句    │  ← 字體名稱用該字體渲染
│ 思源黑體  — 現代 · 清晰易讀      │
│ 粉圓體    — 圓潤 · 活潑可愛      │
│ DM Serif  — 俐落 · 現代襯線     │
│ English                          │
│ Playfair Display  — 高雅 · ...  │
│ ...                              │
└─────────────────────────────────┘
```

- 往下展開優先，下方空間 < 290px 時往上展開
- 動畫：`opacity 0→1 + translateY -5px→0`，0.16s
- 點外部關閉
- 選中高亮 `.fopt.active`

**字體清單（12個）**

| 名稱               | value                         | 說明              |
| ------------------ | ----------------------------- | ----------------- |
| 思源宋體           | `'Noto Serif TC',serif`       | 優雅 · 適合勵志句 |
| 思源黑體           | `'Noto Sans TC',sans-serif`   | 現代 · 清晰易讀   |
| 粉圓體             | `'Jf Open Huninn',sans-serif` | 圓潤 · 活潑可愛   |
| DM Serif           | `'DM Serif Display',serif`    | 俐落 · 現代襯線   |
| Playfair Display   | `'Playfair Display',serif`    | 高雅 · 文藝感強   |
| Cormorant Garamond | `'Cormorant Garamond',serif`  | 古典 · 細緻優美   |
| Lora               | `'Lora',serif`                | 溫潤 · 閱讀舒適   |
| Raleway            | `'Raleway',sans-serif`        | 現代 · 幾何簡潔   |
| Josefin Sans       | `'Josefin Sans',sans-serif`   | 俐落 · 幾何無襯線 |
| Cinzel             | `'Cinzel',serif`              | 莊嚴 · 羅馬風格   |
| DM Mono            | `'DM Mono',monospace`         | 等寬 · 程式美學   |
| Courier New        | `'Courier New',monospace`     | 復古 · 打字機感   |

---

## 顏色歷史

每個背景類型各自獨立：`solid` / `gradient` / `mesh` / `noise`

- 上限 15 筆，超過刪最舊（星號保護除外）
- 星號保護無上限
- 15 筆全為星號時顯示 toast 提示，骰子仍可用但不存入歷史
- 每筆：色票（左上 ✕ 刪除，右上 ★ 星號）
- 骰子：真隨機 hex，骰完立刻存入歷史，同步更新 color picker 值

---

## 歷史紀錄

- 觸發：按下載時自動存一筆
- 上限 15 筆，超過刪最舊（星號保護除外）
- 星號保護無上限，15 筆全星號時 toast 提示
- 儲存失敗時 toast 顯示正確錯誤訊息
- 每筆內容：
  - `thumb`：160×346 JPEG 0.7（canvas toDataURL）
  - `text`：前 20 字
  - `time`：`zh-TW` locale 格式
  - `state`：JSON.stringify(S + \_img)
  - `starred`：boolean
  - `comp`：boolean（是否為壓縮版）
- 「清除無星號紀錄」按鈕：filter 保留 starred，不跳確認

---

## 輸出模型清單

| 名稱                     | 寬     | 高     |
| ------------------------ | ------ | ------ |
| 此裝置                   | SW×DPR | SH×DPR |
| iPhone SE 3              | 750    | 1334   |
| iPhone 13/14             | 1170   | 2532   |
| iPhone 13Pro/14Pro/15/16 | 1179   | 2556   |
| iPhone 15Pro/16Pro       | 1206   | 2622   |
| Samsung S24              | 1080   | 2340   |
| Pixel 8                  | 1080   | 2400   |
| 通用 1080p               | 1080   | 1920   |

橫向模式時寬高對調（只在 `getExportWH()` 一個地方對調，不重複）。

---

## 繪製邏輯

### Preview

```js
draw(pCtx, PW, PH, (ratio = 1));
// pCtx 已 scale(DPR, DPR)，所以繪製座標用 CSS pixel
```

### Export

```js
eCv.width = exportW;
eCv.height = exportH;
eCtx.setTransform(1, 0, 0, 1, 0, 0); // 重置，不需要 DPR scaling
const ratio = exportW / PW;
draw(eCtx, exportW, exportH, ratio);
```

### ratio 的用途

所有與尺寸相關的值都要乘以 ratio：

- `fontSize = S.fs * ratio`
- `shadowBlur = S.shAmt * ratio`
- `shadowOffsetY = 2 * ratio`
- `roundRect radius = 8 * ratio`
- `blur filter = S.blur * ratio`

### 文字排版

- `smart=true`：tx = W*0.5，ty = H*0.62 - totalH/2
- `smart=false`：tx = W*S.tx，ty = H*S.ty - totalH/2
- textAlign left → tx = W*0.08，right → tx = W*0.92
- 文字不可拖出邊緣：tx clamp 0.08–0.92，ty clamp 0.06–0.94
- 拖動文字時自動關閉 smart layout 並 toast 提示

---

## Undo / Redo

- 50 步，記憶體存放（undoStack / redoStack）
- `snap()` 存快照：`JSON.stringify({...S, _img: photoComp||null})`
- 滑桿放開後才存（debounce 300ms），textarea 600ms，顏色 picker 500ms
- 拖動結束後存（mouseup / touchend）
- 切換背景風格、字體、點擊色票等立即存

---

## localStorage

| key       | 內容                           |
| --------- | ------------------------------ |
| `wps4_s`  | S + `_img`（photoComp base64） |
| `wps4_ch` | colorHistories                 |
| `wps4_h`  | historyRecords                 |

- 所有設定即時寫入（每次改動）
- 圖片：photoOrig 只在記憶體，photoComp（壓縮版，最長邊 1200px JPEG 0.75）存 localStorage
- 重新整理後用 photoComp 還原，顯示壓縮版警告
- iOS Safari 無痕模式：SecurityError → toast 警告
- 空間不足：QuotaExceededError → toast 提示「儲存空間不足，請清除部分歷史紀錄」

---

## 配色（CSS Variables）

```css
--bg: #1c1814 /* 主背景（暖灰黑）*/ --bg2: #231f1a /* sidebar 背景 */
  --bg3: #2d2720 /* 輸入框、按鈕背景 */ --bg4: #38302a /* hover 背景 */
  --border: #4a3f36 /* 主邊框 */ --border2: #5c4f44 /* 次邊框 */
  --accent: #d4a96a /* 主強調色（暖金）*/ --accent2: #e8c89a /* 淡強調色 */
  --accent-dim: #8a6a42 /* 暗強調色 */ --text: #f2ead8 /* 主文字（偏暖白）*/
  --text2: #c4b49a /* 次文字 */ --text3: #8a7a68 /* 說明文字 */
  --danger: #d46a6a /* 危險紅 */ --r: 12px /* 預設圓角 */ --fs: 15px
  /* 基礎字體大小 */;
```

---

## 分享

- **Web Share API**：手機優先，分享 PNG 圖片檔
- **不支援時**：降級為下載，toast 提示
- **複製設定連結**：`?s=` + btoa(encodeURIComponent(JSON.stringify(stateWithoutImage)))
  - 不含 isComp，不含 photoComp
  - 載入時 `loadURL()` 解析並 merge 到 S

---

## 其他細節

- 字體大小基準：`--fs: 15px`，所有 UI 文字不低於 0.65rem
- textarea maxlength 150，近限（120）變暖金色，超限變紅
- 橫向模式下 phone frame 圓角改為 20px（直向 40px）
- 抽屜動畫結束後（transitionend）重新計算預覽尺寸
- 選擇尺寸後預覽比例立即更新
- 重置設定保留 selModel（選定的輸出尺寸），清除 photo
