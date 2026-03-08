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

## Step 2. 無中生有：用一行指令建立含全套服務的 Laravel 專案

這時候你一定會問：「我不裝 PHP，那我要怎麼跑 `composer create-project` 建立專案？」
答案是：我們叫 Docker 派一個「臨時工」來幫我們做這件事，做完就讓他原地消失。

這裡提供兩種建立專案的做法，請依據你的「服務需求」來選擇：

### 做法A：一鍵懶人包（適合不需要 MongoDB 的開發者）
這也是剛接觸這套流派最震撼的一刻：**你只需要用一行指令，就能把 Laravel 框架，連同需要的資料庫 (MySQL)、快取 (Redis) 與搜尋引擎 (Meilisearch) 全部一次打包建好。**

打開 WSL 終端機 (Ubuntu)，切換到你想放專案的目錄：
```bash
mkdir -p ~/projects
cd ~/projects
```

執行這串由 Laravel 官方專為 Docker 玩家設計的神奇指令：
```bash
curl -s "https://laravel.build/my-clean-app?with=mysql,redis,meilisearch" | bash
```
*(注意：官方的 `laravel.build` 目前沒有原生支援 MongoDB。若需要 MongoDB，請改用下面的【做法B】！)*

看到 `Thank you! We hope you build something incredible.` 的訊息後就大功告成了。

### 做法B：原汁原味兩步法（適合需要 MongoDB 等進階選配的開發者）
如果你想要安裝官方最近才支援的 MongoDB，就必須先起一個純淨的骨架，再利用 Sail 的互動選單來勾選。

1. 先請臨時工幫你抓下最乾淨的 Laravel 框架：
```bash
mkdir -p ~/projects
cd ~/projects
docker run --rm -u "$(id -u):$(id -g)" -v $(pwd):/var/www/html -w /var/www/html laravelsail/php84-composer:latest composer create-project laravel/laravel my-clean-app
```
2. 進入專案，呼叫 Sail 安裝精靈的互動選單：
```bash
cd my-clean-app
docker run --rm -it -u "$(id -u):$(id -g)" -v $(pwd):/var/www/html -w /var/www/html laravelsail/php84-composer:latest php artisan sail:install
```
這時 Laravel 會跳出一個漂亮的終端機選單，裡面就可以看到 **`mongodb`**！你只要用上下鍵跟空白鍵打勾，按下 Enter 就一次幫你設定到好。

