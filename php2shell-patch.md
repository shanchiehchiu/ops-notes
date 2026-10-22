---
layout: default
title: PHP2Shell 漏洞修補與 Ubuntu 自動更新確認
permalink: /php2shell-patch/
---

# PHP2Shell 漏洞修補與 Ubuntu 自動更新確認

<p class="byline">2026-10-22 ・ 維運紀錄</p>

## 背景

2026/10/21，Ubuntu 發布 USN-8948-1，修補 PHP 的 PHP2Shell 漏洞，對應 CVE-2026-109618、CVE-2026-109619。這是重大安全漏洞，因此我們先確認主機是否已自動更新，再決定是否手動補強。

官方公告：https://ubuntu.com/security/notices/USN-8948-1

## 確認自動更新機制

- 主機使用 `unattended-upgrades`，每日檢查並安裝安全更新。
- 歷史紀錄顯示，PHP 已自動升級 4 次。
- Ubuntu 22.04 的 PHP 8.1 修補版本為 `8.1.2-1ubuntu2.28`。
- 更新前主機為 `8.1.2-1ubuntu2.27`，`.28` 已可取得，但尚未自動安裝。

結論是 Ubuntu 的自動更新機制有效，但等待排程不足以應付重大漏洞，因此改為手動確認並安裝。

## 暫時防護

在安裝修補版本前，先關閉 PHP Session Upload Progress，降低部分攻擊風險。

## 處理步驟

1. 確認系統版本：`lsb_release -a`
2. 確認 PHP 套件來源：`apt-cache policy php8.2-fpm`（PHP 8.1 則對應 `php8.1-fpm`）
3. 判斷來源：
   - 套件來自 `*-security`，且自動更新政策允許，Ubuntu 可自動安裝安全更新。
   - 套件來自 PPA，需確認修補版本，並透過 APT 更新。PPA 預設不一定納入自動更新。
   - 使用 Docker 或自行編譯 PHP，依原本的部署方式更新。不必直接下載 PHP 原始碼。
4. 確認修補版本。PHP 8.2 的官方修補版本為 8.2.35。
5. 模擬升級，確認套件變更內容無誤。
6. 正式更新 PHP 套件。
7. 重啟 PHP-FPM。
8. 驗證 PHP 版本已升級至修補版本。

## 結論

- 已完成：PHP 升級至修補版本。
- 已完成：關閉 PHP Session Upload Progress，作為暫時防護。
- 已確認：主機已啟用自動安全更新，且修補版本已由 Ubuntu 發布。
- 注意：PHP 8.2 與 PHP 8.1 的處理方式不同，應先以 `lsb_release -a` 與 `apt-cache policy` 確認實際版本與來源，再套用對應步驟。
