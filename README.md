<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Tower Layout</title>
  <style>
    body { background: #223; margin: 0; overflow: hidden; }
    #towerCanvas { background: #444; display: block; margin: 0 auto; }
  </style>
</head>
<body>
<canvas id="towerCanvas" width="1024" height="480"></canvas>
<script>
// ==== MAP/LAYOUT DEFINITION ====

// Platforms (ground, steps, walls, tower structure)
const platforms = [
  { x: 0, y: 450, width: 2000, height: 50 },
  { x: 100, y: 370, width: 250, height: 20 },
  { x: 270, y: 300, width: 120, height: 16 },
  { x: 400, y: 250, width: 130, height: 16 },
  { x: 650, y: 320, width: 140, height: 18 },
  { x: 900, y: 390, width: 80, height: 18 },
  { x: 1100, y: 340, width: 150, height: 16 },
  { x: 1400, y: 290, width: 120, height: 18 },
  { x: 1500, y: 220, width: 100, height: 16 },
  { x: 650, y: 220, width: 18, height: 100 },
  { x: 1680, y: 100, width: 170, height: 18 },
];

// Hazards (spikes, traps)
const hazards = [
  { x: 500, y: 465, width: 36, height: 15 },
  { x: 1190, y: 465, width: 36, height: 15 },
  { x: 1520, y: 465, width: 40, height: 15 },
];

// Collectibles (coins, gems, etc.)
const collectibles = [
  { x: 170, y: 340, width: 20, height: 20 },
  { x: 450, y: 225, width: 20, height: 20 },
  { x: 680, y: 297, width: 20, height: 20 },
  { x: 1450, y: 267, width: 20, height: 20 },
  { x: 1710, y: 75, width: 20, height: 20 },
];

// Checkpoints (respawn flags)
const checkpoints = [
  { x: 330, y: 325, width: 24, height: 28 },
  { x: 1120, y: 308, width: 24, height: 28 },
  { x: 1515, y: 186, width: 24, height: 28 },
];

// ==== DRAW HELPERS ====
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
  ctx.lineTo(h.x + h.width/2, h.y);
  ctx.lineTo(h.x + h.width, h.y + h.height);
  ctx.stroke();
}

function drawCollectible(ctx, c) {
  ctx.beginPath();
  ctx.arc(c.x+c.width/2, c.y+c.height/2, 9, 0, 2*Math.PI);
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
  ctx.fillText("✔", cp.x+5, cp.y+18);
}

// ==== RENDER LOOP ====
const canvas = document.getElementById("towerCanvas");
const ctx = canvas.getContext("2d");

function render() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  for (const plat of platforms) drawPlatform(ctx, plat);
  for (const hazard of hazards) drawHazard(ctx, hazard);
  for (const c of collectibles) drawCollectible(ctx, c);
  for (const cp of checkpoints) drawCheckpoint(ctx, cp);

  requestAnimationFrame(render);
}

render();
</script>
</body>
</html>
























