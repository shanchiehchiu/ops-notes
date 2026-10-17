---
layout: default
title: Laravel Pulse 導入與多 Guard 403 排查
permalink: /laravel-pulse/
---

# Laravel Pulse 導入與多 Guard 403 排查

<p class="byline">2026-10-17</p>

## Laravel Pulse 是什麼

Laravel Pulse 是 Laravel 官方的應用程式效能監控套件。它在應用內收集數據、存進自己的資料表，內建一個即時儀表板，掛在 `/pulse` 路徑。

儀表板內建的監控卡片：

- **Servers**：CPU、Memory、Storage（多台主機會分台列出）
- **Application Usage**：最活躍的使用者與路由
- **Slow Requests / Slow Queries / Slow Jobs / Slow Outgoing Requests**：超過門檻（預設 1,000ms）的項目，含最慢時間與次數
- **Queues**：各 queue 的積壓、處理中、失敗數
- **Cache**：命中率與各 key 的 hit/miss
- **Exceptions**：例外類型與發生次數

資料收集靠一支常駐指令 `php artisan pulse:check`；沒跑這支，儀表板開得出來但所有數據都不會更新。過期資料由 Pulse 依 `config/pulse.php` 的 `storage` 設定自行清理。

適用情境：想知道「站台現在卡不卡、卡在哪個路由、哪個 query、cache 命中率多少」，又不想為單一 Laravel 站自建監控系統時，Pulse 是成本最低的選項。

### 跟 Laravel Telescope 差在哪

兩者都是 Laravel 官方套件，都會記錄 request 與 query，但定位不同：

| | Pulse | Telescope |
| :--- | :--- | :--- |
| 定位 | 正式環境**監控** | 開發環境**除錯** |
| 資料粒度 | 聚合統計：次數、最慢、命中率 | 逐筆記錄：每個 request 的 header、payload、response、query、stack trace |
| 系統層 | 有 CPU、Memory、Storage（靠 `pulse:check` 收集） | 無，只看應用層 |
| 開銷 | 小，設計給 production 長期跑 | 大，每個 request 都寫多筆記錄 |
| 敏感資料 | 存統計數字 | 存請求與回應的完整內容（含密碼欄位需自行遮罩） |
| 官方建議環境 | production | local / dev，不建議上正式機 |

簡單分法：**Telescope 回答「這一筆請求發生了什麼」，Pulse 回答「系統現在健不健康、趨勢往哪走」**。Telescope 上正式機有效能與個資外洩疑慮；兩者定位不重疊，可以併用（Telescope 留開發機，Pulse 上正式機）。

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

Pulse 預設只允許本地環境存取，正式環境必須定義 `viewPulse` gate，否則 `/pulse` 對外開放或一律 403。

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

兩個關鍵細節：

1. **第一個參數必須是 `?Authenticatable $user = null`**——多 guard 架構下寫錯這裡會一律 403，根因見文末踩坑紀錄。
2. **`(int) $admin->status`**：欄位沒有 cast 時 MySQL 回字串 `"1"`，嚴格比對 `=== 1` 會失敗。

若是單 guard 的標準應用，官方文件的寫法 `function (User $user)` 即可；自行解析其他 guard 才需要 nullable 參數。

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
| 超管也一律 403 | 檢查 gate 第一參數是否為 `?Authenticatable $user = null`，見下節 |
| 帳號對但 403 | tinker 查 `admin_group_id` 與 `status`；`status` 非 1 是停用，群組不符補 `viewPulse` 權限 |
| `/pulse` 回 404 | `route:list --path=pulse`；路由不在就 `route:clear` 後重建快取 |
| 改完程式碼沒反應 | `systemctl reload php8.1-fpm`，OPcache 咬舊版 |
| 儀表板全無數據 | `pgrep -af "pulse:check"`，Daemon 沒跑就重建 |
| debug log 印不出來 | `.env` 的 `LOG_LEVEL`；`error` 級別會濾掉 `info`/`warning`，臨時 debug 要用 `Log::error` |

站台 `access_log off` 時，用回應 header 判斷請求走到哪一層：`curl -sI` 回應帶 session cookie 與 `XSRF-TOKEN` → 已到 Laravel，問題在 app 層；無 cookie → 沒進 PHP，往 nginx 查。

## 踩坑紀錄：一律 403 的根因

### 症狀

部署後 `/pulse` 一律 403。超管帳號正確、程式確定部署、FPM 與快取都清過，gate 內連 `Log::error` 都沒有輸出。

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
