---
layout: default
title: Linux CPU 使用率監控與異常程序追蹤指南
permalink: /cpu-watch/
---

# Linux CPU 使用率監控與異常程序追蹤指南

<p class="byline">2026-10-01 ・ 2026-10-02 更新（V3.4）</p>

## 背景

主機偶爾會出現短暫的 CPU 尖峰，但一般監控圖只能看到「CPU 有升高」，無法得知當下是哪個程序、甚至是哪一個網址造成的。

因此我們建立一個 CPU Watcher（V3.4），整體流程是：

```text
每 2 秒取樣 CPU
   ↓
CPU >= 15%
   ↓
找出 CPU >= 10% 的 PHP-FPM worker
   ↓
取得 PID
   ↓
用 PID 對照 PHP-FPM access log
   ↓
列出 Laravel 的 request_uri、執行時間、記憶體
   ↓
同時觀察 MariaDB / Nginx 等相關程序
```

PHP worker 的 CPU 不是等整體 CPU 超標才開始算，而是每次取樣都持續透過 `/proc/<pid>/stat` 計算。所以即使只持續一、兩秒的高 CPU request，在觸發時也已經有數值可以對照。

要做到這件事，需要三個東西配合：

1. **PHP-FPM access log**：記錄每個 request 由哪個 PID 處理、真實網址是什麼。
2. **Log 讀取權限**：Watcher 以 `ploi` 身分執行，必須能讀取這份 log。
3. **Logrotate**：Watcher 與 access log 都會持續成長，需要自動輪替。

## 版本演進

| 版本 | 做法 | 代價 |
|---|---|---|
| 初版 | 每 10 秒算一次 CPU，達標才記錄 | 只看得到偵測當下，看不到尖峰之前 |
| V2 | 每秒用 `ps` 保存程序快照，尖峰時一併寫出前約 10 秒 | 每秒執行一次 `ps` |
| V3 | 每秒掃描全部 `/proc/PID/stat` | Watcher 自己實測佔 5～12% CPU，已棄用 |
| V3.1 | 平常只讀 `/proc/stat`，達標才用 `pidstat` 取樣 | 只知道哪個程序忙，無法對應到網址 |
| **V3.4** | 平常讀 `/proc/stat` 與 PHP-FPM worker 的 CPU 時間，達標時用 PID 對照 access log 找出網址 | 需要設定 access log 與權限；移除了 Load、記憶體、連線、MySQL 程序等區塊 |

V3.4 的取捨是：放棄 V3.1 那份「通用的系統狀態快照」，換成能直接回答「**哪個網址**讓 PHP 吃 CPU」。V2、V3.1 的腳本都保留在這個站台的 git 歷史裡。

## 事前準備

### PHP-FPM 現況設定

目前 PHP-FPM 的 pool 設定如下：

```ini
pm = dynamic
pm.max_children = 10
pm.start_servers = 3
pm.min_spare_servers = 2
pm.max_spare_servers = 5
```

也就是最多可以同時有 10 個 PHP worker 處理 request。

確認服務狀態：

```bash
systemctl status php8.1-fpm --no-pager
```

### 開啟 PHP-FPM access log

編輯 pool 設定檔：

```bash
sudo nano /etc/php/8.1/fpm/pool.d/www.conf
```

加入：

```ini
access.log = /var/log/php8.1-fpm-access.log
access.format = "%t pid=%p method=%m request_uri=%{REQUEST_URI}e script=%r status=%s duration=%{milli}dms memory=%{kilo}MKB"
```

這樣每一個 PHP request 都會記錄：

| 欄位 | 內容 |
|---|---|
| `%t` | 收到 request 的時間 |
| `pid` | 處理這個 request 的 worker PID |
| `method` | HTTP Method |
| `request_uri` | 真實的 request URI |
| `script` | PHP 入口腳本 |
| `status` | HTTP 狀態碼 |
| `duration` | 執行時間（毫秒） |
| `memory` | 記憶體使用量（KB） |

