---
layout: default
title: 維運筆記
---

# 維運筆記

<p class="byline">實際維運案例的排查過程與處理紀錄</p>

<ul class="post-list">
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
