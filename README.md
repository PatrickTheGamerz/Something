<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Tower Platformer — Full Updated Build</title>
  <style>
    :root {
      --bg: #0e0f14;
      --canvas: #141725;
      --panel: #1b1f2e;
      --border: #2a3147;
      --text: #e9edf3;
      --muted: #94a3b8;
      --accent: #4ab3ed;
      --accent2: #e4b423;
      --danger: #c63e3e;
      --ok: #27c39f;
      --mc-size: 84px; /* mobile control default size */

      /* Responsive tuning variables */
      --canvas-scale: 1;
      --canvas-shift-x: 0px;
      --ui-scale: 1;
      --menu-scale: 1;
      --menu-offset-x: 0px;
      --toolbar-scale: 1;
      --hint-scale: 1;
      --font-scale: 1;
      --btn-scale: 1;
      --panel-width: 860px;
    }
    html, body { margin:0; height:100%; background:var(--bg); color:var(--text); font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Arial; overflow:hidden; }
    canvas {
      display:block; margin:0 auto; background:var(--canvas);
      transition: transform 0.2s ease;
      transform-origin: center top;
      transform: translateX(var(--canvas-shift-x)) scale(var(--canvas-scale));
    }

    /* UI layers */
    .ui-layer { position:fixed; inset:0; display:grid; place-items:center; pointer-events:none; }
    .panel {
      pointer-events:auto; background:var(--panel); border:1px solid var(--border); border-radius:12px; padding:20px;
      width:var(--panel-width); max-width:calc(100% - 40px); box-shadow:0 20px 60px rgba(0,0,0,0.45);
      max-height: 80vh; overflow-y: auto; /* scrolling */
      transform: scale(var(--menu-scale));
      transform-origin: top left;
      margin-left: var(--menu-offset-x);
    }
    .panel h1 { margin:0 0 8px 0; transform: scale(var(--font-scale)); transform-origin: left top; }
    .row { display:flex; gap:12px; align-items:center; margin:12px 0; flex-wrap: wrap; transform: scale(var(--ui-scale)); transform-origin: left top; }
    .btn {
      background:#222637; border:1px solid var(--border); color:var(--text);
      border-radius:10px; padding:12px 14px; font-weight:600; cursor:pointer;
      transform: scale(var(--btn-scale)); transform-origin: left top;
    }
    .btn:hover { background:#2a3147; }
    .btn.primary { background:linear-gradient(180deg,#2a91c7,#1f78aa); border-color:#2a91c7; }
    .btn.warn { background:linear-gradient(180deg,#b7561f,#9d430f); border-color:#b7561f; }
    .btn.danger { background:linear-gradient(180deg,#c63e3e,#a93131); border-color:#c63e3e; }
    .segmented { display:flex; gap:8px; flex-wrap:wrap; }
    .seg { background:#222637; border:1px solid var(--border); color:var(--text); padding:10px 12px; border-radius:8px; cursor:pointer; }
    .seg.active { background:#2a3147; border-color:#3b4767; }
    .segmented.disabled .seg { opacity: 0.5; pointer-events: none; }
    .sub { margin-top:14px; padding-top:14px; border-top:1px solid #252a3a; }
    .help { color:var(--muted); font-size:13px; transform: scale(var(--font-scale)); transform-origin: left top; }

    /* Editor toolbar */
    .toolbar {
      position:fixed; left:16px; top:16px; display:flex; gap:8px; background:#141725cc; border:1px solid #242a3c;
      border-radius:10px; padding:8px; z-index:10; backdrop-filter: blur(4px);
      transform: scale(var(--toolbar-scale));
      transform-origin: left top;
      margin-left: var(--menu-offset-x);
    }
    .tool { background:#222637; border:1px solid var(--border); color:#d7deea; padding:8px 12px; border-radius:8px; cursor:pointer; }
    .tool.active { background:#2a3147; border-color:#3b4767; }
    .hint {
      position:fixed; right:16px; top:16px; background:#141725cc; color:#cfe2ff; border:1px solid #242a3c;
      padding:8px 12px; border-radius:10px; backdrop-filter: blur(4px); pointer-events:none; z-index:10;
      transform: scale(var(--hint-scale));
      transform-origin: right top;
    }

    /* Mobile controls */
    .mobile-controls { position:fixed; inset:0; pointer-events:none; }
    .mc-btn {
      position:absolute; width:var(--mc-size); height:var(--mc-size); border-radius:50%;
      background: radial-gradient(circle at 30% 30%, #2a3147, #181b26 60%);
      border:1px solid #3b4767; box-shadow: inset 0 6px 12px rgba(0,0,0,0.4), 0 2px 8px rgba(0,0,0,0.4);
      pointer-events:auto; display:flex; align-items:center; justify-content:center; color:#cfe2ff; font-weight:700; font-size:24px;
      user-select:none;
    }
    .mc-btn:active { filter:brightness(1.2); }

    /* Mobile-small layout nudges */
    body.mobile-small .ui-layer { place-items: start; }
    body.mobile-small .panel { margin-left: 12px; }
    body.mobile-small .toolbar { left: 8px; }
  </style>
</head>
<body>
<canvas id="gameCanvas" width="1024" height="480"></canvas>

<!-- Menu -->
<div class="ui-layer" id="menuLayer">
  <div class="panel">
    <h1>Two-Player Tower</h1>
    <p class="help">Sprint, jump, and climb. Build your tower. Play with a friend or a bot.</p>

    <div class="row">
      <button class="btn primary" id="btnPlay">PLAY</button>
      <button class="btn warn" id="btnCreate">CREATE</button>
      <button class="btn" id="btnSettings">SETTINGS</button>
    </div>

    <!-- Settings -->
    <div id="settingsBlock" class="sub" style="display:none;">
      <div class="row">
        <div style="min-width:160px;">Device</div>
        <div class="segmented" id="deviceSeg">
          <div class="seg active" data-device="pc">PC</div>
          <div class="seg" data-device="mobile">Mobile</div>
        </div>
      </div>
      <div class="row">
        <div style="min-width:160px;">Control overlay</div>
        <div class="segmented" id="overlaySeg">
          <div class="seg active" data-overlay="off">Off</div>
          <div class="seg" data-overlay="on">On</div>
        </div>
      </div>
      <div class="row">
        <div style="min-width:160px;">Smaller screen</div>
        <div class="segmented" id="screenSeg">
          <div class="seg active" data-screen="normal">Normal</div>
          <div class="seg" data-screen="small">Smaller</div>
        </div>
      </div>
      <div class="row">
        <div style="min-width:160px;">Other player transparency</div>
        <input type="range" id="alphaSlider" min="0.4" max="1" step="0.05" value="1" />
        <span id="alphaLabel">1.00</span>
      </div>
      <div class="help">Smaller screen scales the canvas and mobile buttons to keep everything accessible.</div>
    </div>

    <!-- Play options -->
    <div id="playOptions" class="sub" style="display:none;">
      <div class="row">
        <div style="min-width:160px;">Mode</div>
        <div class="segmented" id="playModeSeg">
          <div class="seg active" data-playmode="multiplayer">Multiplayer</div>
          <div class="seg" data-playmode="bot">Bot</div>
        </div>
      </div>
      <div id="botDiffRow" class="row" style="display:none;">
        <div style="min-width:160px;">Bot difficulty</div>
        <div class="segmented" id="botDiffSeg">
          <div class="seg active" data-diff="normal">Normal</div>
          <div class="seg" data-diff="tryhard">Tryhard</div>
          <div class="seg" data-diff="master">Tower Master</div>
        </div>
      </div>
      <div class="row">
        <div style="min-width:160px;">Tower</div>
        <div class="segmented" id="towerSeg">
          <div class="seg active" data-tower="default">Default</div>
          <div class="seg" data-tower="custom">Custom (last saved)</div>
        </div>
      </div>
      <div class="row">
        <button class="btn primary" id="btnStartPlay">Start</button>
        <button class="btn" id="btnBackMenu">Back</button>
      </div>
      <div class="help">Bot aims to reach the finish pad. Split-screen activates when players separate.</div>
    </div>
  </div>
</div>

<!-- Editor toolbar -->
<div class="toolbar" id="editorToolbar" style="display:none;">
  <div class="tool active" data-tool="platform">Platform</div>
  <div class="tool" data-tool="hazard">Hazard</div>
  <div class="tool" data-tool="collectible">Gem</div>
  <div class="tool" data-tool="checkpoint">Checkpoint</div>
  <div class="tool" id="gridSnapBtn">Snap: 20</div>
  <div class="tool" id="playtestBtn">Playtest</div>
  <div class="tool" id="clearBtn">Clear</div>
  <div class="tool" id="saveBtn">Save</div>
  <div class="tool" id="loadBtn">Load</div>
  <div class="tool danger" id="exitCreateBtn">Exit</div>
</div>
<div class="hint" id="editorHint" style="display:none;">Create: place with click, move with drag. WASD to pan. Spawn and Finish pad are movable.</div>

<!-- Mobile controls -->
<div class="mobile-controls" id="mobileControls" style="display:none;">
  <!-- Player 1 left side -->
  <div class="mc-btn" id="mcP1Left" style="left: 26px; bottom: 28px;">◀</div>
  <div class="mc-btn" id="mcP1Right" style="left: calc(26px + var(--mc-size) + 16px); bottom: 28px;">▶</div>
  <div class="mc-btn" id="mcP1Jump" style="left: calc(26px + var(--mc-size)/2 + 8px); bottom: calc(28px + var(--mc-size) + 18px);">⤒</div>
  <!-- Player 2 right side -->
  <div class="mc-btn" id="mcP2Left" style="right: calc(26px + var(--mc-size) + 16px); bottom: 28px;">◀</div>
  <div class="mc-btn" id="mcP2Right" style="right: 26px; bottom: 28px;">▶</div>
  <div class="mc-btn" id="mcP2Jump" style="right: calc(26px + var(--mc-size)/2 + 8px); bottom: calc(28px + var(--mc-size) + 18px);">⤒</div>
</div>

<script>
/* ==== CONFIG ==== */
const CANVAS_WIDTH = 1024;
const CANVAS_HEIGHT = 480;
const VIEW_WIDTH = CANVAS_WIDTH / 2;
const VIEW_HEIGHT = CANVAS_HEIGHT;
const WORLD_WIDTH = 1600;
const WORLD_HEIGHT = 2400;
const SPLIT_DISTANCE = 480;
const MERGE_DISTANCE = 360;

/* ==== STATE ==== */
let mode = "menu"; // "menu" | "play" | "create" | "playtest"
let deviceType = "pc"; // "pc" | "mobile"
let overlayEnabled = false;
let screenMode = "normal"; // "normal" | "small"
let playMode = "multiplayer"; // "multiplayer" | "bot"
let botDifficulty = "normal"; // "normal" | "tryhard" | "master"
let towerSelection = "default"; // "default" | "custom"
let currentSplitMode = "merged";
let gridSnap = 20;
let otherAlpha = 1.0; // transparency for other player/bot

/* Mobile-small gameplay tuning */
function getGameplayTuning() {
  const mobileSmall = (deviceType === "mobile" && screenMode === "small");
  return {
    speedMult: mobileSmall ? 1.15 : 1.0,
    jumpVelocity: mobileSmall ? -15.5 : -14.2,
    botReachX: mobileSmall ? { normal: 160, tryhard: 200, master: 220 } : { normal: 140, tryhard: 180, master: 200 },
    botReachY: mobileSmall ? { normal: 120, tryhard: 140, master: 160 } : { normal: 110, tryhard: 130, master: 150 }
  };
}

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
const alphaSlider = document.getElementById("alphaSlider");
const alphaLabel = document.getElementById("alphaLabel");

function segmentedInit(segEl, value, attr, onChange) {
  [...segEl.children].forEach(ch => {
    ch.classList.toggle("active", ch.getAttribute(attr) === value);
    ch.addEventListener("click", () => {
      [...segEl.children].forEach(n => n.classList.remove("active"));
      ch.classList.add("active");
      onChange(ch.getAttribute(attr));
      syncMobileControls();
      // Re-apply screen scale after any relevant change
      applyScreenScale();
    });
  });
}
const deviceSeg = document.getElementById("deviceSeg");
const overlaySeg = document.getElementById("overlaySeg");
const screenSeg = document.getElementById("screenSeg");
const playModeSeg = document.getElementById("playModeSeg");
const botDiffSeg = document.getElementById("botDiffSeg");
const botDiffRow = document.getElementById("botDiffRow");
const towerSeg = document.getElementById("towerSeg");

segmentedInit(deviceSeg, deviceType, "data-device", v => {
  deviceType = v;
  // Control overlay only available on mobile
  const overlayContainer = document.getElementById("overlaySeg");
  if (deviceType === "mobile") {
    overlayContainer.classList.remove("disabled");
  } else {
    overlayEnabled = false;
    [...overlayContainer.children].forEach(n => n.classList.remove("active"));
    overlayContainer.querySelector('[data-overlay="off"]').classList.add("active");
    overlayContainer.classList.add("disabled");
  }
});
segmentedInit(overlaySeg, overlayEnabled ? "on" : "off", "data-overlay", v => overlayEnabled = (v === "on"));
segmentedInit(screenSeg, screenMode, "data-screen", v => { screenMode = v; applyScreenScale(); });
segmentedInit(playModeSeg, playMode, "data-playmode", v => { playMode = v; botDiffRow.style.display = (playMode === "bot") ? "flex" : "none"; });
segmentedInit(botDiffSeg, botDifficulty, "data-diff", v => botDifficulty = v);
segmentedInit(towerSeg, towerSelection, "data-tower", v => towerSelection = v);

alphaSlider.addEventListener("input", () => {
  otherAlpha = parseFloat(alphaSlider.value);
  alphaLabel.textContent = otherAlpha.toFixed(2);
});

btnPlay.addEventListener("click", () => {
  playOptions.style.display = "block";
  settingsBlock.style.display = "none";
  botDiffRow.style.display = (playMode === "bot") ? "flex" : "none";
});
btnBackMenu.addEventListener("click", () => {
  playOptions.style.display = "none";
});
btnSettings.addEventListener("click", () => {
  settingsBlock.style.display = settingsBlock.style.display === "none" ? "block" : "none";
  playOptions.style.display = "none";
});
btnStartPlay.addEventListener("click", () => {
  loadActiveLevel(towerSelection);
  mode = "play";
  menuLayer.style.display = "none";
  editorToolbar.style.display = "none";
  editorHint.style.display = "none";
  resetPlayersToSpawn();
  syncMobileControls();
});
btnCreate.addEventListener("click", () => {
  startCreateFresh();
});

/* ==== MOBILE CONTROLS ==== */
const mcLayer = document.getElementById("mobileControls");
function bindPress(btn, codes) {
  const down = e => { e.preventDefault(); codes.forEach(code => keys[code] = true); };
  const up = e => { e.preventDefault(); codes.forEach(code => keys[code] = false); };
  btn.addEventListener("touchstart", down, { passive:false });
  btn.addEventListener("touchend", up, { passive:false });
  btn.addEventListener("mousedown", down);
  window.addEventListener("mouseup", up);
}
bindPress(document.getElementById("mcP1Left"), ["KeyA"]);
bindPress(document.getElementById("mcP1Right"), ["KeyD"]);
bindPress(document.getElementById("mcP1Jump"), ["KeyW"]);
bindPress(document.getElementById("mcP2Left"), ["ArrowLeft"]);
bindPress(document.getElementById("mcP2Right"), ["ArrowRight"]);
bindPress(document.getElementById("mcP2Jump"), ["ArrowUp"]);

function syncMobileControls() {
  mcLayer.style.display = (deviceType === "mobile" && overlayEnabled && mode !== "menu") ? "block" : "none";
}

/* Enhanced screen scaling logic per device and mode */
function applyScreenScale() {
  const root = document.documentElement;
  const body = document.body;
  const mobile = (deviceType === "mobile");
  const small = (screenMode === "small");

  // Defaults
  root.style.setProperty("--canvas-scale", mobile ? 1 : 1);
  root.style.setProperty("--mc-size", mobile ? "84px" : "84px");
  root.style.setProperty("--canvas-shift-x", "0px");
  root.style.setProperty("--ui-scale", "1");
  root.style.setProperty("--menu-scale", "1");
  root.style.setProperty("--toolbar-scale", "1");
  root.style.setProperty("--hint-scale", "1");
  root.style.setProperty("--font-scale", "1");
  root.style.setProperty("--btn-scale", "1");
  root.style.setProperty("--panel-width", "860px");
  root.style.setProperty("--menu-offset-x", "0px");
  body.classList.remove("mobile-small");

  if (!mobile && small) {
    // PC smaller: same as before
    root.style.setProperty("--canvas-scale", "0.85");
    root.style.setProperty("--mc-size", "84px");
  } else if (mobile && !small) {
    // Mobile normal
    root.style.setProperty("--canvas-scale", "1");
    root.style.setProperty("--mc-size", "84px");
    root.style.setProperty("--ui-scale", "0.95");
    root.style.setProperty("--panel-width", "820px");
  } else if (mobile && small) {
    // Mobile smaller: aggressive scaling + UI compaction + left shift + gameplay tuning handled elsewhere
    body.classList.add("mobile-small");
    root.style.setProperty("--canvas-scale", "0.70");
    root.style.setProperty("--mc-size", "52px");
    root.style.setProperty("--ui-scale", "0.80");
    root.style.setProperty("--menu-scale", "0.86");
    root.style.setProperty("--toolbar-scale", "0.85");
    root.style.setProperty("--hint-scale", "0.85");
    root.style.setProperty("--font-scale", "0.90");
    root.style.setProperty("--btn-scale", "0.90");
    root.style.setProperty("--panel-width", "760px");
    root.style.setProperty("--menu-offset-x", "-20px");
    root.style.setProperty("--canvas-shift-x", "-120px"); // push gameplay left for better reachability
  }
}

/* ==== UTILS ==== */
function clamp(val, min, max) { return Math.max(min, Math.min(val, max)); }
function rectsCollide(a, b) {
  return (a.x < b.x + b.width && a.x + a.width > b.x && a.y < b.y + b.height && a.y + a.height > b.y);
}
function snap(v, s) { return s ? Math.round(v / s) * s : v; }

/* ==== PLAYER ==== */
class Player {
  constructor(x, y, color, controls) {
    this.x = x; this.y = y; this.width = 36; this.height = 48;
    this.color = color; this.vx = 0; this.vy = 0;
    this.baseSpeed = 5.2; this.baseJumpVelocity = -14.2; this.gravity = 0.8; this.maxFall = 18;
    this.onGround = false; this.controls = controls; this.dead = false;
    this.isBot = false;
    this.lastSafe = { x, y }; // for backtracking
    this.stuckTimer = 0;
  }
  get rect() { return { x: this.x, y: this.y, width: this.width, height: this.height }; }
  update(platforms, hazards) {
    const tune = getGameplayTuning();
    const speed = this.baseSpeed * tune.speedMult;
    const jumpV = tune.jumpVelocity; // replaces base when jumping

    if (this.isBot) this.botLogic(platforms, hazards, tune);
    else {
      let move = 0;
      if (keys[this.controls.left]) move -= 1;
      if (keys[this.controls.right]) move += 1;
      this.vx = move * speed;
      if (this.onGround && keys[this.controls.jump]) {
        this.vy = jumpV; this.onGround = false;
      }
    }
    this.vy += this.gravity;
    if (this.vy > this.maxFall) this.vy = this.maxFall;

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
    this.y += this.vy;
    const wasOnGround = this.onGround;
    this.onGround = false;
    for (const p of platforms) {
      if (rectsCollide(this.rect, p)) {
        if (this.vy > 0) { this.y = p.y - this.height; this.vy = 0; this.onGround = true; }
        else if (this.vy < 0) { this.y = p.y + p.height; this.vy = 0; }
      }
    }

    // Hazards kill independently
    for (const h of hazards) if (rectsCollide(this.rect, h)) this.dead = true;

    // Update last safe standing spot
    if (this.onGround) this.lastSafe = { x: this.x, y: this.y };

    // World bounds
    if (this.y > WORLD_HEIGHT) this.dead = true;

    // Detect stuck (not moving vertically while pushing wall)
    if (Math.abs(this.vx) > 0.1 && Math.abs(this.y - prevY) < 0.01 && !this.onGround) this.stuckTimer++;
    else this.stuckTimer = 0;
  }
  botLogic(platforms, hazards, tune) {
    // Heuristic pathing bot: can move left/right, avoid lava, backtrack if stuck/fallen.
    const cfg = {
      speedScale: (botDifficulty === "normal" ? 0.95 : botDifficulty === "tryhard" ? 1.15 : 1.3),
      jumpBias: (botDifficulty === "normal" ? 0.02 : botDifficulty === "tryhard" ? 0.06 : 0.12),
      perfectTiming: (botDifficulty === "master")
    };
    const goalX = finishPad.x + finishPad.width / 2;
    const dx = goalX - (this.x + this.width / 2);
    let dir = Math.sign(dx); // towards goal

    // Sense ground under feet
    const feetY = this.y + this.height + 2;
    let ground = null;
    for (const p of platforms) {
      if (feetY >= p.y - 2 && feetY <= p.y + 10 && this.x + this.width/2 >= p.x && this.x + this.width/2 <= p.x + p.width) {
        ground = p; break;
      }
    }

    // If falling or dead, backtrack to last safe
    if (!this.onGround && this.vy > 10) {
      dir = Math.sign(this.lastSafe.x - this.x);
    }

    // Lookahead ray to avoid hazards and detect edges
    const aheadX = this.x + (dir > 0 ? this.width + 8 : -8);
    const frontRect = { x: aheadX, y: this.y + this.height - 6, width: 12, height: 12 };
    const hazardAhead = hazards.some(h => rectsCollide(frontRect, h));
    const nearEdge = ground ? ((dir > 0 && aheadX > ground.x + ground.width - 10) || (dir < 0 && aheadX < ground.x + 10)) : true;

    // Find candidate platform within jump window (both left and right)
    const reachXMap = tune.botReachX;
    const reachYMap = tune.botReachY;
    const jumpReachX = reachXMap[botDifficulty];
    const jumpReachY = reachYMap[botDifficulty];

    let candidate = null;
    for (const p of platforms) {
      const withinX = (dir > 0)
        ? (p.x <= aheadX + jumpReachX && p.x + p.width >= aheadX)
        : (p.x <= aheadX && p.x + p.width >= aheadX - jumpReachX);
      const aboveOrLevel = (p.y + p.height <= this.y + this.height + 12);
      const withinY = (this.y - p.y) <= jumpReachY + 40;
      if (withinX && aboveOrLevel && withinY) { candidate = p; break; }
    }

    // Decide movement
    this.vx = dir * (this.baseSpeed * tune.speedMult) * cfg.speedScale;

    // Perfect edge-timed jumps (master) or heuristic jumps
    const needJump = nearEdge || hazardAhead || !!candidate;
    const randomJump = Math.random() < cfg.jumpBias;
    const shouldJump = this.onGround && (needJump || randomJump);

    if (cfg.perfectTiming && this.onGround) {
      const edgeThreshold = 6;
      const atEdge = ground && (
        (dir > 0 && (ground.x + ground.width - (this.x + this.width)) < edgeThreshold) ||
        (dir < 0 && (this.x - ground.x) < edgeThreshold)
      );
      const hazardClose = hazards.some(h => {
        const ahead = dir > 0 ? (h.x - (this.x + this.width)) : ((this.x) - (h.x + h.width));
        const sameLevel = Math.abs((h.y) - (this.y + this.height)) < 16;
        return ahead > 0 && ahead < 24 && sameLevel;
      });
      if (atEdge || hazardClose || candidate) {
        this.vy = tune.jumpVelocity; this.onGround = false;
      }
    } else if (shouldJump) {
      this.vy = tune.jumpVelocity; this.onGround = false;
    }

    // Backtrack if stuck (e.g., sliding under platform lip)
    if (this.stuckTimer > 20 && this.onGround) {
      dir = -dir; this.vx = dir * (this.baseSpeed * tune.speedMult) * (cfg.speedScale * 0.8);
      if (Math.random() < 0.5) { this.vy = tune.jumpVelocity; this.onGround = false; }
      this.stuckTimer = 0;
    }
  }
  draw(ctx, alpha = 1.0) {
    ctx.save();
    ctx.globalAlpha = alpha;
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

/* ==== WORLD (DEFAULT: REACHABLE FLOORS) ==== */
const persistentFloor = { x: 0, y: 2350, width: 1600, height: 50, floor: true };
const defaultSpawn = { x: 100, y: 2302, width: 28, height: 32 }; // sits on floor (floor.y - player.height)
const defaultFinishPad = { x: 1480, y: 540, width: 100, height: 24 }; // finish pad at top

const defaultPlatforms = [
  { ...persistentFloor },
  // Floor 1: gentle staircase (vertical delta ~80)
  { x: 160, y: 2220, width: 180, height: 20, floorColor: 1 },
  { x: 360, y: 2140, width: 160, height: 20, floorColor: 1 },
  { x: 560, y: 2060, width: 160, height: 20, floorColor: 1 },
  { x: 760, y: 1980, width: 160, height: 20, floorColor: 1 },
  { x: 960, y: 1900, width: 160, height: 20, floorColor: 1 },

  // Floor 2: a bit harder
  { x: 240, y: 1780, width: 180, height: 20, floorColor: 2 },
  { x: 470, y: 1700, width: 160, height: 20, floorColor: 2 },
  { x: 720, y: 1620, width: 160, height: 20, floorColor: 2 },
  { x: 980, y: 1540, width: 160, height: 20, floorColor: 2 },
  { x: 1240, y: 1460, width: 160, height: 20, floorColor: 2 },

  // Floor 3: tighter gaps, still jumpable (~100 up)
  { x: 300, y: 1340, width: 180, height: 20, floorColor: 1 },
  { x: 560, y: 1240, width: 160, height: 20, floorColor: 1 },
  { x: 820, y: 1140, width: 160, height: 20, floorColor: 1 },
  { x: 1080, y: 1040, width: 160, height: 20, floorColor: 1 },

  // Floor 4: nearing top
  { x: 380, y: 920, width: 180, height: 20, floorColor: 2 },
  { x: 660, y: 820, width: 160, height: 20, floorColor: 2 },
  { x: 940, y: 720, width: 160, height: 20, floorColor: 2 },
  { x: 1220, y: 620, width: 160, height: 20, floorColor: 2 },
];
const defaultHazards = [
  { x: 520, y: 2335, width: 40, height: 15 },
  { x: 900, y: 1975, width: 40, height: 15 },
  { x: 1040, y: 1535, width: 40, height: 15 },
  { x: 1200, y: 1095, width: 40, height: 15 },
];
const defaultCollectibles = [
  { x: 180, y: 2190, width: 20, height: 20 },
  { x: 580, y: 2050, width: 20, height: 20 },
  { x: 980, y: 1890, width: 20, height: 20 },
  { x: 740, y: 1610, width: 20, height: 20 },
  { x: 860, y: 1130, width: 20, height: 20 },
];
const defaultCheckpoints = [
  { x: 580, y: 2050, width: 24, height: 28 },
  { x: 980, y: 1890, width: 24, height: 28 },
  { x: 740, y: 1610, width: 24, height: 28 },
  { x: 860, y: 1130, width: 24, height: 28 },
];

/* ==== ACTIVE LEVEL ==== */
let platforms = [];
let hazards = [];
let collectibles = [];
let checkpoints = [];
let spawnMarker = { ...defaultSpawn };
let finishPad = { ...defaultFinishPad };

function ensureFloor() {
  const hasFloor = platforms.some(p => p.x === persistentFloor.x && p.y === persistentFloor.y && p.width === persistentFloor.width && p.height === persistentFloor.height);
  if (!hasFloor) platforms.unshift({ ...persistentFloor });
}
function clampSpawnToFloor() {
  // Always keep spawn on top of floor
  spawnMarker.x = clamp(spawnMarker.x, 0, WORLD_WIDTH - spawnMarker.width);
  spawnMarker.y = persistentFloor.y - 48; // player height
}
function setDefaultLevel() {
  platforms = JSON.parse(JSON.stringify(defaultPlatforms));
  hazards = JSON.parse(JSON.stringify(defaultHazards));
  collectibles = JSON.parse(JSON.stringify(defaultCollectibles));
  checkpoints = JSON.parse(JSON.stringify(defaultCheckpoints));
  spawnMarker = { ...defaultSpawn };
  finishPad = { ...defaultFinishPad };
  ensureFloor();
  clampSpawnToFloor();
}
function setCustomLevelFromStorage() {
  const data = localStorage.getItem("towerLevel");
  if (data) {
    const parsed = JSON.parse(data);
    platforms = parsed.platforms || [ { ...persistentFloor } ];
    hazards = parsed.hazards || [];
    collectibles = parsed.collectibles || [];
    checkpoints = parsed.checkpoints || [];
    spawnMarker = parsed.spawnMarker || { ...defaultSpawn };
    finishPad = parsed.finishPad || { ...defaultFinishPad };
    ensureFloor();
    clampSpawnToFloor();
  } else {
    setDefaultLevel();
  }
}
function loadActiveLevel(selection) {
  if (selection === "custom") setCustomLevelFromStorage();
  else setDefaultLevel();
}

/* ==== DRAWING ==== */
const canvas = document.getElementById("gameCanvas");
const ctx = canvas.getContext("2d");

function drawPlatform(ctx, p) {
  ctx.save();
  const base1 = p.floorColor === 2 ? "#51576a" : "#6d7b91";
  const base2 = p.floorColor === 2 ? "#34394a" : "#393f52";
  const g = ctx.createLinearGradient(p.x, p.y, p.x, p.y + p.height);
  g.addColorStop(0, p.floor ? "#555" : base1);
  g.addColorStop(1, p.floor ? "#333" : base2);
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
  // Invisible in play; visible in create
  if (mode === "create") {
    ctx.save();
    const g = ctx.createLinearGradient(cp.x, cp.y, cp.x, cp.y + cp.height);
    g.addColorStop(0, "#5acef7"); g.addColorStop(1, "#2a88bf");
    ctx.fillStyle = g; ctx.fillRect(cp.x, cp.y, cp.width, cp.height);
    ctx.fillStyle = "#fff"; ctx.font = "bold 11px sans-serif"; ctx.fillText("✔", cp.x + 5, cp.y + 18);
    ctx.restore();
  }
}
function drawSpawnMarker(ctx, sp) {
  // Visible only in create mode
  if (mode === "create") {
    ctx.save();
    const g = ctx.createLinearGradient(sp.x, sp.y, sp.x, sp.y + sp.height);
    g.addColorStop(0, "#3aeeb4"); g.addColorStop(1, "#1aa57e");
    ctx.fillStyle = g;
    ctx.fillRect(sp.x, sp.y, sp.width, sp.height);
    ctx.fillStyle = "#062a22";
    ctx.font = "bold 11px sans-serif";
    ctx.fillText("SP", sp.x + 6, sp.y + 20);
    ctx.restore();
  }
}
function drawFinishPad(ctx, fp) {
  ctx.save();
  const g = ctx.createLinearGradient(fp.x, fp.y, fp.x, fp.y + fp.height);
  g.addColorStop(0, "#9bff9b"); g.addColorStop(1, "#2bbf57");
  ctx.fillStyle = g; ctx.fillRect(fp.x, fp.y, fp.width, fp.height);
  ctx.strokeStyle = "#0b4f2b"; ctx.strokeRect(fp.x, fp.y, fp.width, fp.height);
  ctx.restore();
}
function drawWorld(ctx) {
  for (const plat of platforms) drawPlatform(ctx, plat);
  for (const hazard of hazards) drawHazard(ctx, hazard);
  for (const c of collectibles) drawCollectible(ctx, c);
  for (const cp of checkpoints) drawCheckpoint(ctx, cp);
  drawFinishPad(ctx, finishPad);
  drawSpawnMarker(ctx, spawnMarker);
}

/* ==== CAMERA ==== */
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

/* ==== PLAYERS ==== */
const player1 = new Player(defaultSpawn.x, defaultSpawn.y, "#23b4e4", { left: "KeyA", right: "KeyD", jump: "KeyW" });
const player2 = new Player(defaultSpawn.x + 60, defaultSpawn.y, "#e4b423", { left: "ArrowLeft", right: "ArrowRight", jump: "ArrowUp" });

function resetPlayersToSpawn() {
  player1.x = spawnMarker.x; player1.y = spawnMarker.y; player1.vx = 0; player1.vy = 0; player1.dead = false; player1.onGround = false;
  player2.x = spawnMarker.x + 60; player2.y = spawnMarker.y; player2.vx = 0; player2.vy = 0; player2.dead = false; player2.onGround = false;
  player2.isBot = (playMode === "bot");
}

/* ==== HUD ==== */
function drawHud(player, x, y, label) {
  ctx.save();
  ctx.globalAlpha = 0.88; ctx.fillStyle = "#111"; ctx.fillRect(x - 10, y - 28, 220, 48);
  ctx.globalAlpha = 1.0; ctx.fillStyle = "#fff"; ctx.font = "16px monospace";
  ctx.fillText(label + " X:" + Math.round(player.x) + " Y:" + Math.round(player.y), x, y);
  ctx.fillText("State: " + (player.dead ? "DEAD" : player.onGround ? "GROUND" : "AIR"), x, y + 22);
  ctx.restore();
}
function drawPlayHUD() {
  drawHud(player1, 28, 36, "P1");
  if (mode === "play") {
    const hx = currentSplitMode === "split" ? VIEW_WIDTH + 28 : 280;
    const label = (playMode === "bot") ? ("BOT-" + botDifficulty.toUpperCase()) : "P2";
    drawHud(player2, hx, 36, label);
  }
}

/* ==== EDITOR STATE & INPUT ==== */
const editorToolbar = document.getElementById("editorToolbar");
const editorHint = document.getElementById("editorHint");
let currentTool = "platform";
[...editorToolbar.querySelectorAll(".tool[data-tool]")].forEach(btn => {
  btn.addEventListener("click", () => {
    [...editorToolbar.querySelectorAll(".tool[data-tool]")].forEach(b => b.classList.remove("active"));
    btn.classList.add("active");
    currentTool = btn.getAttribute("data-tool");
  });
});
document.getElementById("gridSnapBtn").addEventListener("click", () => {
  gridSnap = (gridSnap === 20 ? 40 : gridSnap === 40 ? 0 : 20);
  document.getElementById("gridSnapBtn").textContent = "Snap: " + (gridSnap || "off");
});
document.getElementById("playtestBtn").addEventListener("click", () => {
  mode = "playtest";
  editorToolbar.style.display = "none";
  editorHint.style.display = "none";
  ensureFloor(); clampSpawnToFloor();
  syncMobileControls();
  // Single player only in playtest, independent of bot/multiplayer
  player2.dead = true; player2.isBot = false;
  player1.x = spawnMarker.x; player1.y = spawnMarker.y; player1.vx = 0; player1.vy = 0; player1.dead = false; player1.onGround = false;
});
document.getElementById("clearBtn").addEventListener("click", () => {
  // Fresh level: floor + spawn + finish pad
  platforms = [ { ...persistentFloor } ];
  hazards = []; collectibles = []; checkpoints = [];
  spawnMarker = { ...defaultSpawn }; clampSpawnToFloor();
  finishPad = { ...defaultFinishPad };
});
document.getElementById("saveBtn").addEventListener("click", () => {
  ensureFloor(); clampSpawnToFloor();
  localStorage.setItem("towerLevel", JSON.stringify({ platforms, hazards, collectibles, checkpoints, spawnMarker, finishPad }));
  [...towerSeg.children].forEach(n => n.classList.remove("active"));
  towerSeg.querySelector('[data-tower="custom"]').classList.add("active");
  towerSelection = "custom";
});
document.getElementById("loadBtn").addEventListener("click", () => { setCustomLevelFromStorage(); });
document.getElementById("exitCreateBtn").addEventListener("click", () => {
  mode = "menu"; menuLayer.style.display = "grid";
  editorToolbar.style.display = "none"; editorHint.style.display = "none";
  syncMobileControls();
});

let editorCamX = 0, editorCamY = 0; // pan camera in create mode
function startCreateFresh() {
  mode = "create";
  menuLayer.style.display = "none";
  editorToolbar.style.display = "flex";
  editorHint.style.display = "block";
  // Start fresh: floor + spawn + finish pad
  platforms = [ { ...persistentFloor } ];
  hazards = []; collectibles = []; checkpoints = [];
  spawnMarker = { ...defaultSpawn }; clampSpawnToFloor();
  finishPad = { ...defaultFinishPad };
  editorCamX = 0; editorCamY = 0;
  syncMobileControls();
}
function screenToWorld(sx, sy) {
  if (mode === "create") return [sx + editorCamX, sy + editorCamY];
  return [sx, sy];
}
// Create mode interactions: place & move (including spawn and finish pad)
let dragState = { dragging: false, obj: null, offsetX: 0, offsetY: 0, arr: null, index: -1 };
canvas.addEventListener("mousedown", e => {
  const rect = canvas.getBoundingClientRect();
  let [wx, wy] = screenToWorld(e.clientX - rect.left, e.clientY - rect.top);
  if (mode === "create") {
    // Hit-test spawn marker first
    if (wx >= spawnMarker.x && wx <= spawnMarker.x + spawnMarker.width &&
        wy >= spawnMarker.y && wy <= spawnMarker.y + spawnMarker.height) {
      dragState = { dragging: true, obj: spawnMarker, offsetX: wx - spawnMarker.x, offsetY: wy - spawnMarker.y, arr: null, index: -1 };
      return;
    }
    // Hit-test finish pad next
    if (wx >= finishPad.x && wx <= finishPad.x + finishPad.width &&
        wy >= finishPad.y && wy <= finishPad.y + finishPad.height) {
      dragState = { dragging: true, obj: finishPad, offsetX: wx - finishPad.x, offsetY: wy - finishPad.y, arr: null, index: -1 };
      return;
    }
    // Hit-test existing objects (skip persistent floor)
    const pools = [
      { arr: platforms, type: "platform" },
      { arr: hazards, type: "hazard" },
      { arr: collectibles, type: "collectible" },
      { arr: checkpoints, type: "checkpoint" },
    ];
    for (const pool of pools) {
      for (let i = pool.arr.length - 1; i >= 0; i--) {
        const o = pool.arr[i];
        if (o.floor) continue;
        if (wx >= o.x && wx <= o.x + o.width && wy >= o.y && wy <= o.y + o.height) {
          dragState = { dragging: true, obj: o, offsetX: wx - o.x, offsetY: wy - o.y, arr: pool.arr, index: i };
          return;
        }
      }
    }
    // Place new object
    wx = snap(wx, gridSnap); wy = snap(wy, gridSnap);
    if (currentTool === "platform") platforms.push({ x: wx, y: wy, width: 120, height: 20 });
    if (currentTool === "hazard") hazards.push({ x: wx, y: wy, width: 40, height: 15 });
    if (currentTool === "collectible") collectibles.push({ x: wx, y: wy, width: 20, height: 20 });
    if (currentTool === "checkpoint") checkpoints.push({ x: wx, y: wy, width: 24, height: 28 });
  }
});
canvas.addEventListener("mousemove", e => {
  if (!dragState.dragging) return;
  const rect = canvas.getBoundingClientRect();
  let [wx, wy] = screenToWorld(e.clientX - rect.left, e.clientY - rect.top);
  const nx = snap(wx - dragState.offsetX, gridSnap);
  const ny = snap(wy - dragState.offsetY, gridSnap);
  dragState.obj.x = nx; dragState.obj.y = ny;
  if (dragState.obj === spawnMarker) clampSpawnToFloor();
});
window.addEventListener("mouseup", () => { dragState.dragging = false; });
// Create mode camera pan: WASD keys
function updateEditorCamera() {
  const pan = 12;
  if (keys["KeyA"]) editorCamX = clamp(editorCamX - pan, 0, WORLD_WIDTH - CANVAS_WIDTH);
  if (keys["KeyD"]) editorCamX = clamp(editorCamX + pan, 0, WORLD_WIDTH - CANVAS_WIDTH);
  if (keys["KeyW"]) editorCamY = clamp(editorCamY - pan, 0, WORLD_HEIGHT - CANVAS_HEIGHT);
  if (keys["KeyS"]) editorCamY = clamp(editorCamY + pan, 0, WORLD_HEIGHT - CANVAS_HEIGHT);
}
/* ==== CREATE DRAW WRAPPER ==== */
function drawCreateView() {
  ctx.save();
  ctx.translate(-editorCamX, -editorCamY);
  drawWorld(ctx);
  ctx.restore();
}

/* ==== CAMERA LOGIC IN PLAY ==== */
function drawPlaySplitOrMerged() {
  const p2Active = (playMode === "multiplayer" || playMode === "bot");
  currentSplitMode = p2Active ? getSplitMode(player1, player2) : "merged";

  if (p2Active && currentSplitMode === "split") {
    // Left view (P1)
    ctx.save();
    ctx.beginPath(); ctx.rect(0, 0, VIEW_WIDTH, VIEW_HEIGHT); ctx.clip();
    let [cx1, cy1] = calcCamera(player1.x + player1.width / 2, player1.y + player1.height / 2);
    ctx.translate(-cx1, -cy1);
    drawWorld(ctx); player1.draw(ctx); player2.draw(ctx, otherAlpha);
    ctx.restore();

    // Divider
    ctx.strokeStyle = "#222"; ctx.lineWidth = 4;
    ctx.beginPath(); ctx.moveTo(VIEW_WIDTH, 0); ctx.lineTo(VIEW_WIDTH, VIEW_HEIGHT); ctx.stroke();

    // Right view (P2)
    ctx.save();
    ctx.beginPath(); ctx.rect(VIEW_WIDTH, 0, VIEW_WIDTH, VIEW_HEIGHT); ctx.clip();
    let [cx2, cy2] = calcCamera(player2.x + player2.width / 2, player2.y + player2.height / 2);
    ctx.translate(-cx2 + VIEW_WIDTH, -cy2);
    drawWorld(ctx); player2.draw(ctx, otherAlpha); player1.draw(ctx);
    ctx.restore();
  } else {
    // Single view (merged or single player)
    ctx.save();
    let [cx, cy] = p2Active ? calcMergedCamera(player1, player2)
                            : calcCamera(player1.x + player1.width / 2, player1.y + player1.height / 2);
    ctx.translate(-cx, -cy);
    drawWorld(ctx);
    player1.draw(ctx);
    if (p2Active) player2.draw(ctx, otherAlpha);
    ctx.restore();
  }
}

/* ==== FINISH CHECK ==== */
function checkFinish() {
  const goalRect = finishPad;
  const p1Finished = rectsCollide(player1.rect, goalRect);
  const p2Finished = rectsCollide(player2.rect, goalRect);
  if (p1Finished || (playMode === "bot" && p2Finished)) {
    ctx.save();
    ctx.globalAlpha = 0.9;
    ctx.fillStyle = "#112";
    ctx.fillRect( CANVAS_WIDTH/2 - 160, 20, 320, 40 );
    ctx.fillStyle = "#9bff9b";
    ctx.font = "bold 20px monospace";
    ctx.fillText((p1Finished ? "Player" : "Bot") + " finished the tower!", CANVAS_WIDTH/2 - 150, 46);
    ctx.restore();
  }
}

/* ==== GAME LOOP ==== */
function gameLoop() {
  ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

  if (mode === "play" || mode === "playtest") {
    const p2Active = (mode === "play" && (playMode === "multiplayer" || playMode === "bot"));
    // Independent death/respawn
    if (player1.dead) { player1.x = spawnMarker.x; player1.y = spawnMarker.y; player1.vx = 0; player1.vy = 0; player1.dead = false; player1.onGround = false; }
    if (p2Active && player2.dead) { player2.x = spawnMarker.x + 60; player2.y = spawnMarker.y; player2.vx = 0; player2.vy = 0; player2.dead = false; player2.onGround = false; }

    player1.update(platforms, hazards);
    if (p2Active) {
      player2.isBot = (playMode === "bot");
      player2.update(platforms, hazards);
    }

    drawPlaySplitOrMerged();
    drawPlayHUD();
    checkFinish();
  }

  if (mode === "create") {
    updateEditorCamera();
    drawCreateView();
  }

  requestAnimationFrame(gameLoop);
}

/* ==== STARTUP ==== */
setDefaultLevel();
menuLayer.style.display = "grid";

/* Initialize overlay control availability based on default device (PC) */
document.getElementById("overlaySeg").classList.add("disabled");
overlayEnabled = false;

syncMobileControls();
applyScreenScale();
resetPlayersToSpawn();
gameLoop();
</script>
</body>
</html>
```