Laravel 的所有請求都會經過 `index.php`，所以只看 `script` 會永遠是 `/index.php`。真正的網址要靠 `%{REQUEST_URI}e` 取得，這也是這個設定的重點。

修改後先檢查設定：

```bash
sudo php-fpm8.1 -t
```

成功會看到：

```text
configuration file /etc/php/8.1/fpm/php-fpm.conf test is successful
```

再重新載入：

```bash
sudo systemctl reload php8.1-fpm
```

查看紀錄：

```bash
sudo tail -f /var/log/php8.1-fpm-access.log
```

實際驗證可以看到類似：

```text
02/Oct/2026:02:49:39 +0000 pid=4034905 method=GET request_uri=/backend/dashboard-v2 script=/index.php status=200 duration=703.398ms memory=14336KB
```

> **注意一：`access.log` 是每個 pool 各自生效的設定。** 如果主機上有多個 pool（例如每個網站一個 pool），每個 pool 都要加；而且 V3.4 只會讀取上面設定的這一份 log。

> **注意二：`REQUEST_URI` 含查詢字串。** 網址後面的參數（搜尋關鍵字、token 等）也會被寫進這份 log，所以下一節把權限收緊，並設定輪替避免長期保存。

### 讓 Watcher 能讀取 access log

PHP-FPM 建立的 access log，預設權限是 `-rw------- root root`，只有 root 讀得到。Watcher 以 `ploi` 身分執行，讀不到就會顯示 `PHP access log unavailable`，所以要讓 `ploi` 能唯讀這份 log：

```bash
sudo chown root:ploi /var/log/php8.1-fpm-access.log
sudo chmod 640 /var/log/php8.1-fpm-access.log
```

確認：

```bash
ls -l /var/log/php8.1-fpm-access.log
```

預期：

```text
-rw-r----- 1 root ploi ...
```

用 `ploi` 身分測試，不加 `sudo` 能看到內容就代表 Watcher 讀得到：

```bash
tail -1 /var/log/php8.1-fpm-access.log
```

### 安裝 pidstat

`CURRENT RELATED PROCESSES` 區塊用 `pidstat` 取樣，它在 `sysstat` 套件裡，多數系統預設沒有安裝：

```bash
sudo apt install sysstat -y
```

確認：

```bash
pidstat -V
```

V3.4 沒有安裝 `pidstat` 時不會退回其他做法，這個區塊會直接是空的。其餘功能（尤其是 `HOT PHP REQUESTS`）不依賴它。

### 如果已經在跑舊版

請先停止舊版，再覆蓋腳本：

```bash
pkill -f '/home/ploi/cpu-watch.sh'
```

確認已停止：

```bash
pgrep -af cpu-watch.sh
```

沒有輸出就代表已停止。

Bash 是邊讀邊執行腳本的，在執行中直接覆蓋檔案，可能讓正在跑的舊程序讀到錯亂的內容，所以務必先停再蓋。

V2 留下的快照資料夾已經用不到，可以順手刪除：

```bash
rm -rf ~/perf-logs/cpu-snapshots
```

## 建立監控腳本（V3.4）

直接整段貼上：

