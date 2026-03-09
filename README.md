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

### 安裝頂級引擎：FrankenPHP (Laravel Octane)
因為我們堅持正式機跟開發機都要最乾淨、最高效能，所以我們不用傳統的 Nginx + PHP-FPM，而是直接選用用 Go 寫的現代化伺服器 **FrankenPHP**。它直接內建了 Web Server 跟 PHP 執行環境，讓網頁速度飛快。

剛建好的專案自帶了 Laravel Sail，我們先把它暫時起起來，借用它的環境把 Octane 裝好：

```bash
cd my-clean-app

# 1. 啟動剛建好的基礎專案 (暫時使用 Sail 預設配置)
./vendor/bin/sail up -d

# 2. 安裝 Octane 套件
./vendor/bin/sail composer require laravel/octane

# 3. 選擇 FrankenPHP 作為引擎
./vendor/bin/sail artisan octane:install --server=frankenphp

# 4. 安裝完畢，把暫時的環境關掉，準備我們自己的效能配置
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

### 1. 準備 FrankenPHP 專用 Dockerfile
在專案根目錄建立 `Dockerfile`，負責定義我們應用程式的靈魂：

```dockerfile
# 基礎映像檔：官方 FrankenPHP (自帶 Caddy web server 與 PHP)
FROM dunglas/frankenphp:php8.4

# 加入一些 LABEL 讓同事知道這是什麼 (人類友善)
LABEL maintainer="你的名字 <your.email@example.com>"
LABEL description="Laravel + FrankenPHP 終極純淨版"

# 設定環境變數，讓伺服器知道跑在 80 port
ENV SERVER_NAME=":80"

# 安裝額外的 PHP 擴展
# (FrankenPHP 內建 install-php-extensions 工具，不用自己刻 apt-get，超佛心)
RUN install-php-extensions \
    pdo_mysql \
    gd \
    mbstring \
    xml \
    curl \
    zip \
    mongodb \
    pcntl \
    redis

# 安裝 Composer (從官方映像檔複製過來)
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /app

# 將目前專案原始碼 COPY 進 Image
COPY . /app

# 設定存取權限給 web server
RUN chown -R www-data:www-data /app/storage /app/bootstrap/cache

# 安裝正式版套件 (不含 dev 依賴，保持瘦身)
# 開發時這個動作會被本機 Volume 蓋掉，所以沒關係
RUN composer install --optimize-autoloader --no-dev

# 預設啟動 Laravel Octane 榨出極限效能 (注意：需要指定 admin-port 避免預設算式導致負數報錯)
ENTRYPOINT ["php", "artisan", "octane:start", "--server=frankenphp", "--host=0.0.0.0", "--port=80", "--admin-port=2019"]
```

### 2. 準備統一天下的 `docker-compose.yml`
打開專案內的 `docker-compose.yml`，把裡面原本 Sail 幫你產生的東西全部清空，換成這套配置：

```yaml
services:
    app:
        build:
            context: .
            dockerfile: Dockerfile
        restart: unless-stopped
        ports:
            - "80:80"        # 標準 HTTP
            - "443:443"      # HTTPS
            - "443:443/udp"  # 支援超快的 HTTP/3
        environment:
            - APP_ENV=${APP_ENV:-local}
            - APP_DEBUG=${APP_DEBUG:-true}
            - OCTANE_SERVER=frankenphp
        volumes:
            # 【重要】：開發環境把這行打開，掛載本機目錄
            # 如果是「正式上線」，請把這行註解掉，讓它讀取 Dockerfile 打包好的純淨原始碼
            - .:/app
        # 開發環境必備：傳遞 --watch 參數給 Dockerfile 的 ENTRYPOINT 達成 Hot Reloading
        command: ["--watch"]
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
```

也就是說，我們把原本繁雜的 Nginx、PHP-FPM，加上 Sail 裡面各種雜七雜八的容器，全部收斂成一個高效的 `app` 容器！

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

打開 Antigravity，選擇開啟資料夾 (Open Folder)。
請直接在路徑列輸入 WSL 的網路磁碟機路徑，例如：
`\\wsl$\Ubuntu\home\你的使用者名稱\projects\my-clean-app` 
*(有時候是 `\\wsl.localhost\Ubuntu\...` 視你的 Windows 版本而定)*

Antigravity 會直接連線並讀取 WSL 裡面的檔案。你接下來就在 Antigravity 裡面暢快地寫 Code，存檔的瞬間，因為 Docker volume 檔案掛載的關係，網頁重整就立刻生效。

**這就是我們的日常開發循環：**
1. 啟動 WSL
2. 進入專案資料夾跑 `docker compose up -d`
3. 用 Antigravity 開啟 WSL 上的資料夾進行開發修改
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
   這就是 FrankenPHP 的火力展示：開發跟正式環境完全可以用同一份食譜。回想一下我們在 Step 3 留下的最後一行：
   ```dockerfile
   # 預設啟動 Laravel Octane 榨出極限效能
   ENTRYPOINT ["php", "artisan", "octane:start", "--server=frankenphp", "--host=0.0.0.0", "--port=80", "--admin-port=2019"]
   ```
   **這行就是正式上線時要執行的啟動指令！**
   （開發時，我們是透過 `docker-compose.yml` 塞了 `command: ["--watch"]` 參數去觸發熱重載；正式機我們不傳這個指令，它就會用最純淨、最高效能的 Octane 模式跑起來。）

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
       environment:
         - APP_ENV=production
         - APP_DEBUG=false
         - OCTANE_SERVER=frankenphp
       # ⚠️ 關鍵差異：正式環境「不要」把整包程式碼掛載進去 (勿用 .:/app)
       # 讓它讀取 Dockerfile 打包好的純淨程式碼，效能最快、最安全！
       # 但是！我們「必須」把 storage 資料夾獨立掛載出來，這樣上傳的圖片跟 Log 才不會在更新容器時消失：
       volumes:
         - app-storage:/app/storage

     # (可選) 正式機的資料庫通常建議買雲端託管 (如 AWS RDS Cloud SQL)，若堅持自建才保留這段
     # db:
     #   image: mysql:8.4
     #   ... 放資料庫帳密與掛載設定

   volumes:
     app-storage:
       driver: local
   ```

2. **在 Server 上的起手式**
   把程式碼拉下來後，執行打包與啟動：
   ```bash
   # 1. 建立正式版的 Image (會把你當下的程式碼打包鎖死進去)
   docker compose -f docker-compose.prod.yml build
   
   # 2. 啟動背景服務
   docker compose -f docker-compose.prod.yml up -d
   
   # 3. 執行正式機的資料庫遷移
   docker compose -f docker-compose.prod.yml exec app php artisan migrate --force
   ```

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
