# iPad 借用記錄表 — GitHub Pages + Firebase 部署指南

這個資料夾內的四個檔案（`index.html`、`style.css`、`app.js`、`firebase-config.js`）組成一個完整、獨立的網頁版本，可以直接放上 GitHub Pages 公開瀏覽，**不再需要透過 Claude Artifact**。

## 為什麼需要 Firebase？

原本的版本靠 Claude Artifact 平台的後台幫大家同步借用記錄。GitHub Pages 本身只是「放檔案」的靜態網頁託管，沒有後台，所以無法讓老師和 IT 組看到同一份即時記錄。這裡改用 **Firebase Firestore**（Google 提供的免費雲端資料庫）來擔任這個角色：所有借用記錄、iPad 設定、課節設定都存在 Firestore，每部裝置打開網頁時會即時讀取同一份資料，跟原本的體驗一樣。

Firebase 對一間小學的用量來說完全在免費額度內，也不需要自己維護伺服器。

---

## 第一步：建立 Firebase 專案（約 5 分鐘）

1. 用您的 Google 帳號登入 [Firebase Console](https://console.firebase.google.com/)。
2. 按「新增專案」，輸入專案名稱（例如「lwcps-ipad-booking」），一路按繼續（不需要開啟 Google Analytics）。
3. 專案建立完成後，在左側選單找到 **Build → Firestore Database**，按「建立資料庫」。
   - 位置建議選 `asia-east1`（台灣）或 `asia-east2`（香港），離香港最近、速度較快。
   - 安全性規則先選「正式環境模式」（Production mode），稍後會手動貼規則。
4. 資料庫建立後，切到 Firestore 頁面上方的 **「規則」（Rules）** 分頁，把內容整段換成：

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /ipadBooking/state {
         allow read, write: if true;
       }
       match /{document=**} {
         allow read, write: if false;
       }
     }
   }
   ```

   按「發布」（Publish）。

   > **關於安全性的提醒**：這組規則讓「任何知道網址的人」都能讀寫這一份借用記錄（跟原本 Claude Artifact 版本的行為類似——沒有登入系統，設定密碼只是介面上的一道門檻，不是真正的帳號驗證）。對一間學校內部使用的借用表來說這通常是可接受的取捨，但如果之後想加強保護（例如要求 Google 帳號登入才能寫入），可以再另外設定 Firebase Authentication，屆時请告訴我，我可以協助調整。

5. 回到左上角「專案總覽」，按網頁圖示 `</>` 新增一個「Web app」，名稱隨意（例如「booking-web」），不需要勾選 Firebase Hosting。
6. 這時畫面會顯示一段 `firebaseConfig = {...}` 的設定物件，長得像這樣：

   ```js
   const firebaseConfig = {
     apiKey: "AIzaSy...",
     authDomain: "lwcps-ipad-booking.firebaseapp.com",
     projectId: "lwcps-ipad-booking",
     storageBucket: "lwcps-ipad-booking.appspot.com",
     messagingSenderId: "...",
     appId: "1:...:web:..."
   };
   ```

   把這 6 行的值複製起來，貼到這個資料夾裡的 **`firebase-config.js`** 檔案中對應的位置（取代範例文字），存檔。

---

## 第二步：上傳到 GitHub

1. 在您現有的 GitHub 帳號建立一個新的 Repository（例如命名為 `ipad-booking`），設為 **Public**（GitHub Pages 的免費方案需要公開倉庫；程式碼本身沒有機密資料，Firebase 金鑰在網頁前端本來就會公開，這是正常的）。
2. 把這個資料夾裡的 4 個檔案（`index.html`、`style.css`、`app.js`、已經填好金鑰的 `firebase-config.js`）上傳到這個倉庫的根目錄。
   - 最簡單的方式：在 GitHub 網頁介面按「Add file → Upload files」，把四個檔案拖進去，按「Commit changes」。
3. 到倉庫的 **Settings → Pages**，在「Build and deployment」的 Source 選擇「Deploy from a branch」，Branch 選 `main`（或 `master`）／`/ (root)`，按 Save。
4. 等 1–2 分鐘，GitHub 會給您一個網址，格式通常是：

   ```
   https://<您的GitHub帳號>.github.io/<倉庫名稱>/
   ```

   打開它，應該就能看到跟之前一樣的借用表介面了。把這個網址分享給所有老師和 IT 組即可。

---

## 第三步：測試

1. 用電腦瀏覽器開啟網址，新增一筆借用記錄。
2. 用手機（或另一台電腦／無痕視窗）打開同一個網址，確認剛剛新增的記錄有出現——這就代表 Firebase 同步成功了。
3. 如果畫面最上方出現「離線模式：尚未連接資料庫」的橫幅，通常代表 `firebase-config.js` 裡的設定值還沒填、填錯，或 Firestore 規則還沒發布，請回頭檢查第一步。

---

## 之後要修改網頁內容怎麼辦？

- **修改樣式或版面**：直接編輯 `style.css` 或 `app.js`，重新上傳到 GitHub 覆蓋舊檔即可，GitHub Pages 會自動更新（通常 1 分鐘內生效）。
- **iPad 批次 / 課節設定 / IT 預留 / 清除記錄**：不需要動到程式碼，跟以前一樣直接在網頁右上角齒輪圖示的「系統設定」內操作，設定密碼預設是 `535800`（可以在 `app.js` 開頭的 `defaultState()` 裡修改 `settingsPassword`，但要注意：這個密碼只有在 Firestore 資料庫還是全新、空的時候才會生效一次；之後密碼是存在 `app.js` 本機檔案裡的常數，並不會同步到 Firestore，若要更改密碼，直接修改 `app.js` 內的值再重新上傳即可）。

如果之後想把這個 GitHub Pages 版本换成自訂網域（例如學校自己的網址），或想加上老師登入驗證，都可以再回來討論。
