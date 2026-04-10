# 潔癖工程師的終極開發環境 (前端篇)：Vue / Vite + Docker 零污染實戰

如果你已經看過我們的 [後端 API 開發指南 (Laravel + FrankenPHP)](./README.md)，體驗到了「把所有髒活交給 Docker」的極致快感，那這篇就是為你準備的前端下半場！

前端開發最常遇到的痛點，不外乎是：「這專案要 Node 18，那個老專案要 Node 14... 救命啊！又要切換 nvm 了！」
放心，同樣的「零污染」精神套用在前端，我們**一樣不在電腦上裝任何 Node.js！**

這篇文章將帶你使用 WSL + Docker，無痛建置一個極速、支援 Hot Reloading 的 Vue / Vite 前端開發環境。

---

## 目錄
- [Step 1. 打造純淨地基 (基礎環境準備)](#step-1-打造純淨地基-基礎環境準備)
- [Step 2. 無中生有：用臨時工建立 Vue/Vite 專案](#step-2-無中生有：用臨時工建立-vuevite-專案)
- [Step 3. 專案的 Docker 化：統一天下的配置](#step-3-專案的-docker-化：統一天下的配置)
- [Step 4. 日常開發與 Hot Reloading](#step-4-日常開發與-hot-reloading)
- [Step 5. Antigravity 完美協作起手式](#step-5-antigravity-完美協作起手式)
- [Step 6. 開發完成：測試區與正式區的快速佈署](#step-6-開發完成：測試區與正式區的快速佈署)
- [Step 7. 換電腦怎麼辦？(從 Git Clone 接續開發)](#step-7-換電腦怎麼辦？從-git-clone-接續開發)

---

## Step 1. 打造純淨地基 (基礎環境準備)

如同後端篇，我們在一台全新的 Windows 電腦上，**只准裝 WSL、Docker 和 Antigravity**。
如果還沒裝好，請參考 [後端篇的第一步](./README.md#step-1-打造純淨地基-基礎環境準備)。

*(重要提醒：千萬不要手癢跑去下載 Node.js 安裝檔，忍住！)*

[↑ 回到目錄](#目錄)

---

## Step 2. 無中生有：用臨時工建立 Vue/Vite 專案

在完全沒有裝 Node.js 和 npm 的電腦上，要怎麼跑 `npm create vite@latest`？
沒錯，又是請出 Docker「臨時工」的時候了！

打開 WSL 終端機 (Ubuntu)，切換到你想放專案的目錄：
```bash
mkdir -p ~/projects
cd ~/projects
```

執行這串指令，請 Docker 派一個帶有最新版 Node 的容器，幫我們建立基礎骨架：
```bash
docker run --rm -it \
    -u "$(id -u):$(id -g)" \
    -e npm_config_cache=/tmp/.npm \
    -v $(pwd):/app \
    -w /app \
    node:22-alpine \
    npm create vite@latest my-clean-frontend -- --template vue
```
*(上述指令會建立一個名為 `my-clean-frontend` 的純 Vue 專案。你也可以在互動選單中選擇搭配 TypeScript。)*

建好之後，我們一樣請臨時工幫我們把套件 (`node_modules`) 裝好：
```bash
cd my-clean-frontend

docker run --rm -it \
    -u "$(id -u):$(id -g)" \
    -v $(pwd):/app \
    -w /app \
    node:22-alpine \
    npm install
```

現在，熱騰騰的 Vite 專案跟滿滿的 `node_modules` 都在資料夾裡了，而且我們的 Windows/WSL 依舊乾乾淨淨，連一滴 npm 的痕跡都沒有！

[↑ 回到目錄](#目錄)

---

## Step 3. 專案的 Docker 化：統一天下的配置

跟後端建置的邏輯一樣，我們要寫一份 `Dockerfile` 當作食譜，以及一份 `docker-compose.yml` 當作餐廳經理。

> [!NOTE]
> 前端的 Docker 化跟後端有一個決定性的不同：
> - **開發時**：我們需要 Node.js 環境來跑 `npm run dev`（提供 Hot Reloading 跟即時編譯）。
> - **上線時**：前端程式碼會被編譯成純靜態的 HTML/JS/CSS（`npm run build`），這個階段 **完全不需要 Node.js**，只需要一個輕量的 Web Server（例如 Nginx）把靜態檔案丟給使用者即可。
> 
> 因此，我們接下來寫的 `Dockerfile` 會採用「多階段建置 (Multi-stage Build)」的魔法！

### 1. 準備前端專屬的 Dockerfile
在專案根目錄建立 `Dockerfile`：

```dockerfile
# ---------------------------------------------------------
# Stage 1: 開發與編譯環境 (使用 Node.js)
# ---------------------------------------------------------
FROM node:22-alpine as build-stage

WORKDIR /app

# 先複製 package 檔案，利用 Docker Cache 加速未來的 npm install
COPY package*.json ./
RUN npm install

# 將所有原始碼複製進來
COPY . .

# 執行正式版編譯 (輸出到 /app/dist)
RUN npm run build

# ---------------------------------------------------------
# Stage 2: 正式上線環境 (只留下 Nginx 負責發送靜態檔案)
# ---------------------------------------------------------
FROM nginx:alpine as production-stage

# 把上一階段 (build-stage) 編譯好的 dist 資料夾，複製到 Nginx 的預設網頁目錄
COPY --from=build-stage /app/dist /usr/share/nginx/html

# 開放 80 port
EXPOSE 80

# 啟動 Nginx (背景執行)
CMD ["nginx", "-g", "daemon off;"]
```

### 2. 準備統一天下的 `docker-compose.yml`
打開專案內的 `docker-compose.yml`，貼上這段全能配置：

```yaml
services:
  frontend:
    # 預設使用 Node.js 開發環境的 Image
    image: node:22-alpine
    restart: unless-stopped
    ports:
      - "5173:5173"  # Vite 預設的 dev server port
    volumes:
      # 把本機原始碼掛載進去，這樣存檔才會觸發 Hot Reload
      - .:/app
    working_dir: /app
    # 開發環境下，啟動 Vite 開發伺服器，並允許從外部 (0.0.0.0) 連線
    command: npm run dev -- --host 0.0.0.0
    tty: true
```

[↑ 回到目錄](#目錄)

---

## Step 4. 日常開發與 Hot Reloading

配置完成後，未來你的日常開發就這麼簡單：

### 啟動與即時同步 (Hot Reloading)
在終端機輸入啟動指令：
```bash
docker compose up -d
```
啟動成功後，打開瀏覽器輸入 `http://localhost:5173` 就能看見熱騰騰的 Vue 畫面了。
因為我們有設定 Volume 掛載，所以你在 Antigravity 裡面修改任何 `.vue` 檔案，Vite 都會光速幫你 Hot Reload，瀏覽器免重整直接更新！

### 需要安裝新的 npm 套件？
因為你本機沒有 npm，如果開發到一半想安裝 Vue Router 或 Pinia，同樣呼叫我們由 `docker-compose` 建立的容器來代勞：

> [!WARNING]
> **注意權限問題 (Permission Denied)！** 
> 安裝套件會異動 `package.json` 並產生新的檔案。請務必加上 `-u` 參數明確指定你的平民身分，否則產生的檔案擁有者會變成 `root`，導致編輯器無法存檔喔！

```bash
docker compose exec -u "$(id -u):$(id -g)" frontend npm install vue-router pinia
```

*(萬一忘記加參數導致出現 Permission Denied 報錯，只要在 WSL 敲 `sudo chown -R $USER:$USER .` 就能把權限搶回來。)*

裝完之後，`node_modules` 和 `package.json` 會透過 Volume 自動同步回你的電腦裡。

[↑ 回到目錄](#目錄)

---

## Step 5. Antigravity 完美協作起手式

就跟後端篇一模一樣：
1. 用 Antigravity 打開 WSL 的網路磁碟機路徑：`\\wsl$\Ubuntu\home\你的使用者名稱\projects\my-clean-frontend`
2. `docker compose up -d` 讓前端伺服器跑起來。
3. 舒服地在 Antigravity 寫 code、存檔、看著這療癒的秒速 UI 更新。

[↑ 回到目錄](#目錄)

---

## Step 6. 開發完成：測試區與正式區的快速佈署

還記得我們在 `Dockerfile` 裡面寫的「多階段建置」魔法嗎？到了上線的時候，它就要發揮作用了。

### 準備正式區用的 `docker-compose.prod.yml`
這份配置的目標，是告訴 Docker：「我要建立真正的上線版了，不要再幫我掛載本機資料夾，也不要用 Node.js 啟動 `npm run dev` 了！」

```yaml
services:
  frontend:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    ports:
      - "80:80"   # 正式環境我們改聽預設的 80 port
```

### 在 Server 上的起手式
把程式碼拉下載後：

```bash
# 1. 建置上線版 (會執行 npm run build，並把產生的 HTML/JS 餵給乾淨的 Nginx)
docker compose -f docker-compose.prod.yml build

# 2. 啟動背景服務
docker compose -f docker-compose.prod.yml up -d
```
此時，你的前端 Server 裡面完全沒有 Node.js 的痕跡，只有最輕量、安全的 Nginx 在默默派送編譯好的神速靜態網頁。這就是全端工程師極致浪漫的追求啊！

[↑ 回到目錄](#目錄)

---

## Step 7. 換電腦怎麼辦？(從 Git Clone 接續開發)

我們不需要因為沒有 Node.js 而感到慌張，臨時工隨時都在！

### 1. 把程式碼拉下來
首先，把專案從 Git 拉下來：
```bash
cd ~/projects
git clone <你的-git-repo-網址> my-clean-frontend
cd my-clean-frontend
```

### 2. 用「臨時工」幫你把 Node 模組裝回來
因為 `.gitignore` 不會把 `node_modules` 推進去，所以接手專案的第一步是裝套件：

```bash
docker run --rm -it \
    -u "$(id -u):$(id -g)" \
    -v $(pwd):/app \
    -w /app \
    node:22-alpine \
    npm install
```

### 3. 正常啟動環境
```bash
docker compose up -d
```
搞定！又是一個完全沒有污染、三分鐘內就能完美接手的前端開發環境。

[↑ 回到目錄](#目錄)

---

### 結語
從把龐大的 Laravel 後端裝進單一的 FrankenPHP 容器，到現在讓前端 Vue.js 也能在沒有 Node.js 的 Windows 電腦上高速開發... 我們徹底把所有亂七八糟的環境依賴都關進了 Docker 的鐵籠裡。

這種「隨時隨地，只要有 Docker 就能開發」的確定感，是不是比起以往在那邊 Debug 環境變數要療癒得多了呢？

👉 回頭複習或探索更多：
- [後端 API 開發指南 (Laravel + FrankenPHP)](./README.md)
- [C 語言篇：純淨 Docker 編譯環境](./C_LANG_ENV.md)
