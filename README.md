<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Tower Platformer — Whole Updated Code</title>
  <style>
    :root {
      --bg: #0e0f14;
      --canvas: #141725;
      --panel: #1b1f2e;
      --border: #2a3147;
      --text: #e9edf3;
      --muted: #94a3b8;
      --mc-size: 84px;
      --panel-width: 860px;
      --panel-scale: 1;
      --ui-shift-x: 0px;
    }
    html, body { margin:0; height:100%; background:var(--bg); color:var(--text); font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Arial; overflow:hidden; }
    canvas { display:block; margin:0 auto; background:var(--canvas); transform-origin:center top; }
    .ui-layer { position:fixed; inset:0; display:grid; place-items:center; pointer-events:none; }
    .panel {
      pointer-events:auto; background:var(--panel); border:1px solid var(--border); border-radius:12px; padding:20px;
      width:var(--panel-width); max-width:calc(100% - 40px); max-height:80vh; overflow-y:auto;
      transform:scale(var(--panel-scale)); transform-origin:left top;
      position:relative; left:var(--ui-shift-x);
    }
    .btn { background:#222637; border:1px solid var(--border); color:var(--text); border-radius:10px; padding:12px 14px; font-weight:600; cursor:pointer; margin:4px; }
    .btn:hover { background:#2a3147; }
    .btn.primary { background:linear-gradient(180deg,#2a91c7,#1f78aa); border-color:#2a91c7; }
    .btn.warn { background:linear-gradient(180deg,#b7561f,#9d430f); border-color:#b7561f; }
    .btn.danger { background:linear-gradient(180deg,#c63e3e,#a93131); border-color:#c63e3e; }
    .segmented { display:flex; gap:8px; flex-wrap:wrap; }
    .seg { background:#222637; border:1px solid var(--border); color:var(--text); padding:10px 12px; border-radius:8px; cursor:pointer; }
    .seg.active { background:#2a3147; border-color:#3b4767; }
    .row { display:flex; gap:12px; align-items:center; margin:12px 0; flex-wrap:wrap; }

    /* Editor toolbar */
    .toolbar {
      position:fixed; left:16px; top:16px; display:flex; gap:8px; background:#141725cc; border:1px solid #242a3c;
      border-radius:10px; padding:8px; z-index:10; backdrop-filter: blur(4px);
      transform:scale(var(--panel-scale));
      transform-origin:left top;
      left:calc(16px + var(--ui-shift-x));
    }
    .tool { background:#222637; border:1px solid var(--border); color:#d7deea; padding:8px 12px; border-radius:8px; cursor:pointer; }
    .tool.active { background:#2a3147; border-color:#3b4767; }
    .hint {
      position:fixed; right:16px; top:16px; background:#141725cc; color:#cfe2ff; border:1px solid #242a3c;
      padding:8px 12px; border-radius:10px; backdrop-filter: blur(4px); pointer-events:none; z-index:10;
    }

    /* Mobile controls */
    .mobile-controls { position:fixed; inset:0; pointer-events:none; }
    .mc-btn {
      position:absolute; width:var(--mc-size); height:var(--mc-size); border-radius:50%;
      background: radial-gradient(circle at 30% 30%, #2a3147, #181b26 60%);
      border:1px solid #3b4767; box-shadow: inset 0 6px 12px rgba(0,0,0,0.4), 0 2px 8px rgba(0,0,0,0.4);
      pointer-events:auto; display:flex; align-items:center; justify-content:center; color:#cfe2ff; font-weight:700; font-size:24px;
      user-select:none;
      transform:scale(var(--panel-scale));
      transform-origin:left bottom;
      left:calc(var(--ui-shift-x) + 0px);
    }
    .mc-btn:active { filter:brightness(1.2); }

    .banner {
      position:fixed; top:16px; left:50%; transform:translateX(-50%);
      background:#112; color:#ffd700; padding:10px 16px; border:1px solid #334; border-radius:10px; font: 16px monospace;
      display:none; z-index:20;
    }
  </style>
</head>
<body>
<canvas id="gameCanvas" width="1024" height="480"></canvas>
<div class="banner" id="finishBanner"></div>

<!-- Menu -->
<div class="ui-layer" id="menuLayer">
  <div class="panel">
    <h1>Tower Platformer</h1>
    <div class="row">
      <button class="btn primary" id="btnPlay">PLAY</button>
      <button class="btn warn" id="btnCreate">CREATE</button>
      <button class="btn" id="btnSettings">SETTINGS</button>
    </div>

    <!-- Settings -->
    <div id="settingsBlock" style="display:none;">
      <div class="row">
        <div style="min-width:140px;">Device</div>
        <div class="segmented" id="deviceSeg">
          <div class="seg active" data-device="pc">PC</div>
          <div class="seg" data-device="mobile">Mobile</div>
        </div>
      </div>
      <div class="row" id="overlayRow">
        <div style="min-width:140px;">Control overlay</div>
        <div class="segmented" id="overlaySeg">
          <div class="seg active" data-overlay="off">Off</div>
          <div class="seg" data-overlay="on">On</div>
        </div>
      </div>
      <div class="row">
        <div style="min-width:140px;">Smaller screen</div>
        <div class="segmented" id="screenSeg">
          <div class="seg active" data-screen="normal">Normal</div>
          <div class="seg" data-screen="small">Smaller</div>
        </div>
      </div>
      <div class="help">On PC, control overlay is disabled. On Mobile, Smaller screen reduces UI and expands visible range.</div>
    </div>

    <!-- Play options -->
    <div id="playOptions" style="display:none;">
      <div class="row">
        <div style="min-width:140px;">Mode</div>
        <div class="segmented" id="playModeSeg">
          <div class="seg active" data-playmode="multiplayer">Multiplayer</div>
          <div class="seg" data-playmode="bot">Bot</div>
        </div>
      </div>
      <div class="row" id="botDiffRow" style="display:none;">
        <div style="min-width:140px;">Bot difficulty</div>
        <div class="segmented" id="botDiffSeg">
          <div class="seg active" data-diff="normal">Normal</div>
          <div class="seg" data-diff="tryhard">Tryhard</div>
          <div class="seg" data-diff="master">Tower Master</div>
        </div>
      </div>
      <div class="row">
        <button class="btn primary" id="btnStartPlay">Start</button>
        <button class="btn" id="btnBackMenu">Back</button>
      </div>
    </div>
  </div>
</div>

<!-- Editor toolbar -->
<div class="toolbar" id="editorToolbar" style="display:none;">
  <div class="tool active" data-tool="platform">Platform</div>
  <div class="tool" data-tool="lava">Lava</div>
  <div class="tool" id="rotateBtn">Rotate</div>
  <div class="tool" id="playtestBtn">Playtest</div>
  <div class="tool" id="clearBtn">Clear</div>
  <div class="tool danger" id="exitCreateBtn">Exit</div>
</div>
<div class="hint" id="editorHint" style="display:none;">Create: click to place, drag to move. WASD to pan. Spawn and Finish pad are movable.</div>

<!-- Mobile controls -->
<div class="mobile-controls" id="mobileControls" style="display:none;">
  <!-- Player 1 -->
  <div class="mc-btn" id="mcP1Left" style="left: 26px; bottom: 28px;">◀</div>
  <div class="mc-btn" id="mcP1Right" style="left: 126px; bottom: 28px;">▶</div>
  <div class="mc-btn" id="mcP1Jump" style="left: 76px; bottom: 118px;">⤒</div>
  <div class="mc-btn" id="mcP1Down" style="left: 76px; bottom: 200px; display:none;">▼</div>
  <!-- Player 2 (hidden when bot or in create) -->
  <div class="mc-btn" id="mcP2Left" style="right: 126px; bottom: 28px;">◀</div>
  <div class="mc-btn" id="mcP2Right" style="right: 26px; bottom: 28px;">▶</div>
  <div class="mc-btn" id="mcP2Jump" style="right: 76px; bottom: 118px;">⤒</div>
</div>

<script>
/* ==== CONFIG ==== */
const CANVAS_WIDTH = 1024;
const CANVAS_HEIGHT = 480;
const VIEW_WIDTH = CANVAS_WIDTH / 2;
const VIEW_HEIGHT = CANVAS_HEIGHT;
const WORLD_WIDTH = 1600; // world logical width for create mode panning
const WORLD_HEIGHT = 2400;

let renderScale = 1; // reduced on mobile small for more range view

/* ==== STATE ==== */
let mode = "menu"; // "menu" | "play" | "create" | "playtest"
let deviceType = "pc"; // "pc" | "mobile"
let overlayEnabled = false;
let screenMode = "normal"; // "normal" | "small"
let playMode = "multiplayer"; // "multiplayer" | "bot"
let botDifficulty = "normal"; // "normal" | "tryhard" | "master"
let currentSplitMode = "merged";
let gridSnap = 20;
let rotateMode = false;

let startTimeP1 = null;
let startTimeP2 = null;
let finished = false;
let finishBanner = document.getElementById("finishBanner");

/* ==== INPUT ==== */
const keys = {};
window.addEventListener("keydown", e => keys[e.code] = true);
window.addEventListener("keyup", e => keys[e.code] = false);

/* ==== UI HOOKUP ==== */
const menuLayer = document.getElementById("menuLayer");
const btnPlay = document.getElementById("btnPlay");
const btnSettings = document.getElementById("btnSettings");
const btnCreate = document.getElementById("btnCreate");
const settingsBlock = document.getElementById("settingsBlock");
const playOptions = document.getElementById("playOptions");
const btnStartPlay = document.getElementById("btnStartPlay");
const btnBackMenu = document.getElementById("btnBackMenu");

function segmentedInit(segEl, value, attr, onChange) {
  [...segEl.children].forEach(ch => {
    ch.classList.toggle("active", ch.getAttribute(attr) === value);
    ch.addEventListener("click", () => {
      [...segEl.children].forEach(n => n.classList.remove("active"));
      ch.classList.add("active");
      onChange(ch.getAttribute(attr));
      syncMobileControls();
    });
  });
}
const deviceSeg = document.getElementById("deviceSeg");
const overlaySeg = document.getElementById("overlaySeg");
const overlayRow = document.getElementById("overlayRow");
const screenSeg = document.getElementById("screenSeg");
const playModeSeg = document.getElementById("playModeSeg");
const botDiffSeg = document.getElementById("botDiffSeg");
const botDiffRow = document.getElementById("botDiffRow");

segmentedInit(deviceSeg, deviceType, "data-device", v => {
  deviceType = v;
  // PC disables control overlay, but smaller screen remains available
  if (deviceType === "pc") {
    overlayEnabled = false;
    [...overlaySeg.children].forEach(n => n.classList.remove("active"));
    overlaySeg.querySelector('[data-overlay="off"]').classList.add("active");
    overlayRow.style.display = "none";
    applyScreenScale(); // PC small screen only scales canvas
  } else {
    overlayRow.style.display = "flex";
    applyScreenScale(); // Mobile small screen scales UI and visible range
  }
});
segmentedInit(overlaySeg, overlayEnabled ? "on" : "off", "data-overlay", v => overlayEnabled = (v === "on"));
segmentedInit(screenSeg, screenMode, "data-screen", v => { screenMode = v; applyScreenScale(); });
segmentedInit(playModeSeg, playMode, "data-playmode", v => {
  playMode = v;
  botDiffRow.style.display = (playMode === "bot") ? "flex" : "none";
});
segmentedInit(botDiffSeg, botDifficulty, "data-diff", v => botDifficulty = v);

btnPlay.addEventListener("click", () => {
  playOptions.style.display = "block";
  settingsBlock.style.display = "none";
});
btnBackMenu.addEventListener("click", () => {
  playOptions.style.display = "none";
});
btnSettings.addEventListener("click", () => {
  settingsBlock.style.display = settingsBlock.style.display === "none" ? "block" : "none";
  playOptions.style.display = "none";
});
btnStartPlay.addEventListener("click", () => {
  setDefaultLevel();
  mode = "play";
  menuLayer.style.display = "none";
  editorToolbar.style.display = "none";
  editorHint.style.display = "none";
  resetPlayersToSpawn();
  finished = false;
  startTimeP1 = Date.now();
  startTimeP2 = playMode === "bot" ? Date.now() : (playMode === "multiplayer" ? Date.now() : null);
  syncMobileControls();
});
btnCreate.addEventListener("click", () => {
  startCreateFresh();
});

/* ==== MOBILE CONTROLS ==== */
const mcLayer = document.getElementById("mobileControls");
const mcP1Left = document.getElementById("mcP1Left");
const mcP1Right = document.getElementById("mcP1Right");
const mcP1Jump = document.getElementById("mcP1Jump");
const mcP1Down = document.getElementById("mcP1Down");
const mcP2Left = document.getElementById("mcP2Left");
const mcP2Right = document.getElementById("mcP2Right");
const mcP2Jump = document.getElementById("mcP2Jump");

function bindPress(btn, codes) {
  const down = e => { e.preventDefault(); codes.forEach(code => keys[code] = true); };
  const up = e => { e.preventDefault(); codes.forEach(code => keys[code] = false); };
  btn.addEventListener("touchstart", down, { passive:false });
  btn.addEventListener("touchend", up, { passive:false });
  btn.addEventListener("mousedown", down);
  window.addEventListener("mouseup", up);
}
bindPress(mcP1Left, ["KeyA"]);
bindPress(mcP1Right, ["KeyD"]);
bindPress(mcP1Jump, ["KeyW"]);
bindPress(mcP1Down, ["KeyS"]);
bindPress(mcP2Left, ["ArrowLeft"]);
bindPress(mcP2Right, ["ArrowRight"]);
bindPress(mcP2Jump, ["ArrowUp"]);

function syncMobileControls() {
  const showOverlay = (deviceType === "mobile" && overlayEnabled && mode !== "menu");
  mcLayer.style.display = showOverlay ? "block" : "none";

  // Hide P2 controls if bot or create mode
  const hideP2 = (mode === "create" || (mode === "play" && playMode === "bot") || mode === "playtest");
  mcP2Left.style.display = hideP2 ? "none" : "block";
  mcP2Right.style.display = hideP2 ? "none" : "block";
  mcP2Jump.style.display = hideP2 ? "none" : "block";

  // Down button only in create mode
  mcP1Down.style.display = (mode === "create") ? "block" : "none";
}

function applyScreenScale() {
  const root = document.documentElement;
  if (deviceType === "mobile" && screenMode === "small") {
    renderScale = 0.85; // draw scaled down for more range
    root.style.setProperty("--mc-size", "64px");
    root.style.setProperty("--panel-width", "720px");
    root.style.setProperty("--panel-scale", "0.9");
    root.style.setProperty("--ui-shift-x", "20px");
  } else if (screenMode === "small") {
    renderScale = 1.0; // PC: only UI/canvas size changes
    root.style.setProperty("--mc-size", "72px");
    root.style.setProperty("--panel-width", "780px");
    root.style.setProperty("--panel-scale", "0.95");
    root.style.setProperty("--ui-shift-x", "0px");
  } else {
    renderScale = 1.0;
    root.style.setProperty("--mc-size", "84px");
    root.style.setProperty("--panel-width", "860px");
    root.style.setProperty("--panel-scale", "1");
    root.style.setProperty("--ui-shift-x", "0px");
  }
}

/* ==== UTILS ==== */
function clamp(val, min, max) { return Math.max(min, Math.min(val, max)); }
function rectsCollide(a, b) {
  return (a.x < b.x + b.width && a.x + a.width > b.x && a.y < b.y + b.height && a.y + a.height > b.y);
}
function snap(v, s) { return s ? Math.round(v / s) * s : v; }
function timeFmt(ms) {
  const total = Math.floor(ms / 1000);
  const h = Math.floor(total / 3600);
  const m = Math.floor((total % 3600) / 60);
  const s = total % 60;
  const pad = n => String(n).padStart(2, "0");
  return `${pad(h)}:${pad(m)}:${pad(s)}`;
}

/* ==== PLAYER ==== */
class Player {
  constructor(x, y, color, controls) {
    this.x = x; this.y = y; this.width = 36; this.height = 48;
    this.color = color; this.controls = controls;
    this.vx = 0; this.vy = 0; this.onGround = false; this.dead = false;
    this.isBot = false;
    this.lastSafe = { x, y };
  }
  get rect() { return { x: this.x, y: this.y, width: this.width, height: this.height }; }
  update(platforms, lavas) {
    if (this.isBot) this.botLogic(platforms, lavas);
    else {
      let move = 0;
      if (keys[this.controls.left]) move -= 1;
      if (keys[this.controls.right]) move += 1;
      this.vx = move * 5.2;
      if (this.onGround && keys[this.controls.jump]) { this.vy = -14.2; this.onGround = false; }
      if (mode === "create" && keys["KeyS"]) { /* down key exists only for pan, no crouch */ }
    }

    this.vy += 0.8; if (this.vy > 18) this.vy = 18;

    // Horizontal
    this.x += this.vx;
    for (const p of platforms) {
      if (rectsCollide(this.rect, p)) {
        if (this.vx > 0) this.x = p.x - this.width;
        else if (this.vx < 0) this.x = p.x + p.width;
      }
    }

    // Vertical
    const prevY = this.y;
    this.y += this.vy; this.onGround = false;
    for (const p of platforms) {
      if (rectsCollide(this.rect, p)) {
        if (this.vy > 0) { this.y = p.y - this.height; this.vy = 0; this.onGround = true; this.lastSafe = { x: this.x, y: this.y }; }
        else if (this.vy < 0) { this.y = p.y + p.height; this.vy = 0; }
      }
    }

    // Lava
    for (const l of lavas) { if (rectsCollide(this.rect, l)) this.dead = true; }
    if (this.y > WORLD_HEIGHT) this.dead = true;
  }

  botLogic(platforms, lavas) {
    // Smarter heuristic:
    // 1) If falling badly, steer back to last safe
    // 2) If on ground, choose next target platform above within jump arc; else approach edge
    // 3) Avoid lava by jumping early or detouring

    // Parameters by difficulty
    const cfg = {
      speed: (botDifficulty === "normal" ? 5.0 : botDifficulty === "tryhard" ? 5.8 : 6.2),
      jumpV: -14.2,
      arcX: (botDifficulty === "master" ? 180 : botDifficulty === "tryhard" ? 150 : 130),
      arcY: (botDifficulty === "master" ? 140 : botDifficulty === "tryhard" ? 120 : 110),
    };

    // If falling fast, try to return
    if (!this.onGround && this.vy > 12) {
      const dirBack = Math.sign(this.lastSafe.x - this.x);
      this.vx = dirBack * cfg.speed;
      return;
    }

    // Current ground platform
    let ground = null;
    const feetY = this.y + this.height + 2;
    for (const p of platforms) {
      const onTop = (feetY >= p.y - 2 && feetY <= p.y + 10);
      const midX = this.x + this.width / 2;
      if (onTop && midX >= p.x && midX <= p.x + p.width) { ground = p; break; }
    }

    // Target: prefer platforms ABOVE and towards finishPad.x
    const goalX = finishPad.x + finishPad.width / 2;
    const dirToGoal = Math.sign(goalX - (this.x + this.width / 2));
    let candidate = null;
    let bestScore = -Infinity;
    for (const p of platforms) {
      if (p.y >= this.y + this.height - 6) continue; // only consider above
      const dx = (dirToGoal > 0) ? (p.x - this.x) : ((this.x) - (p.x + p.width));
      if (dx < -20 || dx > cfg.arcX + 80) continue; // roughly ahead
      const dy = (this.y - p.y);
      if (dy > cfg.arcY + 60) continue; // too high
      // Score: closer to goal and moderately above
      const centerDistGoal = Math.abs((p.x + p.width / 2) - goalX);
      const score = -centerDistGoal - dy * 0.5;
      if (score > bestScore) { bestScore = score; candidate = p; }
    }

    // Avoid lava ahead (on same level)
    const aheadRect = {
      x: this.x + (dirToGoal > 0 ? this.width + 6 : -12),
      y: this.y + this.height - 8,
      width: 14, height: 14
    };
    const lavaAhead = lavas.some(l => rectsCollide(aheadRect, l));

    // Movement
    this.vx = dirToGoal * cfg.speed;

    // Jump logic
    const nearEdge = ground ? (
      (dirToGoal > 0 && this.x + this.width > ground.x + ground.width - 6) ||
      (dirToGoal < 0 && this.x < ground.x + 6)
    ) : true;

    const shouldJump =
      (this.onGround && (lavaAhead || nearEdge || !!candidate));

    if (shouldJump) {
      this.vy = cfg.jumpV; this.onGround = false;
    }
  }

  draw(ctx) {
    ctx.save();
    ctx.fillStyle = this.color;
    ctx.fillRect(this.x, this.y, this.width, this.height);
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

/* ==== WORLD (FUN, POSSIBLE OBBY) ==== */
const persistentFloor = { x: 0, y: 2350, width: 1600, height: 50, floor: true };
const defaultSpawn = { x: 60, y: persistentFloor.y - 48, width: 28, height: 32 };
const defaultFinishPad = { x: 1460, y: 620, width: 100, height: 24 };

let platforms = [];
let lavas = [];
let spawnMarker = { ...defaultSpawn };
let finishPad = { ...defaultFinishPad };

// First floor with reachable first platform right off the floor
const defaultPlatforms = [
  { ...persistentFloor },
  { x: 120, y: 2300, width: 160, height: 20 }, // right off the floor
  { x: 320, y: 2230, width: 160, height: 20 },
  { x: 540, y: 2160, width: 160, height: 20 },
  { x: 760, y: 2090, width: 160, height: 20 },
  { x: 980, y: 2020, width: 160, height: 20 },
  // Second floor
  { x: 240, y: 1880, width: 160, height: 20 },
  { x: 480, y: 1810, width: 160, height: 20 },
  { x: 740, y: 1740, width: 160, height: 20 },
  { x: 1000, y: 1670, width: 160, height: 20 },
  // Third floor
  { x: 300, y: 1540, width: 160, height: 20 },
  { x: 580, y: 1470, width: 160, height: 20 },
  { x: 860, y: 1400, width: 160, height: 20 },
  // Fourth floor near finish
  { x: 360, y: 1280, width: 160, height: 20 },
  { x: 660, y: 1200, width: 160, height: 20 },
  { x: 980, y: 1120, width: 160, height: 20 },
  { x: 1260, y: 1040, width: 160, height: 20 },
  { x: 1360, y: 900, width: 180, height: 20 },
  { x: 1430, y: 760, width: 160, height: 20 },
];

const defaultLava = [
  { x: 500, y: 2338, width: 60, height: 12 }, // small hazards along the way
  { x: 820, y: 2088, width: 60, height: 12 },
  { x: 1060, y: 1668, width: 60, height: 12 },
  { x: 1380, y: 748, width: 60, height: 12 },
];

function setDefaultLevel() {
  platforms = JSON.parse(JSON.stringify(defaultPlatforms));
  lavas = JSON.parse(JSON.stringify(defaultLava));
  spawnMarker = { ...defaultSpawn };
  finishPad = { ...defaultFinishPad };
}

/* ==== DRAWING ==== */
const canvas = document.getElementById("gameCanvas");
const ctx = canvas.getContext("2d");

function drawPlatform(ctx, p) {
  ctx.save();
  ctx.fillStyle = "#6d7b91";
  if (p.rot90) {
    ctx.translate(p.x, p.y);
    ctx.rotate(Math.PI / 2);
    ctx.fillRect(0, -p.width, p.height, p.width);
  } else {
    ctx.fillRect(p.x, p.y, p.width, p.height);
  }
  ctx.restore();
}
function drawLava(ctx, l) {
  ctx.save();
  ctx.fillStyle = "#d44b4b";
  if (l.rot90) {
    ctx.translate(l.x, l.y);
    ctx.rotate(Math.PI / 2);
    ctx.fillRect(0, -l.width, l.height, l.width);
  } else {
    ctx.fillRect(l.x, l.y, l.width, l.height);
  }
  ctx.restore();
}
function drawFinishPad(ctx, fp) {
  ctx.save();
  ctx.fillStyle = "#2bbf57";
  ctx.fillRect(fp.x, fp.y, fp.width, fp.height);
  ctx.restore();
}
function drawSpawnMarker(ctx, sp) {
  if (mode === "create") {
    ctx.save();
    ctx.fillStyle = "#3aeeb4";
    ctx.fillRect(sp.x, sp.y, sp.width, sp.height);
    ctx.restore();
  }
}
function drawWorld(ctx) {
  for (const p of platforms) drawPlatform(ctx, p);
  for (const l of lavas) drawLava(ctx, l);
  drawFinishPad(ctx, finishPad);
  drawSpawnMarker(ctx, spawnMarker);
}

/* ==== CAMERA (scaled for mobile small) ==== */
function getViewDims() {
  return [VIEW_WIDTH / renderScale, VIEW_HEIGHT / renderScale];
}
function calcCamera(px, py) {
  const [vw, vh] = getViewDims();
  let camX = Math.round(px - vw / 2);
  let camY = Math.round(py - vh / 2);
  camX = clamp(camX, 0, WORLD_WIDTH - vw);
  camY = clamp(camY, 0, WORLD_HEIGHT - vh);
  return [camX, camY];
}
function calcMergedCamera(p1, p2) {
  const mx = (p1.x + p1.width / 2 + p2.x + p2.width / 2) / 2;
  const my = (p1.y + p1.height / 2 + p2.y + p2.height / 2) / 2;
  return calcCamera(mx, my);
}
function getSplitMode(p1, p2) {
  const dx = (p1.x + p1.width / 2) - (p2.x + p2.width / 2);
  const dy = (p1.y + p1.height / 2) - (p2.y + p2.height / 2);
  const dist = Math.hypot(dx, dy);
  const splitThreshold = 480; const mergeThreshold = 360;
  if (dist > splitThreshold) return "split";
  if (dist < mergeThreshold) return "merged";
  return currentSplitMode;
}

/* ==== PLAYERS ==== */
const player1 = new Player(defaultSpawn.x, defaultSpawn.y, "#23b4e4", { left: "KeyA", right: "KeyD", jump: "KeyW" });
const player2 = new Player(defaultSpawn.x + 60, defaultSpawn.y, "#e4b423", { left: "ArrowLeft", right: "ArrowRight", jump: "ArrowUp" });

function resetPlayersToSpawn() {
  player1.x = spawnMarker.x; player1.y = spawnMarker.y; player1.vx = 0; player1.vy = 0; player1.dead = false; player1.onGround = false;
  player2.x = spawnMarker.x + 60; player2.y = spawnMarker.y; player2.vx = 0; player2.vy = 0; player2.dead = false; player2.onGround = false;
  player2.isBot = (playMode === "bot");
}

/* ==== HUD ==== */
function drawFinishBanner() {
  if (!finished) { finishBanner.style.display = "none"; return; }
  finishBanner.style.display = "block";
}

/* ==== EDITOR STATE & INPUT ==== */
const editorToolbar = document.getElementById("editorToolbar");
const editorHint = document.getElementById("editorHint");
const rotateBtn = document.getElementById("rotateBtn");
const playtestBtn = document.getElementById("playtestBtn");
const clearBtn = document.getElementById("clearBtn");
const exitCreateBtn = document.getElementById("exitCreateBtn");
let currentTool = "platform";
[...editorToolbar.querySelectorAll(".tool[data-tool]")].forEach(btn => {
  btn.addEventListener("click", () => {
    [...editorToolbar.querySelectorAll(".tool[data-tool]")].forEach(b => b.classList.remove("active"));
    btn.classList.add("active");
    currentTool = btn.getAttribute("data-tool");
    rotateMode = false;
    rotateBtn.classList.remove("active");
  });
});
rotateBtn.addEventListener("click", () => {
  rotateMode = !rotateMode;
  rotateBtn.classList.toggle("active", rotateMode);
});
playtestBtn.addEventListener("click", () => {
  mode = "playtest";
  editorToolbar.style.display = "none";
  editorHint.style.display = "none";
  syncMobileControls();
  finished = false;
  player1.x = spawnMarker.x; player1.y = spawnMarker.y; player1.vx = 0; player1.vy = 0; player1.dead = false;
  player2.dead = true; player2.isBot = false;
  startTimeP1 = Date.now();
});
clearBtn.addEventListener("click", () => {
  // Fresh: floor + spawn + finish
  platforms = [ { ...persistentFloor } ];
  lavas = [];
  spawnMarker = { ...defaultSpawn };
  finishPad = { ...defaultFinishPad };
});
exitCreateBtn.addEventListener("click", () => {
  mode = "menu"; menuLayer.style.display = "grid";
  editorToolbar.style.display = "none"; editorHint.style.display = "none";
  syncMobileControls();
});

let editorCamX = 0, editorCamY = 0;
function startCreateFresh() {
  mode = "create";
  menuLayer.style.display = "none";
  editorToolbar.style.display = "flex";
  editorHint.style.display = "block";
  platforms = [ { ...persistentFloor } ];
  lavas = [];
  spawnMarker = { ...defaultSpawn };
  finishPad = { ...defaultFinishPad };
  editorCamX = 0; editorCamY = 0;
  syncMobileControls();
}

function screenToWorld(sx, sy) {
  if (mode === "create") return [sx + editorCamX, sy + editorCamY];
  return [sx, sy];
}

let dragState = { dragging: false, obj: null, offsetX: 0, offsetY: 0, arr: null, index: -1 };
canvas.addEventListener("mousedown", e => {
  const rect = canvas.getBoundingClientRect();
  let [wx, wy] = screenToWorld(e.clientX - rect.left, e.clientY - rect.top);

  if (mode === "create") {
    // Rotate mode: toggle orientation of clicked object
    if (rotateMode) {
      const target = pickObjectAt(wx, wy);
      if (target) { target.obj.rot90 = !target.obj.rot90; }
      return;
    }

    // Spawn marker first
    if (pointInRect(wx, wy, spawnMarker)) {
      dragState = { dragging: true, obj: spawnMarker, offsetX: wx - spawnMarker.x, offsetY: wy - spawnMarker.y };
      return;
    }
    // Finish pad
    if (pointInRect(wx, wy, finishPad)) {
      dragState = { dragging: true, obj: finishPad, offsetX: wx - finishPad.x, offsetY: wy - finishPad.y };
      return;
    }
    // Existing objects
    const target = pickObjectAt(wx, wy);
    if (target) {
      dragState = { dragging: true, obj: target.obj, offsetX: wx - target.obj.x, offsetY: wy - target.obj.y, arr: target.arr, index: target.index };
      return;
    }
    // Place new
    wx = snap(wx, gridSnap); wy = snap(wy, gridSnap);
    if (currentTool === "platform") platforms.push({ x: wx, y: wy, width: 120, height: 20 });
    if (currentTool === "lava") lavas.push({ x: wx, y: wy, width: 60, height: 12 });
  }
});
canvas.addEventListener("mousemove", e => {
  if (!dragState.dragging) return;
  const rect = canvas.getBoundingClientRect();
  let [wx, wy] = screenToWorld(e.clientX - rect.left, e.clientY - rect.top);
  const nx = snap(wx - dragState.offsetX, gridSnap);
  const ny = snap(wy - dragState.offsetY, gridSnap);
  dragState.obj.x = nx; dragState.obj.y = ny;
  // Keep spawn on floor
  if (dragState.obj === spawnMarker) spawnMarker.y = persistentFloor.y - 48;
});
window.addEventListener("mouseup", () => { dragState.dragging = false; });

function pointInRect(x, y, r) { return x >= r.x && x <= r.x + r.width && y >= r.y && y <= r.y + r.height; }
function pickObjectAt(x, y) {
  const pools = [
    { arr: platforms, type: "platform" },
    { arr: lavas, type: "lava" },
  ];
  for (const pool of pools) {
    for (let i = pool.arr.length - 1; i >= 0; i--) {
      const o = pool.arr[i];
      if (o.floor) continue;
      if (pointInRect(x, y, o)) return { arr: pool.arr, index: i, obj: o };
    }
  }
  return null;
}

function updateEditorCamera() {
  const pan = 12;
  if (keys["KeyA"]) editorCamX = clamp(editorCamX - pan, 0, WORLD_WIDTH - CANVAS_WIDTH);
  if (keys["KeyD"]) editorCamX = clamp(editorCamX + pan, 0, WORLD_WIDTH - CANVAS_WIDTH);
  if (keys["KeyW"]) editorCamY = clamp(editorCamY - pan, 0, WORLD_HEIGHT - CANVAS_HEIGHT);
  if (keys["KeyS"]) editorCamY = clamp(editorCamY + pan, 0, WORLD_HEIGHT - CANVAS_HEIGHT);
}
function drawCreateView() {
  ctx.save();
  ctx.translate(-editorCamX, -editorCamY);
  drawWorld(ctx);
  ctx.restore();
}

/* ==== PLAY RENDER ==== */
function drawPlaySplitOrMerged() {
  const p2Active = (playMode === "multiplayer" || playMode === "bot");
  currentSplitMode = p2Active ? getSplitMode(player1, player2) : "merged";

  ctx.save();
  ctx.scale(renderScale, renderScale);

  if (p2Active && currentSplitMode === "split") {
    // Left view (P1)
    ctx.save();
    ctx.beginPath(); ctx.rect(0, 0, VIEW_WIDTH / renderScale, VIEW_HEIGHT / renderScale); ctx.clip();
    let [cx1, cy1] = calcCamera(player1.x + player1.width / 2, player1.y + player1.height / 2);
    ctx.translate(-cx1, -cy1);
    drawWorld(ctx); player1.draw(ctx); if (p2Active) player2.draw(ctx);
    ctx.restore();

    // Divider
    ctx.strokeStyle = "#222"; ctx.lineWidth = 4;
    ctx.beginPath(); ctx.moveTo(VIEW_WIDTH / renderScale, 0); ctx.lineTo(VIEW_WIDTH / renderScale, VIEW_HEIGHT / renderScale); ctx.stroke();

    // Right view (P2)
    ctx.save();
    ctx.beginPath(); ctx.rect(VIEW_WIDTH / renderScale, 0, VIEW_WIDTH / renderScale, VIEW_HEIGHT / renderScale); ctx.clip();
    let [cx2, cy2] = calcCamera(player2.x + player2.width / 2, player2.y + player2.height / 2);
    ctx.translate(-cx2 + VIEW_WIDTH / renderScale, -cy2);
    drawWorld(ctx); if (p2Active) player2.draw(ctx); player1.draw(ctx);
    ctx.restore();
  } else {
    // Single view
    ctx.save();
    let [cx, cy] = p2Active ? calcMergedCamera(player1, player2)
                            : calcCamera(player1.x + player1.width / 2, player1.y + player1.height / 2);
    ctx.translate(-cx, -cy);
    drawWorld(ctx);
    player1.draw(ctx);
    if (p2Active) player2.draw(ctx);
    ctx.restore();
  }

  ctx.restore();
}

/* ==== FINISH CHECK & TIMER ==== */
function checkFinishAndBanner() {
  finishBanner.style.display = "none";
  if (rectsCollide(player1.rect, finishPad)) {
    const elapsed = Date.now() - startTimeP1;
    finishBanner.textContent = `Player completed the tower in ${timeFmt(elapsed)}`;
    finishBanner.style.display = "block";
    finished = true;
  } else if (playMode === "bot" && rectsCollide(player2.rect, finishPad)) {
    const elapsed = Date.now() - startTimeP2;
    finishBanner.textContent = `Bot completed the tower in ${timeFmt(elapsed)}`;
    finishBanner.style.display = "block";
    finished = true;
  }
}

/* ==== GAME LOOP ==== */
function gameLoop() {
  ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

  if (mode === "play" || mode === "playtest") {
    const p2Active = (mode === "play" && (playMode === "multiplayer" || playMode === "bot"));

    // Independent death/respawn; timer resets on death
    if (player1.dead) { player1.x = spawnMarker.x; player1.y = spawnMarker.y; player1.vx = 0; player1.vy = 0; player1.dead = false; player1.onGround = false; startTimeP1 = Date.now(); finished = false; }
    if (p2Active && player2.dead) { player2.x = spawnMarker.x + 60; player2.y = spawnMarker.y; player2.vx = 0; player2.vy = 0; player2.dead = false; player2.onGround = false; startTimeP2 = Date.now(); finished = false; }

    player1.update(platforms, lavas);
    if (p2Active) { player2.isBot = (playMode === "bot"); player2.update(platforms, lavas); }

    drawPlaySplitOrMerged();
    checkFinishAndBanner();
  }

  if (mode === "create") {
    updateEditorCamera();
    drawCreateView();
  }

  requestAnimationFrame(gameLoop);
}

/* ==== STARTUP ==== */
applyScreenScale();
setDefaultLevel();
menuLayer.style.display = "grid";
syncMobileControls();
resetPlayersToSpawn();
gameLoop();
</script>
</body>
</html>
```
