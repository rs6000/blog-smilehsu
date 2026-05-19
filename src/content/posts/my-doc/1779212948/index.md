---
title: "Waline 評論系統 VPS 獨立部署與 Astro 整合實戰指南"
published: 2026-05-20
slug: "vps-waline-sqlite-astro-firefly"
description: "完整記錄如何在 VPS 環境下，使用 Node.js Vanilla 模式、SQLite 資料庫與 PM2 進程管理器獨立部署 Waline 評論系統，並完美整合至 Astro Firefly 主題中，內含常見的 8360 端口佔用與 SQLite 資料表缺失等邊緣錯誤（Edge Cases）排查筆記。"
tags: ["Waline", "Astro", "VPS", "SQLite", "Nginx"]
category: "網頁設計"
---



## 在 VPS 內建立 Waline 資料夾
首先，在你的系統（以 `root` 用戶為例）建立一個專屬的專案資料夾，並同時建立用於存放 SQLite 資料庫的 `data` 目錄：
```bash
mkdir -p /root/waline/data
cd /root/waline
````
---

## 在 waline 資料夾內安裝所需套件

在專案目錄下初始化 Node.js 環境，並安裝 Waline 核心、環境變數工具、資料庫驅動與全域進程管理器：

```
# 初始化專案
npm init -y

# 安裝核心套件與環境變數管理
npm install @waline/vercel dotenv

# 安裝 SQLite 驅動（確保獨立環境下能正常建立與寫入資料）
npm install sqlite3

# 全域安裝 PM2 進程管理器（如果系統以前安裝過可跳過）
npm install -p pm2 -g
```

---

## 設定 `.env` 環境變數

在 `/root/waline` 目錄下建立並編輯 `.env` 檔案：

```
nano .env
```

請將以下內容複製並修改為你的實際資料：

```
# 基礎設定
TZ=Asia/Taipei
SITE_NAME=我的知識庫
# ⚠️ 這裡必須填寫你的「Astro 部落格網址」作為 CORS 安全白名單
SITE_URL=https://your-astro-blog.com  

# 資料庫設定 (SQLite)
# ⚠️ 注意：路徑請指向「純資料夾路徑」，不要帶檔案名稱（如 waline.sqlite），讓系統自行在該目錄下生成
SQLITE_PATH=/root/waline/data

# 安全金鑰
JWT_TOKEN=請替換成一串自訂的很長且隨機的中英文字元作為加密金鑰

# 防機器人與惡意留言安全設定（選填，推薦開啟）
COMMENT_CAPTCHA=true
IP_LIMIT=15
```

## 設定 PM2 的啟動與 Port 佔用檢測

### 4-1 檢測 8360 Port 有無被其他服務佔用

Waline 預設會啟動在 `8360` 端口。在啟動前，請務必檢查該端口是否處於乾淨狀態，避免發生 `EADDRINUSE` 錯誤導致 PM2 無限閃退。

- **檢查端口狀態：**
```
lsof -i :8360
```
- **排查與強制釋放：** 如果畫面有輸出任何佔用的程序（例如舊的 node 或 waline 殘留進程），可以使用以下指令強制清空：
```
# 刪除 PM2 中可能重複登記的服務
pm2 delete waline
# 強制殺死所有正在執行的 node 背景程序
pkill -9 node
```

### 4-2 使用 PM2 乾淨啟動服務

確認端口乾淨後，透過 `dotenv` 掛載環境變數並啟動 Waline 的 Vanilla 入口檔案：

```
pm2 start node_modules/@waline/vercel/vanilla.js --name "waline" --node-args="-r dotenv/config"
```

- **事後查詢啟動指令內容：** 如果你忘記當初是怎麼啟動這個服務的，可以透過 PM2 的記憶體事後回查：
```
pm2 describe waline
```
_(在輸出的表格中，檢查 `script path` 與 `node args` 欄位，即可拼湊還原當初的完整指令。)_

---

## 設定 Nginx 反向代理與網域

為了讓外部能透過安全網域存取你的 Waline 後端（例如 `https://你的網址`），我們需要設定 Nginx 反向代理將流量導向 `8360` 端口。