```bash
cat > ~/cpu-watch.sh <<'EOF'
#!/usr/bin/env bash

LOG_DIR="$HOME/perf-logs"
LOG="$LOG_DIR/cpu-over-15.log"
PHP_ACCESS_LOG="/var/log/php8.1-fpm-access.log"

CPU_THRESHOLD=15
PHP_CPU_THRESHOLD=10
STEAL_THRESHOLD=20

SAMPLE_SECONDS=2
ALIVE_SECONDS=600
DETAIL_COOLDOWN=10

CLK_TCK=$(getconf CLK_TCK)

mkdir -p "$LOG_DIR"

echo "===== CPU WATCH V3.4 START $(date) =====" >> "$LOG"

declare -A PHP_PREV_TICKS
declare -A PHP_CURRENT_CPU

read_cpu() {
    read -r _ user nice system idle iowait irq softirq steal guest guest_nice < /proc/stat

    CPU_USER=$((user + nice))
    CPU_SYSTEM=$system
    CPU_IDLE=$idle
    CPU_IOWAIT=$iowait
    CPU_IRQ=$irq
    CPU_SOFTIRQ=$softirq
    CPU_STEAL=$steal

    CPU_TOTAL=$((user + nice + system + idle + iowait + irq + softirq + steal))
}

get_php_workers() {
    local master_pid

    master_pid=$(pgrep -xo php-fpm8.1 2>/dev/null)

    if [ -n "$master_pid" ]; then
        pgrep -P "$master_pid" php-fpm8.1 2>/dev/null
    fi
}

get_process_ticks() {
    local pid="$1"

    if [ -r "/proc/$pid/stat" ]; then
        awk '{ print $14 + $15 }' "/proc/$pid/stat"
    fi
}

init_php_ticks() {
    local pid ticks

    while read -r pid; do
        [ -z "$pid" ] && continue

        ticks=$(get_process_ticks "$pid")

        if [ -n "$ticks" ]; then
            PHP_PREV_TICKS["$pid"]="$ticks"
        fi
    done < <(get_php_workers)
}

sample_php_cpu() {
    local elapsed="$1"
    local pid ticks prev diff cpu

    PHP_CURRENT_CPU=()

    while read -r pid; do
        [ -z "$pid" ] && continue

        ticks=$(get_process_ticks "$pid")

        [ -z "$ticks" ] && continue

        prev="${PHP_PREV_TICKS[$pid]:-}"

        if [ -n "$prev" ]; then
            diff=$((ticks - prev))

            cpu=$(awk \
                -v ticks="$diff" \
                -v hz="$CLK_TCK" \
                -v seconds="$elapsed" \
                'BEGIN {
                    if (seconds > 0)
                        printf "%.2f", (ticks / hz) * 100 / seconds
                    else
                        printf "0.00"
                }')

            PHP_CURRENT_CPU["$pid"]="$cpu"
        fi

        PHP_PREV_TICKS["$pid"]="$ticks"
    done < <(get_php_workers)

    for pid in "${!PHP_PREV_TICKS[@]}"; do
        if [ ! -d "/proc/$pid" ]; then
            unset 'PHP_PREV_TICKS[$pid]'
        fi
    done
}

find_php_request() {
    local pid="$1"

    if [ ! -r "$PHP_ACCESS_LOG" ]; then
        echo "PHP access log unavailable"
        return
    fi

    local match

    match=$(
        tail -n 1000 "$PHP_ACCESS_LOG" 2>/dev/null \
        | grep -a "pid=${pid} " \
        | tail -1
    )

    if [ -n "$match" ]; then
        echo "$match"
    else
        echo "request_uri: no recent completed request found"
    fi
}

read_cpu
init_php_ticks

prev_total=$CPU_TOTAL
prev_user=$CPU_USER
prev_system=$CPU_SYSTEM
prev_idle=$CPU_IDLE
prev_iowait=$CPU_IOWAIT
prev_irq=$CPU_IRQ
prev_softirq=$CPU_SOFTIRQ
prev_steal=$CPU_STEAL

last_alive=$(date +%s)
last_detail=0
last_sample=$(date +%s.%N)

while true; do
    sleep "$SAMPLE_SECONDS"

    current_sample=$(date +%s.%N)

    elapsed=$(awk \
        -v now="$current_sample" \
        -v prev="$last_sample" \
        'BEGIN { printf "%.3f", now-prev }')

    last_sample="$current_sample"

    read_cpu
    sample_php_cpu "$elapsed"

    diff_total=$((CPU_TOTAL - prev_total))
    diff_user=$((CPU_USER - prev_user))
    diff_system=$((CPU_SYSTEM - prev_system))
    diff_idle=$((CPU_IDLE - prev_idle))
    diff_iowait=$((CPU_IOWAIT - prev_iowait))
    diff_irq=$((CPU_IRQ - prev_irq))
    diff_softirq=$((CPU_SOFTIRQ - prev_softirq))
    diff_steal=$((CPU_STEAL - prev_steal))

    if [ "$diff_total" -gt 0 ]; then

        cpu_used=$(awk -v t="$diff_total" \
            -v idle="$diff_idle" \
            -v wait="$diff_iowait" \
            'BEGIN { printf "%.1f", (t-idle-wait)*100/t }')

        cpu_user=$(awk -v t="$diff_total" -v v="$diff_user" \
            'BEGIN { printf "%.1f", v*100/t }')

        cpu_system=$(awk -v t="$diff_total" -v v="$diff_system" \
            'BEGIN { printf "%.1f", v*100/t }')

        cpu_iowait=$(awk -v t="$diff_total" -v v="$diff_iowait" \
            'BEGIN { printf "%.1f", v*100/t }')

        cpu_softirq=$(awk -v t="$diff_total" -v v="$diff_softirq" \
            'BEGIN { printf "%.1f", v*100/t }')

        cpu_steal=$(awk -v t="$diff_total" -v v="$diff_steal" \
            'BEGIN { printf "%.1f", v*100/t }')

        now=$(date +%s)

        if [ $((now - last_alive)) -ge "$ALIVE_SECONDS" ]; then
            echo "===== CPU WATCH V3.4 ALIVE $(date) CPU=${cpu_used}% STEAL=${cpu_steal}% =====" >> "$LOG"
            last_alive=$now
        fi

        cpu_trigger=0
        steal_trigger=0

        if awk -v c="$cpu_used" -v t="$CPU_THRESHOLD" \
            'BEGIN { exit !(c >= t) }'; then
            cpu_trigger=1
        fi

        if awk -v c="$cpu_steal" -v t="$STEAL_THRESHOLD" \
            'BEGIN { exit !(c >= t) }'; then
            steal_trigger=1
        fi

        if { [ "$cpu_trigger" -eq 1 ] || [ "$steal_trigger" -eq 1 ]; } \
            && [ $((now - last_detail)) -ge "$DETAIL_COOLDOWN" ]; then

            {
                echo "=================================================="
                echo "TIME: $(date)"
                echo "CPU: ${cpu_used}%"
                echo

                echo "===== CPU BREAKDOWN ====="
                echo "user:     ${cpu_user}%"
                echo "system:   ${cpu_system}%"
                echo "iowait:   ${cpu_iowait}%"
                echo "softirq:  ${cpu_softirq}%"
                echo "steal:    ${cpu_steal}%"
                echo

                echo "===== HOT PHP REQUESTS ====="

                hot_count=0

                for pid in "${!PHP_CURRENT_CPU[@]}"; do
                    php_cpu="${PHP_CURRENT_CPU[$pid]}"

                    if awk \
                        -v cpu="$php_cpu" \
                        -v threshold="$PHP_CPU_THRESHOLD" \
                        'BEGIN { exit !(cpu >= threshold) }'; then

                        hot_count=$((hot_count + 1))

                        echo "PID=${pid} CPU=${php_cpu}%"
                        find_php_request "$pid"
                        echo
                    fi
                done

                if [ "$hot_count" -eq 0 ]; then
                    echo "No PHP-FPM worker >= ${PHP_CPU_THRESHOLD}% in trigger interval"
                fi

                echo
                echo "===== CURRENT RELATED PROCESSES ====="

                if command -v pidstat >/dev/null 2>&1; then
                    pidstat -u -p ALL 1 1 2>/dev/null \
                    | awk '
                        $0 ~ /(mariadbd|nginx|snapd|sshd)$/ &&
                        $1 != "Average:" &&
                        $(NF-2) + 0 >= 1 {
                            printf "%-15s PID=%-8s CPU=%s%%\n", $NF, $4, $(NF-2)
                        }
                    '
                fi

                echo

            } >> "$LOG" 2>&1

            last_detail=$now
        fi
    fi

    prev_total=$CPU_TOTAL
    prev_user=$CPU_USER
    prev_system=$CPU_SYSTEM
    prev_idle=$CPU_IDLE
    prev_iowait=$CPU_IOWAIT
    prev_irq=$CPU_IRQ
    prev_softirq=$CPU_SOFTIRQ
    prev_steal=$CPU_STEAL
done
EOF
```

