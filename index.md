---
layout: home
title: Home
---

# Welcome to My Tech Blog

Latest posts about CTF challenges, security notes, technical projects, and more.

<div class="pac-container" aria-hidden="true">

  <div class="pac-row">
    <div class="pac-track" style="animation-delay:-0s">
      <div class="pac-man"><div class="pac-top"></div><div class="pac-bot"></div><div class="pac-eye"></div></div>
      <div class="pac-dots"><span></span><span></span><span></span><span></span><span></span><span></span><span></span><span></span><span></span><span></span></div>
    </div>
  </div>

  <div class="pac-row">
    <div class="pac-track" style="animation-delay:-3s">
      <div class="pac-man"><div class="pac-top"></div><div class="pac-bot"></div><div class="pac-eye"></div></div>
      <div class="pac-dots"><span></span><span></span><span></span><span></span><span></span><span></span><span></span><span></span><span></span><span></span></div>
    </div>
  </div>

  <div class="pac-row">
    <div class="pac-track" style="animation-delay:-6s">
      <div class="pac-man"><div class="pac-top"></div><div class="pac-bot"></div><div class="pac-eye"></div></div>
      <div class="pac-dots"><span></span><span></span><span></span><span></span><span></span><span></span><span></span><span></span><span></span><span></span></div>
    </div>
  </div>

  <div class="pac-row">
    <div class="pac-track" style="animation-delay:-7.5s">
      <div class="pac-man"><div class="pac-top"></div><div class="pac-bot"></div><div class="pac-eye"></div></div>
      <div class="pac-dots"><span></span><span></span><span></span><span></span><span></span><span></span><span></span><span></span><span></span><span></span></div>
    </div>
  </div>

</div>

<style>
/* ── Container: 4 rows ─────────────────────── */
.pac-container {
  width: 100%;
  margin: 0.5rem 0 2rem;
}
.pac-row {
  position: relative;
  height: 46px;
  width: 100%;
  overflow: hidden;
  margin: 3px 0;
}
/* Dashed path line each row runs along */
.pac-row::before {
  content: '';
  position: absolute;
  top: 50%; left: 0; right: 0;
  height: 2px;
  background: repeating-linear-gradient(
    90deg,
    #7aa2f7 0px, #7aa2f7 6px,
    transparent 6px, transparent 14px
  );
  opacity: 0.25;
  transform: translateY(-50%);
}
[data-theme="dark"] .pac-row::before {
  background: repeating-linear-gradient(
    90deg,
    #bb9af7 0px, #bb9af7 6px,
    transparent 6px, transparent 14px
  );
  opacity: 0.2;
}

/* ── Moving track ──────────────────────────── */
.pac-track {
  display: flex;
  align-items: center;
  position: absolute;
  top: 50%;
  transform: translateY(-50%) translateX(-80px);
  animation: pac-march 9s linear infinite;
  white-space: nowrap;
}
@keyframes pac-march {
  0%   { transform: translateY(-50%) translateX(-80px); }
  100% { transform: translateY(-50%) translateX(110vw); }
}

/* ── Pacman body ───────────────────────────── */
.pac-man {
  position: relative;
  width: 34px;
  height: 34px;
  flex-shrink: 0;
  margin-right: 10px;
}
.pac-top, .pac-bot {
  position: absolute;
  width: 0; height: 0;
  left: 0;
  border-left:   17px solid #f7c948;
  border-top:    17px solid #f7c948;
  border-right:  17px solid transparent;
  border-bottom: 17px solid transparent;
}
.pac-top {
  top: 0;
  border-radius: 17px 17px 0 0;
  transform-origin: bottom right;
  animation: chomp-top 0.28s linear infinite alternate;
}
.pac-bot {
  top: 17px;
  border-top-color: transparent;
  border-bottom: 17px solid #f7c948;
  border-radius: 0 0 17px 17px;
  transform-origin: top right;
  animation: chomp-bot 0.28s linear infinite alternate;
}
.pac-eye {
  position: absolute;
  width: 4px; height: 4px;
  background: #0d1117;
  border-radius: 50%;
  top: 6px; left: 8px;
  z-index: 1;
}
@keyframes chomp-top {
  0%   { transform: rotate(-38deg); }
  100% { transform: rotate(0deg); }
}
@keyframes chomp-bot {
  0%   { transform: rotate(38deg); }
  100% { transform: rotate(0deg); }
}

/* ── Dots ──────────────────────────────────── */
.pac-dots {
  display: flex;
  align-items: center;
  gap: 20px;
}
.pac-dots span {
  display: inline-block;
  width: 8px; height: 8px;
  background: #7aa2f7;
  border-radius: 50%;
  flex-shrink: 0;
}
[data-theme="dark"] .pac-dots span { background: #bb9af7; }
[data-theme="dark"] .pac-eye { background: #1a1b2e; }
</style>

## 💬 Have a Question or Topic Request?

Want me to write about something specific? Have questions about cybersecurity, CTF challenges, or Splunk?

**[Ask a Question / Request a Topic →](https://github.com/C4tzm3/C4tzm3.github.io/discussions/new?category=q-a)**

I'll create writeups and guides based on community requests!

---

## Recent Posts
