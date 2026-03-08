# 潔癖工程師的終極開發環境：Windows + WSL + Docker + Antigravity (零污染實戰指南)

做工程很累，有時候比起寫程式，搞定開發環境更讓人崩潰。尤其是 Windows 上面，一下裝 PHP，一下裝 Node.js 環境變數，版本衝突起來真的會想直接重灌電腦。我以為我快把環境搞定了，結果跑別的專案又跳 Error... 這就是人生吧！

這也是為什麼我們最後決定走上這條「絕對純淨」的道路。這篇文章要教你的，是一個非常極端但療癒的流派：
我們在一台全新的 Windows 電腦上，**只准裝 WSL、Docker 和 Antigravity**。什麼本機的 PHP、Node.js、Python，通通不准裝！我們要讓 Windows 客廳保持乾淨，所有的髒活，都交給 Docker 這個貨櫃去處理。

---

## Step 1. 打造純淨地基 (基礎環境準備)

1. **啟用 WSL2 (安裝 Ubuntu)**
   打開 Windows 的 PowerShell (以系統管理員身分執行)，輸入：
   ```powershell
   wsl --install
   ```
   裝完後重新開機，你就有一個純淨的 Ubuntu 環境了。

2. **安裝 Docker Desktop**
   去 Docker 官網下載安裝檔。安裝好之後，進去設定 (Settings) -> Resources -> WSL Integration，確保你的 `Ubuntu` 有被勾起來。這樣 WSL 裡面就能直接呼叫 Docker 了。

3. **安裝 Antigravity**
   這是我們最強大的 AI 協作編輯器。裝好就好，先放著。

*(重要提醒：到目前為止，你的電腦還是香的，千萬不要手癢跑去 WSL 裡面敲什麼 `apt install php`！)*

---

## Step 2. 無中生有：用「拋棄式貨櫃」建立 Laravel 專案

這時候你一定會問：「我不裝 PHP，那我要怎麼跑 `composer create-project` 建立專案？」
答案是：我們叫 Docker 派一個「裝好 PHP 的臨時工」來幫我們做這件事，做完就讓他原地消失。

打開 WSL 終端機 (Ubuntu)，切換到你想放專案的目錄：
```bash
mkdir -p ~/projects
cd ~/projects
```

執行這串神奇的指令 (這是我們的臨時工)：
```bash
docker run --rm \
    -u "$(id -u):$(id -g)" \
    -v $(pwd):/var/www/html \
    -w /var/www/html \
    laravelsail/php83-composer:latest \
    composer create-project laravel/laravel my-clean-app
```
*這行指令的白話文是：跑一個包含 composer 的 PHP 8.3 容器，把目前資料夾掛載進去。下載好 Laravel 後，`--rm` 讓他原地自爆。*

現在 `~/projects/my-clean-app` 裡面就有熱騰騰的 Laravel 專案了，而且你的 WSL 裡面依然一滴 PHP 的痕跡都沒有。乾淨！

---

## Step 3. 專案的 Docker 化與啟動

接下來我們要讓這個專案跑起來。其實 Laravel 新版自帶的 Sail 已經幫我們把 `docker-compose.yml` 寫得差不多了。

進入專案資料夾：
```bash
cd my-clean-app
```

啟動專案的 Docker 環境 (Nginx, PHP, MySQL 等等)：
```bash
./vendor/bin/sail up -d
```
*(這也是叫 Docker 依照設定檔把開發需要的伺服器跑起來。啟動成功後，你打開 Windows 瀏覽器輸入 `http://localhost` 應該就能看到 Laravel 首頁了。)*

### 📁 專案的資料夾架構與檔案樹
在我們的「零污染」流派下，你的 Windows 和 WSL 根目錄都超級乾淨，所有的髒活都在專案資料夾內。你的專案結構看起來會像這樣：

```text
\\wsl$\Ubuntu\home\你的使用者名稱\projects\
└── my-clean-app/                # 你的 Laravel 專案根目錄
    ├── app/                     # Laravel 核心程式碼
    ├── bootstrap/
    ├── config/
    ├── database/
    ├── public/                  # 網站公開目錄 (引導至 index.php)
    ├── resources/               # 前端資源 (視圖、未編譯的 JS/CSS)
    ├── routes/                  # 路由設定 (web.php, api.php 等)
    ├── storage/
    ├── tests/
    ├── vendor/                  # Composer 裝的套件 (裝在 Docker 裡，但檔案存在這)
    ├── .env                     # 你的環境變數設定檔 (例如 DB 連線密碼)
    ├── composer.json            # 擴充套件清單
    ├── docker-compose.yml       # 決定你要起哪些 Docker 容器 (Nginx, MySQL, Redis...)
    └── artisan                  # Laravel 專用指令工具
```
在這裡面，`docker-compose.yml` 就像是專案的「基礎建設草圖」，只要有它，不管換到哪台電腦，下達 `docker compose up -d` 就能原音重現一模一樣的開發環境。