### 可調整的參數

腳本開頭的變數可以依主機調整：

| 變數 | 預設 | 說明 |
|---|---|---|
| `CPU_THRESHOLD` | 15 | 整體 CPU 達此百分比就觸發 |
| `PHP_CPU_THRESHOLD` | 10 | 單一 PHP-FPM worker 達此百分比，才會被列進 `HOT PHP REQUESTS` |
| `STEAL_THRESHOLD` | 20 | steal 達此百分比就觸發（V3.1 是 10） |
| `SAMPLE_SECONDS` | 2 | 取樣間隔（秒） |
| `ALIVE_SECONDS` | 600 | 存活紀錄的間隔（秒） |
| `DETAIL_COOLDOWN` | 10 | 兩次詳細紀錄之間至少相隔的秒數 |

`CPU_THRESHOLD` 與 `STEAL_THRESHOLD` 要一起看：整體 CPU 的數值已經包含 steal（見〈判讀〉），所以只要 steal ≥ 20%，CPU 就必然 ≥ 20%，早就達到 15% 的門檻。目前這組設定下，`STEAL_THRESHOLD` 不會單獨觸發紀錄；只有把它調得比 `CPU_THRESHOLD` 低時才有作用。

另外，腳本裡有幾處是寫死的，換主機時要一併檢查：

