# 📒 InkFlow Notes — 專案開發交接文件 (`PROJECT_HANDOFF.md`)

> [!IMPORTANT]
> **給新對話 AI 助手 (Antigravity Agent) 的首要指令**：
> 本專案已部署於 GitHub 儲存庫 **`CodinSunil/inkflow-notes`**（分支：`main`），並透過 GitHub Pages 自動發布至 **`https://codinsunil.github.io/inkflow-notes/`**。
> 當使用者提出任何功能修改、UI 調整或 Bug 修復時，請直接修改儲存庫根目錄的 **`index.html`**，完成後直接執行 **`git commit` 與 `git push origin main`**，讓使用者的 iPad / iPhone 網頁端自動更新，無需讓使用者手動下載或拖曳檔案。

---

## 1. 專案基本資訊與連結

| 項目 | 內容 / 連結 |
| :--- | :--- |
| **專案名稱** | **InkFlow Notes**（個人專屬 iPad & iPhone 手寫同步筆記 PWA，致敬 GoodNotes / Notability） |
| **GitHub 儲存庫** | `https://github.com/CodinSunil/inkflow-notes` (`main` 分支) |
| **正式上線網址 (PWA)** | `https://codinsunil.github.io/inkflow-notes/` |
| **Google Drive 筆記資料夾** | `https://drive.google.com/drive/folders/1HmVNP7K25JhVuZCHErkvZKlF0AmK0J6O` |
| **Google Drive Folder ID** | `1HmVNP7K25JhVuZCHErkvZKlF0AmK0J6O`（使用者電腦端掛載路徑：`I:\我的雲端硬碟\Notability`） |
| **設計參考靈感** | `https://github.com/Jakeschincariol/replica-skill` |

---

## 2. 核心架構與已實作功能清單

### A. 跨裝置分層儲存與 30 天自動卸載引擎
1. **iPad 模式 (`deviceMode === 'ipad'`)**：
   - 筆記原檔預設儲存在 iPad 本機 (`ipadLocalDB` / `localStorage`)，離線可秒開書寫。
   - **30 天未開啟自動卸載 (`runThirtyDayAutoOffload`)**：當 `now - lastOpenedAt >= 30 天` 時，系統會先將本機筆劃與雲端進行 CRDT 聯集合併，接著清除 iPad 本機的 `pages` 內容（標記 `offloaded: true`）以釋放容量，完整原檔則安全保留在 Google Drive。再次點擊該筆記時會自動從雲端取回。
2. **iPhone 模式 (`deviceMode === 'iphone'`)**：
   - 手機端不佔用本機空間存原檔，平時只載入輕量筆記目錄 (`index`)。
   - 點擊某份筆記時，才動態將該筆記下載至記憶體暫存 (`state.iphoneTempCache`) 供檢視或臨時批註。
3. **三層雲端同步架構（Firestore 熱快取 ＋ Google Drive 冷儲存）**：
   - **熱資料中介層 (< 30 天)**：支援透過 Firebase Firestore REST API (`pushNoteToFirestoreMiddleware`) 進行毫秒級 JSON 同步。
   - **冷資料與永久原檔備份**：透過 Google Apps Script 加密橋接 (`syncViaGasBridge`) 或 Google Drive OAuth Token (`syncNotabilityFolderFromGDrive`)，雙向讀寫使用者的 `Notability` 雲端硬碟資料夾 (`1HmVNP7K25JhVuZCHErkvZKlF0AmK0J6O`)。

### B. 雙端無衝突 CRDT 筆劃合集演算法 ($A \cup B$)
- 實作於 `mergeNotesCRDT(noteA, noteB)`：
  - 每一筆手寫筆劃 (`stroke`) 與文字方塊 (`text`) 皆有獨立 UUID (`id`)、時間戳 (`updatedAt`) 與墓碑標記 (`deleted: boolean`)。
  - 當 iPad 與 iPhone 同時離線修改同一份筆記並丟回雲端時，兩邊筆劃集合會執行**數學聯集 ($A \cup B$)**，並保留 `deleted: true` 墓碑紀錄（防止已擦除的筆劃復活），確保兩邊寫的內容永遠不會互相覆蓋。

### C. iPad / iPhone 觸控與 Apple Pencil 專屬手寫體驗
1. **硬體觸控自動分流 (`e.pointerType`)**：
   - 預設啟用 `fingerAlwaysPan: true`：當偵測到 `e.pointerType === 'pen'`（Apple Pencil）時，0 延遲直接下筆寫字（支援感壓粗細變化與二次貝茲曲線平滑化）。
   - 當偵測到 `e.pointerType === 'touch'`（手指）時，單指自動滑動捲頁、雙指無段縮放畫布（38% ～ 300%），絕不會誤畫出雜線。
2. **Apple Pencil 點兩下切換橡皮擦 (`togglePencilEraserDoubleTap`)**：
   - 支援 Apple Pencil 第二代/Pro 筆身雙擊事件（以及鍵盤 `E` / `F19` 快捷鍵），在「目前畫筆（鋼筆/螢光筆）」與「橡皮擦」之間瞬時切換，並顯示懸浮 HUD 提示。
3. **工具列與版面**：
   - 支援鋼筆、螢光筆（`multiply` 混色不遮字）、墓碑橡皮擦、套索圈選移動工具、文字輸入工具。
   - 支援 5 種筆記紙張模板：方格紙 (`grid`)、康乃爾筆記 (`cornell`)、橫線紙 (`ruled`)、點陣紙 (`dotted`)、空白紙 (`blank`)。
   - 左側筆記本抽屜 (`#notebook-sidebar`) 支援手機與電腦版收合 (`sidebar.style.display = state.sidebarOpen ? 'flex' : 'none'`)。

