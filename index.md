---
layout: default
title: Home
---

<div class="hero">
  <h1>⚡ TeckMagix Docs</h1>
  <p>ICT1012 Operating Systems — lab solutions, exam predictions, and variation practice questions.</p>
</div>

## 📘 Lab 1 — Utilities & Syscalls

<div class="cards">
  <a href="/os_notes/lab1-w1-solutions/" class="card">
    <div class="card-icon">😴</div>
    <div class="card-title">Lab1-w1: sleep & memdump</div>
    <div class="card-desc">sleep via pause syscall and a flexible memory dump formatter.</div>
    <div class="card-footer">
      <span class="badge badge-blue">Week 1</span>
      <span class="badge badge-green">40 pts</span>
    </div>
  </a>
  <a href="/os_notes/lab1-w2-solutions/" class="card">
    <div class="card-icon">👋</div>
    <div class="card-title">Lab1-w2: hello, sixfive & xargs</div>
    <div class="card-desc">Custom kernel syscall, number filtering, and xargs implementation.</div>
    <div class="card-footer">
      <span class="badge badge-blue">Week 2</span>
      <span class="badge badge-green">55 pts</span>
    </div>
  </a>
  <a href="/os_notes/lab1-w3-solutions/" class="card">
    <div class="card-icon">🔌</div>
    <div class="card-title">Lab1-w3: handshake, sniffer & monitor</div>
    <div class="card-desc">Pipe IPC, memory exploit, and syscall tracing.</div>
    <div class="card-footer">
      <span class="badge badge-blue">Week 3</span>
      <span class="badge badge-yellow">65 pts</span>
    </div>
  </a>
</div>

## 📗 Lab 2 — Threads & Synchronisation

<div class="cards">
  <a href="/os_notes/lab2-w5-solutions/" class="card">
    <div class="card-icon">🧵</div>
    <div class="card-title">Lab2-w5: uthread & hash table</div>
    <div class="card-desc">User-level thread context switching in RISC-V assembly + pthreads mutex locking.</div>
    <div class="card-footer">
      <span class="badge badge-blue">Week 5</span>
      <span class="badge badge-yellow">60 pts</span>
    </div>
  </a>
</div>

## 📙 Lab 3 — File System

<div class="cards">
  <a href="/os_notes/lab3-w8-solutions/" class="card">
    <div class="card-icon">💾</div>
    <div class="card-title">Lab3-w8: bigfile & symlinks</div>
    <div class="card-desc">Doubly-indirect inode blocks for large files and symbolic link implementation.</div>
    <div class="card-footer">
      <span class="badge badge-blue">Week 8</span>
      <span class="badge badge-orange">100 pts</span>
    </div>
  </a>
</div>

---

## 🔮 Exam Predictions

Auto-generated exam prediction and variation questions from dropped lab files.

<div class="cards">
{% assign preds = site.predictions | sort: 'title' %}
{% if preds.size > 0 %}
{% for pred in preds %}
  <a href="{{ pred.url | relative_url }}" class="card">
    <div class="card-icon">🔮</div>
    <div class="card-title">{{ pred.title }}</div>
    <div class="card-desc">{{ pred.description | default: "Auto-generated exam predictions and variation questions." }}</div>
    <div class="card-footer">
      <span class="badge badge-purple">Prediction</span>
      {% if pred.lab %}<span class="badge badge-blue">{{ pred.lab | upcase }}</span>{% endif %}
    </div>
  </a>
{% endfor %}
{% else %}
  <div class="card" style="opacity:0.5; cursor:default;">
    <div class="card-icon">📂</div>
    <div class="card-title">No predictions yet</div>
    <div class="card-desc">Drop a lab file into your watch folder — predictions will appear here automatically.</div>
  </div>
{% endif %}
</div>