- `php-fpm8.1`（`pgrep` 找 master 與 worker 用）與 `PHP_ACCESS_LOG` 的路徑，必須跟 PHP 版本、access log 設定一致。
- `CURRENT RELATED PROCESSES` 只會列出名稱為 `mariadbd`、`nginx`、`snapd`、`sshd` 且 CPU ≥ 1% 的程序。

## 語法檢查與啟動

先檢查語法，沒有輸出代表正常：

```bash
bash -n ~/cpu-watch.sh
```

加上執行權限：

```bash
chmod +x ~/cpu-watch.sh
```

確認舊的 Watcher 已停止（見〈如果已經在跑舊版〉），然後背景啟動：

```bash
nohup ~/cpu-watch.sh >/dev/null 2>&1 &
```

即使 SSH 連線關閉，Watcher 仍會繼續執行。

確認是否正在執行，應該只有一個：

```bash
pgrep -af cpu-watch.sh
```

```text
xxxxxxx bash /home/ploi/cpu-watch.sh
```

查看啟動紀錄：

```bash
tail -20 ~/perf-logs/cpu-over-15.log
```

應該會看到：

```text
===== CPU WATCH V3.4 START ...
```

之後每 10 分鐘會寫一筆存活紀錄，同時帶出當下的 CPU 與 steal：

```text
===== CPU WATCH V3.4 ALIVE ... CPU=1.4% STEAL=0.0% =====
```

## 觸發時的紀錄

Watcher 的紀錄檔在：

```text
/home/ploi/perf-logs/cpu-over-15.log
```

當 CPU 超過 15%，例如某個 ERP 頁面造成 PHP 高負載，紀錄會是這樣（範例）：

```text
==================================================
TIME: Fri Oct 2 03:01:22 UTC 2026
CPU: 31.9%

===== CPU BREAKDOWN =====
user:     28.0%
system:   2.9%
iowait:   0.0%
softirq:  0.5%
steal:    0.5%

===== HOT PHP REQUESTS =====
PID=4034905 CPU=67.50%
02/Oct/2026:03:01:21 +0000 pid=4034905 method=GET request_uri=/backend/dashboard-v2 script=/index.php status=200 duration=703.398ms memory=14336KB

===== CURRENT RELATED PROCESSES =====
mariadbd       PID=43036    CPU=15.00%
nginx          PID=122495   CPU=13.00%
```