> [!IMPORTANT]
> **「等等！Sail 幫我選好 MongoDB 後，我可以直接開始寫 Code 了嗎？」**
> 
> **還差最後一步！** Laravel Sail 的互動選單 **「只會幫你準備好 MongoDB 的資料庫伺服器」** (也就是起一個 Docker 容器負責跑資料庫本身)。但是你的 Laravel 專案還需要懂得怎麼跟 MongoDB 溝通的「翻譯官」(Driver)。
> 
> 等到看完這篇文章的 Step 3 與 Step 4 (把設定檔 alias 搞定、容器啟動) 後，請務必記得執行以下指令安裝核心套件：
> ```bash
> sail composer require mongodb/laravel-mongodb
> ```
> 然後照著 [MongoDB 官方文件](https://www.mongodb.com/zh-cn/docs/drivers/php/laravel-mongodb/current/quick-start/download-and-install/) 的指示，修改你的 `config/database.php` 和 `.env` 檔，把連線與 Provider 註冊進去，這樣才算真正的 MongoDB 全解鎖喔！

無論你選做法A還是做法B，現在 `~/projects/my-clean-app` 裡面都有熱騰騰、全副武裝的 Laravel 專案了，重點是你的 WSL 裡面依然一滴 PHP 的痕跡都沒有。乾淨！

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

## Step 4. Sail 進階開發技巧 (強烈建議)

既然你的環境都在 Docker 容器裡，每次要下 PHP 或 Artisan 指令總不能每次都敲那串落落長的 `docker run...`。Laravel Sail 提供了一系列優雅的包裝，讓你可以像在本機開發一樣輕鬆。

### 🌟 強烈建議：設定 Sail Alias
為了不讓你每次都要打 `./vendor/bin/sail`，我們必須要幫 WSL 的終端機設一個短命名的別名 (alias)。

在終端機輸入打開你的設定檔：
```bash
nano ~/.bashrc
```
滑到文件最下面，補上這一行：
```bash
alias sail='sh $([ -f sail ] && echo sail || echo vendor/bin/sail)'
```
存檔離開後，執行 `source ~/.bashrc` 讓它生效。接下來的操作，你只要敲 `sail` 兩個字就能號令天下了！

### 💻 執行 PHP 與 Artisan 指令
應用程式雖然在 Docker 內執行，但透過 Sail 你可以完美遠端遙控：
- **查 PHP 版本或跑腳本：**
  ```bash
  sail php --version
  sail php script.php
  ```
- **執行 Artisan 指令：** (就像在本機敲 `php artisan` 一樣)
  ```bash
  sail artisan queue:work
  sail artisan migrate
  ```

### ➕ 專案開發中途，突然想新增服務？
假設你開發到一半，突然發現需要用 Redis 甚至 Memcached 怎麼辦？完全不用手寫設定檔，只要輸入：
```bash
sail artisan sail:add
```
熟悉的互動式選單又會跑出來，讓你輕鬆勾選需要的服務，它會自動幫你補齊 `docker-compose.yml`。

### 🔄 重建 Sail 映像檔
如果你的專案需要裝新的套件或是底層環境想更新，保險起見最好叫 Docker 清空快取重新打包一次：
```bash
sail down -v
sail build --no-cache
sail up -d
```
*(注意：`-v` 會一併把掛載的 Volume 清理掉，資料庫可能會重置，請在備份或確認不是正式機後再加這個參數！)*

---

## Step 5. Antigravity 完美協作起手式

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

## Step 6. 開發完成：測試區 (Testing) 與正式區 (Production) 的快速佈署

開發完了，要在測試機或正式機上線。這時候你會體會到 Docker 最大的好處：「本機怎麼跑，伺服器就怎麼跑」，永遠不會再有 "But it works on my machine!" 這種懸案。

既然所有的專案都是 Docker 化，我們伺服器端的準備也超級簡單：**Server 同樣只需要安裝好 Docker 就行了。**

> [!WARNING]
> **「為什麼這邊的指令又變回 `docker compose` 了？不能直接用 `sail artisan migrate` 嗎？」**
> 
> 這是一個超關鍵的觀念澄清：**Laravel Sail 是一個純粹的「本機開發輔助工具」！** 
> 
> Sail 的終生使命是讓你在本機開發時舒服一點（幫你掛載本機檔案、自動建立 root 權限讓你好安裝套件）。但在正式上線的大人世界裡，把這種充滿各種後門權限的開發用容器推上線是**非常危險**的。
> 
> 因此，到了 Testing 或 Production 伺服器，我們就不會再帶 Sail 玩了。我們會回歸最純粹、權限鎖緊的標準 `docker compose` 指令。你在本機用 `sail` 得到的開發快樂，在伺服器上就要以正規的 Docker 指令來執行佈署與資料庫遷移。

### 情境 A：快速測試機佈署 (Testing Environment)
如果只是想快速展示給客戶看，或是內部測試，最暴力的做法跟本機開發很像：
1. 透過 Git 把專案 Clone 到 Server 上。
2. 切換到專案目錄。
3. 直接使用掛載 Volume 的模式啟動（適合還要頻繁 git pull 更新的情境）：
   ```bash
   docker compose up -d
   ```
4. 跑一下資料庫更新 (如果有的話)。注意這裡我們沒有 `sail`，你要指定執行在那台名為 `laravel.test` 的容器裡面：
   ```bash
   docker compose exec laravel.test php artisan migrate
   ```
*(對照一下：這行指令相當於你在本機下達 `sail artisan migrate`，只不過我們現在是用 Docker 原生語法，叫 `laravel.test` 那個容器幫我們跑 `php artisan migrate`。)*

### 情境 B：正式環境佈署 (Production Environment)
到了正式機，我們就不建議用掛載 Volume (綁定本機目錄) 的方式了，因為有效能與安全的考量。最好的做法是把程式碼「打包」成正式的 Image。

1. **準備 Production 專用的設定檔**
   在專案根目錄準備一個 `Dockerfile` (專門用來把程式碼 COPY 進去並設定生產環境、關閉 Debug 模式) 和一個正式版的 `docker-compose.prod.yml`。

2. **在 Server 上的起手式**
   把程式碼拉下來後，執行打包與啟動：
   ```bash
   # 1. 建立正式版的 Image (會把你當下的程式碼打包鎖死進去)
   docker compose -f docker-compose.prod.yml build
   
   # 2. 啟動背景服務
   docker compose -f docker-compose.prod.yml up -d
   
   # 3. 執行正式機的資料庫遷移 (這裡指定跑在名為 app 或 laravel.test 的容器內)
   docker compose -f docker-compose.prod.yml exec laravel.test php artisan migrate --force
   ```

**給個小建議**：如果專案規模變大，誠心建議正式區的佈署可以搭配 CI/CD (例如 GitHub Actions 或 GitLab CI)。在 CI 上面把 Docker Image 打包好推到 Registry，你的 Server 只需要單純做 `docker pull` 跟 `docker compose up -d` 就好，這樣是最穩、最不怕髒的標準做法！

---

### 結語
這套「零污染開發流」一開始設定要轉一兩個彎，但相信我，幾個月後你會感謝現在的自己。當有一天你要換電腦，你只要把專案 Copy 過去，裝個 Docker 把機器開起來，三分鐘後你又可以繼續開發了。這就是工程師少數能自我掌控的療癒時刻啊哈哈！
