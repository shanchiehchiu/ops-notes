---
layout: default
title: Laravel Pulse 導入與多 Guard 403 排查
permalink: /laravel-pulse/
---

# Laravel Pulse 導入與多 Guard 403 排查

<p class="byline">2026-10-17</p>

## Laravel Pulse 是什麼

Laravel Pulse 是 Laravel 官方的效能監控套件。它把數據存進自己的資料表，並提供儀表板（路徑 `/pulse`），可以看 CPU / Memory、慢請求、慢 query、queue、cache 命中率和例外。

資料由常駐指令 `php artisan pulse:check` 收集。沒跑這支，儀表板打得開，但數據不會更新。

跟 Telescope 的分工：Telescope 逐筆記錄請求內容，只適合開發機。Pulse 只存聚合統計，開銷小，適合正式機。兩者可以併用。

環境需求：PHP `^8.1`、Laravel `^10.48.4` 起（Pulse v1.8，2026-10）。

## 部署步驟

以下以 Ploi 代管、PHP 8.1 的 Laravel 10 站台為例。

### 1. 安裝套件

```bash
composer require laravel/pulse
php artisan vendor:publish --provider="Laravel\Pulse\PulseServiceProvider"
php artisan migrate
```

`vendor:publish` 會產生 `config/pulse.php` 與 dashboard view。Pulse 需要 PHP `^8.1`。

注意 PHP 版本對齊：若本機 PHP 比正式機新，`composer.json` 的 `config.platform.php` 要釘在正式機版本（例如 `8.1.2`），讓 lock 檔按正式環境解析相依；正式機一律 `composer install` 吃 lock，不跑 `composer update`。

### 2. 設定存取權限

Pulse 預設只允許本地環境存取。正式環境沒定義 `viewPulse` gate 時，`/pulse` 一律 403。

在 `app/Providers/AuthServiceProvider.php` 定義：

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

這段假設後台走獨立的 `backend` guard、模型為 `AdminUser`，放行規則：超管群組直接放行，其他帳號需 `viewPulse` 權限或覆寫，停用帳號與訪客一律 403。

兩個細節：

- 第一個參數必須寫成 `?Authenticatable $user = null`。多 guard 架構寫錯會一律 403，原因見[文末踩坑](#踩坑多-guard-下一律-403)。單 guard 的標準應用用官方的 `function (User $user)` 即可。
- `status` 要用 `(int)` 轉型。欄位沒有 cast 時，MySQL 回字串 `"1"`，`=== 1` 會失敗。

### 3. 部署到正式機

```bash
composer install --no-dev --optimize-autoloader --no-interaction
php artisan migrate --force
php artisan config:clear
php artisan route:clear
sudo systemctl reload php8.1-fpm
```

deploy script 若有 `config:cache` / `route:cache`，必須在 Pulse 裝好後重建，否則舊路由快取裡沒有 `/pulse`。

### 4. 建立資料收集 Daemon

Ploi → Site → Daemons → Add：

| 欄位 | 值 |
| :--- | :--- |
| Command | `php artisan pulse:check` |
| Directory | 專案根目錄 |
| User | `ploi` |
| Processes | `1` |

非 Ploi 環境用 Supervisor 或 systemd 跑同一支指令即可。

### 5. 驗收

- 未登入開 `/pulse` → 403
- 後台登入超管 → `/pulse` → 200
- 等 1–2 分鐘，Servers 卡片出現 CPU、Memory、Storage 數值
- `pgrep -af "pulse:check"` 有一個程序存活

## 日常維運與排查速查

- Daemon 健康：`pgrep -af "pulse:check"`，或在 Ploi Daemons 頁面看 log
- 每次部署後例行：`composer install`、`config:clear`、`route:clear`、`reload php8.1-fpm`
- `Slow Requests` 門檻預設 1,000ms；清快取後的冷啟動請求上榜屬正常，同一路由持續累積才需要排查

| 症狀 | 處理 |
| :--- | :--- |
| 超管也一律 403 | 檢查 gate 第一參數是否為 `?Authenticatable $user = null`，見[踩坑](#踩坑多-guard-下一律-403) |
| 帳號對但 403 | tinker 查 `admin_group_id` 與 `status`；`status` 非 1 是停用，群組不符補 `viewPulse` 權限 |
| `/pulse` 回 404 | `route:list --path=pulse`；路由不在就 `route:clear` 後重建快取 |
| 改完程式碼沒反應 | `systemctl reload php8.1-fpm`，OPcache 咬舊版 |
| 儀表板全無數據 | `pgrep -af "pulse:check"`，Daemon 沒跑就重建 |
| debug log 印不出來 | `.env` 的 `LOG_LEVEL`；`error` 級別會濾掉 `info`/`warning`，臨時 debug 要用 `Log::error` |

站台 `access_log off` 時，用回應 header 判斷請求走到哪一層：`curl -sI` 回應帶 session cookie 與 `XSRF-TOKEN` → 已到 Laravel，問題在 app 層；無 cookie → 沒進 PHP，往 nginx 查。


## 踩坑：多 guard 下一律 403

**症狀**：部署後 `/pulse` 一律 403。超管帳號正確，程式已部署，FPM 與快取都清過，gate 內連 `Log::error` 都沒輸出。

**根因**：gate 最初寫成零參數 closure：

```php
Gate::define('viewPulse', function () {
    $admin = auth()->guard('backend')->user();
    // ...
});
```

Laravel 對訪客只有一個規則：gate closure 的第一個參數允許 null，才會執行 gate。否則直接判 false，closure 不會被呼叫。

Pulse 用預設 `web` guard 解析使用者，但後台管理員登在 `backend` guard。在 `web` guard 眼裡，管理員全是訪客，gate 被無聲跳過。

「連 `Log::error` 都沒輸出」就是關鍵證據：closure 根本沒執行。

**修正**：第一個參數改成 `?Authenticatable $user = null`。Laravel 看的是參數型別「允不允許 null」，不是這次傳的值。參數本身用不到，授權靠自己解析的 `backend` guard。

**測試盲點**：`actingAs('backend')` 會把預設 guard 切成 `backend`，`$user` 不是 null，gate 照常執行，模擬不出訪客路徑。要用真實登入流程（POST 登入端點、帶 session 請求 `/pulse`）才重現得出 403。多 guard 架構測授權，不能只靠 `actingAs`。

**已排除**：`status` 嚴格比對、OPcache、網域與 session 設定、route cache、超管群組設定、`LOG_LEVEL` 過濾。這些都試過，403 依舊。
