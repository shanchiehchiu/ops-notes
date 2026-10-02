---
layout: default
title: Snap 自動更新時段調整說明
permalink: /snap-refresh/
---

# Snap 自動更新時段調整說明

<p class="byline">2026-10-02 ・ 維運紀錄</p>

## 背景

主機原本的 Snap 自動更新排程是：

```text
timer: 00:00~24:00/4
```

意思是系統會在一天之內，隨機安排 4 次 Snap refresh，實際執行時間不固定，有可能落在上班時段。

監控曾抓到 `xdelta3` 與 `snapd` 在更新期間明顯佔用 CPU（監控做法見[《Linux CPU 使用率監控與異常程序追蹤指南》]({{ '/cpu-watch/' | relative_url }})）。為了避免 ERP 使用時段受到背景更新影響，我們把 Snap 的自動更新限制在台灣的凌晨。

## 原始排程

查詢目前的排程：

```bash
snap refresh --time
```

原本看到：

```text
timer: 00:00~24:00/4
last: today at 01:05 UTC
next: today at 06:32 UTC
```

這台主機的系統時區是 UTC，台灣時間比 UTC 快 8 小時，所以：

| UTC | 台灣時間 |
|---|---|
| 01:05 | 09:05 |
| 06:32 | 14:32 |

這兩個時間都落在白天，代表更新可能在上班時間執行。

## 調整方式

把 Snap refresh 限制在這個時段：

| UTC | 台灣時間 |
|---|---|
| 18:00 ～ 20:00 | 02:00 ～ 04:00 |

執行：

```bash
sudo snap set system refresh.timer=18:00-20:00
```

> **換算台灣時間前，請先確認主機時區。** 上面的對照是以主機時區為 UTC 為前提，可以用 `timedatectl` 查看。如果你的主機不是 UTC，同一個設定對應的台灣時間會不一樣。

## 驗證

重新查詢：

```bash
snap refresh --time
```

調整後會看到：

```text
timer: 18:00-20:00
last: today at 01:05 UTC
next: today at 18:00 UTC
```

代表之後 Snap 的自動 refresh 只會安排在：

```text
UTC 18:00-20:00
台灣 02:00-04:00
```

## 查詢最近的更新紀錄

查看 Snap 的變更紀錄：

```bash
snap changes | head -20
```

查看最近 7 天的 `snapd` 紀錄，並只篩出和更新有關的行：

```bash
sudo journalctl -u snapd \
  --since "7 days ago" \
  --no-pager \
  | grep -Ei 'refresh|update|restarting daemon|snapd/'
```

## 注意事項

- **只影響 Snap 的自動更新。** 手動執行 `snap refresh` 不受這個時段限制；系統的 apt 自動更新（`unattended-upgrades`）也有自己的排程，需要另外查：

  ```bash
  systemctl list-timers 'apt-daily*'
  ```

- **確認凌晨 02:00～04:00 沒有其他重工作業。** 如果同一時段已經排了備份或其他排程，更新可能和它們重疊，互相搶資源。

## 結論

原本 Snap 更新可能隨機落在營業時間，監控也曾看到 `snapd`、`xdelta3` 在更新期間造成短暫的 CPU 尖峰。

目前已把自動更新時段改為：

```text
台灣凌晨 02:00～04:00
```

目的是：

> 避免系統背景更新在 ERP 主要使用時段佔用 CPU，降低白天使用者遇到短暫卡頓的機率。
