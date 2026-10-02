---
layout: default
title: Linux CPU 使用率監控與異常程序追蹤指南
permalink: /cpu-watch/
---

# Linux CPU 使用率監控與異常程序追蹤指南

<p class="byline">2026-10-01 ・ 2026-10-02 更新（V2）</p>

## 背景

主機偶爾會出現短暫的 CPU 尖峰，但一般監控圖只能看到「CPU 有升高」，無法得知當下是哪個程序造成的。

因此我們建立一個 CPU Watcher（V2），做到：

- 每 1 秒保存一次程序（process）快照。
- 每 10 秒計算一次整體 CPU 使用率。
- CPU 使用率達 15% 以上時，自動保存異常現場。
- 同時保存尖峰發生前的程序快照。
- 同時記錄 Load、記憶體、PHP-FPM、連線與 MySQL 狀態。
- 每 10 分鐘留下一筆存活紀錄，確認監控仍在執行。

### V2 改進了什麼

初版每 10 秒檢查一次，只在偵測到 CPU 偏高的那一刻才記錄，看不到尖峰「之前」發生了什麼；如果造成尖峰的程序很短暫，偵測到時它可能已經結束。

V2 改為持續保存近期的程序快照，偵測到尖峰時，會把前面約 10 筆快照（約 10 秒）一起寫進紀錄，方便回頭看是誰在尖峰前就開始吃 CPU。

## 建立監控腳本

### 如果已經在跑舊版

請先停止舊版，再覆蓋腳本：

```bash
pkill -f cpu-watch.sh
```

Bash 是邊讀邊執行腳本的，在執行中直接覆蓋檔案，可能讓正在跑的舊程序讀到錯亂的內容。

### 建立 V2 腳本

執行：

```bash
cat > ~/cpu-watch.sh <<'EOF'
#!/usr/bin/env bash

LOG_DIR="$HOME/perf-logs"
LOG="$LOG_DIR/cpu-over-15.log"
SNAPSHOT_DIR="$LOG_DIR/cpu-snapshots"

mkdir -p "$LOG_DIR" "$SNAPSHOT_DIR"

echo "===== CPU WATCH V2 START $(date) =====" >> "$LOG"

last_total=0
last_idle=0
last_check=$(date +%s)
last_alive=$(date +%s)

take_snapshot() {
    local now
    now=$(date +%s)

    {
        echo "TIME: $(date)"
        ps -eo pid,ppid,user,comm,%cpu,%mem,etime --sort=-%cpu | head -30
    } > "$SNAPSHOT_DIR/$now.log"

    # 只保留最近約 1～2 分鐘的快照
    find "$SNAPSHOT_DIR" -type f -name '*.log' -mmin +1 -delete
}

while true; do
    take_snapshot

    now=$(date +%s)

    # 每 10 秒計算一次整體 CPU
    if [ $((now - last_check)) -ge 10 ]; then
        read -r cpu user nice system idle iowait irq softirq steal rest < /proc/stat

        total=$((user + nice + system + idle + iowait + irq + softirq + steal))
        idle_all=$((idle + iowait))

        if [ "$last_total" -ne 0 ]; then
            diff_total=$((total - last_total))
            diff_idle=$((idle_all - last_idle))

            if [ "$diff_total" -gt 0 ]; then
                cpu_used=$(awk -v t="$diff_total" -v i="$diff_idle" \
                    'BEGIN { printf "%.1f", (t-i)*100/t }')

                # 每 10 分鐘留下存活紀錄
                if [ $((now - last_alive)) -ge 600 ]; then
                    echo "===== CPU WATCH ALIVE $(date) CPU=${cpu_used}% =====" >> "$LOG"
                    last_alive=$now
                fi

                # CPU >= 15% 記錄詳細資料
                if awk -v c="$cpu_used" 'BEGIN {exit !(c >= 15)}'; then
                    {
                        echo "=================================================="
                        echo "TIME: $(date)"
                        echo "CPU: ${cpu_used}%"
                        echo

                        echo "===== PROCESS SNAPSHOTS BEFORE SPIKE ====="
                        for f in $(find "$SNAPSHOT_DIR" -type f -name '*.log' | sort | tail -10); do
                            echo
                            echo "--- $f ---"
                            cat "$f"
                        done
                        echo

                        echo "===== CURRENT TOP PROCESSES ====="
                        ps -eo pid,ppid,user,comm,%cpu,%mem,etime --sort=-%cpu | head -30
                        echo

                        echo "===== LOAD ====="
                        uptime
                        echo

                        echo "===== MEMORY ====="
                        free -h
                        echo

                        echo "===== PHP-FPM ====="
                        systemctl status php8.1-fpm --no-pager | head -25
                        echo

                        echo "===== CONNECTION SUMMARY ====="
                        ss -s
                        echo

                        echo "===== MYSQL PROCESS ====="
                        ps -eo pid,user,comm,%cpu,%mem,etime --sort=-%cpu \
                            | grep -E 'mysqld|mariadbd' | head -10
                        echo
                    } >> "$LOG" 2>&1
                fi
            fi
        fi

        last_total=$total
        last_idle=$idle_all
        last_check=$now
    fi

    sleep 1
done
EOF
```

