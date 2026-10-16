#!/data/data/com.termux/files/usr/bin/bash
# Leslie Gauthier's Quest: all-in-one install / serve / deploy for Termux
#   bash install_limbo.sh                  # install + start server
#   bash install_limbo.sh serve            # start server only
#   DEPLOY_REMOTE=https://github.com/USER/leslie-gauthiers-quest.git bash install_limbo.sh deploy

APP="$HOME/leslie-gauthiers-quest"
PORT=8080
MODE="${1:-install}"

say(){ printf '\n\033[1;36m==> %s\033[0m\n' "$1"; }

write_files(){
  mkdir -p "$APP"; cd "$APP" || exit 1

cat > index.html << 'HTML_EOF'
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no,viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-title" content="Leslie's Quest">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="theme-color" content="#000000">
<link rel="manifest" href="manifest.webmanifest">
<link rel="apple-touch-icon" href="icon-192.png">
<title>Leslie Gauthier's Quest</title>
<style>
html,body{margin:0;height:100%;background:#000;overflow:hidden;touch-action:none;user-select:none;-webkit-user-select:none;-webkit-touch-callout:none}
canvas{position:fixed;inset:0;width:100%;height:100%;display:block}
#title{position:fixed;top:12px;left:0;right:0;text-align:center;color:#8a8a8a;font:13px monospace;letter-spacing:2px;pointer-events:none}
#pad{position:fixed;left:0;right:0;bottom:0;padding:10px 14px calc(10px + env(safe-area-inset-bottom));display:flex;justify-content:space-between;align-items:flex-end;pointer-events:none}
.grp{display:flex;gap:12px}
.btn{pointer-events:auto;width:84px;height:84px;border-radius:50%;background:rgba(255,255,255,.07);border:1px solid rgba(255,255,255,.25);color:#ddd;font:600 22px system-ui,sans-serif;display:flex;align-items:center;justify-content:center;touch-action:none}
.btn.on{background:rgba(255,255,255,.25)}
#jump{width:104px;height:104px;font-size:16px;letter-spacing:2px}
</style>
</head>
<body>
<canvas id="c"></canvas>
<div id="pad">
  <div class="grp">
    <div class="btn" data-k="left">&#9664;</div>
    <div class="btn" data-k="right">&#9654;</div>
  </div>
  <div class="grp">
    <div class="btn" id="jump" data-k="jump">JUMP</div>
  </div>
</div>
<script>
const cv = document.getElementById('c'), ctx = cv.getContext('2d');
const VW = 640, VH = 480, WORLD_W = 2400, DEATH_Y = 640;
const G = 1800, SPEED = 180, JUMP = -520, MAX_FALL = 900;
let scale = 1;
let t = 0;

// ---- Level ----
const SOLIDS = [
  {x:-40,  y:400, w:560, h:200},
  {x:600,  y:400, w:200, h:200},
  {x:860,  y:340, w:130, h:18},
  {x:1080, y:400, w:300, h:200},
  {x:1420, y:340, w:200, h:18},
  {x:1700, y:400, w:700, h:200},
];
const SPIKES = [
  {x:640,  y:384, w:44, h:16},
  {x:1160, y:384, w:44, h:16},
  {x:1790, y:384, w:44, h:16},
];
const CPS = [
  {tx:620,  sp:{x:620,  y:370}},
  {tx:1100, sp:{x:1100, y:370}},
  {tx:1740, sp:{x:1740, y:370}},
];
const DOOR = {x:2300, y:340, w:40, h:60};

// ---- State ----
const P = {x:40, y:370, w:18, h:30, vx:0, vy:0, ground:false, face:1};
let spawn = {x:40, y:370};
let deaths = 0, dead = false, deadT = 0, won = false, wonT = 0;
let jumpBuf = 0, jumpPrev = false;

// ---- Input (keyboard + multi-touch buttons) ----
const KEYS = {ArrowLeft:'left', KeyA:'left', ArrowRight:'right', KeyD:'right', Space:'jump', ArrowUp:'jump', KeyW:'jump'};
const keyDown = {left:false, right:false, jump:false};
const touch = {left:new Set(), right:new Set(), jump:new Set()};
const held = k => keyDown[k] || touch[k].size > 0;

addEventListener('keydown', e => { const k = KEYS[e.code]; if (k) { keyDown[k] = true; e.preventDefault(); } });
addEventListener('keyup',   e => { const k = KEYS[e.code]; if (k) { keyDown[k] = false; e.preventDefault(); } });

document.querySelectorAll('.btn').forEach(b => {
  b.addEventListener('pointerdown', e => { e.preventDefault(); touch[b.dataset.k].add(e.pointerId); });
});
const release = e => { for (const k in touch) touch[k].delete(e.pointerId); };
addEventListener('pointerup', release);
addEventListener('pointercancel', release);

// ---- Physics helpers ----
function overlap(a, b) {
  return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y;
}
function respawn() { P.x = spawn.x; P.y = spawn.y; P.vx = 0; P.vy = 0; }
function die() { if (dead || won) return; dead = true; deadT = 0.9; deaths++; }

function update(dt) {
  t += dt;
  if (won) {
    wonT -= dt;
    if (wonT <= 0) { won = false; spawn = {x:40, y:370}; respawn(); }
    return;
  }
  if (dead) {
    deadT -= dt;
    if (deadT <= 0) { dead = false; respawn(); }
    return;
  }

  const l = held('left'), r = held('right'), j = held('jump');
  P.vx = ((r ? 1 : 0) - (l ? 1 : 0)) * SPEED;
  if (P.vx) P.face = Math.sign(P.vx);

  // Jump with a short buffer so taps register on touchscreens
  if (j && !jumpPrev) jumpBuf = 0.12;
  jumpPrev = j;
  jumpBuf -= dt;
  if (jumpBuf > 0 && P.ground) { P.vy = JUMP; jumpBuf = 0; P.ground = false; }

  P.vy = Math.min(P.vy + G * dt, MAX_FALL);

  // Horizontal move + collide
  P.x += P.vx * dt;
  for (const s of SOLIDS) if (overlap(P, s)) {
    if (P.vx > 0) P.x = s.x - P.w; else if (P.vx < 0) P.x = s.x + s.w;
  }

  // Vertical move + collide
  P.y += P.vy * dt;
  P.ground = false;
  for (const s of SOLIDS) if (overlap(P, s)) {
    if (P.vy > 0) { P.y = s.y - P.h; P.ground = true; }
    else if (P.vy < 0) { P.y = s.y + s.h; }
    P.vy = 0;
  }

  // Hazards
  if (P.y > DEATH_Y) die();
  for (const s of SPIKES) if (overlap(P, s)) die();
  if (dead) return;

  // Checkpoints
  for (const c of CPS) if (P.x >= c.tx && c.sp.x > spawn.x) spawn = c.sp;

  // Goal
  if (overlap(P, DOOR)) { won = true; wonT = 2.5; }
}

// ---- Rendering ----
function drawBg(camX) {
  const g = ctx.createLinearGradient(0, 0, 0, VH);
  g.addColorStop(0, '#1c1c1e'); g.addColorStop(0.7, '#080808'); g.addColorStop(1, '#000');
  ctx.fillStyle = g; ctx.fillRect(0, 0, VW, VH);

  // Dead-tree silhouettes (parallax)
  ctx.fillStyle = '#070707';
  for (let i = 0; i < 10; i++) {
    const x = (((i * 190 - camX * 0.35) % 1900) + 1900) % 1900 - 120;
    const h = 140 + (i % 3) * 50;
    ctx.fillRect(x, 400 - h, 12, h);
    ctx.fillRect(x - 30, 400 - h * 0.7, 40, 8);
  }

  // Drifting fog
  ctx.fillStyle = 'rgba(200,200,200,0.04)';
  for (let i = 0; i < 4; i++) {
    const fx = (((t * 12 * (i + 1) + i * 300 - camX * 0.2) % 1000) + 1000) % 1000 - 180;
    ctx.beginPath(); ctx.ellipse(fx, 330 + i * 18, 260, 40, 0, 0, Math.PI * 2); ctx.fill();
  }
}

function drawWorld() {
  for (const s of SOLIDS) {
    ctx.fillStyle = '#111'; ctx.fillRect(s.x, s.y, s.w, s.h);
    ctx.fillStyle = '#262626'; ctx.fillRect(s.x, s.y, s.w, 2);
  }

  ctx.fillStyle = '#3a3a3a';
  for (const s of SPIKES) {
    ctx.beginPath();
    ctx.moveTo(s.x, s.y + s.h); ctx.lineTo(s.x + s.w / 2, s.y); ctx.lineTo(s.x + s.w, s.y + s.h);
    ctx.fill();
  }

  for (const c of CPS) if (c.sp.x <= spawn.x) {
    const g = ctx.createRadialGradient(c.sp.x + 9, c.sp.y + 15, 2, c.sp.x + 9, c.sp.y + 15, 40);
    g.addColorStop(0, 'rgba(255,240,200,0.35)'); g.addColorStop(1, 'rgba(255,240,200,0)');
    ctx.fillStyle = g; ctx.fillRect(c.sp.x - 40, c.sp.y - 25, 100, 100);
  }

  const d = DOOR;
  const dg = ctx.createRadialGradient(d.x + 20, d.y + 30, 4, d.x + 20, d.y + 30, 70);
  dg.addColorStop(0, 'rgba(255,248,220,0.35)'); dg.addColorStop(1, 'rgba(255,248,220,0)');
  ctx.fillStyle = dg; ctx.fillRect(d.x - 60, d.y - 50, 160, 160);
  ctx.fillStyle = '#e8e2c8'; ctx.fillRect(d.x, d.y, d.w, d.h);

  if (!dead) {
    ctx.fillStyle = '#000'; ctx.fillRect(P.x, P.y, P.w, P.h);
    ctx.strokeStyle = 'rgba(255,255,255,0.18)'; ctx.lineWidth = 1;
    ctx.strokeRect(P.x + 0.5, P.y + 0.5, P.w - 1, P.h - 1);
    const ex = P.face > 0 ? 11 : 4;
    ctx.fillStyle = '#f2f2f2';
    ctx.fillRect(P.x + ex, P.y + 9, 2, 2);
    ctx.fillRect(P.x + ex + 3, P.y + 9, 2, 2);
  }
}

function draw() {
  ctx.setTransform(1, 0, 0, 1, 0, 0);
  ctx.fillStyle = '#000'; ctx.fillRect(0, 0, cv.width, cv.height);

  const ox = (cv.width - VW * scale) / 2, oy = (cv.height - VH * scale) / 2;
  ctx.setTransform(scale, 0, 0, scale, ox, oy);

  const camX = Math.max(0, Math.min(WORLD_W - VW, P.x + P.w / 2 - VW / 2));
  drawBg(camX);
  ctx.save(); ctx.translate(-camX, 0); drawWorld(); ctx.restore();

  const v = ctx.createRadialGradient(VW/2, VH/2, VH*0.35, VW/2, VH/2, VW*0.65);
  v.addColorStop(0, 'rgba(0,0,0,0)'); v.addColorStop(1, 'rgba(0,0,0,0.85)');
  ctx.fillStyle = v; ctx.fillRect(0, 0, VW, VH);

  if (dead) { ctx.fillStyle = 'rgba(0,0,0,0.5)'; ctx.fillRect(0, 0, VW, VH); }

  ctx.textBaseline = 'alphabetic';
  ctx.fillStyle = '#8a8a8a'; ctx.font = '14px monospace'; ctx.textAlign = 'left';
  ctx.fillText('DEATHS ' + deaths, 14, 24);
  if (t < 7) {
    ctx.fillStyle = 'rgba(180,180,180,' + Math.min(1, 7 - t).toFixed(2) + ')';
    ctx.textAlign = 'center';
    ctx.fillText("LESLIE GAUTHIER'S QUEST", VW / 2, 48);
  }
  if (won) {
    ctx.fillStyle = '#e8e2c8'; ctx.font = '28px serif'; ctx.textAlign = 'center';
    ctx.fillText('ESCAPED', VW / 2, VH / 2);
  }
}

// ---- Boot ----
function resize() {
  const dpr = window.devicePixelRatio || 1;
  cv.width = innerWidth * dpr; cv.height = innerHeight * dpr;
  scale = Math.min(innerWidth / VW, innerHeight / VH) * dpr;
}
addEventListener('resize', resize);
resize();

let last = performance.now();
function frame(now) {
  const dt = Math.min(0.033, (now - last) / 1000);
  last = now;
  update(dt);
  draw();
  document.querySelectorAll('.btn').forEach(b => b.classList.toggle('on', held(b.dataset.k)));
  requestAnimationFrame(frame);
}
respawn();
requestAnimationFrame(frame);

if ('serviceWorker' in navigator) navigator.serviceWorker.register('sw.js').catch(() => {});
</script>
</body>
</html>
HTML_EOF

cat > manifest.webmanifest << 'JSON_EOF'
{
  "name": "Leslie Gauthier's Quest",
  "short_name": "Leslie's Quest",
  "start_url": "./",
  "scope": "./",
  "display": "fullscreen",
  "orientation": "landscape",
  "background_color": "#000000",
  "theme_color": "#000000",
  "icons": [
    {"src": "icon-192.png", "sizes": "192x192", "type": "image/png"},
    {"src": "icon-512.png", "sizes": "512x512", "type": "image/png"}
  ]
}
JSON_EOF

cat > sw.js << 'JS_EOF'
const CACHE = 'leslie-quest-v1';
const FILES = ['./', 'index.html', 'manifest.webmanifest', 'icon-192.png', 'icon-512.png'];
self.addEventListener('install', e =>
  e.waitUntil(caches.open(CACHE).then(c => c.addAll(FILES))));
self.addEventListener('fetch', e =>
  e.respondWith(caches.match(e.request).then(r => r || fetch(e.request))));
JS_EOF

cat > make_icons.py << 'PY_EOF'
import zlib, struct

def chunk(tag, data):
    return struct.pack('>I', len(data)) + tag + data + struct.pack('>I', zlib.crc32(tag + data) & 0xffffffff)

def make(size, path):
    rows = []
    r = size * 0.36
    for y in range(size):
        row = bytearray([0])
        for x in range(size):
            dx, dy = x - size / 2, y - size / 2
            if dx * dx + dy * dy < r * r:
                c = (232, 232, 232)
                if abs(dx) < size * 0.07 and abs(dy) < size * 0.18:
                    c = (10, 10, 10)
            else:
                c = (10, 10, 10)
            row += bytes(c)
        rows.append(bytes(row))
    data = zlib.compress(b''.join(rows), 9)
    with open(path, 'wb') as f:
        f.write(b'\x89PNG\r\n\x1a\n')
        f.write(chunk(b'IHDR', struct.pack('>IIBBBBB', size, size, 8, 2, 0, 0, 0)))
        f.write(chunk(b'IDAT', data))
        f.write(chunk(b'IEND', b''))

make(192, 'icon-192.png')
make(512, 'icon-512.png')
PY_EOF

  python3 make_icons.py
}

serve(){
  [ -d "$APP" ] || { echo "Not installed. Run: bash install_limbo.sh"; exit 1; }
  cd "$APP" || exit 1
  command -v termux-wake-lock >/dev/null 2>&1 && termux-wake-lock
  IP=$(ip -4 addr show wlan0 2>/dev/null | awk '/inet /{print $2}' | cut -d/ -f1)
  echo ""
  echo "Leslie Gauthier's Quest"
  echo "On this phone:   http://localhost:$PORT"
  [ -n "$IP" ] && echo "Other devices:   http://$IP:$PORT  (same Wi-Fi)"
  echo "Add to home screen from Chrome (Android) or Safari (iPhone) for app mode."
  echo "Stop with Ctrl+C"
  python3 -m http.server "$PORT" --bind 0.0.0.0
}

deploy(){
  [ -d "$APP" ] || { echo "Not installed. Run install first."; exit 1; }
  if [ -z "$DEPLOY_REMOTE" ]; then
    echo "Set your repo URL first, e.g.:"
    echo "  DEPLOY_REMOTE=https://github.com/USER/leslie-gauthiers-quest.git bash install_limbo.sh deploy"
    exit 1
  fi
  pkg install -y git openssh >/dev/null 2>&1
  cd "$APP" || exit 1

  case "$DEPLOY_REMOTE" in
    git@*|ssh://*)
      if [ ! -f "$HOME/.ssh/id_ed25519" ]; then
        say "Creating SSH key. Add this public key to GitHub (Settings > SSH keys):"
        mkdir -p "$HOME/.ssh"
        ssh-keygen -t ed25519 -N "" -f "$HOME/.ssh/id_ed25519" -q
        cat "$HOME/.ssh/id_ed25519.pub"
        read -r -p "Press Enter after adding the key..." _
      fi
      ;;
  esac

  [ -d .git ] || git init -q -b main
  git config user.name  >/dev/null 2>&1 || git config user.name  "Leslie Dev"
  git config user.email >/dev/null 2>&1 || git config user.email "dev@example.com"
  git add -A
  git commit -qm "Deploy Leslie Gauthier's Quest $(date +%F_%T)" || echo "No changes to commit."
  git remote remove origin >/dev/null 2>&1
  git remote add origin "$DEPLOY_REMOTE"

  say "Pushing to $DEPLOY_REMOTE"
  if git push -u origin main; then
    echo ""
    echo "Pushed. To publish on GitHub Pages:"
    echo "  Repo > Settings > Pages > Deploy from branch: main / (root)"
    echo "Your game will be at: https://USER.github.io/REPO/"
    echo "Open that link on Android or iPhone and use 'Add to Home Screen'."
  else
    echo "Push failed. For HTTPS use a GitHub personal access token as the password."
  fi
}

case "$MODE" in
  install)
    say "Updating Termux packages"
    pkg update -y && pkg upgrade -y
    say "Installing python, git, openssh, iproute2"
    pkg install -y python git openssh iproute2 || true
    say "Writing Leslie Gauthier's Quest to $APP"
    write_files
    say "Installed. Starting server."
    serve
    ;;
  serve)  serve ;;
  deploy) deploy ;;
  *) echo "Usage: bash install_limbo.sh [install|serve|deploy]"; exit 1 ;;
esac
