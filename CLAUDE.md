# 壁紙工作室 — 技術架構文件

> 供 AI 對話恢復記憶使用。單一 HTML 檔案，純 vanilla HTML/CSS/JS，無框架。

---

## 專案概覽

- **檔案**：`index.html`（全部 CSS / JS inline）
- **網址**：https://wenhsi.github.io/wallpaper-studio/
- **作者**：wenhsi（kevinwen2014@gmail.com）
- **版本**：v1.0.1
- **字體**：Google Fonts CDN（Noto Serif TC、Noto Sans TC、LXGW WenKai TC、Zen Maru Gothic、M PLUS Rounded 1c、DotGothic16、DM Serif Display、DM Mono、Playfair Display、Cormorant Garamond、Lora、Raleway、Josefin Sans、Cinzel）
- **儲存**：localStorage（key prefix: `wps4_`）

---

## Layout

### 桌機版（視窗寬度 > 700px）

```
┌──────────────────┬─────────────────────────────────┐
│  Sidebar 320px   │         Preview Area             │
│  ┌────────────┐  │   [↩] [↪] [⟲橫向]               │
│  │  Tab Nav   │  │                                  │
│  ├────────────┤  │      ┌──────────────┐            │
│  │            │  │      │  Phone Frame │            │
│  │  Panel     │  │      │   Canvas     │            │
│  │ (scroll)   │  │      └──────────────┘            │
│  │            │  │      model name · W×H px         │
│  └────────────┘  │                                  │
│                  │   [⬇ 下載桌布]                    │
└──────────────────┴─────────────────────────────────┘
```

### 手機版（視窗寬度 ≤ 700px）

```
┌──────────────────────────┐
│       Preview Area        │
│  [↩][↪][⟲橫向][⬇下載]   │
│   ┌───────────────────┐  │
│   │    Phone Frame    │  │
│   └───────────────────┘  │
│    model name · W×H px   │
├──────────────────────────┤
│  ▔▔▔▔ (handle bar)       │
│  [文字][背景][排版][匯出][歷史] │
│                           │
│   Panel content (scroll)  │
└──────────────────────────┘
```

- 底部抽屜預設收起（50px，只顯示 handle + tab nav）
- 抽屜高度 = `window.innerHeight * 0.46`
- 點擊 handle 切換；點擊預覽背景空白區自動關閉
- `transitionend` 後重新計算預覽尺寸

---

## State 結構

```js
const DS = () => ({
  // 文字
  text: '每一天，都是重新開始的機會。',
  font: "'LXGW WenKai TC',serif",
  fw: '400',        // 字重：300 / 400 / 700
  fs: 22,           // 字體大小 px（UI 範圍 14–80）
  tc: '#1c1814',    // 文字顏色
  to: 95,           // 文字透明度 20–100
  lh: 2.0,          // 行距倍數 1.0–3.0
  ls: 1,            // 字距 px 0–20
  ta: 'center',     // 對齊：left / center / right
  shOn: false,      // 文字陰影開關
  shAmt: 15,        // 陰影強度 0–40
  bgOn: false,      // 文字背景色塊開關
  bgCol: '#000000', // 文字背景顏色
  bgOp: 40,         // 文字背景透明度 5–95

  // 背景
  bgs: 'solid',     // solid / gradient / mesh / noise / photo
  sc: '#f0eae0',    // 單色背景顏色（預設暖米白）
  g1: '#1c1814',    // 漸層起色
  g2: '#3d2b1a',    // 漸層終色
  gd: 'to bottom',  // 漸層方向
  mc: [...],        // 色塊漸層（4 個 hex）
  nc: '#1c1814',    // 噪點底色
  na: 20,           // 噪點強度 5–60

  // 圖片
  blur: 0,          // 模糊 0–20
  dim: 30,          // 遮暗 0–80
  ix: 0,            // 圖片 X 偏移（比例）
  iy: 0,            // 圖片 Y 偏移（比例）
  iscale: 1,        // 圖片縮放 0.3–5

  // 排版
  tx: 0.5,          // 文字 X 位置（比例）
  ty: 0.65,         // 文字 Y 位置（比例）
  smart: true,      // 智能排版
  land: false,      // 橫向模式

  // 其他
  isComp: false,    // 使用壓縮圖片
  selModel: 0,      // 選定的輸出尺寸 index
});
let S = DS();
```

---

## 顏色歷史結構

```js
let cHist = {
  solid: [],      // 單色歷史
  gradient: [],   // 漸層歷史
  mesh: [],       // 色塊歷史
  noise: [],      // 噪點歷史
  text: [],       // 文字顏色歷史
  pairs: [],      // 配色組合 [{tc, sc, starred}]
};
```