各區塊的意思：

1. 時間與當下的 CPU 使用率。
2. `CPU BREAKDOWN`：CPU 使用率的細項拆解。
3. `HOT PHP REQUESTS`：取樣區間內 CPU ≥ 10% 的 PHP-FPM worker，以及它最近處理過的 request。
4. `CURRENT RELATED PROCESSES`：同一時間 MariaDB、Nginx 等相關程序的 CPU 使用率。

這樣就把「CPU 高 → 哪個 PHP worker → 哪個網址 → 執行多久、用多少記憶體 → 資料庫與 Nginx 是否也在忙」串在同一筆紀錄裡。

SSH 暴力登入造成尖峰的案例，見[《SSH 暴力登入與 Fail2ban 防護》]({{ '/ssh-fail2ban/' | relative_url }})。

## 實際案例：`/backend/sales/order`

Watcher 曾經抓到這樣的情況：

| 項目 | 內容 |
|---|---|
| 整體 CPU | 36.8% |
| PHP-FPM worker | PID=4034905，CPU=51.96% |
| MariaDB | CPU=13% |

同一時間，PHP-FPM access log 裡該 PID 的 request 是 `GET /backend/sales/order`。同一個功能在 access log 裡還出現過下面這些紀錄：

| 來源 | request_uri | duration | memory |
|---|---|---|---|
| Watcher 抓到的那次（03:04:40） | `/backend/sales/order` | 1322.854 ms | 38912 KB（約 38 MB） |
| access log 另見 | `/backend/sales/order?ajax=1...` | 2574.913 ms | 105688 KB（約 103 MB） |
| access log 另見 | `/backend/sales/order?...` | 2873.612 ms | 103640 KB（約 101 MB） |

目前觀察到的是：`/backend/sales/order` 這類 request 會伴隨較明顯的 PHP CPU 與 MariaDB 負載，執行時間約 1.3～2.9 秒，帶查詢參數的版本記憶體用量也高出不少。

這只是「同時出現」的相關性，還不是根因。Watcher 的工作到「找出是哪個網址」為止，所以接下來不需要再強化 Watcher，而是針對這個反覆出現的高負載網址，往 Laravel 層查：

- Controller 的處理流程
- SQL query（有沒有慢查詢、重複查詢）
- DataTable 的查詢
- 關聯資料的載入成本

## 判讀

### 「CPU」數值包含 steal

腳本算出的整體 CPU 是「100% 減去 idle 與 iowait」，所以 steal 也算在裡面。

**steal** 是虛擬機（VM）才有的數值，代表這台 VM 想用 CPU，卻被底層的宿主機排不到、被別的 VM 搶走的時間。只要 steal 偏高，就算你的程式什麼都沒做，「CPU」也會被墊高、觸發紀錄。

### 看懂 CPU 細項

| 偏高的欄位 | 通常代表 |
|---|---|
| `user` | 應用程式在運算（PHP、MySQL 查詢等） |
| `system` | 核心層的工作（網路、系統呼叫、檔案操作等） |
| `iowait` | 在等磁碟 I/O |
| `softirq` | 網路封包處理 |
| `steal` | 宿主機資源不足，瓶頸在 VM 之外 |

如果 `steal` 很高而 `user` 很低，問題多半出在主機商的宿主機，而不是你的程式，這時往下追查程序沒有意義，應該改去向主機商反映，或評估更換主機。

想手動確認 steal，可以執行：

```bash
vmstat 1 5
```

看最右邊附近的 `st` 欄。

### 怎麼讀 `HOT PHP REQUESTS`

