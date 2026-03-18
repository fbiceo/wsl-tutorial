# 潔癖工程師的終極開發環境：Windows + WSL + Docker + Antigravity (零污染實戰指南)

做工程很累，有時候比起寫程式，搞定開發環境更讓人崩潰。尤其是 Windows 上面，一下裝 PHP，一下裝 Node.js 環境變數，版本衝突起來真的會想直接重灌電腦。我以為我快把環境搞定了，結果跑別的專案又跳 Error... 這就是人生吧！

這也是為什麼我們最後決定走上這條「絕對純淨」的道路。這篇文章要教你的，是一個非常極端但療癒的流派：
我們在一台全新的 Windows 電腦上，**只准裝 WSL、Docker 和 Antigravity**。什麼本機的 PHP、Node.js、Python，通通不准裝！我們要讓 Windows 客廳保持乾淨，所有的髒活，都交給 Docker 這個貨櫃去處理。

> [!NOTE]
> 這是專為 **Backend (Laravel + FrankenPHP)** 打造的開發指南。
> 如果你負責的是前端 (Vue / Vite) 介面開發，請前往：
> 👉 **[前端篇：Vue / Vite + Docker 零污染實戰](./VUE_VITE_FRONTEND.md)**

---

## 目錄
- [Step 1. 打造純淨地基 (基礎環境準備)](#step-1-打造純淨地基-基礎環境準備)
- [Step 2. 無中生有：建立全套 Laravel 專案與安裝 FrankenPHP](#step-2-無中生有：建立全套-laravel-專案與安裝-frankenphp)
- [Step 3. 專案的 Docker 化：統一天下的配置](#step-3-專案的-docker-化：統一天下的配置)
- [Step 4. 日常開發與進階技巧](#step-4-日常開發與進階技巧)
- [Step 5. Antigravity 完美協作起手式](#step-5-antigravity-完美協作起手式)
- [Step 6. 開發完成：測試區 (Testing) 與正式區 (Production) 的快速佈署](#step-6-開發完成：測試區-testing-與正式區-production-的快速佈署)
- [Step 7. 換電腦怎麼辦？(從 Git Clone 接續開發)](#step-7-換電腦怎麼辦？從-git-clone-接續開發)

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

[↑ 回到目錄](#目錄)

---

## Step 2. 無中生有：建立全套 Laravel 專案與安裝 FrankenPHP

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

> [!WARNING]
> **修復 Sail 無法執行的問題**
> 
> 因為我們是用純 Docker 的方式建立專案，不管你選做法 A 還是 B，都會出現 sail 無法執行的問題。請進入專案目錄，並執行下面解決方式才能順利運作：
> ```bash
> cd my-clean-app
> 
> # 1. 先把 sail 套件裝回來
> docker run --rm -it -u "$(id -u):$(id -g)" -v $(pwd):/var/www/html -w /var/www/html laravelsail/php84-composer:latest composer require laravel/sail --dev --ignore-platform-reqs
> 
> # 2. 接著執行你剛剛跑的 sail:install 來初始化設定檔 (docker-compose.yml 等)
> docker run --rm -it -u "$(id -u):$(id -g)" -v $(pwd):/var/www/html -w /var/www/html laravelsail/php84-composer:latest php artisan sail:install
> 
> # 3. 初始化完成後，就可以順利啟動了
> ./vendor/bin/sail up -d
> ```

### 安裝頂級引擎：FrankenPHP (Laravel Octane)
因為我們堅持正式機跟開發機都要最乾淨、最高效能，所以我們不用傳統的 Nginx + PHP-FPM，而是直接選用用 Go 寫的現代化伺服器 **FrankenPHP**。它直接內建了 Web Server 跟 PHP 執行環境，讓網頁速度飛快。

剛建好的專案已透過上方步驟啟動 Laravel Sail，我們接著借用它的環境把 Octane 裝好：

```bash
# 確保你還在 my-clean-app 目錄下

# 1. 安裝 Octane 套件
./vendor/bin/sail composer require laravel/octane

# 2. 選擇 FrankenPHP 作為引擎
./vendor/bin/sail artisan octane:install --server=frankenphp

# 3. 安裝完畢，把暫時的環境關掉，準備我們自己的效能配置
./vendor/bin/sail down
```

[↑ 回到目錄](#目錄)

---

## Step 3. 專案的 Docker 化：統一天下的配置

為了達成從開發到上線都能保持極致輕量與一致，我們要徹底拋棄原本 Sail 那套依賴許多服務的配置，改寫自己的 **FrankenPHP 專用 Dockerfile**，並用一套 `docker-compose.yml` 統一天下！

> [!NOTE]
> **等等，`Dockerfile` 跟 `docker-compose.yml` 到底差在哪？**
> 
> 如果用開餐廳來比喻：
> - **`Dockerfile` 是一份「廚師的食譜與訓練手冊」**：它負責描述「單一個貨櫃 (Container)」要怎麼被做出來。例如我們這份食譜寫著：要用 Go 語言的 FrankenPHP 當基底、要安裝哪些 PHP 擴充、要把 Laravel 程式碼放進哪個資料夾。建置出來的東西叫做 `Image`（映像檔）。
> - **`docker-compose.yml` 則是「餐廳的經理與營運計畫」**：它負責統籌整間餐廳要請幾個廚師、誰負責做什麼。你可以用它來規定：我要起一個命名為 `app` 的服務（並且指定使用剛才那份 `Dockerfile` 來建立出來）、要開放哪幾個 Port 給外面的客人連線、還要不要掛載資料庫 `db` 服務。
>
> 簡單來說：**`Dockerfile` 決定了「內容物是什麼」，而 `docker-compose.yml` 決定了這些內容物要「怎麼合作跟對外營業」。**

### 1. 準備 FrankenPHP 專用 Dockerfile（多階段建置版）
在專案根目錄建立 `Dockerfile`，這份食譜使用了 Docker 的「**多階段建置 (Multi-stage Build)**」技巧：先請 Node.js 臨時工編譯前端資源 (Vite)，再把成果搬進最終的 FrankenPHP 映像檔，讓正式版 Image 不會殘留 `node_modules` 等開發垃圾，保持極致瘦身：

```dockerfile
# ==========================================
# 階段 1：編譯前端資源 (使用 Node.js)
# ==========================================
FROM node:20-alpine AS frontend

WORKDIR /app

# 複製 package 檔案以利用 Docker 快取 (只要 package.json 沒變，就不用重新 npm install)
COPY package.json package-lock.json* ./
RUN npm install

# 複製其餘原始碼進行打包 (例如 Laravel Vite 產出的 public/build)
COPY . .
RUN npm run build


# ==========================================
# 階段 2：建立正式執行環境 (FrankenPHP)
# ==========================================
FROM dunglas/frankenphp:php8.4

# 加入一些 LABEL 讓同事知道這是什麼 (人類友善)
LABEL maintainer="你的名字 <your.email@example.com>"
LABEL description="Laravel + FrankenPHP 終極純淨版"

# 設定環境變數，讓 FrankenPHP 知道跑在 8000 port
ENV SERVER_NAME=":8000"

# 安裝額外的 PHP 擴展
# (FrankenPHP 內建 install-php-extensions 工具，不用自己刻 apt-get，超佛心)
RUN install-php-extensions \
    pdo_mysql \
    gd \
    mbstring \
    xml \
    curl \
    intl \
    zip \
    mongodb \
    pcntl \
    redis

# 安裝 Composer (從官方映像檔複製過來)
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /app

# 將目前專案原始碼 COPY 進 Image
COPY . /app

# 🔑 【關鍵】從「階段 1」將編譯好的前端靜態資源直接複製過來
# 這樣正式版就自帶編譯完的 CSS/JS，不需要在伺服器上裝 Node.js！
COPY --from=frontend /app/public/build /app/public/build

# 複製 Entrypoint 腳本並設定可執行權限
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

# 執行正式版套件安裝 (不含 dev 依賴，保持瘦身)
RUN composer install --optimize-autoloader --no-dev
RUN php artisan storage:link

# 設定 Entrypoint 腳本，讓它在每次容器啟動時自動修復 Storage 權限
ENTRYPOINT ["docker-entrypoint.sh"]

# ⚡ 預設啟動 Laravel Octane 榨出極限效能
CMD ["php", "artisan", "octane:frankenphp", "--host=0.0.0.0", "--port=8000", "--admin-port=2019"]
```

> [!NOTE]
> **「ENTRYPOINT 跟 CMD 到底差在哪？為什麼要拆成兩行？」**
>
> 簡單來說：`ENTRYPOINT` 是「每次容器啟動時一定會先跑的前置腳本」，而 `CMD` 則是「前置腳本跑完後，接著要執行的主程式」。
> 拆開的好處是：我們可以在 `docker-entrypoint.sh` 裡面做一些必要的準備工作（像是修復 Storage 權限），做完之後才把控制權交給 Octane 伺服器。
> 而在開發環境裡，我們還能透過 `docker-compose.yml` 的 `command` 參數輕鬆覆寫整段 `CMD`，例如加上 `--watch` 來啟用熱重載，非常彈性！

### 2. 準備 Entrypoint 啟動腳本
在專案根目錄建立 `docker-entrypoint.sh`，這個腳本會在每次容器啟動時自動執行，幫我們處理 Storage 目錄的初始化與權限修復：

```bash
#!/bin/sh
set -e

# ==========================================
# Laravel Startup Preparation Script
# ==========================================

echo "🚀 Starting Laravel startup preparation..."

# 1. 確保 /app/storage 目錄存在
# (如果是第一次掛載全新的 volume，資料夾可能不全)
mkdir -p /app/storage/app/public
mkdir -p /app/storage/framework/cache/data
mkdir -p /app/storage/framework/sessions
mkdir -p /app/storage/framework/testing
mkdir -p /app/storage/framework/views
mkdir -p /app/storage/logs

# 2. 自動修復 storage 目錄權限
# 這一步非常重要！如果是開發者用 root 建立的檔案或 Docker Volume
# 會導致 www-data 無法寫入 Log 或是上傳圖片
echo "🔒 Fixing permissions for /app/storage..."
chown -R www-data:www-data /app/storage
chown -R www-data:www-data /app/bootstrap/cache

echo "✅ Storage initialization completed."

# ==========================================
# 啟動原本的 CMD (把控制權交給 Octane)
# ==========================================
echo "✅ Preparation complete. Starting server..."
exec "$@"
```

> [!WARNING]
> **別忘了這個腳本的換行格式！**
> 如果你是在 Windows 上編輯這個檔案，請確認存檔時使用 **LF** 換行（不是 CRLF），否則 Linux 容器會無法執行它。在 Antigravity 底部的狀態列可以點擊切換換行格式。

### 3. 準備統一天下的 `docker-compose.yml`
打開專案內的 `docker-compose.yml`，把裡面原本 Sail 幫你產生的東西全部清空，換成這套配置：

```yaml
services:
    app:
        build:
            context: .
            dockerfile: Dockerfile
        restart: unless-stopped
        ports:
            - "8000:8000"    # HTTP (對應 Dockerfile 的 SERVER_NAME / --port)
        environment:
            - APP_ENV=${APP_ENV:-local}
            - APP_DEBUG=${APP_DEBUG:-true}
        volumes:
            # 【重要】：開發環境把這行打開，掛載本機目錄
            # 如果是「正式上線」，請把這行註解掉，讓它讀取 Dockerfile 打包好的純淨原始碼
            - .:/app
        # 開發環境必備：覆寫 Dockerfile 指令加入 --watch 參數達成 Hot Reloading
        command: [ "php", "artisan", "octane:frankenphp", "--host=0.0.0.0", "--port=8000", "--admin-port=2019", "--watch" ]
        tty: true
        networks:
            - sail

    mysql:
        image: 'mysql:8.4'
        ports:
            - '${FORWARD_DB_PORT:-3306}:3306'
        environment:
            MYSQL_ROOT_PASSWORD: '${DB_PASSWORD}'
            MYSQL_ROOT_HOST: '%'
            MYSQL_DATABASE: '${DB_DATABASE}'
            MYSQL_USER: '${DB_USERNAME}'
            MYSQL_PASSWORD: '${DB_PASSWORD}'
            MYSQL_ALLOW_EMPTY_PASSWORD: 1
            MYSQL_EXTRA_OPTIONS: '${MYSQL_EXTRA_OPTIONS:-}'
        volumes:
            - 'sail-mysql:/var/lib/mysql'
            - './vendor/laravel/sail/database/mysql/create-testing-database.sh:/docker-entrypoint-initdb.d/10-create-testing-database.sh'
        networks:
            - sail
        healthcheck:
            test:
                - CMD
                - mysqladmin
                - ping
                - '-p${DB_PASSWORD}'
            retries: 3
            timeout: 5s

    redis:
        image: 'redis:alpine'
        ports:
            - '${FORWARD_REDIS_PORT:-6379}:6379'
        volumes:
            - 'sail-redis:/data'
        networks:
            - sail
        healthcheck:
            test:
                - CMD
                - redis-cli
                - ping
            retries: 3
            timeout: 5s

    meilisearch:
        image: 'getmeili/meilisearch:latest'
        ports:
            - '${FORWARD_MEILISEARCH_PORT:-7700}:7700'
        environment:
            MEILI_NO_ANALYTICS: '${MEILISEARCH_NO_ANALYTICS:-false}'
        volumes:
            - 'sail-meilisearch:/meili_data'
        networks:
            - sail
        healthcheck:
            test:
                - CMD
                - wget
                - '--no-verbose'
                - '--spider'
                - 'http://127.0.0.1:7700/health'
            retries: 3
            timeout: 5s

    mongodb:
        image: 'mongo:8.0'
        ports:
            - '${FORWARD_MONGODB_PORT:-27017}:27017'
        environment:
            MONGO_INITDB_ROOT_USERNAME: '${MONGODB_USERNAME:-root}'
            MONGO_INITDB_ROOT_PASSWORD: '${MONGODB_PASSWORD:-secret}'
            MONGO_INITDB_DATABASE: '${MONGODB_DATABASE:-laravel}'
        volumes:
            - 'sail-mongodb:/data/db'
        networks:
            - sail
        healthcheck:
            test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
            retries: 3
            timeout: 5s

networks:
    sail:
        driver: bridge

volumes:
    sail-mysql:
        driver: local
    sail-redis:
        driver: local
    sail-meilisearch:
        driver: local
    sail-mongodb:
        driver: local
```

也就是說，我們把原本繁雜的 Nginx、PHP-FPM，加上 Sail 裡面各種雜七雜八的容器，全部收斂成一個高效的 `app` 容器！

> [!WARNING]
> **MongoDB 啟動失敗？(不斷閃退)**
> 
> MongoDB 容器啟動失敗，通常是因為它規定只要有設定 `MONGO_INITDB_ROOT_USERNAME`，就「必須」要給它初始密碼，不能留空。
> 因為你的 `.env` 裡可能還沒有設定 `MONGODB_PASSWORD`（導致原本寫法被當作空字串），所以才引發報錯。
> 
> **解決方式：**
> 如上方的配置，我們已經直接在 `docker-compose.yml` 替密碼加上了預設值 `secret` (`${MONGODB_PASSWORD:-secret}`)。修改後重新啟動容器（`docker compose up -d`），現在它應該已經順利起飛了！如果有裝 GUI 工具也可以直接連看看。

[↑ 回到目錄](#目錄)

---

## Step 4. 日常開發與進階技巧

配置完成後，未來你的日常開發就不再需要依賴 Sail 的腳本，直接回歸最標準的 Docker 指令：

### 啟動與即時同步 (Hot Reloading)
在終端機輸入啟動指令：
```bash
docker compose up -d
```
啟動成功後，打開瀏覽器輸入 `http://localhost` 就能看見極速的 Laravel。
因為我們有設定 Volume 掛載以及 `--watch` 參數，你在程式裡寫的任何 Controller 或 Route，FrankenPHP 都會立刻幫你重載 worker，重整網頁立刻生效！

### 執行 PHP 與 Artisan 指令
既然沒有了 Sail，我們要怎麼下 Artisan 指令呢？很簡單，直接叫我們的 `app` 容器幫忙代勞：
```bash
# 執行 artisan 指令 (單純執行，不產生檔案的指令)
docker compose exec app php artisan migrate

# ⚠️ 當需要「建立新檔案」(例如 make:controller) 時，務必加上使用者身分，避免產生的檔案變成 root 權限導致編輯器報錯無法存檔：
docker compose exec -u "$(id -u):$(id -g)" app php artisan make:controller TestController

# 查 PHP 版本
docker compose exec app php --version
```

### 執行背景任務 (Queue Worker) 的優雅姿勢
做專案難免會碰到要寄信、轉檔這種很花時間的任務，有時候看畫面一直轉圈圈真的會很焦慮... 所以這時候就會用到 Laravel 的 Queue 啦。

以前在傳統主機上，我們通常要另外安裝 `supervisor` 來確保 Queue 行程中斷時能自動重啟。**但是！既然我們都上了 Docker 這艘船，最乾淨優雅的做法就是「完全不要裝 supervisor」，直接利用 Docker 引擎本身來當我們的進程守護神！**

#### 做法一：本機開發的手動除錯模式
平常自己開發，只是想快速看一下 Job 執行的 log，你可以多開一個 WSL 終端機視窗，直接叫 `app` 容器在前景幫你跑：
```bash
docker compose exec app php artisan queue:work
```
*(小提醒：這個指令敲下去就會卡在前景印出執行狀況，要停止就按 `Ctrl+C`。)*

#### 做法二：正式上線 / 長期執行的終極做法 (Docker Native Supervisor)
如果是在正式上線，或是你本機不想一直留一個終端機黑畫面，可以直接在 `docker-compose.yml` (以及 `.prod.yml`) 裡，**多起一個專門當無情打工機器的容器**！

打開你的 `docker-compose.yml`，在 `services:` 底下，原本的 `app` 區塊下方加入這個 `queue` 服務：

```yaml
    queue:
        # 完全沿用 app 的食譜與程式碼，不用重新打包，超快！
        build:
            context: .
            dockerfile: Dockerfile
        # 🔑 【關鍵】：這行就是我們的 Supervisor！出錯當機或重開機，Docker 都會秒速幫你重啟
        restart: unless-stopped
        environment:
            - APP_ENV=${APP_ENV:-local}
            - APP_DEBUG=${APP_DEBUG:-true}
        volumes:
            - .:/app
        # 覆寫原本的啟動指令，讓他從 Web Server 變成專職的 Queue Worker
        command: [ "php", "artisan", "queue:work", "--tries=3", "--timeout=90" ]
        networks:
            - sail
#### 進階做法：大型專案的救星「Laravel Horizon」
如果你的專案開始變大，有很多種不同的 Queue（例如分出 `emails`、`video_processing`、`reports`），而且開始需要：
1. **看報表**：想知道 Queue 現在塞車多嚴重、每分鐘消化多少任務。
2. **看失敗原因**：有沒有任務一直 Retry 失敗卡住。
3. **動態調度心力**：當 `emails` 塞爆時，自動派更多 Worker 去處理信件。

這時候，**真的非常強烈建議改用 Laravel Horizon**！
它不是一個全新的 Queue 系統，而是建構在 Redis 之上的**「Queue 視覺化管理後台與超級調度員」**。

**改成 Horizon 後，我們的設定會變成這樣：**
1. 專案內安裝 Horizon (`composer require laravel/horizon` 再 `php artisan horizon:install`)。
2. 在 `docker-compose.yml` 裡面，不再寫死一堆不同 `--queue=xxx` 的容器，而是直接全部交給 Horizon 管：
```yaml
    queue:
        build:
            context: .
            dockerfile: Dockerfile
        restart: unless-stopped
        environment:
            - APP_ENV=${APP_ENV:-local}
            - APP_DEBUG=${APP_DEBUG:-true}
        volumes:
            - .:/app
        # 🔑 【關鍵】：啟動指令從 queue:work 改成 horizon
        command: [ "php", "artisan", "horizon" ]
        networks:
            - sail
```

只要起了這個 `queue` 容器跑 `horizon`，它就會去讀你專案裡的 `config/horizon.php`，幫你「自動」在背景生出對應數量的 Worker，而且還送你一個超有質感的網頁儀表板（預設在 `你的網址/horizon`）！
做專案看到這種儀表板在跑數據，真的有一種「哇，我寫的系統好高級」的錯覺哈哈，非常推薦在正式專案使用！

> [!WARNING]
> **「等等！那如果有程式碼更新，Horizon 不就全部一起重啟了？不能只重啟某個 Queue 嗎？」**
>
> 沒錯，這正是你需要權衡的點！Horizon 是一個**整體**。當你執行 `php artisan horizon:terminate`（通常在佈署腳本 `Deploy` 階段為了載入新程式碼而下達的指令）時，**Horizon 會把它底下管理的所有 Queue 全部優雅地重啟一次**。
> 目前 Laravel 官方的 Horizon 設計上，**沒辦法單獨只對某個特定的 Queue 下達「重新載入程式碼」的指令**。只要重啟，就是全家桶一起重啟。
> 
> 如果你的系統有一種 Queue 是「**絕對不能被其他 Queue 的更新干擾**」（例如：一個跑下去要 3 小時的超大報表匯出、或是極度敏感的交易對帳），那把它們**全部**塞給同一個 Horizon 管可能就不是好主意。
>
> **遇到這種狀況的解法：**
> 回歸「做法二」的本質精神，我們在 `docker-compose.yml` 裡面，把它們拆開成不同的容器，各管各的：
> ```yaml
>     # 負責處理一般信件、通知等可以一起重啟的輕量任務
>     horizon:
>         command: [ "php", "artisan", "horizon" ]
> 
>     # 負責不能隨便被重啟的敏感耗時大任務，獨立出來跑原生的 queue:work
>     queue-critical:
>         command: [ "php", "artisan", "queue:work", "--queue=critical", "--timeout=10800" ]
> ```
> 這樣就能同時享受 Horizon 視覺化管理輕量任務的美好，又保證核心大任務在佈署更新時，只要我們不手動去 `docker restart queue-critical`，它就不會被意外打斷囉！
> 
> 
> **💡 延伸思考：那寫在 `docker-compose.yml` 裡的這些 Queue，遇到我們常見的 `build` 然後 `up -d` 更新程式碼時，會有影響嗎？**
> 
> **答案是：還是會有影響！**
> 雖然 `up -d` 不會像 `down` 那樣粗暴地把所有東西關掉再開，但它的運作原理是：「發現 Image 更新了 → **刪除舊容器** → 用新 Image **建立新容器**」。
>
> 也就是說，當你打完 `docker compose build` 然後跑 `docker compose up -d` 時，Docker Compose 會發現 `queue-critical` 依賴的 Image 換新了，它就會強制把你正在跑的那個舊 `queue-critical` 容器「刪掉」，然後換一個新的給你。
> 這個「刪舊建新」的瞬間，那個跑了兩個小時的報表任務一樣會被**直接腰斬中斷**（因為舊容器的記憶體跟進程直接消失了）！
> 
> **正式環境更新程式碼的「防腰斬」安全做法 (Zero Downtime 小技巧)：**
> 1. 先跑 `docker compose build app` (把新程式碼打包好)。
> 2. 接著跑 `docker compose up -d app horizon` (告訴 Docker：請幫我**只更新** Web 跟 Horizon 容器，那些不怕中斷或有完善 retry 機制的任務換新無所謂)。
> 3. ***刻意不要去 up 那個 `queue-critical` 容器***！只要你不點名它，Docker 就不會去重建它，它就會安靜地在背景繼續執行舊版程式碼把手上幾小時的大任務做完。直到確認它閒置了，我們再手動單獨去 `docker compose up -d queue-critical` 讓它也載入新版程式碼。

> [!WARNING]
> **「編輯器突然不能存檔了？Permission Denied 怎麼辦？」**
> 
> 當你忘記加 `-u "$(id -u):$(id -g)"` 參數去建立檔案（或者是系統自動生成的 log 檔），這些檔案預設擁有者會標記為 `root`，導致你一般使用者的編輯器無法修改與存檔。
> 遇到這個狀況，請在 WSL 終端機對專案目錄執行：
> `sudo chown -R $USER:$USER .`
> 即可把檔案擁有權搶回來！

> [!TIP]
> **嫌指令太長？設個 Alias 吧！**  
> 你可以在 WSL 的 `~/.bashrc` 加上：
> `alias dc='docker compose'`  
> `alias dce='docker compose exec -u "$(id -u):$(id -g)"'`  
> 這樣以後只要敲 `dce app php artisan make:controller TestController` 就舒服多了，而且絕對不會出現權限問題！

### 重建映像檔
如果你修改了 `Dockerfile`（例如想裝新的 PHP 擴展），請記得叫 Docker 清空快取重新打包一次：
```bash
docker compose build --no-cache
docker compose up -d
```

[↑ 回到目錄](#目錄)

---

## Step 5. Antigravity 完美協作起手式

現在網頁跑起來了，我們要開始改 Code。以往可能會想用 devcontainer，但我們直接走更穩定直白的路線。

### 1. 開啟專案資料夾
打開 Antigravity，選擇開啟資料夾 (Open Folder)。
請直接在路徑列輸入 WSL 的網路磁碟機路徑，例如：
`\\wsl$\Ubuntu\home\你的使用者名稱\projects\my-clean-app` 
*(有時候是 `\\wsl.localhost\Ubuntu\...` 視你的 Windows 版本而定)*

### 2. 注入 AI 靈魂：安裝 Laravel Boost (必做！)
為了讓 Antigravity 能夠 100% 發揮 AI 寫 Code 的功力（讓它可以自己去查資料庫結構、看 Log、讀 Route），我們必須安裝官方專門為 AI 打造的神器：**Laravel Boost (MCP Server)**。

既然走零污染流派，安裝與初始化一樣派出 Docker 代勞，請在 WSL 終端機執行：
```bash
# 安裝 Boost 套件 (記得加上使用者權限，避免產生 root 檔案)
docker compose exec -u "$(id -u):$(id -g)" app composer require laravel/boost --dev

# 執行初始化安裝 (指令加上參數直接完成，避免互動選單出現咬死終端機)
docker compose exec -it -u "$(id -u):$(id -g)" app php artisan boost:install --mcp --guidelines --skills
```

**設定 AI 編輯器的 MCP 連線指令（最重要的一步）：**
因為我們要在 Windows 全域環境中，讓 AI 同時掌握多個不同專案的貨櫃，所以設定必須寫在 **全域 MCP 設定檔** 中。

📍 **全域設定檔位置：**
`~/.gemini/antigravity/mcp_config.json` (Windows 上通常位在 `C:\Users\你的使用者名稱\.gemini\antigravity\mcp_config.json`)

請打開這個檔案，並確保：
1. **命名區隔**：每個專案的 Server Key 必須是唯一的（例如 `autoparts-boost`、`blog-boost`）。
2. **指定專案路徑**：因為指令從全域觸發，`docker compose` 必須加上 `--project-directory <絕對路徑>`，讓容器知道要在哪個資料夾下執行。

**📝 設定範例 (同時設定兩個專案)：**
```json
{
  "mcpServers": {
    "autoparts-boost": {
      "command": "docker",
      "args": [
        "compose",
        "--project-directory",
        "/home/user/projects/autoparts",
        "exec",
        "-T",
        "app",
        "php",
        "artisan",
        "boost:mcp"
      ]
    },
    "blog-boost": {
      "command": "docker",
      "args": [
        "compose",
        "--project-directory",
        "/home/user/projects/blog",
        "exec",
        "-T",
        "app",
        "php",
        "artisan",
        "boost:mcp"
      ]
    }
  }
}
```

> [!IMPORTANT]
> **注意 1：替換正確的專案路徑**
> JSON 裡面的 `/home/user/projects/autoparts` 請務必改成**你各個專案在 WSL 裡面的絕對路徑**！

> [!TIP]
> **注意 2：不要漏掉 `-T` 參數哦！** 
> AI 和 MCP 溝通是靠終端機標準輸入輸出 (stdio) 的。如果不加 `-T` 把 TTY 關掉，貨櫃回傳的資料會夾帶一堆 Linux 終端機控制碼，AI 會被亂碼搞到崩潰。

### 🧠 AI 如何知道要用哪一個專案的大腦？
由於所有 Server 的設定都在全域，AI 啟動時會將**所有的工具**都載入。
但是不用擔心，AI 在執行任務時，會自動根據以下兩點聰明地切換：
1. **當前工作目錄（CWD）**：AI 會偵測你目前所在處理的專案路徑。
2. **Server 命名與上下文**：AI 會比對名稱最符合當前專案上下文的 MCP Server（例如，人在 `autoparts` 目錄寫 code，AI 就會知道要選用來自 `autoparts-boost` 的工具）。

設定完成後，**強烈建議「重新載入視窗」或重啟編輯器**，並在 MCP 面板點擊 **Refresh**。現在你的 AI 已經拿到了所有專案的萬能鑰匙，遇到靈異 Bug，你甚至可以霸氣地對 AI 說：「你自己進貨櫃查錯誤 Log」，然後它就會乖乖幫你把問題挖出來啦！

### 3. 日常開發循環
Antigravity 會直接讀取 WSL 裡面的檔案並在背景透過 Laravel Boost 即時懂你的架構。存檔瞬間，因為 Docker Volume 掛載與 FrankenPHP 熱重載的關係，網頁重整立刻生效。

兼具潔癖與極致 AI 生產力，**這就是我們的日常開發循環：**
1. 啟動 WSL
2. 進入專案資料夾跑 `docker compose up -d`
3. 用 Antigravity 開啟 WSL 上的資料夾進行開發 (AI 已火力全開)
4. 下班關機前敲個 `docker compose down`，乾乾淨淨。

[↑ 回到目錄](#目錄)

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
4. 跑一下資料庫更新 (如果有的話)。因為我們統籌了配置，所以你只要指定執行在那台名為 `app` 的容器裡面：
   ```bash
   docker compose exec app php artisan migrate --force
   ```

### 情境 B：正式環境佈署 (Production Environment)
到了正式機，我們就不建議用掛載 Volume (綁定本機目錄) 的方式了，因為有效能與安全的考量。最好的做法是把程式碼「打包」成正式的 Image。

1. **準備 Production 專用的設定檔**
   在專案根目錄準備一個 `Dockerfile` (專門用來把程式碼 COPY 進去並安裝依賴) 和一個正式版的 `docker-compose.prod.yml`。

   **📝 範例：正式環境用的 `Dockerfile`**
   其實，**你在 Step 3 寫的那個 `Dockerfile` 就已經是 Production Ready 了！**
   這就是多階段建置 + FrankenPHP 的火力展示：開發跟正式環境完全可以用同一份食譜。回想一下我們在 Step 3 的設計：
   - `ENTRYPOINT` 負責每次啟動時先跑 `docker-entrypoint.sh` 修復 Storage 權限
   - `CMD` 則是正式啟動 Octane 伺服器
   - 前端資源在「階段 1」就已經編譯打包好了
   
   **開發時**，我們透過 `docker-compose.yml` 的 `command` 覆寫整段 `CMD` 並加上 `--watch` 來觸發熱重載；**正式機**不傳這個參數，它就會用最純淨、最高效能的 Octane 模式跑起來。

   **📝 範例：正式環境用的 `docker-compose.prod.yml`**
   *(正式區不需要再去搞 Nginx 跟 PHP-FPM 啦！直接起單一個包含了 FrankenPHP 的 app 映像檔即可，極致簡潔！)*
   ```yaml
   services:
     app:
       build:
         context: .
         dockerfile: Dockerfile
       restart: unless-stopped
       # ⚡ 效能極致化：改用 host 網路模式，讓容器直接綁定伺服器網卡
       # 拔除 ports 設定，省去 Docker NAT 轉換效能損耗，也更容易拿到用戶真實 IP！
       network_mode: "host"
       # ⚠️ 為什麼這裡還要寫 environment？
       # 雖然我們下面有掛載 env_file，但在 compose 檔裡寫死這幾個變數，層級會「最高」
       # 就算你不小心把本機寫著 APP_ENV=local 的 .env 拿丟上正式機，
       # 這裡的設定也會強行把它蓋掉，確保正式機「絕對」是用 production 跑，這是一種防呆保險！
       environment:
         - APP_ENV=production
         - APP_DEBUG=false
       # ⚠️ 關鍵差異：正式環境「不要」把整包程式碼掛載進去 (勿用 .:/app)
       # 讓它讀取 Dockerfile 打包好的純淨程式碼，效能最快、最安全！
       # 但是！我們「必須」把 storage 資料夾獨立掛載出來，這樣上傳的圖片跟 Log 才不會在更新容器時消失：
       volumes:
         - ./storage:/app/storage
       # ⚠️ 正式機的 .env 怎麼辦？
       # 直接在伺服器專案根目錄建立一份 `.env` 檔案，Docker Compose 自動會讀取進去！
       # 不要把資料庫密碼寫死在下面的 docker-compose.prod.yml 裡，這樣推上 Git 會外洩喔。
       env_file:
         - path: .env
           required: true # (Docker Compose v2.24+) 如果外面沒有 .env 檔，會直接報錯拒絕啟動！強迫你一定要建好它！

     # 背景任務調度總管 (Laravel Horizon)
     horizon:
       build:
         context: .
         dockerfile: Dockerfile
       restart: unless-stopped
       network_mode: "host"
       environment:
         - APP_ENV=production
         - APP_DEBUG=false
       volumes:
         - ./storage:/app/storage
       env_file:
         - path: .env
           required: true
       # 🔑 覆寫 CMD，讓它不要跑 Web Server，改跑 Horizon
       command: [ "php", "artisan", "horizon" ]

     # (可選) 搜尋引擎：如果有使用 Laravel Scout + Meilisearch
     meilisearch:
       image: getmeili/meilisearch:latest
       restart: unless-stopped
       # ⚠️ 網路防呆：因為上面的 app 用了 host 模式，它們不在同一個 Docker 虛擬網段了！
       # 所以 app 必須連線 `127.0.0.1:7700` 來找它。
       # 為了安全，我們這裡只把 Port 綁定在主機的 `127.0.0.1` 內網，絕對不要只寫 `- "7700:7700"`，否則全世界都能連你的搜尋引擎！
       ports:
         - "127.0.0.1:7700:7700"
       environment:
         # 正式區千萬別用預設空密碼！要在 .env 裡自行設定一個長度夠的 MEILI_MASTER_KEY
         MEILI_MASTER_KEY: '${MEILISEARCH_KEY}'
         MEILI_ENV: 'production'
       volumes:
         - meili-data:/meili_data

   volumes:
     meili-data:
       driver: local
   ```

   > [!IMPORTANT]
   > **Storage 掛載方式：`./storage` vs Named Volume**
   >
   > 注意到了嗎？這裡我們用的是 `./storage:/app/storage`（Bind Mount，直接映射主機目錄），而不是 `app-storage:/app/storage`（Named Volume）。
   > 兩者的差別：
   > - **Bind Mount (`./storage`)** → 檔案直接存在伺服器的專案目錄裡，你可以 `ls ./storage/logs` 直接查看 Log，直覺又方便。
   > - **Named Volume (`app-storage`)** → 檔案由 Docker 管理存放在 Docker 內部路徑，需要 `docker volume inspect` 才能找到實體位置。
   >
   > 我們選擇 Bind Mount 是因為正式環境偵錯時，能直接在伺服器上 `cat storage/logs/laravel.log` 超級方便！

2. **在 Server 上的標準佈署流程 (第一次上線與後續推 Code 更新通用)**
   因為我們正式環境**沒有掛載目錄**，程式碼是「烤」在 Image 裡面的。所以無論是第一次把專案 clone 下來，還是日後改了 Code 要上線，請統一依照這個「黃金四步」：

   ```bash
   # 1. 取得最新程式碼 (如果是第一次，這步叫做 git clone)
   git pull
   
   # 2. 叫 Docker 重新把程式碼烤成正式版 Image
   docker compose -f docker-compose.prod.yml build
   
   # 3. 無縫啟動/重啟服務 (Docker 會自動開新容器或平滑替換舊容器)
   docker compose -f docker-compose.prod.yml up -d
   
   # 4. 執行正式機的資料庫更新 (如果有的話)
   docker compose -f docker-compose.prod.yml exec app php artisan migrate --force
   ```
   
   > [!NOTE]
   > **「咦？為什麼 migrate 要加 `--force`？會把資料刪掉嗎？」**
   > 別擔心，`--force` **不會**刪除你的資料庫。當 Laravel 偵測到身處 `production` 環境時，執行任何 migrate 都會跳出警告問「你確定要執行嗎？(Y/n)」，加上 `--force` 只是為了告訴系統「對，我確定，跳過詢問直接跑完」，是安全且必備的選項。*(如果要刪除重建那叫 `migrate:fresh`，在正式機千萬別亂下！)*

   > [!WARNING]
   > **🚨 故障排除：正式區啟動失敗怎麼解？**
   > 如果你下 `docker compose up -d` 後網頁打不開，通常是因為 Laravel 在啟動時遇到 Fatal Error 或是 .env 設定錯誤。
   > 常見原因與解法如下：
   > 1. **檢查環境變數 .env**：確認 `.env` 有建立，是否有缺少 `APP_KEY`，或者參數(如 production) 有沒打錯。
   > 2. **查看真實錯誤報告**：如果 Worker 狂閃退，可以直接跑這行指令，讓它在前景把實際的 PHP 錯誤吐出來：
   > ```bash
   > docker compose -f docker-compose.prod.yml exec app php artisan octane:frankenphp --host=0.0.0.0 --port=8000
   > ```
   > *(看到錯誤就一目了然，修好後再重啟容器即可！)*

**給個小建議**：如果專案規模變大，誠心建議正式區的佈署可以搭配 CI/CD (例如 GitHub Actions 或 GitLab CI)。在 CI 上面把 Docker Image 打包好推到 Registry，你的 Server只需要單純做 `docker pull` 跟 `docker compose up -d` 就好，這樣是最穩、最不怕髒的標準做法！

[↑ 回到目錄](#目錄)

---

## Step 7. 換電腦怎麼辦？(從 Git Clone 接續開發)

當你換了一台新電腦，或是團隊有新同事加入時，該如何無痛接手這個「零污染」專案呢？
我們不需要因為沒有 PHP 而感到慌張，因為 Docker 會搞定一切！

### 1. 把程式碼拉下來
首先，在你的新電腦 (或新同事的電腦) 裡的 WSL 中，把專案從 Git 拉下來：
```bash
cd ~/projects
git clone <你的-git-repo-網址> my-clean-app
cd my-clean-app
```

### 2. 準備 `.env` 檔
專案通常不會把 `.env` 放進版本控制，所以你要複製一份給它：
```bash
cp .env.example .env
```

### 3. 用「臨時工」幫你裝滿整車的 `vendor` 套件
現在我們專案裡空空的，連 `sail` 都沒有。如果你敲 `composer install` 系統會無情地跟你說 `command not found` (因為我們很乾淨，沒裝 PHP 啊！)。

所以我們要再派一次 Docker 臨時工出場，請他以本機使用者的權限，幫我們掛載目錄並執行 Composer：
```bash
docker run --rm \
    -u "$(id -u):$(id -g)" \
    -v $(pwd):/var/www/html \
    -w /var/www/html \
    laravelsail/php84-composer:latest \
    composer install --ignore-platform-reqs
```
*(注意：這裡的 `--ignore-platform-reqs` 是告訴 Composer 不要機車去檢查本機有沒有 PHP 跟對應的版本，直接乖乖把 package 下載好放到 vendor 裡面就好！)*

### 4. 啟動環境與初始化
感謝臨時工幫忙把套件載完，現在你的專案又是一尾活龍。接下來就回歸我們標準的啟動流程：

```bash
# 啟動背景服務
docker compose up -d

# 產生加密密鑰
docker compose exec app php artisan key:generate

# 執行資料庫遷移與資料填充
docker compose exec app php artisan migrate:fresh --seed
```

只要簡單四步，新電腦上的開發環境就 100% 復活了！

[↑ 回到目錄](#目錄)

---

### 結語
這套「零污染開發流」一開始設定要轉一兩個彎，但相信我，幾個月後你會感謝現在的自己。尤其是搭上 FrankenPHP 這個黑科技，當你換電腦只要裝個 Docker 就能三分鐘回到完美的開發狀態... 這種把混亂收斂到極致的感覺，就是我們工程師少數能自我掌控的療癒時刻啊哈哈！

👉 準備好開發其他領域了嗎？請前往：
- **[前端篇：Vue / Vite + Docker 零污染實戰](./VUE_VITE_FRONTEND.md)**
- **[C 語言篇：純淨 Docker 編譯環境](./C_LANG_ENV.md)**