- 各類型上限 15 筆，超過刪最舊（starred 保護）
- `pairs` 儲存文字色 + 背景色配對，一鍵套用兩個值

---

## 預覽尺寸計算

```js
const BASE_PW = 180; // ratio 基準寬度

function computeSize() {
  const m = getSelModel();
  // 橫向：AR = m.h/m.w（高/寬，產生寬畫布）
  // 直向：AR = m.w/m.h（寬/高）
  const AR = S.land ? (m.h / m.w) : (m.h > 0 ? m.w / m.h : 9 / 19);
  // 在可用空間內最大化比例
  // 橫向限制：w≤700, h≤380；直向限制：w≤380, h≤800
}

function renderP() {
  // ratio 使用短邊，避免橫向字體膨脹
  draw(pCtx, PW, PH, (S.land ? PH : PW) / BASE_PW);
}
```

- `PW`、`PH`：當前預覽畫布 CSS 像素寬高
- canvas.width = PW × DPR（高清）
- 繪製前 `ctx.scale(DPR, DPR)`

---

## 裝置模型（MODELS）

```js
const MODELS = [
  { name: '此裝置', w: REAL_W, h: REAL_H, notch: 'auto' },
  { name: 'iPhone SE 3', w: 750, h: 1334, notch: 'none' },
  { name: 'iPhone 13 / 14', w: 1170, h: 2532, notch: 'notch' },
  { name: 'iPhone 13 Pro Max / 14 Plus', w: 1284, h: 2778, notch: 'notch' },
  { name: 'iPhone 14 Pro / 15 / 15 Pro / 16', w: 1179, h: 2556, notch: 'island' },
  { name: 'iPhone 15+ / 15 Pro Max / 16 Plus', w: 1290, h: 2796, notch: 'island' },
  { name: 'iPhone 16 Pro', w: 1206, h: 2622, notch: 'island' },
  { name: 'iPhone 16 Pro Max', w: 1320, h: 2868, notch: 'island' },
  { name: 'Samsung S24 / S24+ / S25', w: 1080, h: 2340, notch: 'hole' },
  { name: 'Samsung S24 Ultra / S25 Ultra', w: 1440, h: 3088, notch: 'hole' },
  { name: 'Pixel 8 / 9', w: 1080, h: 2400, notch: 'hole' },
  { name: '通用 1080p', w: 1080, h: 1920, notch: 'clear' },
];
```

### Notch 類型（getNotchType）

| 類型 | 說明 | overlay 行為 |
|------|------|-------------|
| `island` | 動態島（iPhone 14 Pro+） | 小膠囊形狀 |
| `notch` | 瀏海（iPhone 13/14） | 寬矩形缺口 |
| `hole` | 打孔（Samsung/Pixel） | 小圓形孔 |
| `none` | 無缺口（iPhone SE） | 無 island，頂部留空 |
| `clear` | 無 overlay | 隱藏整個 lock overlay |
| `auto` | 此裝置，依解析度對應 | map 查詢，找不到 fallback `clear` |

---

## Lock Screen Overlay

HTML 結構：
```html
<div class="lock-overlay" id="lockOverlay" data-notch="...">
  <div class="lo-island" id="loIsland"></div>
  <div class="lo-text">          <!-- 日期+時鐘 wrapper -->
    <div class="lo-date" id="loDate">週日 ☁ 6月8日</div>
    <div class="lo-clock" id="loClock">9:41</div>
  </div>
  <div class="lo-widgets">...</div>
  <div class="lo-spacer"></div>
  <div class="lo-bottom">        <!-- 閃光燈 + 相機按鈕 -->
    <div class="lo-btn">...</div>
    <div class="lo-btn">...</div>
  </div>
  <div class="lo-homebar"></div>
</div>
```

所有元素尺寸用 `--pw` CSS 變數縮放（portrait=PW, landscape=PH）。

### 橫向模式 overlay（resizePreview 中執行）

```js
if (S.land) {
  // 整個 .lock-overlay 旋轉 -90°，模擬手機拿橫的效果
  lo.style.width = PH + 'px';
  lo.style.height = PW + 'px';
  lo.style.left = (PW - PH) / 2 + 'px';
  lo.style.top = -(PW - PH) / 2 + 'px';
  lo.style.transform = 'rotate(-90deg)';
  lo.style.setProperty('--pw', PH + 'px');
} else {
  // 還原
  lo.style.setProperty('--pw', PW + 'px');
}
```