<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Two-Player Platformer (Players & Logic)</title>
  <style>
    body { background: #223; margin: 0; overflow: hidden; }
    #gameCanvas { background: #444; display: block; margin: 0 auto; }
  </style>
</head>
<body>
<canvas id="gameCanvas" width="1024" height="480"></canvas>
<script>
// ==== CONFIGURATION ====
const CANVAS_WIDTH = 1024;
const CANVAS_HEIGHT = 480;
const VIEW_WIDTH = CANVAS_WIDTH / 2;
const VIEW_HEIGHT = CANVAS_HEIGHT;
const WORLD_WIDTH = 2000;
const WORLD_HEIGHT = 1000;
const SPLIT_DISTANCE = 420;
const MERGE_DISTANCE = 340;

// ==== INPUT HANDLING ====
const keys = {};
window.addEventListener('keydown', e => { keys[e.code] = true; });
window.addEventListener('keyup', e => { keys[e.code] = false; });

// ==== UTILITY ====
function clamp(val, min, max) { return Math.max(min, Math.min(val, max)); }
function rectsCollide(a, b) {
  return (
    a.x < b.x + b.width &&
    a.x + a.width > b.x &&
    a.y < b.y + b.height &&
    a.y + a.height > b.y
  );
}

// ==== PLAYER CLASS ====
class Player {
  constructor(x, y, color, controls) {
    this.x = x; this.y = y;
    this.width = 36; this.height = 48;
    this.color = color;
    this.vx = 0; this.vy = 0;
    this.speed = 5.2;
    this.jumpVelocity = -13.7;
    this.maxFall = 17;
    this.gravity = 0.7;
    this.onGround = false;
    this.controls = controls;
    this.checkpoint = { x, y };
    this.collected = 0;
    this.dead = false;
  }
  get rect() { return { x: this.x, y: this.y, width: this.width, height: this.height }; }
  respawn() {
    this.x = this.checkpoint.x;
    this.y = this.checkpoint.y;
    this.vx = 0; this.vy = 0;
    this.dead = false;
  }
  update(platforms, hazards, collectibles, checkpoints) {
    // Input
    let move = 0;
    if (keys[this.controls.left]) move -= 1;
    if (keys[this.controls.right]) move += 1;
    this.vx = move * this.speed;

    if (this.onGround && keys[this.controls.jump]) {
      this.vy = this.jumpVelocity;
      this.onGround = false;
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
        if (this.vy > 0) {
          this.y = plat.y - this.height;
          this.vy = 0; this.onGround = true;
        } else if (this.vy < 0) {
          this.y = plat.y + plat.height;
          this.vy = 0;
        }
      }
    }

    // Bounds
    this.x = clamp(this.x, 0, WORLD_WIDTH - this.width);
    if (this.y > WORLD_HEIGHT) this.dead = true;

    // Hazards
    for (const h of hazards) if (rectsCollide(this.rect, h)) this.dead = true;

    // Collectibles
    for (let i = collectibles.length - 1; i >= 0; i--) {
      if (rectsCollide(this.rect, collectibles[i])) {
        collectibles.splice(i, 1);
        this.collected++;
      }
    }

    // Checkpoints
    for (const cp of checkpoints) {
      if (rectsCollide(this.rect, cp)) this.checkpoint = { x: cp.x, y: cp.y };
    }
  }
  draw(ctx) {
    ctx.fillStyle = this.color;
    ctx.fillRect(this.x, this.y, this.width, this.height);
    // Eyes
    ctx.fillStyle = "#fff";
    ctx.fillRect(this.x+7, this.y+12, 8, 8);
    ctx.fillRect(this.x+21, this.y+12, 8, 8);
    ctx.fillStyle = "#444";
    ctx.fillRect(this.x+10, this.y+16, 3, 3);
    ctx.fillRect(this.x+24, this.y+16, 3, 3);
  }
}

// ==== CAMERA LOGIC ====
function getSplitMode(p1, p2) {
  const dx = (p1.x+p1.width/2) - (p2.x+p2.width/2);
  const dy = (p1.y+p1.height/2) - (p2.y+p2.height/2);
  const dist = Math.hypot(dx, dy);
  if (dist > SPLIT_DISTANCE) return "split";
  else if (dist < MERGE_DISTANCE) return "merged";
  else return currentSplitMode;
}
function calcCamera(px, py) {
  let cx = Math.round(px - VIEW_WIDTH/2);
  let cy = Math.round(py - VIEW_HEIGHT/2);
  cx = clamp(cx, 0, WORLD_WIDTH - VIEW_WIDTH);
  cy = clamp(cy, 0, WORLD_HEIGHT - VIEW_HEIGHT);
  return [cx, cy];
}
function calcMergedCamera(p1, p2) {
  let mx = ((p1.x+p1.width/2)+(p2.x+p2.width/2))/2;
  let my = ((p1.y+p1.height/2)+(p2.y+p2.height/2))/2;
  return calcCamera(mx, my);
}

