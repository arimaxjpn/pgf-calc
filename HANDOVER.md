# PGF Calc Pro - 引き継ぎメモ（最新版）

## アプリ概要
高速道路中央分離帯のプレキャストガードフェンス（PGF）設置時の水準測量補助ツール。
ファイル: `pgf.html`（単一ファイル、外部依存なし）

---

## デプロイ情報（PWA）

- **GitHub リポジトリ**: https://github.com/arimaxjpn/pgf-calc
- **公開URL**: https://arimaxjpn.github.io/pgf-calc/pgf.html
- **git ユーザー**: arimaxjpn / arimaxjpr@gmail.com
- **認証**: Git Credential Manager（Windows）で自動処理済み

### 更新手順
```bash
git add pgf.html
git commit -m "変更内容"
git push
```
push後1〜2分でiPhoneに反映。アプリを完全に閉じて再起動で最新版が読み込まれる。

---

## PWA構成ファイル

| ファイル | 役割 |
|---|---|
| `pgf.html` | メインアプリ（PWA対応済み） |
| `manifest.json` | アプリ名・アイコン・display:standalone・theme_color:#222222 |
| `sw.js` | cache-first戦略でオフラインキャッシュ（pgf.html, manifest.json） |
| `icon-192.png` | PWAアイコン 192×192（Python+Pillowで生成） |
| `icon-512.png` | PWAアイコン 512×512（Python+Pillowで生成） |

### pgf.html に含まれるPWA設定
- `<link rel="manifest" href="manifest.json">`
- `<meta name="theme-color" content="#222222">`
- iOS用メタタグ（apple-mobile-web-app-capable等）
- Service Worker登録スクリプト
- iOSバウンス防止（CSS: `overscroll-behavior:none` + JS: touchmove preventDefault）

---

## 現在のレイアウト構成（上から順）

1. **グリッド**（`generateGrid()` で動的生成、横スクロール対応）
   - Row 1: **統合エリア**（勾配入力 → 距離入力 → SVG勾配図）← 設定セクション
   - Row 2: 奥（Back）読み値 × 5列（基準点 + スパン1〜4）
   - Row 3: 手前（Front）読み値 × 5列

2. **フッター**
   - 読値リセット / 観測点変更

> **注意**: `h = 0` に固定（PGF高の入力欄は削除済み）。

---

## 統合エリア（Row 1）の構成

```
┌──────────────────────────────────────────────────────┐
│ [勾配(%)]  [入力1] [入力2] [入力3] [入力4]            │  .int-top (padding: 20px 0 0 0)
│ [距離(m)]  [入力1] [入力2] [入力3] [入力4]            │  .int-bottom (padding-bottom: 2px)
│ ────●──────●──────●──────●──────●────  (SVG 500×90) │
└──────────────────────────────────────────────────────┘
```

- 左端に 50px の「ラベルスペーサー」（高さ34px）
- 4つの勾配入力欄（幅100px）、4つの距離入力欄（幅100px）
- **カスケードルール**: スパン i に勾配を入力すると、i 以降のスパンに適用される
- カスケード適用中スパンのプレースホルダーは `=`（未入力の先頭は `0`）
- `isReversed=false` → 基準点セルは最左
- `isReversed=true`  → 基準点セルは最右
- SVG: `drawSlope()` でリアルタイム描画

---

## 現在の完全なCSS（主要部分）

```css
/* body */
body {
  overscroll-behavior: none; /* iOSバウンス防止 */
}

/* 統合エリア */
.int-cell {
  grid-column: 1 / -1;
  display: flex;
  flex-direction: column;
  background: #222;
  border-top: 1px solid #444;
  border-bottom: 1px solid #444;
}
.int-top {
  display: flex;
  align-items: center;
  padding: 20px 0 0 0;
}
.int-slope-inp {
  width: 100px;
  padding: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
}
.int-bottom {
  display: flex;
  padding-bottom: 2px;
}
.int-dist-cell {
  width: 100px;
  min-width: 100px;
  height: 34px;
  background: transparent;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
}

/* セル高さ */
.cell-h { height: 68px; }
.cell-m { height: 50px; }

/* パネル */
.panel {
  background: #222;
  border: 1px solid #444;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  margin: -1px 0 0 -1px;
  position: relative;
  padding: 4px;
}

/* フォーカス枠：なし（カーソル点滅で代替） */
input:focus { outline: none; }
.int-slope-inp input:focus,
.int-dist-cell input:focus { outline: none; }
```