PHP-FPM access log 是在 request **結束時**才寫入一行，`%t` 記錄的則是**收到 request 的時間**。所以 Watcher 找到的，是這個 worker「最近一筆已經完成」的 request，這代表：

- **最常見的情況**：造成尖峰的 request 很短（例如這個例子的 703 ms），在 Watcher 查詢時已經結束，這時找到的就是兇手。
- **要小心的情況**：如果某個 request 還在執行中（例如一個要跑 30 秒的報表），它還沒寫進 log，這時看到的會是同一個 worker「上一個」request，不是正在吃 CPU 的那個。

因此請把時間與 `duration` 一起看：request 的收到時間，加上 `duration`，應該要落在尖峰發生的時間附近；如果對不上，就要懷疑那是上一筆。

出現 `request_uri: no recent completed request found` 時，常見原因有：

- 這個 worker 最近的 request 還沒完成（尖峰期間常見）。
- 流量大，Watcher 只查看 access log 最後 1000 行，該 worker 的最後一筆已經被擠出範圍。
- 該 PID 不是由這份 access log 對應的 pool 處理（見前面關於多個 pool 的注意事項）。

出現 `PHP access log unavailable` 代表 `ploi` 讀不到這份 log，回頭檢查權限。

出現 `No PHP-FPM worker >= 10% in trigger interval` 則代表這次尖峰不是 PHP 造成的，往 `CPU BREAKDOWN` 與 `CURRENT RELATED PROCESSES` 找。

### 紀錄裡出現 `binary file matches`

如果 Watcher 的紀錄裡出現：

```text
grep: (standard input): binary file matches
```

同時 `request_uri` 又顯示找不到，代表 access log 裡有 `grep` 判定為非文字的內容（例如請求網址帶有無效編碼的位元組，掃描程式的探測請求常會這樣），`grep` 就只回報「有符合」，而不輸出那一行。新版 `grep` 會把這則訊息寫到 stderr，所以 request 查不到；較舊的 `grep` 則會把 `Binary file (standard input) matches` 當成一般輸出，結果這句話會被當成 request 印在紀錄裡。

腳本裡的 `grep -a` 就是為了這件事：強制把 access log 當文字處理，才找得到 request。如果你是從舊版腳本升級，請確認 `find_php_request` 裡用的是 `grep -a`。

### 其他限制

- `HOT PHP REQUESTS` 的順序是依 PID 任意排列，不是依 CPU 由高到低。
- PHP-FPM 是 `dynamic` 模式，worker 會視負載新增與消失。兩次取樣之間才出現、又消失的 worker，不會被看到；新出現的 worker 也要到下一次取樣才有數值。
- `CURRENT RELATED PROCESSES` 的 PID 取自 `pidstat` 輸出的第 4 欄，這只在時間欄位是 `HH:MM:SS AM/PM`（佔兩欄）的格式下才正確；如果主機的語系讓時間只佔一欄，PID 欄位會取成 `%usr` 的數值。看到 PID 明顯不對時，先檢查這點。

## Logrotate

Watcher 的紀錄與 PHP access log 都會持續成長，兩個都要設定輪替。以下這組設定已在實際主機上測試成功。

### Watcher 紀錄

建立設定檔：

```bash
sudo nano /etc/logrotate.d/cpu-watch
```

內容：

```conf
/home/ploi/perf-logs/cpu-over-15.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    maxsize 20M
    su ploi ploi
}
```

### PHP-FPM access log

建立設定檔：

```bash
sudo nano /etc/logrotate.d/php8.1-fpm-access
```

內容：

```conf
/var/log/php8.1-fpm-access.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    maxsize 50M
    su root root
}
```

### 各項設定的意思

