# 潔癖工程師的終極開發環境 (C 語言篇)：捨棄 MinGW，擁抱純淨 Docker

如果你已經受夠了在 Windows 上面為了寫 C 語言去安裝 MinGW、Cygwin，又或者是常常為了環境變數搞到懷疑人生... 恭喜你，這篇就是你的救星。

我們延續「**把所有髒活交給 Docker**」的極簡哲學。這次，我們一樣不在電腦上裝任何 `gcc` 或 `clang`。我們要讓古老的 C 語言開發環境也變得輕量、純淨、隨插即用。

---

## 目錄
- [Step 1. 打造純淨地基 (基礎環境準備)](#step-1-打造純淨地基-基礎環境準備)
- [Step 2. 寫下第一段 C 語言](#step-2-寫下第一段-c-語言)
- [Step 3. 呼叫 Docker 臨時工來幫忙編譯與執行](#step-3-呼叫-docker-臨時工來幫忙編譯與執行)
- [Step 4. 進階：用 Makefile 與 Docker Compose 統管專案](#step-4-進階：用-makefile-與-docker-compose-統管專案)
- [結語](#結語)

---

## Step 1. 打造純淨地基 (基礎環境準備)

跟之前一樣，我們只准電腦裡有 **WSL、Docker 和 Antigravity**。
如果你還沒裝好，請參考 [後端篇的第一步](./README.md#step-1-打造純淨地基-基礎環境準備)。

*(重要提醒：請忍住你的衝動，千萬不要在 WSL 裡面下 `apt install build-essential` 或是 `gcc`！保持它的貞潔！)*

[↑ 回到目錄](#目錄)

---

## Step 2. 寫下第一段 C 語言

打開你的 WSL 終端機，建立一個新的專案資料夾：
```bash
mkdir -p ~/projects/c-clean-env
cd ~/projects/c-clean-env
```

接著用 Antigravity 打開這個資料夾 (`\\wsl$\Ubuntu\home\你的使用者名稱\projects\c-clean-env`)。

在裡面建立一個最經典的 `main.c` 檔案：
```c
#include <stdio.h>

int main() {
    printf("Hello, 零污染的 C 語言世界！\n");
    return 0;
}
```

這時候你的編輯器可能會沒有完整的 AutoComplete 或 IntelliSense 提示 (因為你本機沒有標頭檔)，沒關係，我們工程師靠的是心智編譯器 (笑)，先繼續往下走。

[↑ 回到目錄](#目錄)

---

## Step 3. 呼叫 Docker 臨時工來幫忙編譯與執行

沒有 `gcc`，我們要怎麼編譯？沒錯，就是請出 Docker 官方準備好的 `gcc` 映像檔。

### 1. 編譯程式碼
在終端機輸入以下指令，請臨時工幫我們把 `main.c` 編譯成執行檔 `app`：

```bash
docker run --rm \
    -u "$(id -u):$(id -g)" \
    -v $(pwd):/app \
    -w /app \
    gcc:latest \
    gcc main.c -o app
```
*(解釋：我們把當前目錄掛載到容器的 `/app`，然後叫容器裡的 `gcc` 把 `main.c` 編譯出來。這裡指名用當前使用者的身份 `id -u` 去建檔，免得建出來的檔案權限變成 root 砍不掉。)*

### 2. 執行程式碼
編譯好之後，資料夾會多出一個 `app` 執行檔。
其實因為我們已經在 WSL (Linux 環境) 裡面了，你可以直接敲 `./app` 就會通。但為了貫徹「連執行都要靠 Docker」的浪漫，你可以這樣做：

```bash
docker run --rm \
    -u "$(id -u):$(id -g)" \
    -v $(pwd):/app \
    -w /app \
    ubuntu:22.04 \
    ./app
```

[↑ 回到目錄](#目錄)

---

## Step 4. 進階：用 Docker Compose 統管專案

每次都要打那麼一長串 Docker 指令實在太痛苦了。這時候我們就可以寫一個 `docker-compose.yml` 把這些複雜度藏起來。

### 1. 建立統治級別的 `docker-compose.yml`
我們起一個長期運行的開發容器：

```yaml
services:
  c-dev:
    image: gcc:latest
    # 保留 TTY 讓容器持續運行不中斷
    tty: true
    volumes:
      - .:/app
    working_dir: /app
```

### 2. 日常開發流程
以後你只要進入資料夾，敲這行啟動容器：
```bash
docker compose up -d
```

現在，這個名為 `c-dev` 的全能 C 語言環境就在背景待命了。
只要你想編譯或執行，就把指令丟進去：

> [!WARNING]
> **注意權限問題 (Permission Denied)！**
> 預設用 `docker compose exec` 去編譯出來的檔案會變成 `root` 權限，若未來要手動刪除或用編輯器修改會跳權限不足。
> 請記得加上 `-u "$(id -u):$(id -g)"` 選項，確保產出的執行檔屬於你的平民帳號：
> *(如果不幸已經變 root 砍不掉，老樣子，在 WSL 終端機敲 `sudo chown -R $USER:$USER .` 就好)*

```bash
# 編譯
docker compose exec -u "$(id -u):$(id -g)" c-dev gcc main.c -o app

# 執行 (不產生新檔案時可以省略 -u)
docker compose exec c-dev ./app

# 如果你寫了 Makefile，甚至可以直接下
docker compose exec -u "$(id -u):$(id -g)" c-dev make
```

你甚至可以在容器裡面直接下 `gdb ./app` 來除錯！
下班的時候一樣敲個 `docker compose down`，就什麼多餘的行程都不留下了。

> [!TIP]
> **如果覺得沒 IntelliSense 很難受怎麼辦？**
> 
> 如果專案變大，真的很需要 Antigravity 的 C/C++ 語法提示跟自動完成，這裡偷偷教個進階做法：
> 直接使用微軟的 **Dev Containers** 擴充功能！我們可以在資料夾內建立 `.devcontainer/devcontainer.json`，讓編輯器直接連接進 `c-dev` 容器裡面進行開發，這樣既有「完美的語法提示」，又維持了本機「不污染」的教條。

[↑ 回到目錄](#目錄)

---

### 結語
寫 C 語言可以不用搞一堆繁雜的環境變數、不用跟 Windows 的路徑搏鬥。這就是我們透過 WSL + Docker 所追求的極致開發自由。

不管今天是要寫 PHP、Vue 還是最古早味的 C 語言，這套環境都能讓你用最乾淨的方式完成任務。

👉 回頭複習：
- [後端篇：Laravel + FrankenPHP (預設)](./README.md)
- [前端篇：Vue / Vite + Docker 實戰](./VUE_VITE_FRONTEND.md)
