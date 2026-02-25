---
layout: home
title: Home
---

# Welcome to My Tech Blog

Latest posts about CTF challenges, security notes, technical projects, and more.

<div class="pac-container" aria-hidden="true">
  <div class="pac-track">
    <div class="pac-man">
      <div class="pac-top"></div>
      <div class="pac-bot"></div>
      <div class="pac-eye"></div>
    </div>
    <div class="pac-dots">
      <span></span><span></span><span></span><span></span>
      <span></span><span></span><span></span><span></span>
      <span></span><span></span><span></span><span></span>
    </div>
  </div>
</div>

<style>
.pac-container {
  width: 100%;
  overflow: hidden;
  height: 44px;
  margin: 0.25rem 0 1.5rem;
  position: relative;
}
.pac-track {
  display: flex;
  align-items: center;
  position: absolute;
  left: 0;
  animation: pac-march 5s linear infinite;
  white-space: nowrap;
}
@keyframes pac-march {
  0%   { transform: translateX(-80px); }
  100% { transform: translateX(110vw); }
}
.pac-man {
  position: relative;
  width: 36px;
  height: 36px;
  flex-shrink: 0;
  margin-right: 10px;
}
.pac-top, .pac-bot {
  position: absolute;
  width: 0; height: 0;
  left: 0;
  border-left:   18px solid #f7c948;
  border-top:    18px solid #f7c948;
  border-right:  18px solid transparent;
  border-bottom: 18px solid transparent;
}
.pac-top {
  top: 0;
  border-radius: 18px 18px 0 0;
  transform-origin: bottom right;
  animation: chomp-top 0.22s linear infinite alternate;
}
.pac-bot {
  top: 18px;
  border-top-color: transparent;
  border-bottom: 18px solid #f7c948;
  border-radius: 0 0 18px 18px;
  transform-origin: top right;
  animation: chomp-bot 0.22s linear infinite alternate;
}
.pac-eye {
  position: absolute;
  width: 5px; height: 5px;
  background: #1a1b2e;
  border-radius: 50%;
  top: 7px; left: 9px;
  z-index: 1;
}
@keyframes chomp-top {
  0%   { transform: rotate(-35deg); }
  100% { transform: rotate(0deg); }
}
@keyframes chomp-bot {
  0%   { transform: rotate(35deg); }
  100% { transform: rotate(0deg); }
}
.pac-dots {
  display: flex;
  align-items: center;
  gap: 18px;
}
.pac-dots span {
  display: inline-block;
  width: 9px; height: 9px;
  background: #7aa2f7;
  border-radius: 50%;
  flex-shrink: 0;
  animation: dot-pulse 0.44s linear infinite alternate;
}
.pac-dots span:nth-child(2)  { animation-delay: 0.04s; }
.pac-dots span:nth-child(3)  { animation-delay: 0.08s; }
.pac-dots span:nth-child(4)  { animation-delay: 0.12s; }
.pac-dots span:nth-child(5)  { animation-delay: 0.16s; }
.pac-dots span:nth-child(6)  { animation-delay: 0.20s; }
.pac-dots span:nth-child(7)  { animation-delay: 0.24s; }
.pac-dots span:nth-child(8)  { animation-delay: 0.28s; }
.pac-dots span:nth-child(9)  { animation-delay: 0.32s; }
.pac-dots span:nth-child(10) { animation-delay: 0.36s; }
.pac-dots span:nth-child(11) { animation-delay: 0.40s; }
.pac-dots span:nth-child(12) { animation-delay: 0.44s; }
@keyframes dot-pulse {
  0%   { opacity: 1;   transform: scale(1); }
  100% { opacity: 0.3; transform: scale(0.7); }
}
[data-theme="dark"] .pac-eye { background: #1a1b2e; }
[data-theme="dark"] .pac-dots span { background: #bb9af7; }
</style>

## 💬 Have a Question or Topic Request?

Want me to write about something specific? Have questions about cybersecurity, CTF challenges, or Splunk?

**[Ask a Question / Request a Topic →](https://github.com/C4tzm3/C4tzm3.github.io/discussions/new?category=q-a)**

I'll create writeups and guides based on community requests!

---

## Recent Posts
