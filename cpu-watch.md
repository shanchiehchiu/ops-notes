---
layout: default
title: Linux CPU 使用率監控與異常程序追蹤指南
permalink: /cpu-watch/
---

# Linux CPU 使用率監控與異常程序追蹤指南

<p class="byline">2026-10-01 ・ 2026-10-02 更新（V3.1）</p>

## 背景

主機偶爾會出現短暫的 CPU 尖峰，但一般監控圖只能看到「CPU 有升高」，無法得知當下是哪個程序造成的。

因此我們建立一個 CPU Watcher（V3.1 輕量版），核心做法是：

> 平常只讀 `/proc/stat`，幾乎不掃描程序；只有 CPU ≥ 15% 或 steal ≥ 10% 時，才抓一次詳細的程序資訊。

具體來說：

- 每 2 秒讀一次 `/proc/stat`，計算整體 CPU 使用率，並拆解成 user、system、iowait、irq、softirq、steal。
- CPU ≥ 15% 或 steal ≥ 10% 時，自動保存異常現場。
- 觸發後有 10 秒冷卻時間，避免尖峰期間反覆執行重型檢查。
- 同時記錄 Load、記憶體、PHP-FPM、連線與 MySQL 狀態。
- 每 10 分鐘留下一筆存活紀錄，確認監控仍在執行。

## 版本演進

| 版本 | 做法 | 代價 |
|---|---|---|
| 初版 | 每 10 秒算一次 CPU，達標才記錄 | 只看得到偵測當下，看不到尖峰之前 |
| V2 | 每秒用 `ps` 保存程序快照，尖峰時一併寫出前約 10 秒 | 每秒執行一次 `ps` |
| V3 | 每秒掃描全部 `/proc/PID/stat` | Watcher 自己實測佔 5～12% CPU，已棄用 |
| **V3.1** | 平常只讀 `/proc/stat`，達標才取樣一次 | **沒有尖峰前的程序快照** |

V3.1 用「不保存尖峰前快照」換取極低的監控負擔，並且新增了 steal 的偵測（見下方〈判讀〉）。如果你更需要看到尖峰「之前」的程序，可以回頭參考 V2 的做法；這個站台的 git 歷史裡保留了 V2 的腳本。

## 事前準備

### 安裝 pidstat

觸發時的程序取樣使用 `pidstat`，它在 `sysstat` 套件裡，多數系統預設沒有安裝：

```bash
sudo apt install sysstat -y
```

確認：

```bash
pidstat -V
```

沒有安裝也能運作，腳本會退回使用 `ps`，但 `ps` 的 `%CPU` 是程序「從啟動到現在」的平均值，不夠準確（見〈判讀〉）。

### 如果已經在跑舊版

請先停止舊版，再覆蓋腳本：

```bash
pkill -f cpu-watch.sh
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

## 建立監控腳本

直接整段貼上：

```bash
cat > ~/cpu-watch.sh <<'EOF'
#!/usr/bin/env bash

LOG_DIR="$HOME/perf-logs"
LOG="$LOG_DIR/cpu-over-15.log"

CPU_THRESHOLD=15
STEAL_THRESHOLD=10
SAMPLE_SECONDS=2
ALIVE_SECONDS=600
DETAIL_COOLDOWN=10

mkdir -p "$LOG_DIR"

echo "===== CPU WATCH V3.1 START $(date) =====" >> "$LOG"

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

read_cpu

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