### D. 純前端 Notability `.note` 深度解析引擎
- 實作於 `unzipNotabilityArchive`、`parseBinaryPlist`、`collectAllPlistsFromBytes`、`convertNotabilityPlistToInkFlow`、`enrichNoteWithEmbeddedPdfAndImages`：
  1. **瀏覽器端直接解壓縮 `.note` (ZIP 封包)**：使用非阻塞 `DecompressionStream('deflate-raw')` 解開 `.plist`、`.pdf` 與 `.png`/`.jpg` 圖片。
  2. **完整解開 `NSKeyedArchiver` (`bplist00`) 的 `CF$UID` (`0x80`) 間接參照**：自動讀取根字典的 `$objects` 陣列，透過 `deref()` 與 `resolveDataBuffer()` 穿透多層 `UID` 指標與巢狀 `bplist00` 子封包，精準還原 `curvespoints`、`curvesnumpoints`、`curveswidths`、`curvescolors`、`curvesfractionalwidths` 與文字區塊。
  3. **通用 `Float32` 筆跡軌跡探測器 (Fallback)**：若遇到新版 Notability 改動字典 Key 名稱，自動掃描所有 `NSData` 緩衝區中的連續 2D `(x, y)` 座標軌跡並重建筆劃。
  4. **內嵌 PDF 講義與圖片渲染 (`pdf.js`)**：自動將 `.note` 內的 `PDFs/*.pdf` 逐頁渲染成筆記背景 (`page.bgImage`)，並將手寫筆跡疊加於對應頁面上。

---

## 3. Google Apps Script 私人密碼鎖雲端橋接腳本 (`google-drive-bridge.gs`)

若使用者需要重新部署或修改 Google Drive 免 Token 自動同步橋接，請使用以下完整腳本（貼至 `https://script.google.com/` 部署為網頁應用程式）：

```javascript
const NOTABILITY_FOLDER_ID = '1HmVNP7K25JhVuZCHErkvZKlF0AmK0J6O';
const SECRET_PASSKEY = 'inkflow-sunil-2026'; // 使用者自訂的專屬通行密碼鎖

function verifyPasskey(reqKey) {
  return reqKey && reqKey === SECRET_PASSKEY;
}

function doGet(e) {
  try {
    const key = (e && e.parameter && e.parameter.key) || '';
    if (!verifyPasskey(key)) {
      return jsonOut({ ok: false, error: '403 Forbidden: 通行密碼錯誤，拒絕存取雲端硬碟' });
    }
    const action = (e && e.parameter && e.parameter.action) || 'list';
    const folder = DriveApp.getFolderById(NOTABILITY_FOLDER_ID);
    if (action === 'list') {
      const files = folder.getFiles();
      const items = [];
      while (files.hasNext()) {
        const f = files.next();
        const name = f.getName();
        if (name.endsWith('.note') || name.endsWith('.zip') || name.endsWith('.inkflow.json')) {
          items.push({ id: f.getId(), name: name, updatedAt: f.getLastUpdated().getTime(), size: f.getSize() });
        }
      }
      return jsonOut({ ok: true, folderName: folder.getName(), files: items });
    }
    if (action === 'get') {
      const file = DriveApp.getFileById(e.parameter.fileId);
      const name = file.getName();
      if (name.endsWith('.inkflow.json')) {
        return jsonOut({ ok: true, type: 'inkflow_json', name: name, data: JSON.parse(file.getBlob().getDataAsString('UTF-8')) });
      } else {
        return jsonOut({ ok: true, type: 'note_binary', name: name, base64: Utilities.base64Encode(file.getBlob().getBytes()) });
      }
    }
    return jsonOut({ ok: false, error: 'Unknown action' });
  } catch (err) {
    return jsonOut({ ok: false, error: String(err) });
  }
}

function doPost(e) {
  try {
    const payload = JSON.parse(e.postData.contents);
    if (!verifyPasskey(payload.key)) {
      return jsonOut({ ok: false, error: '403 Forbidden: 通行密碼錯誤，拒絕寫入雲端硬碟' });
    }
    const folder = DriveApp.getFolderById(NOTABILITY_FOLDER_ID);
    const fileName = (payload.noteData.title || payload.noteId) + '.inkflow.json';
    const existing = folder.getFilesByName(fileName);
    const content = JSON.stringify(payload.noteData);
    if (existing.hasNext()) {
      const f = existing.next();
      f.setContent(content);
      return jsonOut({ ok: true, fileId: f.getId(), updated: true });
    } else {
      const f = folder.createFile(fileName, content, MimeType.PLAIN_TEXT);
      return jsonOut({ ok: true, fileId: f.getId(), created: true });
    }
  } catch (err) {
    return jsonOut({ ok: false, error: String(err) });
  }
}

function jsonOut(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj)).setMimeType(ContentService.MimeType.JSON);
}
```

---

## 4. 給使用者的交接步驟（如何在 Project 中接續開發）

1. 在 Antigravity 左側或首頁建立/進入你的 **Project**。
2. 在 Project 的 **Repositories（連結儲存庫）** 中，選取你剛建好的 **`CodinSunil/inkflow-notes`**。
3. 在該 Project 內開啟一個新對話，直接把這份 [PROJECT_HANDOFF.md](file:///home/antigravity/.gemini/jetski/brain/v1/default/jetski_remote_agents_3p/782929ca-d8fc-4fba-894c-9a5a7d423ae3/PROJECT_HANDOFF.md) 的內容貼給新的 AI 助手（或上傳這份 `.md` 檔）。
4. 之後你在 iPad / iPhone 實際使用上有任何想改的地方，只要跟新對話說一聲，AI 就會直接修改 `CodinSunil/inkflow-notes` 裡的 `index.html` 並自動 Push 更新你的網頁 App！
