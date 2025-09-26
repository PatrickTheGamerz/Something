<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Two-Player Tower Platformer — Menu + Settings + Create</title>
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
    canvas { display: block; margin: 0 auto; background: #171923; }
    /* UI */
    .ui-layer {
      position: fixed; inset: 0; pointer-events: none; display: grid; place-items: center;
    }
    .panel {
      pointer-events: auto;
      background: linear-gradient(180deg, var(--panel), var(--panel-2));
      border: 1px solid #1f2433; border-radius: 12px; box-shadow: 0 20px 60px rgba(0,0,0,0.45);
      padding: 20px; width: 720px; max-width: calc(100% - 40px);
    }
    .panel h1 { margin: 0 0 8px 0; font-size: 28px; letter-spacing: 0.4px; }
    .panel p { margin: 0 0 16px 0; color: var(--muted); }
    .menu-grid {
      display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; margin-top: 12px;
    }
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
    .segmented {
      display:flex; gap:8px;
    }
    .segmented .seg {
      background:#222637; border:1px solid #2a3147; color:var(--text); padding:10px 12px; border-radius:8px; cursor:pointer;
    }
    .seg.active { background:#2a3147; border-color:#3b4767; }
    .help {
      margin-top: 10px; font-size: 13px; color: var(--muted);
      border-top: 1px dashed #2a3147; padding-top: 10px;
    }
    /* Editor toolbar */
    .toolbar {
      position: fixed; left: 16px; top: 16px; pointer-events:auto;
      display:flex; gap:8px; background:#141725cc; border:1px solid #242a3c; border-radius:10px; padding:8px;
      backdrop-filter: blur(4px);
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
  </style>
</head>
<body>
<canvas id="gameCanvas" width="1024" height="480"></canvas>

<!-- UI Layer -->
<div class="ui-layer" id="menuLayer">
  <div class="panel">
    <h1>Two-Player Tower</h1>
    <p>Sprint, jump, and climb. Build your own levels. Play your way.</p>
    <div class="menu-grid">
      <button class="btn primary" id="btnPlay">PLAY</button>
      <button class="btn" id="btnSettings">SETTINGS</button>
      <button class="btn warn" id="btnCreate">CREATE</button>
    </div>
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
      <div class="help">Tip: Mobile overlay shows on-screen arrows and jump. You can still use a keyboard on PC.</div>
    </div>
  </div>
</div>

<!-- Editor toolbar (hidden until CREATE) -->
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

<!-- Mobile controls (toggle) -->
<div class="mobile-controls" id="mobileControls" style="display:none;">
  <div class="mc-btn" id="mcLeft" style="left: 28px; bottom: 28px;">
    <div class="mc-label">◀</div>
  </div>
  <div class="mc-btn" id="mcRight" style="left: 128px; bottom: 28px;">
    <div class="mc-label">▶</div>
  </div>
  <div class="mc-btn mc-small" id="mcA" style="right: 128px; bottom: 28px;">
    <div class="mc-label">A</div>
  </div>
  <div class="mc-btn" id="mcJump" style="right: 28px; bottom: 28px;">
    <div class="mc-label">⤒</div>
  </div>
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
let mode = "menu"; // "menu" | "play" | "create"
let deviceType = "pc"; // "pc" | "mobile"
let overlayEnabled = false; // show on-screen controls
let currentSplitMode = "merged";
let gridSnap = 20;

// ==== INPUT HANDLING ====
const keys = {};
window.addEventListener("keydown", e => keys[e.code] = true);
window.addEventListener("keyup", e => keys[e.code] = false);

// Utility to set active class on segmented controls
function segmentedInit(segEl, value, attr) {
  [...segEl.children].forEach(ch => {
    ch.classList.toggle("active", ch.getAttribute(attr) === value);
    ch.addEventListener("click", () => {
      [...segEl.children].forEach(n => n.classList.remove("active"));
      ch.classList.add("active");
      const v = ch.getAttribute(attr);
      if (attr === "data-device") deviceType = v;
      if (attr === "data-overlay") overlayEnabled = (v === "on");
      syncMobileControls();
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

btnPlay.addEventListener("click", () => {
  mode = "play";
  menuLayer.style.display = "none";
  editorToolbar.style.display = "none";
  syncMobileControls();
});
btnSettings.addEventListener("click", () => {
  settingsBlock.style.display = settingsBlock.style.display === "none" ? "block" : "none";
});
btnCreate.addEventListener("click", () => {
  mode = "create";
  menuLayer.style.display = "none";
  editorToolbar.style.display = "flex";
  syncMobileControls();
});

segmentedInit(deviceSeg, deviceType, "data-device");
segmentedInit(overlaySeg, overlayEnabled ? "on" : "off", "data-overlay");

// ==== MOBILE CONTROLS ====
const mcLayer = document.getElementById("mobileControls");
const mcLeft = document.getElementById("mcLeft");
const mcRight = document.getElementById("mcRight");
const mcJump = document.getElementById("mcJump");
const mcA = document.getElementById("mcA");

// Map mobile buttons to player1 controls (WASD) and arrows for player2
function bindTouch(btn, codesDown, codesUp) {
  const down = (e) => { e.preventDefault(); codesDown.forEach(code => keys[code] = true); };
  const up = (e) => { e.preventDefault(); codesUp.forEach(code => keys[code] = false); };
  btn.addEventListener("touchstart", down);
  btn.addEventListener("mousedown", down);
  window.addEventListener("touchend", up);
  window.addEventListener("mouseup", up);
}
bindTouch(mcLeft, ["KeyA", "ArrowLeft"], ["KeyA", "ArrowLeft"]);
bindTouch(mcRight, ["KeyD", "ArrowRight"], ["KeyD", "ArrowRight"]);
bindTouch(mcJump, ["KeyW", "ArrowUp"], ["KeyW", "ArrowUp"]);
bindTouch(mcA, ["KeyS"], ["KeyS"]);

function syncMobileControls() {
  mcLayer.style.display = (deviceType === "mobile" && overlayEnabled && mode !== "menu") ? "block" : "none";
}

// ==== UTILITY ====
function clamp(val, min, max) { return Math.max(min, Math.min(val, max)); }
function rectsCollide(a, b) {
  return (a.x < b.x + b.width && a.x + a.width > b.x && a.y < b.y + b.height && a.y + a.height > b.y);
}
function snap(v, s) { return Math.round(v / s) * s; }

// ==== PLAYER CLASS ====
class Player {
  constructor(x, y, color, controls) {
    this.x = x; this.y = y; this.width = 36; this.height = 48;
    this.color = color; this.vx = 0; this.vy = 0;
    this.speed = 5.4; this.jumpVelocity = -14.2; this.gravity = 0.8; this.maxFall = 18;
    this.onGround = false; this.controls = controls;
    this.checkpoint = { x, y }; this.collected = 0; this.dead = false;
  }
  get rect() { return { x: this.x, y: this.y, width: this.width, height: this.height }; }
  respawn() { this.x = this.checkpoint.x; this.y = this.checkpoint.y; this.vx = 0; this.vy = 0; this.dead = false; }
  update(platforms, hazards, collectibles, checkpoints) {
    let move = 0;
    if (keys[this.controls.left]) move -= 1;
    if (keys[this.controls.right]) move += 1;
    this.vx = move * this.speed;

    if (this.onGround && keys[this.controls.jump]) {
      this.vy = this.jumpVelocity; this.onGround = false;
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
    // Body with subtle shading and outline
    ctx.save();
    // Shadow
    ctx.fillStyle = "rgba(0,0,0,0.25)";
    ctx.beginPath();
    ctx.ellipse(this.x + this.width/2, this.y + this.height + 6, this.width*0.45, 6, 0, 0, Math.PI*2);
    ctx.fill();

    // Body
    const grd = ctx.createLinearGradient(this.x, this.y, this.x, this.y + this.height);
    grd.addColorStop(0, this.color);
    grd.addColorStop(1, "#0c0f14");
    ctx.fillStyle = grd;
    ctx.strokeStyle = "#0a0d12";
    ctx.lineWidth = 2;
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

// ==== WORLD (DEFAULT) ====
let basePlatforms = [
  { x: 0, y: 2350, width: 1600, height: 50 },
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
let baseHazards = [
  { x: 500, y: 2335, width: 40, height: 15 },
  { x: 850, y: 1985, width: 40, height: 15 },
  { x: 700, y: 1585, width: 40, height: 15 },
];
let baseCollectibles = [
  { x: 220, y: 2170, width: 20, height: 20 },
  { x: 620, y: 2070, width: 20, height: 20 },
  { x: 1020, y: 1970, width: 20, height: 20 },
  { x: 820, y: 1770, width: 20, height: 20 },
  { x: 1220, y: 1670, width: 20, height: 20 },
];
let baseCheckpoints = [
  { x: 620, y: 2070, width: 24, height: 28 },
  { x: 820, y: 1770, width: 24, height: 28 },
  { x: 1220, y: 1670, width: 24, height: 28 },
];

// Active level data (changes in editor/playtest)
let platforms = JSON.parse(JSON.stringify(basePlatforms));
let hazards = JSON.parse(JSON.stringify(baseHazards));
let collectibles = JSON.parse(JSON.stringify(baseCollectibles));
let checkpoints = JSON.parse(JSON.stringify(baseCheckpoints));

// ==== EDITOR STATE ====
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
  platforms = []; hazards = []; collectibles = []; checkpoints = [];
});
document.getElementById("saveBtn").addEventListener("click", () => {
  localStorage.setItem("towerLevel", JSON.stringify({ platforms, hazards, collectibles, checkpoints }));
});
document.getElementById("loadBtn").addEventListener("click", () => {
  const data = localStorage.getItem("towerLevel");
  if (data) {
    const parsed = JSON.parse(data);
    platforms = parsed.platforms || [];
    hazards = parsed.hazards || [];
    collectibles = parsed.collectibles || [];
    checkpoints = parsed.checkpoints || [];
  }
});
document.getElementById("playtestBtn").addEventListener("click", () => {
  mode = "play";
  editorToolbar.style.display = "none";
  syncMobileControls();
});

// ==== EDITOR INPUT ====
const canvas = document.getElementById("gameCanvas");
const ctx = canvas.getContext("2d");
canvas.addEventListener("click", e => {
  if (mode !== "create") return;
  const rect = canvas.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const y = e.clientY - rect.top;
  // Convert to world coords (no camera in editor, just raw)
  let wx = x, wy = y;
  if (gridSnap) { wx = snap(wx, gridSnap); wy = snap(wy, gridSnap); }
  if (currentTool === "platform") platforms.push({ x: wx, y: wy, width: 100, height: 20 });
  if (currentTool === "hazard") hazards.push({ x: wx, y: wy, width: 40, height: 15 });
  if (currentTool === "collectible") collectibles.push({ x: wx, y: wy, width: 20, height: 20 });
  if (currentTool === "checkpoint") checkpoints.push({ x: wx, y: wy, width: 24, height: 28 });
});

// ==== DRAW FUNCTIONS ====
function drawPlatform(ctx, p) {
  ctx.fillStyle = "#666";
  ctx.fillRect(p.x, p.y, p.width, p.height);
  ctx.strokeStyle = "#333";
  ctx.strokeRect(p.x, p.y, p.width, p.height);
}
function drawHazard(ctx, h) {
  ctx.fillStyle = "#b43a3a";
  ctx.fillRect(h.x, h.y, h.width, h.height);
  ctx.strokeStyle = "#fff";
  ctx.beginPath();
  ctx.moveTo(h.x, h.y + h.height);
  ctx.lineTo(h.x + h.width / 2, h.y);
  ctx.lineTo(h.x + h.width, h.y + h.height);
  ctx.stroke();
}
function drawCollectible(ctx, c) {
  ctx.beginPath();
  ctx.arc(c.x + c.width / 2, c.y + c.height / 2, 9, 0, 2 * Math.PI);
  ctx.fillStyle = "#eeca3a";
  ctx.fill();
  ctx.strokeStyle = "#e5cc77";
  ctx.lineWidth = 2;
  ctx.stroke();
}
function drawCheckpoint(ctx, cp) {
  ctx.fillStyle = "#4ab3ed";
  ctx.fillRect(cp.x, cp.y, cp.width, cp.height);
  ctx.fillStyle = "#fff";
  ctx.font = "bold 11px sans-serif";
  ctx.fillText("✔", cp.x + 5, cp.y + 18);
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

// ==== GAME LOOP ====
function gameLoop() {
  ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

  if (mode === "play") {
    if (player1.dead) player1.respawn();
    if (player2.dead) player2.respawn();
    player1.update(platforms, hazards, collectibles, checkpoints);
    player2.update(platforms, hazards, collectibles, checkpoints);
    currentSplitMode = getSplitMode(player1, player2);

    if (currentSplitMode === "split") {
      // Left view
      ctx.save();
      ctx.beginPath(); ctx.rect(0, 0, VIEW_WIDTH, VIEW_HEIGHT); ctx.clip();
      let [cx1, cy1] = calcCamera(player1.x + player1.width/2, player1.y + player1.height/2);
      ctx.translate(-cx1, -cy1);
      drawWorld(ctx); player1.draw(ctx); player2.draw(ctx);
      ctx.restore();

      // Divider
      ctx.strokeStyle = "#222"; ctx.lineWidth = 4;
      ctx.beginPath(); ctx.moveTo(VIEW_WIDTH, 0); ctx.lineTo(VIEW_WIDTH, VIEW_HEIGHT); ctx.stroke();

      // Right view
      ctx.save();
      ctx.beginPath(); ctx.rect(VIEW_WIDTH, 0, VIEW_WIDTH, VIEW_HEIGHT); ctx.clip();
      let [cx2, cy2] = calcCamera(player2.x + player2.width/2, player2.y + player2.height/2);
      ctx.translate(-cx2 + VIEW_WIDTH, -cy2);
      drawWorld(ctx); player2.draw(ctx); player1.draw(ctx);
      ctx.restore();
    } else {
      ctx.save();
      let [cx, cy] = calcMergedCamera(player1, player2);
      ctx.translate(-cx, -cy);
      drawWorld(ctx); player1.draw(ctx); player2.draw(ctx);
      ctx.restore();
    }
  }

  if (mode === "create") {
    // Editor just shows world
    drawWorld(ctx);
  }

  requestAnimationFrame(gameLoop);
}
gameLoop();
</script>
</body>
</html>
