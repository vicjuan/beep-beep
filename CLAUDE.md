# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

這份文件是給接手開發此專案的 Claude session 看的。先讀完再開工。

## 專案核心

爸爸（vic）的兒子看到爸爸在開發倉儲掃描功能，跟著想要在「辦家家酒」時拿手機假扮成
送貨員、收銀員、圖書館員 ⋯ 任何掃條碼的職業。所有遊戲的共通儀式感是
**「掃條碼然後逼一聲」**。

這個 repo 就是要做出那個小網站。

## 使用者 / 受眾

- **主使用者**：vic 的小孩（年齡待 vic 確認）
- **次使用者**：vic 自己（demo 給孩子 / 跟孩子一起玩）
- **不是**：給成年人用的 POS 系統 / 真實庫存管理

## 核心約束（先別動）

1. **手機優先** — 主要 device 是手機，桌機只是次要 fallback
2. **零學習曲線** — 孩子打開頁面就要能玩，不需要 onboarding / 教學
3. **觸控目標巨大** — 全部按鈕 ≥ 56px，按錯比沒按嚴重
4. **逼一聲是核心 feedback** — 聲音 + 視覺閃光不能省略，這是遊戲的高潮
5. **隱私** — 條碼資料不上傳任何 server，全部在 device 上
6. **不需要登入** — 進來就能玩
7. **HTTPS 必須** — getUserMedia 沒 HTTPS 不能用 camera

## 從 parent app (zebulun5_backoffice) 能直接拿什麼