### 5-1 建立 Nginx 設定檔：
```
nano /etc/nginx/sites-available/waline
```
### 5-2 貼入以下設定：
```
server {
    listen 80;
    server_name 你的網址; # 替換成你的 Waline 專屬子網域
    location / {
        proxy_pass [http://127.0.0.1:8360](http://127.0.0.1:8360);
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
### 5-3  啟用設定並重啟 Nginx：
```
ln -s /etc/nginx/sites-available/waline /etc/nginx/sites-enabled/
nginx -t
systemctl restart nginx
```

---

## 加掛 SSL 憑證

使用 Certbot（Let's Encrypt）為你的 Waline 獨立網域加上 HTTPS 加密：

```
# 安裝 Certbot（若尚未安裝）
apt install certbot python3-certbot-nginx

# 申請並自動設定 SSL
certbot --nginx -d 你的網址
```

依照畫面提示輸入 Email 並同意條款，完成後 Certbot 會自動修改 Nginx 設定並將 HTTP 流量重導向至安全加密的 HTTPS。

---

## 打開網頁與註冊管理員帳號

1.  使用瀏覽器打開註冊專用網域網址： `https://你的網址/ui/register`
2.  **填寫註冊表單**：輸入你預計使用的管理員使用者名稱、Email 和密碼，並點擊送出。
3.  **完成初始化**：第一個在該系統註冊的用戶會被 Waline 後端預設賦予 `administrator`（超級管理員）權限。註冊成功後，即可直接進入管理後台檢視評論與進行系統設定。

---

## 把 Waline 加入目前 Astro 網站使用的 Firefly 主題當中

Firefly 主題已經內建封裝了 Waline 的前端接口，**不需動到任何頁面組件（Layout）**，直接修改專案內的設定檔即可。

1.  打開你 Astro 專案中的評論設定檔，路徑通常為：`src/config/comment.ts`。
2.  將配置修改為以下結構：
    
```
export const commentConfig = {
  // 1. 啟用評論系統
  enable: true,
  
  // 2. 指定使用 waline
  type: 'waline',
  
  // 3. 填入 Waline 的具體參數
  waline: {
    // 必填：填入你在 VPS 上透過 Nginx 對外導出的 HTTPS 網址
    serverURL: 'https://你的網址', 
    
    // 選填：其他自訂引數
    lang: 'zh-TW',                      // 語系設定為繁體中文
    visitor: true,                      // 是否啟用文章瀏覽量統計
    dark: 'auto',                       // 暗黑模式跟隨系統或主題切換
    placeholder: '歡迎留下你的評論與想法！', // 輸入框預設提示字
    imageUploader: false                // 💡 關鍵：若不想開放訪客上傳圖片，可直接在此關閉前端按鈕
  }
}
```

存檔後，在 Astro 本地執行 `pnpm dev` 或重新編譯部署，即可在每篇文章底部看到完全獨立、自動依據文章 URL (Path) 切換的專屬評論區。

---

## 其他注意事項或備註說明

### 9-1. **500: SQLITE\_ERROR: no such table: wl\_Users 邊緣錯誤排查** 
如果你在第 7 步點擊註冊時噴出此錯誤，代表你所使用的開源 Waline 核心（如 ThinkJS 4.0 Alpha 測試版）未能透過 `vanilla.js` 觸發自動建立資料表的機制。
**解決辦法**：在 `/root/waline-server` 底下建立一個臨時的 `init-db.js` 腳本，利用 Node.js 獨立載入 `sqlite3` 驅動，手動執行 `CREATE TABLE IF NOT EXISTS wl_Users ...`、`wl_Comment`、`wl_Counter` 等核心 SQL 語句。手動將資料表建好後，執行 `pm2 restart waline`，即可順利繞過此問題完成註冊。

#### [如何手動建立sqlite檔案](https://gist.github.com/rs6000/233d5f71337a828cab9bb7bf9dcd296c)

#### 更簡單的做法:到 walink的 github倉庫下在空白的 [walink.sqlite](https://github.com/walinejs/waline/blob/main/assets/waline.sqlite)

### 9-2. **網址填寫對照表（核心邏輯）**
`.env` 檔案中的 `SITE_URL`：必須填寫 **Astro 部落格的網址**（用於後端通知跳轉與 CORS 安全驗證）。
Astro 前端設定中的 `serverURL`：必須填寫 **Waline 後端的網址**（`https://你的網址`）。

### 9-3. **評論資料遷移備忘** 
Waline 是嚴格認「網址路徑 (URL Path)」來區分文章評論的。未來若在 Astro 更改了文章的 `slug` 或檔名，舊文章的留言會因為路徑不匹配而隱藏。此時需登入 `https://你的網址/ui/` 後台，手動將該評論的 `Waline Path` 欄位修改為新路徑即可完成無損遷移。



---
