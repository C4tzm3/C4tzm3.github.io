---
layout: home
title: Home
---

<div class="matrix-wrap">
  <canvas id="matrix-canvas"></canvas>
  <div class="matrix-hero">
    <h1 class="matrix-title">Welcome to My Tech Blog</h1>
    <p class="matrix-sub">Latest posts about CTF challenges, security notes, technical projects, and more.</p>
  </div>
  <div class="matrix-hint">// hover or click to interact</div>
</div>

<script>
(function() {
  var canvas = document.getElementById('matrix-canvas');
  var wrap   = canvas.parentElement;
  var ctx    = canvas.getContext('2d');
  var chars  = '01 アイウエカキクケサシスセタチツテナニヌネ ABCDEF 0123456789 <>{}[]#@!'.split('');
  var fontSize = 13;
  var cols, drops, bright;
  var mouseCol = -1;

  function resize() {
    canvas.width  = wrap.offsetWidth;
    canvas.height = 200;
    cols  = Math.floor(canvas.width / fontSize);
    drops = [];
    bright = [];
    for (var i = 0; i < cols; i++) {
      drops[i]  = Math.random() * -80;
      bright[i] = 0;
    }
  }
  resize();
  window.addEventListener('resize', resize);

  canvas.addEventListener('mousemove', function(e) {
    var r = canvas.getBoundingClientRect();
    mouseCol = Math.floor((e.clientX - r.left) / fontSize);
    for (var i = Math.max(0, mouseCol-3); i < Math.min(cols, mouseCol+4); i++) {
      bright[i] = Math.min(1, bright[i] + 0.3);
    }
  });

  canvas.addEventListener('mouseleave', function() { mouseCol = -1; });

  canvas.addEventListener('click', function(e) {
    var r = canvas.getBoundingClientRect();
    var c = Math.floor((e.clientX - r.left) / fontSize);
    for (var i = Math.max(0, c-6); i < Math.min(cols, c+7); i++) {
      drops[i]  = 0;
      bright[i] = 1;
    }
  });

  function draw() {
    var dark = document.documentElement.getAttribute('data-theme') === 'dark';
    ctx.fillStyle = dark ? 'rgba(26,27,46,0.12)' : 'rgba(240,240,248,0.18)';
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.font = fontSize + 'px "JetBrains Mono",monospace';

    for (var i = 0; i < cols; i++) {
      var ch = chars[Math.floor(Math.random() * chars.length)];
      var x  = i * fontSize;
      var y  = drops[i] * fontSize;

      if (bright[i] > 0) {
        ctx.fillStyle = dark ? '#bb9af7' : '#7c3aed';
        bright[i] = Math.max(0, bright[i] - 0.02);
      } else {
        ctx.fillStyle = dark ? '#7aa2f7' : '#2959aa';
      }

      ctx.globalAlpha = 0.55 + Math.random() * 0.45;
      ctx.fillText(ch, x, y);
      ctx.globalAlpha = 1;

      if (y > canvas.height && Math.random() > 0.975) drops[i] = 0;
      drops[i] += 0.4;
    }
  }

  setInterval(draw, 45);
})();
</script>

<style>
.matrix-wrap {
  position: relative;
  width: 100%;
  border-radius: 10px;
  overflow: hidden;
  margin: 0.5rem 0 2rem;
  cursor: crosshair;
  border: 1px solid var(--border);
  background: var(--bg2);
}
#matrix-canvas { display: block; width: 100%; }
.matrix-hint {
  position: absolute;
  bottom: 8px; right: 12px;
  font-size: 0.68rem;
  font-family: 'JetBrains Mono', monospace;
  color: var(--text-muted);
  opacity: 0.6;
  pointer-events: none;
}
.matrix-hero {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  pointer-events: none;
  z-index: 2;
  background: rgba(26, 27, 46, 0.55);
  padding: 1.1rem 2.2rem;
  border-radius: 10px;
  backdrop-filter: blur(3px);
  border: 1px solid rgba(122, 162, 247, 0.25);
  width: max-content;
  max-width: 88%;
}
[data-theme="light"] .matrix-hero {
  background: rgba(240, 240, 248, 0.65);
  border-color: rgba(41, 89, 170, 0.2);
}
.matrix-title {
  font-size: 1.5rem;
  font-weight: 700;
  color: #c0caf5;
  margin: 0 0 0.35rem;
  letter-spacing: 0.01em;
  border: none;
  padding: 0;
}
[data-theme="light"] .matrix-title { color: #1e3a6e; }
.matrix-sub {
  font-size: 0.85rem;
  color: #a9b1d6;
  margin: 0;
  line-height: 1.5;
}
[data-theme="light"] .matrix-sub { color: #4a5568; }
</style>

## 💬 Have a Question or Topic Request?

Want me to write about something specific? Have questions about cybersecurity, CTF challenges, or Splunk?

**[Ask a Question / Request a Topic →](https://github.com/C4tzm3/C4tzm3.github.io/discussions/new?category=q-a)**

I'll create writeups and guides based on community requests!

---

## Recent Posts
