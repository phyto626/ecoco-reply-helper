# Google Sheets CMS 建立指南

**對應功能**：F-06 後台管理  
**目標**：建立 Google Sheets 作為回覆範本資料庫，搭配 Apps Script API，讓後台變更永久儲存。

---

## 一、建立 Google Sheets

### 步驟 1：新增試算表

1. 前往 [Google Sheets](https://sheets.google.com) → 新增空白試算表
2. 命名為：**ECOCO 回覆範本庫**

### 步驟 2：建立 10 個工作表（分頁）

在試算表底部依序建立以下 10 個分頁，名稱**完全對應**：

| 分頁名稱 | 對應類別 |
|---------|---------|
| `cat-01` | 💬 聊天互動 / 正面回饋 |
| `cat-02` | 🗑️ 滿倉 / 換袋問題 |
| `cat-03` | 🔧 機台維修 / 狀態查詢 |
| `cat-04` | ❓ 使用操作問題 |
| `cat-05` | 📱 APP / 帳號問題 |
| `cat-06` | 🎁 點數 / 優惠券問題 |
| `cat-07` | 📍 設站建議 / 設備合作 |
| `cat-08` | ♻️ 回收規則 / 品項問題 |
| `cat-09` | 🏪 公益商家 / 兌換優惠 |
| `cat-10` | ⚠️ 負面意見 / 抱怨處理 |

### 步驟 3：設定欄位標題

每個分頁的**第一列**填入以下標題（A～C 欄）：

| A 欄 | B 欄 | C 欄 |
|------|------|------|
| `reply_id` | `title` | `body` |

### 步驟 4：填入現有資料

將 `replies.json` 的內容複製到對應分頁。  
以 `cat-01` 為例：

| A | B | C |
|---|---|---|
| reply_id | title | body |
| reply-01-001 | 感謝支持（通用） | 謝謝你的支持與鼓勵！💚\n我們會持續努力… |

> ⚠️ `body` 欄的換行在試算表中直接按 **Alt+Enter** 輸入，不用輸入 `\n` 符號。

---

## 二、建立 Apps Script

### 步驟 1：開啟 Apps Script

在試算表頁面 →【擴充功能】→【Apps Script】

### 步驟 2：貼上以下程式碼

刪除預設內容，完整貼上：

```javascript
const SHEET_ID = SpreadsheetApp.getActiveSpreadsheet().getId();

const CATEGORIES_META = [
  { id: 'cat-01', name: '💬 聊天互動 / 正面回饋', description: '感謝留言、正面互動' },
  { id: 'cat-02', name: '🗑️ 滿倉 / 換袋問題',     description: '機台滿了、何時清運' },
  { id: 'cat-03', name: '🔧 機台維修 / 狀態查詢',  description: '機器壞掉、何時維修' },
  { id: 'cat-04', name: '❓ 使用操作問題',          description: '退瓶、投電池、去膠膜' },
  { id: 'cat-05', name: '📱 APP / 帳號問題',        description: '無法登入、快取問題' },
  { id: 'cat-06', name: '🎁 點數 / 優惠券問題',     description: '點數未入帳、無法兌換' },
  { id: 'cat-07', name: '📍 設站建議 / 設備合作',   description: '建議設點、合作洽談' },
  { id: 'cat-08', name: '♻️ 回收規則 / 品項問題',   description: '哪些可以回收、瓶蓋規則' },
  { id: 'cat-09', name: '🏪 公益商家 / 兌換優惠',   description: '哪些商家可以兌換' },
  { id: 'cat-10', name: '⚠️ 負面意見 / 抱怨處理',   description: '公司形象維護、正面回應負評' },
];

// ── GET：回傳所有回覆資料（前台讀取）──
function doGet(e) {
  const ss = SpreadsheetApp.openById(SHEET_ID);
  const result = CATEGORIES_META.map(meta => {
    const sheet = ss.getSheetByName(meta.id);
    const replies = [];
    if (sheet && sheet.getLastRow() > 1) {
      const data = sheet.getRange(2, 1, sheet.getLastRow() - 1, 3).getValues();
      data.forEach(row => {
        if (row[0]) replies.push({ id: row[0], title: row[1], body: row[2] });
      });
    }
    return { ...meta, replies };
  });

  return ContentService
    .createTextOutput(JSON.stringify(result))
    .setMimeType(ContentService.MimeType.JSON);
}

// ── POST：寫入變更（後台操作）──
function doPost(e) {
  try {
    const payload = JSON.parse(e.postData.contents);
    const { action, category_id, reply } = payload;
    const ss = SpreadsheetApp.openById(SHEET_ID);
    const sheet = ss.getSheetByName(category_id);
    if (!sheet) throw new Error('找不到分頁：' + category_id);

    if (action === 'upsert') {
      const lastRow = sheet.getLastRow();
      let found = false;
      if (lastRow > 1) {
        const ids = sheet.getRange(2, 1, lastRow - 1, 1).getValues().flat();
        const rowIdx = ids.indexOf(reply.id);
        if (rowIdx >= 0) {
          sheet.getRange(rowIdx + 2, 1, 1, 3).setValues([[reply.id, reply.title, reply.body]]);
          found = true;
        }
      }
      if (!found) {
        sheet.appendRow([reply.id, reply.title, reply.body]);
      }
    } else if (action === 'delete') {
      const lastRow = sheet.getLastRow();
      if (lastRow > 1) {
        const ids = sheet.getRange(2, 1, lastRow - 1, 1).getValues().flat();
        const rowIdx = ids.indexOf(reply.id);
        if (rowIdx >= 0) sheet.deleteRow(rowIdx + 2);
      }
    } else if (action === 'reorder') {
      const { replies } = payload;
      if (sheet.getLastRow() > 1) sheet.deleteRows(2, sheet.getLastRow() - 1);
      replies.forEach(r => sheet.appendRow([r.id, r.title, r.body]));
    }

    return ContentService
      .createTextOutput(JSON.stringify({ success: true }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService
      .createTextOutput(JSON.stringify({ success: false, error: err.message }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```

### 步驟 3：部署為 Web App

1. 右上角點【部署】→【新增部署作業】
2. 類型選【網路應用程式】
3. 設定如下：
   - 說明：`ECOCO 回覆平台 API`
   - 以誰的身份執行：**我（你的 Google 帳號）**
   - 誰可以存取：**所有人**（若要限制，可改為「在組織內的所有人」）
4. 點【部署】→ 複製產生的 **Web App 網址**

> 網址格式：`https://script.google.com/macros/s/AKfycb.../exec`

---

## 三、更新 index.html 串接 Sheets API

取得 Web App 網址後，修改 `index.html` 中的 `SHEETS_API` 變數：

找到以下這一行：
```javascript
const SHEETS_API = 'https://script.google.com/macros/s/【原有網址】/exec';
```

改為你自己部署的網址：
```javascript
const SHEETS_API = 'https://script.google.com/macros/s/【貼上你的網址】/exec';
```

---

## 四、機台狀態查詢網址

點擊側邊欄「機台狀態」會另開新分頁，預設網址設定於 `index.html` 的 `DEFAULT_LOOKER_URL`：

```javascript
const DEFAULT_LOOKER_URL = 'https://100.ecocogroup.com/stores';
```

若需更換查詢網址，直接修改此變數值即可。

> ℹ️ 目前機台狀態系統需登入帳號才能查看，因此採用另開新分頁方式，不在平台內嵌入。

---

## 五、測試

### 測試 GET（讀取）

在瀏覽器直接開啟 Web App 網址，應回傳完整的 JSON 資料。

### 測試 POST（寫入）

使用 curl 或 Postman：
```bash
curl -X POST \
  'https://script.google.com/macros/s/【你的網址】/exec' \
  -H 'Content-Type: application/json' \
  -d '{"action":"upsert","category_id":"cat-01","reply":{"id":"reply-01-999","title":"測試標題","body":"測試內文"}}'
```

執行後確認 Google Sheets `cat-01` 分頁是否新增一筆資料。

---

## 六、注意事項

- **每次修改 Apps Script 程式碼後，必須重新部署**（新版本），舊網址不會自動更新
- **CORS**：Apps Script Web App 預設允許跨域請求，不需額外設定
- **快取**：Apps Script 有快取機制，更新後若資料未即時反映，可在網址後加 `?t=${Date.now()}` 強制重新抓取

---

*建立日期：2026/03/21　最後更新：2026/03/27*