這個版本直接讀取 `/proc/stat` 來計算 CPU 使用率，不依賴 `top` 的輸出。早期用 `top` 的版本會因為欄位抓錯，而誤判成 `CPU: 100%`。

## 加上執行權限

```bash
chmod +x ~/cpu-watch.sh
```

## 背景啟動

```bash
nohup ~/cpu-watch.sh >/dev/null 2>&1 &
```

即使 SSH 連線關閉，Watcher 仍會繼續執行。

確認是否正在執行：

```bash
pgrep -af cpu-watch.sh
```

正常會看到類似：

```text
3471234 /usr/bin/env bash /home/ploi/cpu-watch.sh
```

## 紀錄位置

異常紀錄會寫入：

```text
~/perf-logs/cpu-over-15.log
```

每秒一次的程序快照則放在：

```text
~/perf-logs/cpu-snapshots/
```

快照只保留最近約 1～2 分鐘，舊的會自動刪除，不會持續佔用硬碟。

啟動時，紀錄檔裡會先出現：

```text
===== CPU WATCH V2 START Thu Oct 1 13:00:00 UTC 2026 =====
```

之後每 10 分鐘會寫一筆：

```text
===== CPU WATCH ALIVE Thu Oct 1 13:10:00 UTC 2026 CPU=2.7% =====
```

看到這行，代表監控仍在正常執行。

## CPU 超過 15% 時

只要 CPU 使用率達 15% 以上，就會自動保存異常現場。每一筆紀錄依序包含：

1. 時間與當下的 CPU 使用率。
2. `PROCESS SNAPSHOTS BEFORE SPIKE`：尖峰發生前的程序快照（最近 10 筆）。
3. `CURRENT TOP PROCESSES`：偵測到尖峰當下，CPU 使用率最高的程序。
4. Load、記憶體、PHP-FPM 狀態、連線摘要與 MySQL 程序。

例如：

```text
==================================================
TIME: Thu Oct 1 01:38:20 PM UTC 2026
CPU: 28.4%

===== PROCESS SNAPSHOTS BEFORE SPIKE =====

--- /home/ploi/perf-logs/cpu-snapshots/<時間戳>.log ---
...

===== CURRENT TOP PROCESSES =====
PID      PPID USER    COMMAND       %CPU %MEM ELAPSED
3476648  ...  root    sshd          43.5  0.2 ...
3476631  ...  root    sshd          21.1  0.2 ...
3470811  ...  ploi    php-fpm8.1     0.4  2.3 ...
43036    ...  mysql   mariadbd       0.1  9.2 ...
```

這次就是在 CPU 28.4% 時抓到：主要來源是 `sshd`，PHP-FPM 與 MariaDB 的使用率都很低。後續追查見[《SSH 暴力登入與 Fail2ban 防護》]({{ '/ssh-fail2ban/' | relative_url }})。

同樣的方式，未來若是 `php-fpm8.1`、`mariadbd`、`redis-server` 或其他程序吃 CPU，也都會被記錄下來。

### 判讀時的注意事項

`ps` 顯示的 `%CPU` 是該程序「從啟動到現在」的平均使用率，不是當下瞬間的數值。因此請搭配 `ETIME`（已執行時間）一起看：

- 剛啟動不久的程序（例如新建立的 SSH 連線），`%CPU` 會特別明顯。
- 已執行很久的程序（例如 `mariadbd`），即使剛才突然變忙，平均值也不太會立刻拉高。

## 即時查看

平常不用盯著畫面，讓 Watcher 在背景執行即可。

如果正在測試，或客戶反映「現在很卡」，可以用下面的指令即時追蹤：

```bash
tail -f ~/perf-logs/cpu-over-15.log
```

有新的 CPU 異常紀錄時，會直接顯示在畫面上。

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

異常紀錄因為包含多筆快照，單筆會比較長，必要時可以多看幾行。

## 停止 Watcher

```bash
pkill -f cpu-watch.sh
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

> 當 CPU 異常升高時，自動保留當下，以及尖峰發生前，是哪個程序在使用 CPU。

可用來追查：

- PHP / Laravel 程序異常
- MySQL / MariaDB 高負載
- Redis
- SSH
- cron 排程
- 備份程序
- 系統背景程序
- 其他不明程序

建議的使用方式：

1. 平常讓 Watcher 在背景執行。
2. 客戶反映卡頓時，用 `tail -f ~/perf-logs/cpu-over-15.log` 即時查看。
3. 依當下與尖峰前 CPU 最高的程序，再往下一層追查。
