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

## 一、PHP 8.1 更新經驗（Ubuntu 22.04）

主機環境為 Ubuntu 22.04，PHP 版本為 8.1。

**確認自動更新**

- 主機使用 `unattended-upgrades`，每日檢查並安裝安全更新。
- 歷史紀錄顯示，PHP 已自動升級 4 次。

**確認修補版本**

- USN-8948-1 列出 22.04（jammy）的 php8.1 修補版本為 `8.1.2-1ubuntu2.28`。
- 更新前主機為 `8.1.2-1ubuntu2.27`，`.28` 已可取得，但尚未自動安裝。
- 因為是重大漏洞，我們沒有只等待自動排程，而是手動安裝 `.28`，並重啟 PHP-FPM。

**暫時防護（後備方案）**

在無法立即更新時，可先關閉 Session Upload Progress。此設定只封鎖已確認的 session 攻擊路徑，不能取代更新。

```ini
; php.ini，或 FPM pool / 伺服器設定（需在請求解析前套用）
session.upload_progress.enabled = 0
```

注意事項：

- 必須寫在 `php.ini`、FPM pool 或伺服器設定中。`ini_set()` 與 FPM 的 `.user.ini` 無效。
- 修改後，必須重啟所有 PHP worker。
- 本次我們已關閉此設定，作為暫時防護。

**PHP 官方提到的其他緩解措施（僅供參考，同樣不能取代更新）**

- CVE-2026-109618：可考慮設定 `memory_limit=-1`（建議以 `php_admin_value` 設定），並調整 `post_max_size` 與 `upload_max_filesize`；避免使用 `RLIMIT_AS` 或 `RLIMIT_DATA`。
- 隱藏錯誤訊息（`error_reporting`、`display_errors`、`@`）無法防護這兩個漏洞。

**經驗小結**

- 自動更新機制有效，但重大漏洞應主動確認修補版本並手動安裝。
- 22.04 的 PHP 8.1 仰賴 Ubuntu 回補修補。PHP 官方已不再為 8.1 發布更新，長期應規劃升級至仍受支援的版本。

## 二、PHP 8.2 建議更新方式

PHP 8.2 的官方修補版本為 `8.2.35`。USN-8948-1 未列出 php8.2，因此不能假設 Ubuntu 安全源會自動提供 8.2 的修補。

建議依下列順序處理：

1. 確認系統版本：`lsb_release -a`
2. 確認 PHP 8.2 套件來源：`apt-cache policy php8.2-fpm`
3. 依來源判斷：
   - 來自 Ubuntu 官方套件（`*-security`）：依 USN 的版本更新，並確認自動更新政策是否涵蓋。
   - 來自 Ondřej PPA：PPA 不一定納入自動更新，需確認 PPA 已發布 `8.2.35` 等修補版本，再透過 APT 手動更新。
   - 使用 Docker 或自行編譯：依原本的部署方式更新，不必直接下載 PHP 原始碼。
4. 模擬升級，確認套件變更內容無誤。
5. 正式更新 PHP 套件。
6. 重啟 PHP-FPM 或所有 PHP worker。
7. 驗證 PHP 版本已升級至修補版本。

在更新完成前，可先套用上方的 `session.upload_progress.enabled = 0` 作為後備方案。

## 結論

- 已完成：主機 PHP 8.1 升級至 `8.1.2-1ubuntu2.28`（依 USN-8948-1）。
- 已完成：關閉 Session Upload Progress，作為暫時防護。
- 已確認：主機已啟用自動安全更新，但重大漏洞應手動確認與安裝。
- 未確認：PHP 8.2 的修補來源。依上方步驟 2 確認後再決定。
- 長期建議：規劃將 PHP 8.1 升級至受支援版本。
