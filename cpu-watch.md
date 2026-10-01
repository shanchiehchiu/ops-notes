---
layout: default
title: CPU 使用率監控與異常程序追蹤指南
permalink: /cpu-watch/
---

# CPU 使用率監控與異常程序追蹤指南

<p class="byline">2026-10-01 ・ 維運紀錄</p>

## 背景

主機偶爾會出現短暫的 CPU 尖峰，但一般監控圖只能看到「CPU 有升高」，無法得知當下是哪個程序造成的。

因此我們建立一個 CPU Watcher，做到：

- 每 10 秒檢查一次 CPU。
- CPU 使用率達 15% 以上時，自動記錄當下的系統狀態。
- 每 10 分鐘寫一筆存活紀錄，確認監控仍在執行。
- 可用來追查 PHP、MySQL、Redis、SSH、排程或其他 Linux 程序。

## 建立監控腳本

執行：

```bash
cat > ~/cpu-watch.sh <<'EOF'
#!/usr/bin/env bash

mkdir -p "$HOME/perf-logs"
LOG="$HOME/perf-logs/cpu-over-15.log"

echo "===== CPU WATCH START $(date) =====" >> "$LOG"

last_total=0
last_idle=0
last_alive=$(date +%s)

while true; do
    read -r cpu user nice system idle iowait irq softirq steal rest < /proc/stat

    total=$((user + nice + system + idle + iowait + irq + softirq + steal))
    idle_all=$((idle + iowait))

    if [ "$last_total" -ne 0 ]; then
        diff_total=$((total - last_total))
        diff_idle=$((idle_all - last_idle))

        if [ "$diff_total" -gt 0 ]; then
            cpu_used=$(awk -v t="$diff_total" -v i="$diff_idle" \
                'BEGIN { printf "%.1f", (t-i)*100/t }')

            now=$(date +%s)

            # 每 10 分鐘寫一筆存活紀錄
            if [ $((now - last_alive)) -ge 600 ]; then
                echo "===== CPU WATCH ALIVE $(date) CPU=${cpu_used}% =====" >> "$LOG"
                last_alive=$now
            fi

            # CPU >= 15% 才寫詳細資料
            if awk -v c="$cpu_used" 'BEGIN {exit !(c >= 15)}'; then
                {
                    echo "=================================================="
                    echo "TIME: $(date)"
                    echo "CPU: ${cpu_used}%"
                    echo

                    echo "===== TOP PROCESSES ====="
                    ps -eo pid,ppid,user,comm,%cpu,%mem --sort=-%cpu | head -20
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
                    ps -eo pid,user,comm,%cpu,%mem --sort=-%cpu \
                        | grep -E 'mysqld|mariadbd' | head -10
                    echo
                } >> "$LOG" 2>&1
            fi
        fi
    fi

    last_total=$total
    last_idle=$idle_all

    sleep 10
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

紀錄會寫入：

```text
~/perf-logs/cpu-over-15.log
```

啟動時，檔案裡會先出現：

```text
===== CPU WATCH START Thu Oct 1 13:00:00 UTC 2026 =====
```

之後每 10 分鐘會寫一筆：

```text
===== CPU WATCH ALIVE Thu Oct 1 13:10:00 UTC 2026 CPU=2.7% =====
```

看到這行，代表監控仍在正常執行。

## CPU 超過 15% 時

只要 CPU 使用率達 15% 以上，就會自動記錄詳細資料，例如：

```text
==================================================
TIME: Thu Oct 1 01:38:20 PM UTC 2026
CPU: 28.4%

===== TOP PROCESSES =====
PID      PPID USER    COMMAND       %CPU %MEM
3476648  ...  root    sshd          43.5  0.2
3476631  ...  root    sshd          21.1  0.2
3470811  ...  ploi    php-fpm8.1     0.4  2.3
43036    ...  mysql   mariadbd       0.1  9.2
```

這次就是在 CPU 28.4% 時抓到：主要來源是 `sshd`，PHP-FPM 與 MariaDB 的使用率都很低。後續追查見[《主機卡頓排查紀錄》]({{ '/ssh-fail2ban/' | relative_url }})。

同樣的方式，未來若是 `php-fpm8.1`、`mariadbd`、`redis-server` 或其他程序吃 CPU，也都會被記錄下來。

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

> 當 CPU 異常升高時，自動保留當下是哪個程序在使用 CPU。

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
3. 依當下 CPU 最高的程序，再往下一層追查。
