---
layout: default
title: Laravel Pulse 導入與多 Guard 403 排查
permalink: /laravel-pulse/
---

# Laravel Pulse 導入與多 Guard 403 排查

<p class="byline">2026-10-17</p>

## 背景

一套 Laravel 10 應用（Ploi 代管、PHP 8.1）導入官方 Laravel Pulse 做效能監控，取代自建儀表板。`/pulse` 沿用後台的登入 session 與權限系統，不對外開放。

部署後遇到一個很隱密的問題：後台管理員開 `/pulse` 一律 403，gate 裡面放的 debug log 也完全沒有輸出。追查後發現是 Laravel Gate 的「訪客跳過機制」在多 guard 架構下的坑，本文記錄部署方式、權限設計與排查過程。

## 名詞

- **Daemon**：Ploi 管理的常駐程序，停止後 Ploi 會重新啟動。
- **guard**：Laravel 的認證通道。本例前台用預設 `web` guard，後台用 `backend` guard。
- **訪客（guest）**：在當前 guard 下沒有登入使用者的請求。
- **超級管理員**：屬於超管群組清單的後台帳號。

## 架構與權限

Pulse 路由掛 `web` middleware，經過 `Laravel\Pulse\Http\Middleware\Authorize` 檢查 `viewPulse` gate。

權限在 `app/Providers/AuthServiceProvider.php` 定義。重點：Pulse 用預設 `web` guard 解析使用者，但後台登在 `backend` guard，所以 gate 必須**自己解析 backend guard**，不能依賴傳入的 `$user`。

```php
Gate::define('viewPulse', function (?\Illuminate\Contracts\Auth\Authenticatable $user = null) {
    $admin = auth()->guard('backend')->user();

    if (! $admin instanceof \App\Models\AdminUser || (int) $admin->status !== 1) {
        return false;
    }

    return in_array($admin->admin_group_id, config('permission.super_admin_groups', []))
        || $admin->permissions()->where('name', 'viewPulse')->where('status', 1)->exists()
        || $admin->permission_overrides()->where('name', 'viewPulse')->where('status', 1)->exists();
});
```

放行規則：

- 超管群組直接放行
- 其他帳號需 `viewPulse` 權限或權限覆寫
- 停用帳號與訪客一律 403

兩個細節：

1. **第一個參數必須是 `?Authenticatable $user = null`**，這是本次 403 的根因修正，見文末附錄。
2. **`(int) $admin->status`**：欄位沒有 cast 時 MySQL 回字串 `"1"`，`=== 1` 嚴格比對會失敗。

## 部署步驟

在專案根目錄依序執行：

```bash
# 1. 安裝相依（正式機一律 install，吃 lock 檔）
composer install --no-dev --optimize-autoloader --no-interaction

# 2. 建立 pulse_* 資料表
php artisan migrate --force

# 3. 清快取
php artisan config:clear
php artisan route:clear

# 4. 重載 PHP-FPM（避免 OPcache 咬舊版）
sudo systemctl reload php8.1-fpm
```

再到 Ploi → Site → Daemons 新增：

| 欄位 | 值 |
| :--- | :--- |
| Command | `php artisan pulse:check` |
| Directory | 專案根目錄 |
| User | `ploi` |
| Processes | `1` |

沒有這支 Daemon，儀表板開得出來但所有數據都不會更新。

另外注意：deploy script 若有 `config:cache` / `route:cache`，必須在 Pulse 裝好後重建，否則舊的路由快取裡沒有 `/pulse`。

## 驗收

- 未登入開 `/pulse` → 403
- `/backend` 登入超管 → `/pulse` → 200，儀表板正常渲染
- 等 1–2 分鐘，Servers 卡片出現 CPU、Memory、Storage 數值
- `pgrep -af "pulse:check"` 有一個程序存活

## 日常維運

- Daemon 健康：`pgrep -af "pulse:check"`，或在 Ploi Daemons 頁面看 log
- 每次部署後例行：`composer install`、`config:clear`、`route:clear`、`reload php8.1-fpm`
- Pulse 依 `config/pulse.php` 的 `storage` 設定自行清理過期資料，不用手動清表
- `Slow Requests` 門檻預設 1,000ms；清快取後的冷啟動請求上榜屬正常，同一路由持續累積才需要排查