---

## Step 4. Antigravity 完美協作起手式

現在網頁跑起來了，我們要開始改 Code。以往可能會想用 devcontainer，但我們直接走更穩定直白的路線。

打開 Antigravity，選擇開啟資料夾 (Open Folder)。
請直接在路徑列輸入 WSL 的網路磁碟機路徑，例如：
`\\wsl$\Ubuntu\home\你的使用者名稱\projects\my-clean-app` 
*(有時候是 `\\wsl.localhost\Ubuntu\...` 視你的 Windows 版本而定)*

Antigravity 會直接連線並讀取 WSL 裡面的檔案。你接下來就在 Antigravity 裡面暢快地寫 Code，存檔的瞬間，因為 Docker volume 檔案掛載的關係，網頁重整就立刻生效。

**這就是我們的日常開發循環：**
1. 啟動 WSL
2. 進入專案資料夾跑 `sail up -d` (或者你自己寫好的 `docker compose up -d`)
3. 用 Antigravity 開啟 WSL 上的資料夾進行開發修改
4. 下班關機前敲個 `sail down`，乾乾淨淨。

---

## Step 5. 開發完成：測試區 (Testing) 與正式區 (Production) 的快速佈署

開發完了，要在測試機或正式機上線。這時候你會體會到 Docker 最大的好處：「本機怎麼跑，伺服器就怎麼跑」，永遠不會再有 "But it works on my machine!" 這種懸案。

既然所有的專案都是 Docker 化，我們伺服器端的準備也超級簡單：**Server 同樣只需要安裝好 Docker 就行了。**

### 情境 A：快速測試機佈署 (Testing Environment)
如果只是想快速展示給客戶看，或是內部測試，最暴力的做法跟本機開發很像：
1. 透過 Git 把專案 Clone 到 Server 上。
2. 切換到專案目錄。
3. 直接使用掛載 Volume 的模式啟動（適合還要頻繁 git pull 更新的情境）：
   ```bash
   docker compose up -d
   ```
4. 跑一下資料庫更新 (如果有的話)：
   ```bash
   docker compose exec app php artisan migrate
   ```
效果立竿見影，客戶馬上看得到。

### 情境 B：正式環境佈署 (Production Environment)
到了正式機，我們就不建議用掛載 Volume (綁定本機目錄) 的方式了，因為有效能與安全的考量。最好的做法是把程式碼「打包」成正式的 Image。

1. **準備 Production 專用的設定檔**
   在專案根目錄準備一個 `Dockerfile` (專門用來把程式碼 COPY 進去並設定生產環境的 PHP/Nginx) 和一個 `docker-compose.prod.yml`。

2. **在 Server 上的起手式**
   把程式碼拉下來後，執行打包與啟動：
   ```bash
   # 1. 建立正式版的 Image (會把你當下的程式碼打包進去)
   docker compose -f docker-compose.prod.yml build
   
   # 2. 啟動背景服務
   docker compose -f docker-compose.prod.yml up -d
   
   # 3. 執行正式機的資料庫遷移 (通常只需做一次)
   docker compose -f docker-compose.prod.yml exec app php artisan migrate --force
   ```

**給個小建議**：如果專案規模變大，誠心建議正式區的佈署可以搭配 CI/CD (例如 GitHub Actions 或 GitLab CI)。在 CI 上面把 Docker Image 打包好推到 Registry，你的 Server 只需要單純做 `docker pull` 跟 `docker compose up -d` 就好，這樣是最穩、最不怕髒的標準做法！

---

### 結語
這套「零污染開發流」一開始設定要轉一兩個彎，但相信我，幾個月後你會感謝現在的自己。當有一天你要換電腦，你只要把專案 Copy 過去，裝個 Docker 把機器開起來，三分鐘後你又可以繼續開發了。這就是工程師少數能自我掌控的療癒時刻啊哈哈！
