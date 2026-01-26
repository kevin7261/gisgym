# 📊 Google Sheets 寫入功能設置說明

本功能會自動將每次生成的題目寫入到指定的 Google Sheets 中。

## 🚀 快速設置步驟

### 1. 打開 Google Sheets

訪問你的 Google Sheets：
```
https://docs.google.com/spreadsheets/d/1JTrp2bRtwkv0pgTt9pIvmcmYRkEzgMCXA9tSdOAbc6M/edit
```

### 2. 創建 Google Apps Script 專案

有兩種方式：

**方式 A：從 Google Sheets 創建**
1. 在 Google Sheets 中，點擊「擴充功能」>「Apps Script」
2. 會自動打開 Google Apps Script 編輯器

**方式 B：直接訪問**
1. 訪問：https://script.google.com
2. 點擊「新增專案」

### 3. 複製並貼上代碼

將以下代碼複製到 Apps Script 編輯器中：

```javascript
function doPost(e) {
  try {
    // 打開 Google Sheets（使用 Sheet ID）
    const sheetId = '1JTrp2bRtwkv0pgTt9pIvmcmYRkEzgMCXA9tSdOAbc6M';
    const sheet = SpreadsheetApp.openById(sheetId).getActiveSheet();
    
    // 解析接收到的數據（支援表單數據和 JSON）
    // 注意：使用 FormData 時，數據在 e.parameter 中
    // 使用 JSON 時，數據在 e.postData.contents 中
    let data;
    
    // 優先檢查表單數據（FormData）
    if (e.parameter && Object.keys(e.parameter).length > 0) {
      data = e.parameter;
    } 
    // 如果沒有表單數據，檢查 JSON 數據
    else if (e.postData && e.postData.contents) {
      try {
        data = JSON.parse(e.postData.contents);
      } catch (parseError) {
        // JSON 解析失敗，使用空對象
        data = {};
      }
    } 
    // 都沒有，使用空對象
    else {
      data = {};
    }
    
    // 如果第一行是空的，添加標題行
    if (sheet.getLastRow() === 0) {
      sheet.appendRow(['時間', '題目類型', '難度', '題目內容', '提示', '目標檔案']);
      // 設置標題行格式（可選）
      const headerRange = sheet.getRange(1, 1, 1, 6);
      headerRange.setFontWeight('bold');
      headerRange.setBackground('#4285f4');
      headerRange.setFontColor('#ffffff');
    }
    
    // 添加數據行
    sheet.appendRow([
      data.timestamp ? new Date(data.timestamp) : new Date(),
      data.qtype || '',
      data.level || '',
      data.question_content || '',
      data.hint || '',
      data.target_filename || ''
    ]);
    
    // 自動調整列寬（可選）
    sheet.autoResizeColumns(1, 6);
    
    return ContentService.createTextOutput(JSON.stringify({
      success: true,
      message: '數據已成功寫入'
    })).setMimeType(ContentService.MimeType.JSON);
    
  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({
      success: false,
      error: error.toString()
    })).setMimeType(ContentService.MimeType.JSON);
  }
}

// 可選：添加 doGet 用於測試
function doGet(e) {
  return ContentService.createTextOutput('Google Sheets Web App 運行正常')
    .setMimeType(ContentService.MimeType.TEXT);
}
```

### 4. 保存專案

1. 點擊「儲存」按鈕（或按 `Ctrl+S` / `Cmd+S`）
2. 為專案命名（例如：「GISGym 題目記錄」）

### 5. 部署為 Web App

1. 點擊「部署」>「新增部署作業」
2. 選擇類型：「網頁應用程式」
3. 配置設置：
   - **說明**：可選，例如「GISGym 題目寫入服務」
   - **執行身份**：選擇「我」
   - **具有存取權的使用者**：選擇「任何人」（重要！）
4. 點擊「部署」
5. **首次部署**：Google 會要求授權，請：
   - 點擊「授權存取權限」
   - 選擇你的 Google 帳號
   - 點擊「進階」>「前往 [專案名稱]（不安全）」
   - 點擊「允許」