// ==== INITIALIZATION ====
const canvas = document.getElementById("gameCanvas");
const ctx = canvas.getContext("2d");

const player1 = new Player(100, 400, "#23b4e4", { left:"KeyA", right:"KeyD", jump:"KeyW" });
const player2 = new Player(160, 400, "#e4b423", { left:"ArrowLeft", right:"ArrowRight", jump:"ArrowUp" });

let allCollectibles = []; // will be filled from tower
let currentSplitMode = "merged";

// ==== HUD ====
function drawHud(player, x, y) {
  ctx.save();
  ctx.globalAlpha = 0.88;
  ctx.fillStyle = "#111";
  ctx.fillRect(x-10, y-28, 102, 44);
  ctx.globalAlpha = 1.0;
  ctx.fillStyle = "#fff";
  ctx.font = "18px monospace";
  ctx.fillText("Collected: " + player.collected, x, y);
  ctx.fillText("Checkpoint: " + Math.round(player.checkpoint.x) + "," + Math.round(player.checkpoint.y), x, y+24);
  ctx.restore();
}

// ==== GAME LOOP ====
function gameLoop() {
  if (player1.dead) player1.respawn();
  if (player2.dead) player2.respawn();

  player1.update(platforms, hazards, allCollectibles, checkpoints);
  player2.update(platforms, hazards, allCollectibles, checkpoints);

  currentSplitMode = getSplitMode(player1, player2);

  ctx.clearRect(0,0,CANVAS_WIDTH,CANVAS_HEIGHT);

  if (currentSplitMode === "split") {
    // Left viewport
    ctx.save();
    ctx.beginPath(); ctx.rect(0,0,VIEW_WIDTH,VIEW_HEIGHT); ctx.clip();
    let [cx1, cy1] = calcCamera(player1.x+player1.width/2, player1.y+player1.height/2);
    ctx.translate(-cx1, -cy1);
    drawWorld(ctx);
    player1.draw(ctx); player2.draw(ctx);
    ctx.restore();

    // Divider
    ctx.strokeStyle="#222"; ctx.lineWidth=4;
    ctx.beginPath(); ctx.moveTo(VIEW_WIDTH,0); ctx.lineTo(VIEW_WIDTH,VIEW_HEIGHT); ctx.stroke();

    // Right viewport
    ctx.save();
    ctx.beginPath(); ctx.rect(VIEW_WIDTH,0,VIEW_WIDTH,VIEW_HEIGHT); ctx.clip();
    let [cx2, cy2] = calcCamera(player2.x+player2.width/2, player2.y+player2.height/2);
    ctx.translate(-cx2 + VIEW_WIDTH, -cy2);

    drawWorld(ctx);
    player2.draw(ctx);
    player1.draw(ctx); // draw secondary for awareness
    ctx.restore();

    // HUD overlays
    drawHud(player1, 28, 36);
    drawHud(player2, VIEW_WIDTH + 28, 36);

  } else {
    // Merged view
    ctx.save();
    let [cx, cy] = calcMergedCamera(player1, player2);
    ctx.translate(-cx, -cy);

    drawWorld(ctx);
    player1.draw(ctx);
    player2.draw(ctx);
    ctx.restore();

    drawHud(player1, 28, 36);
    drawHud(player2, 28 + 240, 36);
  }

  requestAnimationFrame(gameLoop);
}

// ==== WORLD DRAW WRAPPER ====
// This assumes you’ve imported or inlined the Tower.js definitions
function drawWorld(ctx) {
  for (const plat of platforms) drawPlatform(ctx, plat);
  for (const hazard of hazards) drawHazard(ctx, hazard);
  for (const c of allCollectibles) drawCollectible(ctx, c);
  for (const cp of checkpoints) drawCheckpoint(ctx, cp);
}

// ==== START GAME ====
allCollectibles = collectibles.map(c => ({ ...c })); // clone from tower
gameLoop();
</script>
</body>
</html>
