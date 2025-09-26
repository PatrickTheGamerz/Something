<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Two-Player Tower Platformer — Menu, Settings, Create, Bot</title>
  <style>
    :root {
      --bg: #0e0f14;
      --panel: #171923;
      --panel-2: #10121a;
      --accent: #4ab3ed;
      --accent-2: #e4b423;
      --text: #e9edf3;
      --muted: #94a3b8;
      --danger: #b43a3a;
      --ok: #27c39f;
    }
    html, body {
      margin: 0; height: 100%; background: var(--bg); color: var(--text); font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Arial;
      overflow: hidden;
    }
    canvas { display: block; margin: 0 auto; background: #141725; }
    /* UI */
    .ui-layer { position: fixed; inset: 0; pointer-events: none; display: grid; place-items: center; }
    .panel {
      pointer-events: auto;
      background: linear-gradient(180deg, var(--panel), var(--panel-2));
      border: 1px solid #1f2433; border-radius: 12px; box-shadow: 0 20px 60px rgba(0,0,0,0.45);
      padding: 20px; width: 720px; max-width: calc(100% - 40px);
    }
    .panel h1 { margin: 0 0 8px 0; font-size: 28px; letter-spacing: 0.4px; }
    .panel p { margin: 0 0 16px 0; color: var(--muted); }
    .menu-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; margin-top: 12px; }
    .btn {
      background: #222637; border: 1px solid #2a3147; color: var(--text);
      border-radius: 10px; padding: 14px 16px; font-weight: 600; letter-spacing: 0.3px; cursor: pointer;
      transition: transform 0.06s ease, background 0.2s ease;
    }
    .btn:hover { transform: translateY(-2px); background: #262c42; }
    .btn.primary { background: linear-gradient(180deg, #2a91c7, #1f78aa); border-color: #2a91c7; }
    .btn.warn { background: linear-gradient(180deg, #b7561f, #9d430f); border-color: #b7561f; }
    .btn.danger { background: linear-gradient(180deg, #c63e3e, #a93131); border-color: #c63e3e; }
    .row { display:flex; gap:12px; align-items:center; margin: 10px 0 4px 0; }
    .segmented { display:flex; gap:8px; flex-wrap: wrap; }
    .segmented .seg {
      background:#222637; border:1px solid #2a3147; color:var(--text); padding:10px 12px; border-radius:8px; cursor:pointer;
    }
    .seg.active { background:#2a3147; border-color:#3b4767; }
    .help { margin-top: 10px; font-size: 13px; color: var(--muted); border-top: 1px dashed #2a3147; padding-top: 10px; }
    /* Secondary panel for play options */
    .sub-panel { margin-top: 14px; padding-top: 14px; border-top: 1px solid #252a3a; }

    /* Editor toolbar */
    .toolbar {
      position: fixed; left: 16px; top: 16px; pointer-events:auto;
      display:flex; gap:8px; background:#141725cc; border:1px solid #242a3c; border-radius:10px; padding:8px;
      backdrop-filter: blur(4px); z-index: 10;
    }
    .tool { background:#222637; border:1px solid #2a3147; color:#d7deea; padding:8px 12px; border-radius:8px; cursor:pointer; }
    .tool.active { background:#2a3147; border-color:#3b4767; }
    .toolbar .label { color:#9fb2cc; padding:8px 12px; }

    /* Mobile controls */
    .mobile-controls { position: fixed; inset: 0; pointer-events: none; }
    .mc-btn {
      position: absolute; width: 84px; height: 84px; border-radius: 50%;
      background: radial-gradient(circle at 30% 30%, #2a3147, #181b26 60%);
      border: 1px solid #3b4767; box-shadow: inset 0 6px 12px rgba(0,0,0,0.4), 0 2px 8px rgba(0,0,0,0.4);
      pointer-events: auto; touch-action: none; user-select: none;
    }
    .mc-btn:active { filter: brightness(1.2); }
    .mc-small { width: 64px; height: 64px; font-size: 12px; }
    .mc-label {
      position:absolute; left:50%; top:50%; transform:translate(-50%,-50%);
      color:#cfe2ff; font-weight:700; text-shadow: 0 1px 2px rgba(0,0,0,0.8);
    }

    /* Selection highlight in editor */
    .hint { position: fixed; right: 16px; top: 16px; background:#141725cc; color:#cfe2ff; border:1px solid #242a3c; padding:8px 12px; border-radius:10px; backdrop-filter: blur(4px); pointer-events:none; }
  </style>
</head>
<body>
<canvas id="gameCanvas" width="1024" height="480"></canvas>

<!-- Main menu -->
<div class="ui-layer" id="menuLayer">
  <div class="panel">
    <h1>Two-Player Tower</h1>
    <p>Sprint, jump, and climb. Build your own levels. Play your way.</p>
    <div class="menu-grid">
      <button class="btn primary" id="btnPlay">PLAY</button>
      <button class="btn" id="btnSettings">SETTINGS</button>
      <button class="btn warn" id="btnCreate">CREATE</button>
    </div>

    <!-- Settings -->
    <div id="settingsBlock" style="margin-top:16px; display:none;">
      <div class="row">
        <div style="min-width:120px;">Device</div>
        <div class="segmented" id="deviceSeg">
          <div class="seg" data-device="pc">PC</div>
          <div class="seg" data-device="mobile">Mobile</div>
        </div>
      </div>
      <div class="row">
        <div style="min-width:120px;">Control overlay</div>
        <div class="segmented" id="overlaySeg">
          <div class="seg" data-overlay="off">Off</div>
          <div class="seg" data-overlay="on">On</div>
        </div>
      </div>
      <div class="help">Tip: Mobile overlay shows on-screen controls for both players.</div>
    </div>

    <!-- Play options -->
    <div id="playOptions" class="sub-panel" style="display:none;">
      <div class="row">
        <div style="min-width:140px;">Mode</div>
        <div class="segmented" id="playModeSeg">
          <div class="seg active" data-playmode="multiplayer">Multiplayer</div>
          <div class="seg" data-playmode="bot">With Bot</div>
        </div>
      </div>
      <div class="row">
        <div style="min-width:140px;">Tower</div>
        <div class="segmented" id="towerSeg">
          <div class="seg active" data-tower="default">Default</div>
          <div class="seg" data-tower="custom">Custom (last saved)</div>
        </div>
      </div>
      <div class="row">
        <button class="btn primary" id="btnStartPlay">Start</button>
        <button class="btn" id="btnBackMenu">Back</button>
      </div>
      <div class="help">Multiplayer uses split-screen when players are far apart. Bot mode adds a simple AI companion as Player 2.</div>
    </div>
  </div>
</div>

<!-- Editor toolbar -->
<div class="toolbar" id="editorToolbar" style="display:none;">
  <div class="label">Tool:</div>
  <button class="tool active" data-tool="platform">Platform</button>
  <button class="tool" data-tool="hazard">Lava</button>
  <button class="tool" data-tool="collectible">Gem</button>
  <button class="tool" data-tool="checkpoint">Checkpoint</button>
  <div class="label">Grid:</div>
  <button class="tool" id="gridSnapBtn">Snap: 20</button>
  <button class="tool" id="clearBtn">Clear</button>
  <button class="tool" id="saveBtn">Save</button>
  <button class="tool" id="loadBtn">Load</button>
  <button class="tool" id="playtestBtn">Playtest</button>
</div>
<div class="hint" id="editorHint" style="display:none;">Click to place. Drag to move. Floor is permanent.</div>

<!-- Mobile controls: left cluster = Player 1 (WASD), right cluster = Player 2 (Arrows) -->
<div class="mobile-controls" id="mobileControls" style="display:none;">
  <!-- Player 1 left side -->
  <div class="mc-btn" id="mcP1Left" style="left: 26px; bottom: 28px;"><div class="mc-label">A</div></div>
  <div class="mc-btn" id="mcP1Right" style="left: 126px; bottom: 28px;"><div class="mc-label">D</div></div>
  <div class="mc-btn" id="mcP1Jump" style="left: 76px; bottom: 118px;"><div class="mc-label">W</div></div>
  <div class="mc-btn mc-small" id="mcP1Down" style="left: 76px; bottom: 18px;"><div class="mc-label">S</div></div>

  <!-- Player 2 right side -->
  <div class="mc-btn" id="mcP2Left" style="right: 126px; bottom: 28px;"><div class="mc-label">◀</div></div>
  <div class="mc-btn" id="mcP2Right" style="right: 26px; bottom: 28px;"><div class="mc-label">▶</div></div>
  <div class="mc-btn" id="mcP2Jump" style="right: 76px; bottom: 118px;"><div class="mc-label">⤒</div></div>
  <div class="mc-btn mc-small" id="mcP2Down" style="right: 76px; bottom: 18px;"><div class="mc-label">▾</div></div>
</div>

<script>
// ==== CONFIGURATION ====
const CANVAS_WIDTH = 1024;
const CANVAS_HEIGHT = 480;
const VIEW_WIDTH = CANVAS_WIDTH / 2;
const VIEW_HEIGHT = CANVAS_HEIGHT;
const WORLD_WIDTH = 1600;
const WORLD_HEIGHT = 2400;
const SPLIT_DISTANCE = 480;
const MERGE_DISTANCE = 360;

// ==== GAME STATE ====
let mode = "menu"; // "menu" | "play" | "create" | "playtest"
let deviceType = "pc"; // "pc" | "mobile"
let overlayEnabled = false; // on-screen controls
let currentSplitMode = "merged";
let gridSnap = 20;

let playMode = "multiplayer"; // "multiplayer" | "bot"
let towerSelection = "default"; // "default" | "custom"

// ==== INPUT HANDLING ====
const keys = {};
window.addEventListener("keydown", e => keys[e.code] = true);
window.addEventListener("keyup", e => keys[e.code] = false);

// Utility segmented
function segmentedInit(segEl, value, attr, onChange) {
  [...segEl.children].forEach(ch => {
    ch.classList.toggle("active", ch.getAttribute(attr) === value);
    ch.addEventListener("click", () => {
      [...segEl.children].forEach(n => n.classList.remove("active"));
      ch.classList.add("active");
      const v = ch.getAttribute(attr);
      onChange(v);
    });
  });
}

// ==== UI HOOKUP ====
const menuLayer = document.getElementById("menuLayer");
const btnPlay = document.getElementById("btnPlay");
const btnSettings = document.getElementById("btnSettings");
const btnCreate = document.getElementById("btnCreate");
const settingsBlock = document.getElementById("settingsBlock");
const deviceSeg = document.getElementById("deviceSeg");
const overlaySeg = document.getElementById("overlaySeg");

const playOptions = document.getElementById("playOptions");
const playModeSeg = document.getElementById("playModeSeg");
const towerSeg = document.getElementById("towerSeg");
const btnStartPlay = document.getElementById("btnStartPlay");
const btnBackMenu = document.getElementById("btnBackMenu");

btnPlay.addEventListener("click", () => {
  playOptions.style.display = "block";
  settingsBlock.style.display = "none";
});
btnBackMenu.addEventListener("click", () => {
  playOptions.style.display = "none";
});
btnStartPlay.addEventListener("click", () => {
  // Load tower selection
  loadActiveLevel(towerSelection);
  // Start appropriate mode
  mode = "play";
  menuLayer.style.display = "none";
  editorToolbar.style.display = "none";
  syncMobileControls();
  // Reset players to start
  resetPlayersToStart();
});

btnSettings.addEventListener("click", () => {
  settingsBlock.style.display = settingsBlock.style.display === "none" ? "block" : "none";
  playOptions.style.display = "none";
});
btnCreate.addEventListener("click", () => {
  mode = "create";
  menuLayer.style.display = "none";
  editorToolbar.style.display = "flex";
  document.getElementById("editorHint").style.display = "block";
  syncMobileControls();
  // Ensure floor exists
  ensureFloor();
});

segmentedInit(deviceSeg, deviceType, "data-device", v => { deviceType = v; syncMobileControls(); });
segmentedInit(overlaySeg, overlayEnabled ? "on" : "off", "data-overlay", v => { overlayEnabled = (v === "on"); syncMobileControls(); });
segmentedInit(playModeSeg, playMode, "data-playmode", v => { playMode = v; });
segmentedInit(towerSeg, towerSelection, "data-tower", v => { towerSelection = v; });

// ==== MOBILE CONTROLS ====
const mcLayer = document.getElementById("mobileControls");
function bindTouch(btn, codesDown, codesUp) {
  const down = (e) => { e.preventDefault(); codesDown.forEach(code => keys[code] = true); };
  const up = (e) => { e.preventDefault(); codesUp.forEach(code => keys[code] = false); };
  btn.addEventListener("touchstart", down); btn.addEventListener("mousedown", down);
  window.addEventListener("touchend", up); window.addEventListener("mouseup", up);
}
// Player 1 cluster
bindTouch(document.getElementById("mcP1Left"), ["KeyA"], ["KeyA"]);
bindTouch(document.getElementById("mcP1Right"), ["KeyD"], ["KeyD"]);
bindTouch(document.getElementById("mcP1Jump"), ["KeyW"], ["KeyW"]);
bindTouch(document.getElementById("mcP1Down"), ["KeyS"], ["KeyS"]);
// Player 2 cluster
bindTouch(document.getElementById("mcP2Left"), ["ArrowLeft"], ["ArrowLeft"]);
bindTouch(document.getElementById("mcP2Right"), ["ArrowRight"], ["ArrowRight"]);
bindTouch(document.getElementById("mcP2Jump"), ["ArrowUp"], ["ArrowUp"]);
bindTouch(document.getElementById("mcP2Down"), ["ArrowDown"], ["ArrowDown"]);

function syncMobileControls() {
  const show = (deviceType === "mobile" && overlayEnabled && mode !== "menu");
  mcLayer.style.display = show ? "block" : "none";
}

// ==== UTILITY ====
function clamp(val, min, max) { return Math.max(min, Math.min(val, max)); }
function rectsCollide(a, b) {
  return (a.x < b.x + b.width && a.x + a.width > b.x && a.y < b.y + b.height && a.y + a.height > b.y);
}
function snap(v, s) { return s ? Math.round(v / s) * s : v; }

// ==== PLAYER CLASS ====
class Player {
  constructor(x, y, color, controls) {
    this.x = x; this.y = y; this.width = 36; this.height = 48;
    this.color = color; this.vx = 0; this.vy = 0;
    this.speed = 5.4; this.jumpVelocity = -14.2; this.gravity = 0.8; this.maxFall = 18;
    this.onGround = false; this.controls = controls;
    this.checkpoint = { x, y }; this.collected = 0; this.dead = false;
    this.isBot = false;
  }
  get rect() { return { x: this.x, y: this.y, width: this.width, height: this.height }; }
  respawn() { this.x = this.checkpoint.x; this.y = this.checkpoint.y; this.vx = 0; this.vy = 0; this.dead = false; }
  update(platforms, hazards, collectibles, checkpoints) {
    // Bot AI: simple follow Player 1 horizontally, jump occasionally if stuck
    if (this.isBot) {
      const targetX = player1.x;
      const dx = targetX - this.x;
      const dir = Math.sign(dx);
      this.vx = dir * this.speed * 0.9; // a bit slower
      // Jump if near an edge or not moving horizontally for a while
      if (this.onGround && Math.abs(dx) > 20) {
        // Simple heuristic: random small chance or if hazard in front
        const frontRect = { x: this.x + dir * (this.width + 4), y: this.y + this.height, width: 6, height: 6 };
        const nearHazard = hazards.some(h=>rectsCollide(frontRect, h));
        if (nearHazard || Math.random() < 0.02) {
          this.vy = this.jumpVelocity; this.onGround = false;
        }
      }
    } else {
      let move = 0;
      if (keys[this.controls.left]) move -= 1;
      if (keys[this.controls.right]) move += 1;
      this.vx = move * this.speed;
      if (this.onGround && keys[this.controls.jump]) {
        this.vy = this.jumpVelocity; this.onGround = false;
      }
    }

    this.vy += this.gravity;
    if (this.vy > this.maxFall) this.vy = this.maxFall;

    // Horizontal
    this.x += this.vx;
    for (const plat of platforms) {
      if (rectsCollide(this.rect, plat)) {
        if (this.vx > 0) this.x = plat.x - this.width;
        else if (this.vx < 0) this.x = plat.x + plat.width;
      }
    }

    // Vertical
    this.y += this.vy;
    this.onGround = false;
    for (const plat of platforms) {
      if (rectsCollide(this.rect, plat)) {
        if (this.vy > 0) { this.y = plat.y - this.height; this.vy = 0; this.onGround = true; }
        else if (this.vy < 0) { this.y = plat.y + plat.height; this.vy = 0; }
      }
    }

    // World bounds
    if (this.y > WORLD_HEIGHT) this.dead = true;

    // Hazards
    for (const hazard of hazards) if (rectsCollide(this.rect, hazard)) this.dead = true;

    // Collectibles
    for (let i = collectibles.length - 1; i >= 0; i--) {
      if (rectsCollide(this.rect, collectibles[i])) {
        collectibles.splice(i, 1); this.collected += 1;
      }
    }

    // Checkpoints
    for (const cp of checkpoints) if (rectsCollide(this.rect, cp)) this.checkpoint = { x: cp.x, y: cp.y };
  }
  draw(ctx) {
    ctx.save();
    // Body (no shadow)
    const grd = ctx.createLinearGradient(this.x, this.y, this.x, this.y + this.height);
    grd.addColorStop(0, this.color); grd.addColorStop(1, "#0c0f14");
    ctx.fillStyle = grd; ctx.strokeStyle = "#0a0d12"; ctx.lineWidth = 2;
    ctx.fillRect(this.x, this.y, this.width, this.height);
    ctx.strokeRect(this.x+0.5, this.y+0.5, this.width-1, this.height-1);
    // Eyes
    ctx.fillStyle = "#fff";
    ctx.fillRect(this.x + 7, this.y + 12, 8, 8);
    ctx.fillRect(this.x + 21, this.y + 12, 8, 8);
    ctx.fillStyle = "#364a63";
    ctx.fillRect(this.x + 10, this.y + 16, 3, 3);
    ctx.fillRect(this.x + 24, this.y + 16, 3, 3);
    ctx.restore();
  }
}

// ==== DEFAULT WORLD ====
const defaultPlatforms = [
  { x: 0, y: 2350, width: 1600, height: 50 }, // floor
  { x: 200, y: 2200, width: 200, height: 20 },
  { x: 600, y: 2100, width: 180, height: 20 },
  { x: 1000, y: 2000, width: 220, height: 20 },
  { x: 400, y: 1900, width: 160, height: 20 },
  { x: 800, y: 1800, width: 180, height: 20 },
  { x: 1200, y: 1700, width: 200, height: 20 },
  { x: 600, y: 1600, width: 180, height: 20 },
  { x: 300, y: 1500, width: 160, height: 20 },
  { x: 900, y: 1400, width: 200, height: 20 },
];
const defaultHazards = [
  { x: 500, y: 2335, width: 40, height: 15 },
  { x: 850, y: 1985, width: 40, height: 15 },
  { x: 700, y: 1585, width: 40, height: 15 },
];
const defaultCollectibles = [
  { x: 220, y: 2170, width: 20, height: 20 },
  { x: 620, y: 2070, width: 20, height: 20 },
  { x: 1020, y: 1970, width: 20, height: 20 },
  { x: 820, y: 1770, width: 20, height: 20 },
  { x: 1220, y: 1670, width: 20, height: 20 },
];
const defaultCheckpoints = [
  { x: 620, y: 2070, width: 24, height: 28 },
  { x: 820, y: 1770, width: 24, height: 28 },
  { x: 1220, y: 1670, width: 24, height: 28 },
];

// Persistent floor (cannot be removed)
const persistentFloor = { x: 0, y: 2350, width: 1600, height: 50 };

// Active level data (changes in editor/play/playtest)
let platforms = JSON.parse(JSON.stringify(defaultPlatforms));
let hazards = JSON.parse(JSON.stringify(defaultHazards));
let collectibles = JSON.parse(JSON.stringify(defaultCollectibles));
let checkpoints = JSON.parse(JSON.stringify(defaultCheckpoints));

function ensureFloor() {
  const hasFloor = platforms.some(p => p.x === persistentFloor.x && p.y === persistentFloor.y && p.width === persistentFloor.width && p.height === persistentFloor.height);
  if (!hasFloor) platforms.unshift(JSON.parse(JSON.stringify(persistentFloor)));
}

function loadActiveLevel(selection) {
  if (selection === "custom") {
    const data = localStorage.getItem("towerLevel");
    if (data) {
      const parsed = JSON.parse(data);
      platforms = parsed.platforms && parsed.platforms.length ? parsed.platforms : [JSON.parse(JSON.stringify(persistentFloor))];
      hazards = parsed.hazards || [];
      collectibles = parsed.collectibles || [];
      checkpoints = parsed.checkpoints || [];
    } else {
      platforms = JSON.parse(JSON.stringify(defaultPlatforms));
      hazards = JSON.parse(JSON.stringify(defaultHazards));
      collectibles = JSON.parse(JSON.stringify(defaultCollectibles));
      checkpoints = JSON.parse(JSON.stringify(defaultCheckpoints));
    }
  } else {
    platforms = JSON.parse(JSON.stringify(defaultPlatforms));
    hazards = JSON.parse(JSON.stringify(defaultHazards));
    collectibles = JSON.parse(JSON.stringify(defaultCollectibles));
    checkpoints = JSON.parse(JSON.stringify(defaultCheckpoints));
  }
  ensureFloor();
}

// ==== DRAW FUNCTIONS ====
function drawPlatform(ctx, p) {
  ctx.save();
  const g = ctx.createLinearGradient(p.x, p.y, p.x, p.y + p.height);
  g.addColorStop(0, "#6d7b91"); g.addColorStop(1, "#393f52");
  ctx.fillStyle = g;
  ctx.fillRect(p.x, p.y, p.width, p.height);
  ctx.strokeStyle = "#2a3147";
  ctx.strokeRect(p.x, p.y, p.width, p.height);
  ctx.restore();
}
function drawHazard(ctx, h) {
  ctx.save();
  const g = ctx.createLinearGradient(h.x, h.y, h.x, h.y + h.height);
  g.addColorStop(0, "#d44b4b"); g.addColorStop(1, "#8b2a2a");
  ctx.fillStyle = g;
  ctx.fillRect(h.x, h.y, h.width, h.height);
  ctx.strokeStyle = "#ffe5e5";
  ctx.beginPath();
  ctx.moveTo(h.x, h.y + h.height);
  ctx.lineTo(h.x + h.width / 2, h.y);
  ctx.lineTo(h.x + h.width, h.y + h.height);
  ctx.stroke();
  ctx.restore();
}
function drawCollectible(ctx, c) {
  ctx.save();
  ctx.beginPath();
  ctx.arc(c.x + c.width / 2, c.y + c.height / 2, 9, 0, 2 * Math.PI);
  const g = ctx.createRadialGradient(c.x + c.width / 2, c.y + c.height / 2, 3, c.x + c.width / 2, c.y + c.height / 2, 9);
  g.addColorStop(0, "#ffe57a"); g.addColorStop(1, "#caa541");
  ctx.fillStyle = g; ctx.fill();
  ctx.strokeStyle = "#f8d98f"; ctx.lineWidth = 2; ctx.stroke();
  ctx.restore();
}
function drawCheckpoint(ctx, cp) {
  ctx.save();
  const g = ctx.createLinearGradient(cp.x, cp.y, cp.x, cp.y + cp.height);
  g.addColorStop(0, "#5acef7"); g.addColorStop(1, "#2a88bf");
  ctx.fillStyle = g; ctx.fillRect(cp.x, cp.y, cp.width, cp.height);
  ctx.fillStyle = "#fff"; ctx.font = "bold 11px sans-serif"; ctx.fillText("✔", cp.x + 5, cp.y + 18);
  ctx.restore();
}
function drawWorld(ctx) {
  for (const plat of platforms) drawPlatform(ctx, plat);
  for (const hazard of hazards) drawHazard(ctx, hazard);
  for (const c of collectibles) drawCollectible(ctx, c);
  for (const cp of checkpoints) drawCheckpoint(ctx, cp);
}

// ==== CAMERA LOGIC ====
function getSplitMode(p1, p2) {
  const dx = (p1.x + p1.width / 2) - (p2.x + p2.width / 2);
  const dy = (p1.y + p1.height / 2) - (p2.y + p2.height / 2);
  const dist = Math.hypot(dx, dy);
  if (dist > SPLIT_DISTANCE) return "split";
  else if (dist < MERGE_DISTANCE) return "merged";
  else return currentSplitMode;
}
function calcCamera(px, py) {
  let camX = Math.round(px - VIEW_WIDTH / 2);
  let camY = Math.round(py - VIEW_HEIGHT / 2);
  camX = clamp(camX, 0, WORLD_WIDTH - VIEW_WIDTH);
  camY = clamp(camY, 0, WORLD_HEIGHT - VIEW_HEIGHT);
  return [camX, camY];
}
function calcMergedCamera(p1, p2) {
  let mx = (p1.x + p1.width / 2 + p2.x + p2.width / 2) / 2;
  let my = (p1.y + p1.height / 2 + p2.y + p2.height / 2) / 2;
  return calcCamera(mx, my);
}

// ==== PLAYERS ====
const player1 = new Player(100, 2200, "#23b4e4", { left: "KeyA", right: "KeyD", jump: "KeyW" });
const player2 = new Player(160, 2200, "#e4b423", { left: "ArrowLeft", right: "ArrowRight", jump: "ArrowUp" });

function resetPlayersToStart() {
  player1.x = 100; player1.y = 2200; player1.vx = 0; player1.vy = 0; player1.dead = false; player1.onGround = false; player1.collected = 0;
  player2.x = 160; player2.y = 2200; player2.vx = 0; player2.vy = 0; player2.dead = false; player2.onGround = false; player2.collected = 0;
  player1.checkpoint = { x: player1.x, y: player1.y };
  player2.checkpoint = { x: player2.x, y: player2.y };
  player2.isBot = (playMode === "bot");
}

// ==== HUD ====
const canvas = document.getElementById("gameCanvas");
const ctx = canvas.getContext("2d");
function drawHud(player, x, y) {
  ctx.save();
  ctx.globalAlpha = 0.88; ctx.fillStyle = "#111"; ctx.fillRect(x - 10, y - 28, 160, 44);
  ctx.globalAlpha = 1.0; ctx.fillStyle = "#fff"; ctx.font = "16px monospace";
  ctx.fillText("Collected: " + player.collected, x, y);
  ctx.fillText("Checkpoint: " + Math.round(player.checkpoint.x) + "," + Math.round(player.checkpoint.y), x, y + 22);
  ctx.restore();
}

// ==== EDITOR STATE + INPUT (place & move) ====
let currentTool = "platform";
const editorToolbar = document.getElementById("editorToolbar");
const toolButtons = editorToolbar.querySelectorAll(".tool[data-tool]");
toolButtons.forEach(btn => {
  btn.addEventListener("click", () => {
    toolButtons.forEach(b => b.classList.remove("active"));
    btn.classList.add("active");
    currentTool = btn.dataset.tool;
  });
});

document.getElementById("gridSnapBtn").addEventListener("click", () => {
  gridSnap = (gridSnap === 20 ? 40 : gridSnap === 40 ? 0 : 20);
  document.getElementById("gridSnapBtn").textContent = "Snap: " + (gridSnap || "off");
});
document.getElementById("clearBtn").addEventListener("click", () => {
  // Clear but keep floor
  platforms = [JSON.parse(JSON.stringify(persistentFloor))];
  hazards = []; collectibles = []; checkpoints = [];
});
document.getElementById("saveBtn").addEventListener("click", () => {
  ensureFloor();
  localStorage.setItem("towerLevel", JSON.stringify({ platforms, hazards, collectibles, checkpoints }));
  // Also allow selecting custom tower in play menu
  [...towerSeg.children].forEach(n => n.classList.remove("active"));
  towerSeg.querySelector('[data-tower="custom"]').classList.add("active");
  towerSelection = "custom";
});
document.getElementById("loadBtn").addEventListener("click", () => {
  const data = localStorage.getItem("towerLevel");
  if (data) {
    const parsed = JSON.parse(data);
    platforms = parsed.platforms || [JSON.parse(JSON.stringify(persistentFloor))];
    hazards = parsed.hazards || [];
    collectibles = parsed.collectibles || [];
    checkpoints = parsed.checkpoints || [];
    ensureFloor();
  }
});
document.getElementById("playtestBtn").addEventListener("click", () => {
  mode = "playtest";
  editorToolbar.style.display = "none";
  document.getElementById("editorHint").style.display = "none";
  ensureFloor();
  syncMobileControls();
  // Single player only in playtest
  player2.dead = true; // effectively disable second player
  player2.isBot = false;
  player1.x = 100; player1.y = 2200; player1.vx = 0; player1.vy = 0; player1.dead = false; player1.onGround = false; player1.collected = 0;
});

// Editor placement & movement
let dragState = {
  dragging: false,
  targetArray: null,
  targetIndex: -1,
  offsetX: 0,
  offsetY: 0
};
function pickObjectAt(worldX, worldY) {
  // Try hazards, collectibles, checkpoints, platforms (but skip floor)
  const arrays = [
    { arr: hazards, type: "hazard" },
    { arr: collectibles, type: "collectible" },
    { arr: checkpoints, type: "checkpoint" },
    { arr: platforms, type: "platform" }
  ];
  for (const entry of arrays) {
    for (let i = entry.arr.length - 1; i >= 0; i--) {
      const o = entry.arr[i];
      if (entry.type === "platform" && o.x === persistentFloor.x && o.y === persistentFloor.y && o.width === persistentFloor.width && o.height === persistentFloor.height) {
        continue; // skip floor
      }
      if (worldX >= o.x && worldX <= o.x + o.width && worldY >= o.y && worldY <= o.y + o.height) {
        return { arr: entry.arr, index: i, obj: o };
      }
    }
  }
  return null;
}
canvas.addEventListener("mousedown", e => {
  if (mode !== "create") return;
  const rect = canvas.getBoundingClientRect();
  let wx = e.clientX - rect.left;
  let wy = e.clientY - rect.top;

  const picked = pickObjectAt(wx, wy);
  if (picked) {
    dragState.dragging = true;
    dragState.targetArray = picked.arr;
    dragState.targetIndex = picked.index;
    dragState.offsetX = wx - picked.obj.x;
    dragState.offsetY = wy - picked.obj.y;
  } else {
    // Place new
    wx = snap(wx, gridSnap); wy = snap(wy, gridSnap);
    if (currentTool === "platform") platforms.push({ x: wx, y: wy, width: 100, height: 20 });
    if (currentTool === "hazard") hazards.push({ x: wx, y: wy, width: 40, height: 15 });
    if (currentTool === "collectible") collectibles.push({ x: wx, y: wy, width: 20, height: 20 });
    if (currentTool === "checkpoint") checkpoints.push({ x: wx, y: wy, width: 24, height: 28 });
  }
});
canvas.addEventListener("mousemove", e => {
  if (mode !== "create" || !dragState.dragging) return;
  const rect = canvas.getBoundingClientRect();
  let wx = e.clientX - rect.left - dragState.offsetX;
  let wy = e.clientY - rect.top - dragState.offsetY;
  wx = snap(wx, gridSnap); wy = snap(wy, gridSnap);
  const obj = dragState.targetArray[dragState.targetIndex];
  obj.x = wx; obj.y = wy;
});
window.addEventListener("mouseup", () => {
  dragState.dragging = false;
});

// ==== HUD (slim) for play modes ====
function drawPlayHUD() {
  drawHud(player1, 28, 36);
  if (mode === "play" && playMode !== "bot") {
    // show both players
    drawHud(player2, currentSplitMode === "split" ? VIEW_WIDTH + 28 : 280, 36);
  }
}

// ==== GAME LOOP ====
function gameLoop() {
  ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

  if (mode === "play" || mode === "playtest") {
    // Active players
    const p2Active = (mode === "play" && (playMode === "multiplayer" || playMode === "bot"));

    if (player1.dead) player1.respawn();
    if (p2Active && player2.dead) player2.respawn();

    player1.update(platforms, hazards, collectibles, checkpoints);
    if (p2Active) player2.update(platforms, hazards, collectibles, checkpoints);

    currentSplitMode = p2Active ? getSplitMode(player1, player2) : "merged";

    if (p2Active && currentSplitMode === "split") {
      // Left view (P1)
      ctx.save();
      ctx.beginPath(); ctx.rect(0, 0, VIEW_WIDTH, VIEW_HEIGHT); ctx.clip();
      let [cx1, cy1] = calcCamera(player1.x + player1.width/2, player1.y + player1.height/2);
      ctx.translate(-cx1, -cy1);
      drawWorld(ctx); player1.draw(ctx); player2.draw(ctx);
      ctx.restore();

      // Divider
      ctx.strokeStyle = "#222"; ctx.lineWidth = 4;
      ctx.beginPath(); ctx.moveTo(VIEW_WIDTH, 0); ctx.lineTo(VIEW_WIDTH, VIEW_HEIGHT); ctx.stroke();

      // Right view (P2)
      ctx.save();
      ctx.beginPath(); ctx.rect(VIEW_WIDTH, 0, VIEW_WIDTH, VIEW_HEIGHT); ctx.clip();
      let [cx2, cy2] = calcCamera(player2.x + player2.width/2, player2.y + player2.height/2);
      ctx.translate(-cx2 + VIEW_WIDTH, -cy2);
      drawWorld(ctx); player2.draw(ctx); player1.draw(ctx);
      ctx.restore();
    } else {
      // Single view
      ctx.save();
      let [cx, cy] = p2Active ? calcMergedCamera(player1, player2)
                              : calcCamera(player1.x + player1.width/2, player1.y + player1.height/2);
      ctx.translate(-cx, -cy);
      drawWorld(ctx);
      player1.draw(ctx);
      if (p2Active) player2.draw(ctx);
      ctx.restore();
    }

    drawPlayHUD();
  }

  if (mode === "create") {
    ensureFloor();
    drawWorld(ctx);
  }

  requestAnimationFrame(gameLoop);
}
gameLoop();
</script>
</body>
</html>
```