| 設定 | 作用 |
|---|---|
| `daily` / `rotate 14` | 每天輪替，保留 14 份 |
| `compress` / `delaycompress` | 舊檔壓縮，但最新的一份先不壓縮 |
| `missingok` / `notifempty` | 檔案不存在或是空的就跳過 |
| `copytruncate` | 複製後把原檔清空，不需要讓 Watcher 或 PHP-FPM 重新開檔，不會中斷 |
| `su ploi ploi` | 以 `ploi` 身分輪替，因為 Watcher 的紀錄放在 `ploi` 的家目錄 |
| `maxsize` | 超過指定大小時，在**下一次 logrotate 執行時**輪替 |

> **`maxsize` 不會讓檔案在一天之內提前輪替。** logrotate 預設由系統排程每天只執行一次，`maxsize` 只在它執行的那一刻才會檢查，所以在 `daily` 的設定下，它實際上沒有額外效果。如果想真的限制單日內的大小，需要讓 logrotate 更頻繁地執行（例如另外排一個每小時執行的工作），或是調高 `CPU_THRESHOLD`、拉長 `DETAIL_COOLDOWN` 來減少 Watcher 的寫入量。

可以用下面的指令確認 logrotate 的執行排程：

```bash
systemctl list-timers | grep logrotate
```

### 測試

先用 dry-run 檢查設定（不會真的輪替）：

```bash
sudo logrotate -d /etc/logrotate.d/cpu-watch
sudo logrotate -d /etc/logrotate.d/php8.1-fpm-access
```

需要時再強制執行一次：

```bash
sudo logrotate -f /etc/logrotate.d/cpu-watch
sudo logrotate -f /etc/logrotate.d/php8.1-fpm-access
```

查看輪替後的檔案：

```bash
sudo ls -lh /var/log/php8.1-fpm-access*
```

## 即時查看

平常不用盯著畫面，讓 Watcher 在背景執行即可。

如果正在測試，或客戶反映「現在很卡」，可以用下面的指令即時追蹤：

```bash
tail -f ~/perf-logs/cpu-over-15.log
```

有新的異常紀錄時，會直接顯示在畫面上。

要離開時按 `Ctrl + C`。這只會停止 `tail -f`，**不會停止背景的 Watcher**。

## 事後查看

看最後 100 行：

```bash
tail -n 100 ~/perf-logs/cpu-over-15.log
```

看最後 200 行：

```bash
tail -n 200 ~/perf-logs/cpu-over-15.log
```

只想看有哪些 PHP request 被抓到：

```bash
grep -A1 '^PID=' ~/perf-logs/cpu-over-15.log
```

## 停止 Watcher

```bash
pkill -f '/home/ploi/cpu-watch.sh'
```

確認是否已停止：

```bash
pgrep -af cpu-watch.sh
```

沒有輸出就代表已停止。

## 重新啟動

```bash
nohup ~/cpu-watch.sh >/dev/null 2>&1 &
```

再確認一次：

```bash
pgrep -af cpu-watch.sh
```

## 用途

這個 Watcher 不只是用來查 SSH 攻擊，它的目的是：

> 當 CPU 異常升高時，自動保留當下是哪個 PHP request、哪個程序在使用 CPU，以及資源是被自己用掉、還是被宿主機搶走。

可用來追查：

- 哪個 Laravel 網址造成 PHP 高負載
- MySQL / MariaDB 高負載
- Nginx、SSH 等相關程序
- 宿主機資源不足（steal）

建議的使用方式：

1. 平常讓 Watcher 在背景執行。
2. 客戶反映卡頓時，用 `tail -f ~/perf-logs/cpu-over-15.log` 即時查看。
3. 先看 `CPU BREAKDOWN`：如果 `steal` 高，瓶頸在主機商那一層。
4. 如果 `steal` 不高，看 `HOT PHP REQUESTS` 找出是哪個網址，並比對 `duration` 與時間。
5. 沒有 PHP worker 偏高時，再看 `CURRENT RELATED PROCESSES` 的 MariaDB、Nginx 往下追查。
6. 同一個網址反覆出現時，就不再是 Watcher 的問題，改往 Laravel 的 controller、SQL 與資料載入查（見〈實際案例〉）。