## 故障排查

站台通常 `access_log off`，請求記錄查不到，改用回應 header 判斷請求走到哪一層：

```bash
curl -sI https://<站台網域>/pulse
```

- 回應帶 `laravel_session`（或自訂的 session cookie）與 `XSRF-TOKEN` → 請求已到 Laravel，問題在 app 層
- 純 403、無 cookie → 沒進 PHP，查 nginx 與 Ploi 自訂片段（`/etc/nginx/ploi/<站台>/after/*`）
- `server: cloudflare` 只代表經過 CF，不是 CF 擋的證據

| 症狀 | 處理 |
| :--- | :--- |
| 超管也一律 403 | 檢查 gate 第一參數是否為 `?Authenticatable $user = null`，見附錄 |
| 帳號對但 403 | tinker 查 `admin_group_id` 與 `status`；`status` 非 1 是停用，群組不符補 `viewPulse` 權限 |
| `/pulse` 回 404 | `route:list --path=pulse`；路由不在就 `route:clear` 後重建快取 |
| 改完程式碼沒反應 | `systemctl reload php8.1-fpm`，OPcache 咬舊版 |
| 儀表板全無數據 | `pgrep -af "pulse:check"`，Daemon 沒跑就重建 |
| debug log 印不出來 | `.env` 的 `LOG_LEVEL`；`error` 級別會濾掉 `info`/`warning`，臨時 debug 要用 `Log::error` |

## 附錄：一律 403 的根因

### 症狀

`/pulse` 一律 403。超管帳號正確、程式確定部署、FPM 與快取都清過，gate 內連 `Log::error` 都沒有輸出。

### 根因

gate 最初寫成零參數 closure：

```php
Gate::define('viewPulse', function () {
    $admin = auth()->guard('backend')->user();
    // ...
});
```

Laravel 的機制：**請求在預設 guard 下是訪客時，只有 gate closure 第一個參數允許 null，Laravel 才會執行該 gate**。否則直接判定 false，closure 根本不被呼叫。

而 Pulse 用預設 `web` guard 解析使用者，後台管理員登在 `backend` guard——在 `web` guard 眼裡全部是訪客。於是 gate 被無聲跳過：一律 403，且 gate 內任何 log 都印不出來。最後正是「連 `Log::error` 都沒有」這點證明程式根本沒執行，問題在 gate 被呼叫之前。

修正方式是把第一個參數宣告成可空型別：`?Authenticatable $user = null`。`?` 是簽名上的宣告，Laravel 用 reflection 讀它來決定訪客時要不要執行；`= null` 是預設值。Laravel 看的是「型別允不允許 null」，不是「這次傳的是不是 null」。參數本身我們不用，授權完全靠自己解析的 `backend` guard。

### 測試盲點

`actingAs('backend')` 會把預設 guard 切成 `backend`，`$user` 非 null，gate 正常執行——模擬不到正式環境的訪客路徑。改用真實登入測試（實際 POST 登入端點建立 session，再請求 `/pulse`）才在本地複製出一模一樣的 403。

**多 guard 架構下測試授權，必須用真實登入流程驗證訪客路徑，不能只靠 `actingAs`。**

### 排查時排除的假線索

| 假設 | 結果 |
| :--- | :--- |
| `status === 1` 嚴格比對失敗 | MySQL 回字串 `"1"`，確有其事但非根因；改 `(int)` 後 403 依舊 |
| OPcache 咬舊版 | restart PHP-FPM 無效 |
| 網域不同站台 | `server_name` 確認兩網域同站台 |
| session domain 錯誤 | 登入同網域，cookie 正常 |
| route cache 過舊 | `route:list` 顯示路由存在，`route:clear` 無效 |
| 超管群組設定不同 | tinker 查證設定正確 |
| debug log 沒寫 | `Log::info` 被 `LOG_LEVEL=error` 濾掉；改 `Log::error` 仍無輸出 → 證明 closure 沒執行 |