vic 的 parent 倉儲管理系統 (`~/git/zebulun5_backoffice`) 已經有**相機掃描條碼**功能
在生產跑了，路徑：[`grails-app/views/admin/packageWarehousing.gsp`](https://gitlab.atonkevin.ai/zebulun5/zebulun5_backoffice/-/blob/main/grails-app/views/admin/packageWarehousing.gsp)
搜「相機掃描條碼」這節。整套都已踩過坑，**直接抄整套架構就對了**。

### 技術選擇 — 沿用 html5-qrcode

parent app 用 [html5-qrcode](https://github.com/mebjas/html5-qrcode) 函式庫
（內部包了 ZXing-js + camera 生命週期管理），CDN 載入：

```html
<script src="https://unpkg.com/html5-qrcode" defer></script>
```

優於自己 wrap `BarcodeDetector` + ZXing fallback：lib 已處理跨瀏覽器、權限、
HTTPS 檢查、camera lifecycle，少寫好幾百行。

### 直接複製的 code 片段

#### 1. WebAudio 逼聲（成功 / 失敗音調區分）
```js
function beep(ok) {
    try {
        var ctx = new (window.AudioContext || window.webkitAudioContext)();
        var osc = ctx.createOscillator();
        var gain = ctx.createGain();
        osc.connect(gain); gain.connect(ctx.destination);
        osc.type = 'square';
        osc.frequency.value = ok ? 880 : 220;  // 成功高音 / 失敗低音
        gain.gain.value = 0.15;
        osc.start();
        osc.stop(ctx.currentTime + (ok ? 0.08 : 0.4));  // 成功短促 / 失敗長嗶
    } catch (e) { /* AudioContext 失敗就算了 */ }
}
```

#### 2. 相機初始化（支援格式 + 後鏡頭 + 取景框）
```js
scanner = new Html5Qrcode('cameraReader', {
    formatsToSupport: [
        Html5QrcodeSupportedFormats.CODE_128,
        Html5QrcodeSupportedFormats.CODE_39,
        Html5QrcodeSupportedFormats.EAN_13,
        Html5QrcodeSupportedFormats.EAN_8,
        Html5QrcodeSupportedFormats.UPC_A,
        Html5QrcodeSupportedFormats.UPC_E,
        Html5QrcodeSupportedFormats.ITF,
        Html5QrcodeSupportedFormats.QR_CODE
    ]
});
scanner.start(
    { facingMode: 'environment' },              // 後鏡頭
    { fps: 12, qrbox: { width: 280, height: 120 } },  // 12fps 省電 + 取景框
    onScanned,                                  // 掃到 callback
    function () { /* 每幀辨識失敗，忽略 */ }
);
```

#### 3. 觸覺回饋（手機震動）
```js
if (navigator.vibrate) {
    try { navigator.vibrate(100); } catch (e) {}
}
```
搭配 `beep()` 一起觸發，視覺 + 聽覺 + 觸覺三重 feedback。

### parent app 已踩過 / 解掉的坑（必看）

1. **iOS Safari getUserMedia 必須在 user gesture handler 內同步呼叫**
   ```js
   $('#btnOpenCameraScan').on('click', function () {
       startScan();  // 在 click handler 同步呼叫，不能 setTimeout 或 await 後
   });
   ```

2. **錯誤訊息要分類給使用者看**（不要丟原始 exception）
   ```js
   if (/Permission|NotAllowed/i.test(msg)) showStatus('相機權限被拒…', false);
   else if (/NotFound|DevicesNotFound/i.test(msg)) showStatus('找不到可用的相機', false);
   else if (location.protocol !== 'https:' && location.hostname !== 'localhost')
       showStatus('必須在 HTTPS 環境才能使用相機', false);
   ```

3. **頁面切走時要關相機**（不關會持續耗電 + 隱私）
   ```js
   $(window).on('pagehide beforeunload', function () { stopScanner(); });
   document.addEventListener('visibilitychange', function () {
       if (document.hidden) stopScanner();
   });
   ```

4. **防 double-fire**：掃到後立刻 `scanning = false`，避免同條碼觸發兩次
   ```js
   function onScanned(decodedText) {
       if (!scanning) return;
       scanning = false;
       // ...
   }
   ```

### parent app **不要**抄的部分

- **Carrier tracking number 正規化邏輯**（UPS / FedEx / USPS / Amazon / DHL regex）
  — 那是真實物流系統用的，beep-beep 是家家酒，**所有條碼都當有效就好**
- **POST 給後端的 doScan 流程** — beep-beep 沒後端，掃到就 beep + 顯示就完工
- **scan-to-commit 入庫狀態機** — 不用

### parent app 的 UX 流程仍可參考

- Scan 完立刻 `beep()` + vibrate 三重 feedback
- 失敗音調 220Hz 長 400ms / 成功 880Hz 短 80ms，耳朵就能分辨
- Modal 設計：黑底 backdrop + 中間 video + 下方 status 文字 + 取消鈕
- 「請將條碼對準框內」hint text 引導使用者
- 桌機 / iOS 第一次按下「開始掃描」前不 init AudioContext / Html5Qrcode

## 技術建議（可商量，但有理由）

### 建議 stack
- **Vite + 純 TS**（不要 React/Vue，這個專案太小）
- **html5-qrcode** 函式庫 — 跟 parent app 同款（已生產驗證、處理掉 BarcodeDetector
  跨瀏覽器差異 + camera 生命週期），CDN 載入或 npm install 都行
- **WebAudio API** 做逼逼聲（不要載音檔，動態合成即可，code 參考上一節）
- **PWA**（manifest + service worker），讓孩子可以加到主畫面當 app 用
- **GitHub Pages 部署**（免費、HTTPS 自動、跟 repo 整合好）

### 不建議
- React/Next.js（這個專案完全不需要 component framework）
- 後端 / 資料庫（沒任何資料要存）
- 真實 POS / 庫存 API（這是家家酒，不是真的系統）
- TailwindCSS（CSS 量小到不值得 build step；普通 CSS 寫就好）

### 為什麼這樣選
parent app (zebulun5_backoffice) 用的是 jQuery + jqWidgets + 大量 inline JS，但這個專案
是全新的、單一頁面、沒包袱，可以用現代但極簡的工具。

## 第一個 milestone：「能逼一聲」

最小可玩版本應該長這樣：
- 開頁面 → 看到「按這裡開始掃描」大按鈕
- 按下 → 開相機（要請使用者授權）
- 掃到任何條碼 → 螢幕閃光 + 逼一聲 + 顯示條碼數字
- 按關掉 → 回首頁

**這就是 v0**。不要在這個 milestone 加角色扮演、不要加多種職業介面、不要加歷史紀錄。
能逼一聲再說。

## 之後的方向（按優先序，跟 vic 確認）

1. **角色情境** — 收銀員 / 送貨員 / 圖書館員 各自的 UI 主題 + 對話框
2. **掃描歷史** — 一回合掃過的東西（localStorage，重整不消失）
3. **總計** — 收銀員模式可以加總「金額」（隨機 fake 價格 or 從條碼 hash）
4. **音效變化** — 不同角色逼聲不同 / 加上 TTS 報「歡迎光臨」之類
5. **多語** — 英文 / 日文之類（vic 可能想讓孩子順便接觸）

## 一定要先問 vic 的問題

開工前先確認，**不要自己決定**：
- [ ] 孩子年齡？這決定字級 / 語言複雜度 / 是否需要注音
- [ ] 第一階段要哪一種角色？（收銀員、送貨員、還是先做「通用掃描」）
- [ ] 部署到哪？預設 GitHub Pages，但可能想 Cloudflare Pages / Vercel
- [ ] 要不要支援多語言？還是只繁體中文？
- [ ] 孩子用自己的裝置還是跟爸爸共用？影響有沒有需要切換 profile

## 常見坑

### iOS Safari camera
- **必須 HTTPS**（localhost 例外）。GitHub Pages 自動 HTTPS，dev 用 `vite --host --https` 或 `mkcert`
- `getUserMedia({ video: { facingMode: 'environment' } })` 拿後鏡頭
- 第一次按下「開始」按鈕時呼叫 `getUserMedia`，不要 page load 就呼叫（會被擋）

### html5-qrcode
- 用 lib 跟 parent app 一致，跨瀏覽器差異 lib 內部處理掉了，不必碰 BarcodeDetector
- 支援格式要 explicit 指定 `formatsToSupport`（看上一節 code），不指定會全掃很慢
- 取景框 `qrbox` 是視覺引導，**不是辨識範圍**（lib 仍掃整張 frame）
- `fps: 12` 是省電 sweet spot，太高 (30+) 手機會發燙

### PWA on iOS
- iOS 的 PWA 限制很多（不能背景跑、storage 容易被清）
- 但加到主畫面當 app icon 開仍然 OK，這個用途夠了

### WebAudio
- iOS 要使用者「第一次互動後」才能播聲音。第一次按「開始」時順便 init `AudioContext`
- 不要用 `<audio>` 載音檔 — 額外網路請求 + 延遲；動態合成 oscillator 就好

## Repo 慣例

- 中文 commit message（vic 的偏好，看 vic 其他 repo 都是繁中）
- Branch 工作流：直接 push main 即可（個人專案，不必開 PR）
- 不寫 unit test（這專案太小，手測就好）

## 如果你卡住

不確定的事 ASK，不要瞎猜。vic 是這個專案的 product owner，
不是「協助你 implement」的對象 — 他知道孩子要什麼，你不知道。