### スペーサー（ラベル付き、generateGrid内）
```javascript
const slopeSpacer = `<div style="width:50px;min-width:50px;height:34px;display:flex;flex-direction:column;align-items:center;justify-content:center;text-align:center;"><span style="font-size:10px;color:#aaa;line-height:1.3;">勾配<br>(%)</span></div>`;
const distSpacer  = `<div style="width:50px;min-width:50px;height:34px;display:flex;flex-direction:column;align-items:center;justify-content:center;text-align:center;"><span style="font-size:10px;color:#aaa;line-height:1.3;">距離<br>(m)</span></div>`;
```

SVGタグ: `<svg id="slope-svg" width="500" height="90" style="display:block">`

---

## 読値セルの構造（現在）

```
┌────────────────────┐
│ [入力欄 読値]  [×] │  ← inp-group（height: 34px）
│ 目標値    アシスト │  ← height: 26px, space-between
└────────────────────┘
```

**データセル（JavaScript テンプレート）:**
```javascript
`<div class="panel cell-h">
  <div class="inp-group">
    <input type="number" id="in_rb_${idx}" class="inp-sm" placeholder="読値"
      oninput="chkDiff('rb', ${idx})" onkeydown="chkEnt(event)">
    <button class="btn-clear" onclick="clearInp('in_rb_${idx}')">×</button>
  </div>
  <div style="display:flex;width:100%;justify-content:space-between;align-items:center;height:26px;">
    <span id="out_rb_${idx}" class="res-line" style="color:#666">-</span>
    <span id="diff_rb_${idx}" class="diff-line" style="color:#888">-</span>
  </div>
</div>`
```
※ `txt-orange` クラスは廃止。色は `style.color` を直接操作（動的カラーリング）。

**基準点セル（背景色なし）:**
```javascript
`<div class="panel cell-h">
  <input type="number" id="val_rb" class="inp-sm" placeholder="読値"
    oninput="runCalc()" onkeydown="chkEnt(event)">
  <div style="height:26px;width:100%"></div>
</div>`
```
※ 基準点は目標値・差分表示なし。`bg-accent` クラス削除済み。

---

## SVG（drawSlope）の構成

```
SVGサイズ: width=500, height=90
pad=50（左右余白）
tickTop=8, tickBot=22, arrowY=15（寸法線エリア）
centerY=58（水平基準線Y座標）
```

描画順序（重ね順）:
1. 白い寸法線（ティックマーク + 基準線 + 内向き矢印）stroke="#e0e0e0" stroke-width="0.5"
2. 水平基準線（破線、rgba 0.2）
3. ドロップライン（各点→下端、破線、rgba 0.25）
4. 勾配折れ線（青 #2196F3）
5. ドット（基準点: 赤 #f44336 r=5、測点: 黄 #FFD600 r=4）stroke="#111" stroke-width="0.5"
6. 矢印三角形: fill="#aaa"

---

## グリッド仕様

```css
.scroll-container {
  display: grid;
  grid-template-columns: 100px repeat(4, 100px); /* 5列 × 100px = 500px */
  overflow-x: auto;
}
.grid-wrapper.reversed .scroll-container {
  grid-template-columns: repeat(4, 100px) 100px;
}
```

SVGの `drawSlope()` では `W=500, pad=50` で計算。
スペーサーが50px → 勾配・距離入力欄がスパンの「線分中心」に揃う。

---

## 主要変数・関数

| 名前 | 役割 |
|---|---|
| `SPANS = 4` | スパン数（固定） |
| `isReversed` | 観測方向（false=上り側, true=下り側） |
| `memTargets` | 計算済み目標値の辞書 |
| `generateGrid()` | グリッドHTML生成・再描画 |
| `getEffectiveSlopes()` | カスケード勾配配列を返す（slopes[0]=スパン1用） |
| `runCalc()` | メイン計算（h=0固定）、末尾でdrawSlope呼び出し |
| `drawSlope()` | SVG勾配図描画（W=500, H=90, pad=50） |
| `doFlip()` | 観測点変更（値保存→再生成→値復元）、奥手前スワップ |
| `doReset()` | 読値のみリセット（距離・勾配は保持） |
| `updateSlopePlaceholders()` | カスケード適用スパンのplaceholderを `=` に更新 |
| `onSlopeInput(idx)` / `clearSlope(idx)` | 勾配入力イベント |
| `onDistInput(idx)` / `clearDist(idx)` | 距離入力イベント |

---

## 入力フィールドID一覧

| ID | 内容 | 備考 |
|---|---|---|
| `val_rb` | 基準点・奥読み値 | 動的生成 |
| `val_rf` | 基準点・手前読み値 | 動的生成 |
| `val_slope_{1-4}` | 各スパンの勾配(%) | 動的生成、カスケードルール適用 |
| `in_rb_{1-4}` | 各スパン・奥読み値 | 動的生成 |
| `in_rf_{1-4}` | 各スパン・手前読み値 | 動的生成 |
| `in_dist_{1-4}` | 各スパンの区間距離(m) | 動的生成、デフォルト5 |