while true; do
    sleep "$SAMPLE_SECONDS"

    read_cpu

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

        cpu_irq=$(awk -v t="$diff_total" -v v="$diff_irq" \
            'BEGIN { printf "%.1f", v*100/t }')

        cpu_softirq=$(awk -v t="$diff_total" -v v="$diff_softirq" \
            'BEGIN { printf "%.1f", v*100/t }')

        cpu_steal=$(awk -v t="$diff_total" -v v="$diff_steal" \
            'BEGIN { printf "%.1f", v*100/t }')

        now=$(date +%s)

        # 每 10 分鐘留下存活紀錄
        if [ $((now - last_alive)) -ge "$ALIVE_SECONDS" ]; then
            echo "===== CPU WATCH V3.1 ALIVE $(date) CPU=${cpu_used}% STEAL=${cpu_steal}% =====" >> "$LOG"
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

        # 避免尖峰期間每 2 秒都跑重型詳細檢查
        if { [ "$cpu_trigger" -eq 1 ] || [ "$steal_trigger" -eq 1 ]; } \
            && [ $((now - last_detail)) -ge "$DETAIL_COOLDOWN" ]; then

            {
                echo "=================================================="
                echo "TIME: $(date)"
                echo "CPU: ${cpu_used}%"
                echo

                echo "===== TRIGGER ====="

                if [ "$cpu_trigger" -eq 1 ]; then
                    echo "CPU >= ${CPU_THRESHOLD}%"
                fi

                if [ "$steal_trigger" -eq 1 ]; then
                    echo "STEAL >= ${STEAL_THRESHOLD}%"
                fi

                echo
                echo "===== CPU BREAKDOWN ====="
                echo "user:     ${cpu_user}%"
                echo "system:   ${cpu_system}%"
                echo "iowait:   ${cpu_iowait}%"
                echo "irq:      ${cpu_irq}%"
                echo "softirq:  ${cpu_softirq}%"
                echo "steal:    ${cpu_steal}%"
                echo

                echo "===== PROCESS CPU SAMPLE ====="

                if command -v pidstat >/dev/null 2>&1; then
                    pidstat -u -p ALL 1 1
                else
                    echo "pidstat not found; fallback to ps"
                    ps -eo pid,ppid,user,comm,%cpu,%mem,etime \
                        --sort=-%cpu | head -30
                fi

                echo
                echo "===== LOAD ====="
                uptime
                echo

                echo "===== MEMORY ====="
                free -h
                echo

                echo "===== PHP-FPM ====="
                systemctl status php8.1-fpm --no-pager | head -20
                echo

                echo "===== CONNECTION SUMMARY ====="
                ss -s
                echo

                echo "===== MYSQL ====="
                ps -eo pid,user,comm,%cpu,%mem,etime \
                    --sort=-%cpu \
                    | grep -E 'mysqld|mariadbd' \
                    | head -10

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
| `STEAL_THRESHOLD` | 10 | steal 達此百分比就觸發 |
| `SAMPLE_SECONDS` | 2 | 讀取 `/proc/stat` 的間隔（秒） |
| `ALIVE_SECONDS` | 600 | 存活紀錄的間隔（秒） |
| `DETAIL_COOLDOWN` | 10 | 兩次詳細紀錄之間至少相隔的秒數 |

另外，腳本裡寫死了 `php8.1-fpm`，如果主機上的 PHP 版本不同，請改成對應的服務名稱。

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

所有紀錄都寫入同一個檔案：

```text
~/perf-logs/cpu-over-15.log
```

啟動時，紀錄檔裡會先出現：

```text
===== CPU WATCH V3.1 START Fri Oct 2 ... UTC 2026 =====
```

之後每 10 分鐘會寫一筆，同時帶出當下的 CPU 與 steal：

```text
===== CPU WATCH V3.1 ALIVE ... CPU=1.4% STEAL=0.0% =====
```

看到這行，代表監控仍在正常執行。

## 觸發時的紀錄

只要 CPU ≥ 15% 或 steal ≥ 10%，就會自動保存異常現場。每一筆紀錄依序包含：

1. 時間與當下的 CPU 使用率。
2. `TRIGGER`：是哪個條件觸發的（CPU、steal，或兩者）。
3. `CPU BREAKDOWN`：CPU 使用率的細項拆解。
4. `PROCESS CPU SAMPLE`：1 秒內各程序實際的 CPU 使用率。
5. Load、記憶體、PHP-FPM 狀態、連線摘要與 MySQL 程序。

例如 steal 偏高的情況：

```text
CPU: 43.8%

===== TRIGGER =====
CPU >= 15%
STEAL >= 10%

===== CPU BREAKDOWN =====
user:      3.1%
system:   15.0%
iowait:    0.0%
irq:       0.0%
softirq:   0.2%
steal:    25.5%
```

同樣的方式，未來若是 `php-fpm8.1`、`mariadbd`、`redis-server` 或其他程序吃 CPU，也都會被記錄下來。SSH 暴力登入造成尖峰的案例，見[《SSH 暴力登入與 Fail2ban 防護》]({{ '/ssh-fail2ban/' | relative_url }})。

## 判讀

### 「CPU」數值包含 steal

腳本算出的整體 CPU 是「100% 減去 idle 與 iowait」，所以 steal 也算在裡面。上面的範例中：

```text
3.1 + 15.0 + 0.2 + 25.5 = 43.8
```

也就是那 43.8% 裡，有 25.5% 是 steal，並不是主機自己的程式在用。

**steal** 是虛擬機（VM）才有的數值，代表這台 VM 想用 CPU，卻被底層的宿主機排不到、被別的 VM 搶走的時間。也因此，只要 steal 偏高，就算你的程式什麼都沒做，「CPU」也會被墊高、觸發紀錄。

### 看懂 CPU 細項

| 偏高的欄位 | 通常代表 |
|---|---|
| `user` | 應用程式在運算（PHP、MySQL 查詢等） |
| `system` | 核心層的工作（網路、系統呼叫、檔案操作等） |
| `iowait` | 在等磁碟 I/O |
| `softirq` / `irq` | 網路封包或硬體中斷處理 |
| `steal` | 宿主機資源不足，瓶頸在 VM 之外 |

如果 `steal` 很高而 `user` 很低，問題多半出在主機商的宿主機，而不是你的程式，這時往下追查程序沒有意義，應該改去向主機商反映，或評估更換主機。

想手動確認 steal，可以執行：

```bash
vmstat 1 5
```

看最右邊附近的 `st` 欄。

### 程序取樣的限制

- `pidstat` 取樣的是 1 秒內的實際 CPU 使用率，比 `ps` 準確。沒有安裝 `pidstat` 而退回 `ps` 時，`%CPU` 是程序「從啟動到現在」的平均值，要搭配 `etime`（已執行時間）一起看。
- 程序取樣是在「偵測到尖峰之後」才進行，所以持續時間極短的尖峰，取樣時可能已經結束。這是 V3.1 為了降低負擔所做的取捨。
- `pidstat -p ALL` 會列出所有程序，單筆紀錄可能偏長。

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

只想找出有哪些時間點被觸發：

```bash
grep -E '^TIME:|^STEAL >=|^CPU >=' ~/perf-logs/cpu-over-15.log
```

### 注意紀錄檔大小

如果 steal 或 CPU 長時間偏高，每 10 秒就會寫一筆較長的紀錄，紀錄檔會成長得很快。建議定期檢查：

```bash
du -h ~/perf-logs/cpu-over-15.log
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

> 當 CPU 或 steal 異常升高時，自動保留當下是哪個程序在使用 CPU，以及資源是被自己用掉、還是被宿主機搶走。

可用來追查：

- PHP / Laravel 程序異常
- MySQL / MariaDB 高負載
- Redis
- SSH
- cron 排程
- 備份程序
- 系統背景程序
- 宿主機資源不足（steal）
- 其他不明程序

建議的使用方式：

1. 平常讓 Watcher 在背景執行。
2. 客戶反映卡頓時，用 `tail -f ~/perf-logs/cpu-over-15.log` 即時查看。
3. 先看 `STEAL=`：如果 steal 高，瓶頸在主機商那一層。
4. 如果 steal 不高，再依 `PROCESS CPU SAMPLE` 裡 CPU 最高的程序往下追查。
