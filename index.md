---
layout: default
title: 維運筆記
---

# 維運筆記

<p class="byline">實際維運案例的排查過程與處理紀錄</p>

<ul class="post-list">
  <li>
    <a href="{{ '/php2shell-patch/' | relative_url }}">
      <span class="post-date">2026-10-22</span>
      <span class="post-title">PHP2Shell 漏洞修補與 Ubuntu 自動更新確認</span>
      <span class="post-desc">確認 Ubuntu 自動更新機制，並完成 PHP 修補版本的安裝與暫時防護。</span>
    </a>
  </li>
  <li>
    <a href="{{ '/laravel-pulse/' | relative_url }}">
      <span class="post-date">2026-10-17</span>
      <span class="post-title">Laravel Pulse 導入與多 Guard 403 排查</span>
      <span class="post-desc">導入官方 Pulse 監控，並追查多 guard 架構下 gate 被訪客機制跳過、一律 403 的根因。</span>
    </a>
  </li>
  <li>
    <a href="{{ '/snap-refresh/' | relative_url }}">
      <span class="post-date">2026-10-02</span>
      <span class="post-title">Snap 自動更新時段調整說明</span>
      <span class="post-desc">把 Snap 自動更新限制在台灣凌晨，避免更新佔用 CPU 影響白天使用。</span>
    </a>
  </li>
  <li>
    <a href="{{ '/ssh-fail2ban/' | relative_url }}">
      <span class="post-date">2026-10-01</span>
      <span class="post-title">SSH 暴力登入與 Fail2ban 防護</span>
      <span class="post-desc">從 CPU 尖峰追到 SSH 外部攻擊，並以 Fail2ban 自動封鎖。</span>
    </a>
  </li>
  <li>
    <a href="{{ '/cpu-watch/' | relative_url }}">
      <span class="post-date">2026-10-01</span>
      <span class="post-title">Linux CPU 使用率監控與異常程序追蹤指南</span>
      <span class="post-desc">CPU 升高時，自動找出是哪個 PHP 請求（網址）造成，並搭配 PHP-FPM access log 與 logrotate 長期運作。</span>
    </a>
  </li>
</ul>