---

## 智能排版

### 直向（getSmartTY）

```js
function getSmartTY() {
  if (S.land) return 0.5; // 橫向垂直置中
  const nt = getNotchType(getSelModel());
  if (nt === 'clear') return 0.50;
  if (nt === 'none')  return 0.70; // iPhone SE，無缺口，文字較低
  if (nt === 'notch') return 0.61;
  if (nt === 'hole')  return 0.59;
  return 0.62; // island
}
```

### 橫向安全區（getLandSafeZone）

```js
function getLandSafeZone() {
  // [leftRatio, rightRatio]
  // 左側：時鐘/island/widgets（portrait top → landscape left）
  // 右側：閃光+相機按鈕（portrait bottom → landscape right）
  const nt = getNotchType(getSelModel());
  if (nt === 'island' || nt === 'notch') return [0.43, 0.87];
  if (nt === 'hole'   || nt === 'none')  return [0.37, 0.93];
  return [0.05, 0.95]; // clear，全域置中
}

function getSmartTX() {
  if (!S.land) return 0.5;
  const z = getLandSafeZone();
  return (z[0] + z[1]) / 2;
}
```

---

## 繪製邏輯（drawTXT）

```js
function drawTXT(ctx, W, H, ratio) {
  const fs = S.fs * ratio;
  // 橫向智能模式：mw 限制在安全區寬度內，避免右側溢出
  const mw = (S.smart && S.land)
    ? W * (getLandSafeZone()[1] - getLandSafeZone()[0]) * 0.94
    : W * 0.84;
  const lines = wrap(ctx, t, mw);
  const lh = fs * S.lh;
  // totH = 前(n-1)行行距 + 最後一行字高（避免行距影響中心點）
  const totH = (lines.length - 1) * lh + fs;
  let tx, ty;
  if (S.smart) { tx = W * getSmartTX(); ty = H * getSmartTY() - totH / 2; }
  else         { tx = W * S.tx;         ty = H * S.ty - totH / 2; }
}
```

Export 時 ratio = `(S.land ? h : w) / BASE_PW`（短邊/基準）。

---

## Undo / Redo

- 50 步，記憶體存放（`undoStack` / `redoStack`）
- `snap()`：存 `JSON.stringify({...S, _img: photoComp || null})`
- debounce：滑桿 300ms，textarea 600ms，顏色 picker 500ms
- 拖動結束後（mouseup / touchend）立即存

---

## localStorage

| key | 內容 |
|-----|------|
| `wps4_s` | 當前 state + `_imgFp`（照片 fingerprint） |
| `wps4_ch` | colorHistories（含 pairs） |
| `wps4_h` | 歷史紀錄（最多 15 筆） |
| `wps4_img_{fp}` | 各張照片壓縮版（base64，單獨存） |

- 照片以 fingerprint 去重，歷史紀錄只存 `_imgFp` 引用
- `lsSet`：先 removeItem 再 setItem，避免覆寫失敗
- 滿時（QuotaExceededError）：toast 提示清除歷史

### 初始化順序

```js
loadS();    // 從 localStorage 載入
loadURL();  // 從 URL ?s= 載入，覆蓋 localStorage（分享連結優先）
loadCH();
loadH();
```

---

## 分享連結

```js
window.copyLink = function () {
  const s = { ...S }; delete s.isComp;
  const url = location.href.split('?')[0] + '?s=' + btoa(encodeURIComponent(JSON.stringify(s)));
  navigator.clipboard.writeText(url);
};
```

- 包含所有 S 欄位（含 text、font、colors 等），不含圖片
- `loadURL` 在 `loadS` 之後執行，URL 參數優先

---

## CSS 主題（亮色）

```css
:root {
  --bg: #f5f5f5;
  --bg2: #ffffff;
  --bg3: #f0f0f0;
  --bg4: #e8e8e8;
  --border: rgba(0,0,0,0.09);
  --border2: rgba(0,0,0,0.16);
  --accent: #1a1a1a;
  --text: #111;
  --text2: #555;
  --text3: #999;
  --danger: #d03030;
  --r: 12px;
  --fs: 16px;
}
```

---

## 版本歷史摘要

| 版本 | 主要內容 |
|------|---------|
| v1.0.1 | 手機版點擊預覽空白區關閉抽屜 |
| v1.0.0 | 正式版：配色組合、橫向智能排版修正（getLandSafeZone）、分享連結載入順序修正 |
| beta | localStorage 照片去重、橫向預覽 AR、iOS 存相簿、歷史紀錄、UI 打磨 |
