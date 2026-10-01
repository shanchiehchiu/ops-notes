---
layout: default
title: 主機卡頓排查紀錄：SSH 暴力登入與 Fail2ban 防護
---

# 主機卡頓排查紀錄：SSH 暴力登入與 Fail2ban 防護

<p class="byline">2026-10-01 ・ 維運紀錄</p>

## 背景

客戶反映系統在晚間偶爾卡頓。我們先檢查主機的 CPU、記憶體、硬碟與網路使用狀況：整體資源沒有長時間過載，但即時監控抓到了短暫的 CPU 尖峰。

其中一次監控的輸出如下：

```text
CPU: 28.4%

TOP PROCESSES
sshd        43.5%
sshd        21.1%
php-fpm8.1   0.4%
mariadbd     0.1%
```

當時 CPU 主要被 SSH 服務佔用，而不是網站（php-fpm）或資料庫（mariadbd）。

## 發現 SSH 外部攻擊

接著查看 SSH 服務在尖峰時段的紀錄：

```bash
sudo journalctl -u ssh \
  --since "2026-10-01 13:36:30" \
  --until "2026-10-01 13:39:00" \
  --no-pager
```

輸出中會看到類似下列的內容：

```text
Failed password for root from 125.129.127.204
Invalid user admin from 125.129.127.204
Invalid user deploy from 137.184.79.87
maximum authentication attempts exceeded
```

從紀錄可以確認，多個外部 IP 持續嘗試以 `root`、`admin`、`worker`、`ftpuser` 等常見帳號登入。部分來源被斷線後，會立刻重新連線繼續嘗試。這類大量的登入嘗試，正是 `sshd` CPU 升高的原因。

## 安裝 Fail2ban

Fail2ban 會監看登入失敗的紀錄，自動封鎖反覆失敗的來源 IP。

### 安裝

```bash
sudo apt update
sudo apt install fail2ban -y
```

安裝成功時，輸出會包含：

```text
Reading package lists... Done
Building dependency tree... Done
Setting up fail2ban ...
```

### 建立設定

```bash
sudo nano /etc/fail2ban/jail.local
```

填入以下內容：

```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 你的公開IP
findtime = 10m
maxretry = 5
bantime = 1h
backend = systemd

[sshd]
enabled = true
port = ssh
filter = sshd
```

這份設定的意思是：

> 同一個 IP 在 10 分鐘內 SSH 登入失敗 5 次，就封鎖 1 小時。

> **注意**：請把 `你的公開IP` 換成自己常用的連線 IP，避免操作失誤時把自己鎖在門外。

## 檢查並啟用

先驗證設定檔是否正確：

```bash
sudo fail2ban-client -t
```

正常會顯示：

```text
OK: configuration test is successful
```

重啟服務並設為開機自動啟動：

```bash
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban
```

最後查看 SSH 防護的運作狀態：

```bash
sudo fail2ban-client status sshd
```

實際輸出類似：

```text
Status for the jail: sshd
|- Filter
|  |- Currently failed: 1
|  |- Total failed: 78
|  `- Journal matches: _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 5
   |- Total banned: 5
   `- Banned IP list: 2.57.121.112 45.148.10.151 125.129.127.204 137.184.79.87 62.60.130.253
```

`Filter` 區塊是偵測端的統計，`Actions` 區塊則是實際封鎖的結果：

- `Currently failed`／`Total failed`：目前仍在計數視窗內、以及累計的登入失敗次數。
- `Currently banned`／`Total banned`：目前被封鎖、以及累計封鎖的 IP 數。
- `Banned IP list`：目前被封鎖的 IP，以空白分隔。

啟用後，Fail2ban 已開始自動封鎖反覆嘗試登入的來源，前面紀錄中出現的 `125.129.127.204`、`137.184.79.87` 也都在封鎖清單內。

## 結論

- 已確認主機存在 SSH 暴力登入行為，並且會造成 CPU 短暫升高。
- 已部署 Fail2ban：同一來源短時間內多次登入失敗，系統會自動封鎖，降低 SSH 攻擊對主機資源的影響。