> **削除済み**: `val_h`（PGF高）→ `h = 0` にハードコード

---

## 計算式

```
effectiveSlopes = getEffectiveSlopes() // カスケード適用済み配列
cumDrop += dVal * 1000 * (effectiveSlopes[i-1] / 100)
目標値 = 基準読み値 - cumDrop + h   // h=0
差分 = 目標値 - 実測値
  > 0 → "X 高い"（地盤が高い、切土）
  < 0 → "X 低い"（地盤が低い、盛土）
  = 0 → "OK"
```

---

## doFlip() で保存・復元するもの

- `val_rb`, `val_rf`（スワップして復元：奥→手前、手前→奥）
- `in_rb_{1-4}`, `in_rf_{1-4}`（スワップして復元）
- `in_dist_{1-4}`（そのまま復元）
- `val_slope_{1-4}`（そのまま復元）

**重要**: すべて動的生成フィールドのため、`generateGrid()` 前に退避、後に復元。

---

## 確定済み設定

- 勾配入力フォーマット: `%` 入力のまま（`3.33` = 3.33%）、変更不要
- フォーカス枠: なし（`outline: none`）、カーソル点滅で代替

---

## 現在の配色（確定）

| 要素 | 色 |
|---|---|
| 勾配・距離 input 背景 | `#333` |
| 勾配・距離 input 文字 | `#e0e0e0` |
| 勾配・距離 input ボーダー | `#555`（右辺なし） |
| 寸法線・ティック・矢印線 | `#e0e0e0` / stroke-width 0.5 |
| 矢印三角形 fill | `#aaa` |
| 基準点ドット | `#f44336`（赤） |
| 測点ドット | `#FFD600`（黄） |
| 目標値テキスト | `#666` |
| 差分デフォルト `-` | `#888` |
| 差分 高い | `#f44336`（赤） |
| 差分 低い | `#2196F3`（青） |
| 差分 OK | `#FFD600`（黄） |

---

## フッター下の計算式表示

```html
<div class="formula">
  累積落差 (mm) ＝ 距離 (m) × 勾配 (%) × 10　※スパンごとに加算
  目標値 ＝ 基準読み値 − 累積落差
  差分 ＝ 目標値 − 実測値　( + → 高い / − → 低い )
</div>
```
スタイル: `background:#1a1a1a; border:1px solid #333; font-size:11px; margin-top:15px`

---

## chkDiff() カラーリングロジック

```javascript
if (target === undefined || inpEl.value === '') {
  outEl.innerText = '-'; outEl.style.color = '#888'; return;
}
if (diff > 0)      { outEl.innerText = Math.abs(diff) + ' 高い'; outEl.style.color = '#f44336'; }
else if (diff < 0) { outEl.innerText = Math.abs(diff) + ' 低い'; outEl.style.color = '#2196F3'; }
else               { outEl.innerText = 'OK';                     outEl.style.color = '#FFD600'; }
```

runCalc()内でのリセット:
```javascript
const dRb = document.getElementById(`diff_rb_${i}`); dRb.innerText = '-'; dRb.style.color = '#888';
const dRf = document.getElementById(`diff_rf_${i}`); dRf.innerText = '-'; dRf.style.color = '#888';
```

---

## 勾配・距離 input のダークテーマCSS

```css
.int-slope-inp input,
.int-dist-cell input {
  background: #333;
  color: #e0e0e0;
  border-color: #555;
  border-radius: 0;
  border-right: none;
}
.int-slope-inp input::placeholder,
.int-dist-cell input::placeholder { color: #777; }
.int-slope-inp input:focus,
.int-dist-cell input:focus { outline: none; }
```

---

## フッター・計算式CSS

```css
.footer { display: flex; gap: 8px; margin-top: 15px; height: 50px; }
.formula {
  margin-top: 15px; padding: 10px 14px;
  background: #1a1a1a; border: 1px solid #333; border-radius: 6px;
  text-align: left; font-size: 11px; color: #777; line-height: 1.8;
}
.formula span { color: #aaa; }
```

計算式HTML:
```html
<div class="formula">
  <span>累積落差 (mm)</span>　＝　距離 (m) × 勾配 (%) × 10　　※スパンごとに加算<br>
  <span>目標値</span>　＝　基準読み値 − 累積落差<br>
  <span>差分</span>　＝　目標値 − 実測値　　( + → 高い　/ − → 低い )
</div>
```
