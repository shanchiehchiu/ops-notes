---
layout: default
title: PHP2Shell 漏洞修補與 Ubuntu 自動更新確認
permalink: /php2shell-patch/
---

# PHP2Shell 漏洞修補與 Ubuntu 自動更新確認

<p class="byline">2026-10-22 ・ 維運紀錄</p>

## 背景

2026/10/21，PHP 官方與 Ubuntu 同日發布安全更新，修補兩個可導致未授權遠端程式碼執行（RCE）的漏洞：

- CVE-2026-109618：記憶體不足（OOM）時，請求資料只完成部分初始化，清理流程仍將無效資料當成有效資料處理。
- CVE-2026-109619：請求專屬的內部狀態在記憶體釋放後仍被保留，同一 worker 的後續請求可能使用到過期狀態。

PHP 官方公告：https://www.php.net/archive/2026.php#2026-10-21-5z

Ubuntu 安全公告：https://ubuntu.com/security/notices/USN-8948-1

## 立即處置：關閉 Session Upload Progress

在無法立即更新時，先關閉 Session Upload Progress。此設定只封鎖已確認的 session 攻擊路徑，不能取代更新。

```ini
; php.ini，或 FPM pool / 伺服器設定（需在請求解析前套用）
session.upload_progress.enabled = 0
```

- 必須寫在 `php.ini`、FPM pool 或伺服器設定中。`ini_set()` 與 FPM 的 `.user.ini` 無效。
- 修改後，必須重啟所有 PHP worker。
- 本次已關閉此設定，作為暫時防護。

PHP 官方提到的其他緩解措施（同樣不能取代更新）：

- CVE-2026-109618：可考慮設定 `memory_limit=-1`（建議以 `php_admin_value` 設定），並調整 `post_max_size` 與 `upload_max_filesize`；避免使用 `RLIMIT_AS` 或 `RLIMIT_DATA`。
- 隱藏錯誤訊息（`error_reporting`、`display_errors`、`@`）無法防護這兩個漏洞。

## 一、PHP 8.1 更新（Ubuntu 22.04）

目標：從 `8.1.2-1ubuntu2.27` 更新至 Ubuntu Security 的 `8.1.2-1ubuntu2.28`，不切換到 Ondřej PPA。

**自動更新確認**

- 主機使用 `unattended-upgrades`，每日檢查並安裝安全更新。
- 歷史紀錄顯示，PHP 已自動升級 4 次。
- 自動排程安裝 `.28` 可以接受，不需要額外手動操作。
- 若要縮短暴露時間，可依下列步驟手動安裝 `.28`。

**更新步驟（手動安裝，選用）**

1. 檢查目前版本（唯讀）：`php -v`、`php-fpm8.1 -v`
2. 更新套件索引：`sudo apt update`（不升級套件，不重啟服務）
3. 確認修補版本（唯讀）：

   ```bash
   apt-cache policy php8.1-fpm php8.1-common
   ```

   應看到 `8.1.2-1ubuntu2.28`，來源為 `security.ubuntu.com/ubuntu jammy-security`。`Candidate` 可能顯示 Ondřej PPA 的 `8.1.34`，此時不得執行 `apt upgrade`。
4. 模擬安裝，不修改系統：

   ```bash
   sudo apt-get -s install \
     php8.1-fpm=8.1.2-1ubuntu2.28 \
     php8.1-common=8.1.2-1ubuntu2.28
   ```

   確認沒有相依衝突、`Remv` 或非預期的 PPA 套件。
5. 正式安裝。PHP 核心及標準擴充套件可固定版本安裝；Redis、Memcached 等另外打包的擴充套件不得強制套用 `.28`。
6. 測試設定並重啟 PHP-FPM：

   ```bash
   sudo php-fpm8.1 -t
   sudo systemctl restart php8.1-fpm
   ```

   重啟可能造成短暫服務中斷。

**驗證**

```bash
php -v
dpkg-query -W php8.1-fpm php8.1-common php8.1-opcache
systemctl is-active php8.1-fpm
php-fpm8.1 -i | grep session.upload_progress.enabled
```

預期結果：套件版本為 `8.1.2-1ubuntu2.28`，服務為 `active`，`session.upload_progress.enabled => Off => Off`。

## 二、PHP 8.2 建議更新方式

PHP 8.2 的官方修補版本為 `8.2.35`。USN-8948-1 未列出 php8.2，因此不能假設 Ubuntu 安全源會提供 8.2 的修補。

**本主機現況**

- 主機的 `php8.2-fpm` 未安裝（`Installed: (none)`）。目前運作的是 PHP 8.1，本節不影響現有服務。
- 若之後要使用 PHP 8.2，`Candidate` 為 Ondřej PPA 的 `8.2.35-1+ubuntu22.04.1+deb.sury.org+1`，需安裝，不是更新。

**步驟**

1. 確認系統版本（唯讀）：

   ```bash
   lsb_release -a
   ```

2. 確認沒有已安裝的 PHP 8.2 套件（唯讀）：

   ```bash
   dpkg-query -W 'php8.2*'
   ```

   無輸出代表未安裝。若已安裝，依既有版本逐項確認是否為 `8.2.35`。

3. 確認套件來源（唯讀）：

   ```bash
   apt-cache policy php8.2-fpm
   ```

   應看到來源為 `ppa.launchpadcontent.net/ondrej/php/ubuntu jammy/main`。

4. 模擬安裝，不修改系統：

   ```bash
   sudo apt-get -s install php8.2-fpm=8.2.35-1+ubuntu22.04.1+deb.sury.org+1
   ```

   確認沒有相依衝突，且沒有移除 `php8.1-*` 套件。

5. 正式安裝：

   ```bash
   sudo apt-get install php8.2-fpm=8.2.35-1+ubuntu22.04.1+deb.sury.org+1
   ```

6. 測試設定並重啟 PHP 8.2-FPM。只有在網站 pool 已切換到 8.2 時才執行重啟：

   ```bash
   sudo php-fpm8.2 -t
   sudo systemctl restart php8.2-fpm
   ```

7. 驗證：

   ```bash
   php-fpm8.2 -v
   systemctl is-active php8.2-fpm
   ```

   預期結果：版本為 `8.2.35`，服務為 `active`。

在安裝或切換完成前，先套用上方的 `session.upload_progress.enabled = 0`。

**注意**

- 安裝 PHP 8.2 不會自動讓網站改用 8.2。網站仍由 Nginx 指向的 PHP-FPM socket 決定。
- 若 PPA 未加入，需先加入 `ondrej/php` PPA，此步驟會改變套件來源，應先與維運確認。

## 結論

- 已完成：關閉 Session Upload Progress，作為暫時防護。
- 已確認：PHP 8.1 修補版本為 `8.1.2-1ubuntu2.28`（依 USN-8948-1）。安裝方式為自動排程或手動安裝，兩者皆可接受。
- 已確認：主機已啟用 `unattended-upgrades`。
- 未確認：PHP 8.2 的修補來源，依第二節步驟 2 確認。
- 長期建議：規劃將 PHP 8.1 升級至受支援版本。PHP 官方已不再為 8.1 發布更新。