6. 複製 **Web App URL**（格式類似：`https://script.google.com/macros/s/.../exec`）

### 6. 配置到應用程式

1. 打開 `src/utils/googleSheets.js`
2. 找到 `GOOGLE_SHEETS_WEB_APP_URL` 常數
3. 將你複製的 Web App URL 貼上：

```javascript
const GOOGLE_SHEETS_WEB_APP_URL = 'https://script.google.com/macros/s/YOUR_SCRIPT_ID/exec';
```

### 7. 測試功能

1. 重新啟動應用程式
2. 在應用中生成一個題目
3. 檢查 Google Sheets 是否出現新的一行數據

## ✅ 驗證設置

### 測試 Web App

你可以直接在瀏覽器中訪問 Web App URL（使用 `doGet` 函數）：
```
https://script.google.com/macros/s/YOUR_SCRIPT_ID/exec
```

應該會看到：「Google Sheets Web App 運行正常」

### 檢查 Google Sheets

生成題目後，檢查 Google Sheets 是否：
- 第一行有標題（時間、題目類型、難度、題目內容、提示、目標檔案）
- 新生成的題目出現在新的一行
- 數據格式正確

## 🔧 故障排除

### 問題 1：寫入失敗，顯示「未配置 Web App URL」

**解決方案**：
- 確認已將 Web App URL 配置到 `googleSheets.js` 中
- 確認 URL 格式正確（以 `/exec` 結尾）

### 問題 2：寫入失敗，顯示「HTTP 403」或「權限錯誤」

**解決方案**：
- 確認 Web App 的「具有存取權的使用者」設置為「任何人」
- 重新部署 Web App
- 確認 Google Sheets 的分享設置允許編輯（至少對你的 Google 帳號）

### 問題 3：寫入失敗，顯示「找不到試算表」

**解決方案**：
- 確認 Sheet ID 正確（在 Apps Script 代碼中）
- 確認你有權限訪問該 Google Sheets

### 問題 4：數據沒有寫入

**解決方案**：
- 檢查瀏覽器控制台是否有錯誤訊息
- 確認 Apps Script 代碼正確
- 嘗試手動執行 Apps Script 中的 `doPost` 函數進行測試

## 📝 數據格式

寫入的數據包含以下欄位：

| 欄位 | 說明 | 範例 |
|------|------|------|
| 時間 | 題目生成的時間戳 | 2026-01-25T10:30:00.000Z |
| 題目類型 | 題目類型（如：實作題） | 實作題 |
| 難度 | 難度等級（如：進階） | 進階 |
| 題目內容 | 完整的題目內容 | 請實作一個函數... |
| 提示 | 題目提示 | 可以使用... |
| 目標檔案 | 目標檔案名稱 | test.js |

## 🔒 安全性說明

- Web App URL 是公開的，但只有知道 URL 的人才能使用
- 建議定期檢查 Google Sheets 的分享權限
- 如果需要更安全的方案，可以添加 API Key 驗證（需要修改 Apps Script 代碼）

## 📚 相關文件

- [Google Apps Script 文檔](https://developers.google.com/apps-script)
- [Google Sheets API 文檔](https://developers.google.com/sheets/api)
- [ContentService 文檔](https://developers.google.com/apps-script/reference/content/content-service)

## 💡 進階配置

### 自定義欄位

如果需要添加更多欄位，可以：

1. 修改 Apps Script 代碼中的標題行
2. 修改 `appendRow` 中的數據順序
3. 在 `WorkTab.vue` 中傳遞更多數據到 `writeQuestionToSheets`

### 添加數據驗證

可以在 Apps Script 中添加數據驗證邏輯，例如：
- 檢查必填欄位
- 驗證數據格式
- 過濾重複數據

### 自動格式化

Apps Script 代碼已經包含了基本的格式化：
- 標題行加粗、藍色背景、白色文字
- 自動調整列寬

你可以根據需要添加更多格式化選項。
