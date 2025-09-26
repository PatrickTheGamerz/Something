<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Tower Platformer Updated</title>
  <style>
    body { margin:0; background:#0e0f14; overflow:hidden; font-family:sans-serif; color:#fff; }
    canvas { display:block; margin:0 auto; background:#141725; }
    .ui-layer { position:fixed; inset:0; display:grid; place-items:center; pointer-events:none; }
    .panel { pointer-events:auto; background:#1b1f2e; border:1px solid #2a3147; border-radius:12px; padding:20px; width:720px; max-width:calc(100% - 40px); }
    .btn { background:#222637; border:1px solid #2a3147; color:#fff; border-radius:10px; padding:14px 16px; font-weight:600; cursor:pointer; margin:4px; }
    .btn:hover { background:#2a3147; }
    .toolbar { position:fixed; left:16px; top:16px; display:flex; gap:8px; background:#141725cc; border:1px solid #242a3c; border-radius:10px; padding:8px; z-index:10; }
    .tool { background:#222637; border:1px solid #2a3147; color:#fff; padding:8px 12px; border-radius:8px; cursor:pointer; }
    .tool.active { background:#2a3147; }
    .mobile-controls { position:fixed; inset:0; pointer-events:none; }
    .mc-btn { position:absolute; width:84px; height:84px; border-radius:50%; background:#2a3147; border:1px solid #3b4767; pointer-events:auto; display:flex; align-items:center; justify-content:center; color:#cfe2ff; font-weight:bold; font-size:24px; }
    .mc-btn:active { filter:brightness(1.2); }
  </style>
</head>
<body>
<canvas id="gameCanvas" width="1024" height="480"></canvas>

<!-- Menu -->
<div class="ui-layer" id="menuLayer">
  <div class="panel">
    <h1>Tower Platformer</h1>
    <button class="btn" id="btnPlay">Play</button>
    <button class="btn" id="btnCreate">Create</button>
    <button class="btn" id="btnSettings">Settings</button>
    <div id="playOptions" style="display:none;">
      <p>Choose Mode:</p>
      <button class="btn" id="btnMultiplayer">Multiplayer</button>
      <button class="btn" id="btnBotNormal">Bot: Normal</button>
      <button class="btn" id="btnBotTryhard">Bot: Tryhard</button>
      <button class="btn" id="btnBotMaster">Bot: Tower Master</button>
    </div>
  </div>
</div>

<!-- Editor toolbar -->
<div class="toolbar" id="editorToolbar" style="display:none;">
  <button class="tool" id="toolPlatform">Platform</button>
  <button class="tool" id="toolHazard">Hazard</button>
  <button class="tool" id="toolCollectible">Gem</button>
  <button class="tool" id="toolCheckpoint">Checkpoint</button>
  <button class="tool" id="toolExit">Exit</button>
</div>

<!-- Mobile controls -->
<div class="mobile-controls" id="mobileControls" style="display:none;">
  <!-- Player 1 -->
  <div class="mc-btn" id="mcP1Left" style="left:26px; bottom:28px;">◀</div>
  <div class="mc-btn" id="mcP1Right" style="left:126px; bottom:28px;">▶</div>
  <div class="mc-btn" id="mcP1Jump" style="left:76px; bottom:118px;">⤒</div>
  <!-- Player 2 -->
  <div class="mc-btn" id="mcP2Left" style="right:126px; bottom:28px;">◀</div>
  <div class="mc-btn" id="mcP2Right" style="right:26px; bottom:28px;">▶</div>
  <div class="mc-btn" id="mcP2Jump" style="right:76px; bottom:118px;">⤒</div>
</div>

<script>
// ==== CONFIG ====
const CANVAS_WIDTH=1024, CANVAS_HEIGHT=480, VIEW_WIDTH=CANVAS_WIDTH/2, VIEW_HEIGHT=CANVAS_HEIGHT;
const WORLD_WIDTH=1600, WORLD_HEIGHT=2400;
const SPLIT_DISTANCE=480, MERGE_DISTANCE=360;

// ==== STATE ====
let mode="menu"; // menu, play, create
let playMode="multiplayer"; // multiplayer, bot
let botDifficulty="normal"; // normal, tryhard, master
let currentSplitMode="merged";

// ==== INPUT ====
const keys={};
window.addEventListener("keydown",e=>keys[e.code]=true);
window.addEventListener("keyup",e=>keys[e.code]=false);

// ==== UTILS ====
function clamp(v,min,max){return Math.max(min,Math.min(v,max));}
function rectsCollide(a,b){return a.x<b.x+b.width&&a.x+a.width>b.x&&a.y<b.y+b.height&&a.y+a.height>b.y;}

// ==== PLAYER ====
class Player{
  constructor(x,y,color,controls){this.x=x;this.y=y;this.width=36;this.height=48;this.color=color;this.vx=0;this.vy=0;this.speed=5;this.jumpVelocity=-14;this.gravity=0.8;this.maxFall=18;this.onGround=false;this.controls=controls;this.dead=false;this.isBot=false;}
  get rect(){return{x:this.x,y:this.y,width:this.width,height:this.height};}
  update(platforms){
    if(this.isBot){this.botLogic(platforms);}else{
      let move=0;if(keys[this.controls.left])move-=1;if(keys[this.controls.right])move+=1;this.vx=move*this.speed;if(this.onGround&&keys[this.controls.jump]){this.vy=this.jumpVelocity;this.onGround=false;}
    }
    this.vy+=this.gravity;if(this.vy>this.maxFall)this.vy=this.maxFall;
    this.x+=this.vx;for(const p of platforms){if(rectsCollide(this.rect,p)){if(this.vx>0)this.x=p.x-this.width;else if(this.vx<0)this.x=p.x+p.width;}}
    this.y+=this.vy;this.onGround=false;for(const p of platforms){if(rectsCollide(this.rect,p)){if(this.vy>0){this.y=p.y-this.height;this.vy=0;this.onGround=true;}else if(this.vy<0){this.y=p.y+p.height;this.vy=0;}}}
    if(this.y>WORLD_HEIGHT)this.dead=true;
  }
  botLogic(platforms){
    // Simplified AI: jumps periodically, moves right
    this.vx=this.speed*0.8;
    if(this.onGround&&Math.random()<0.05){this.vy=this.jumpVelocity;this.onGround=false;}
    if(botDifficulty==="tryhard"&&this.onGround&&Math.random()<0.1){this.vy=this.jumpVelocity;}
    if(botDifficulty==="master"&&this.onGround){this.vy=this.jumpVelocity;}
  }
  draw(ctx){
    ctx.fillStyle=this.color;ctx.fillRect(this.x,this.y,this.width,this.height);
    ctx.fillStyle="#fff";ctx.fillRect(this.x+7,this.y+12,8,8);ctx.fillRect(this.x+21,this.y+12,8,8);
    ctx.fillStyle="#364a63";ctx.fillRect(this.x+10,this.y+16,3,3);ctx.fillRect(this.x+24,this.y+16,3,3);
  }
}

// ==== WORLD ====
let platforms = [
  // Permanent floor
  { x: 0, y: 2350, width: 1600, height: 50, floor: true },

  // Floor 1 platforms
  { x: 200, y: 2200, width: 200, height: 20 },
  { x: 500, y: 2100, width: 180, height: 20 },
  { x: 800, y: 2000, width: 200, height: 20 },

  // Floor 2 platforms
  { x: 300, y: 1850, width: 200, height: 20 },
  { x: 600, y: 1750, width: 180, height: 20 },
  { x: 950, y: 1650, width: 200, height: 20 },

  // Floor 3 platforms
  { x: 400, y: 1500, width: 200, height: 20 },
  { x: 700, y: 1400, width: 180, height: 20 },
  { x: 1000, y: 1300, width: 200, height: 20 },

  // Floor 4 platforms
  { x: 500, y: 1150, width: 200, height: 20 },
  { x: 800, y: 1050, width: 180, height: 20 },
  { x: 1100, y: 950, width: 200, height: 20 },

  // Floor 5 platforms (near end)
  { x: 600, y: 800, width: 200, height: 20 },
  { x: 900, y: 700, width: 180, height: 20 },
  { x: 1200, y: 600, width: 200, height: 20 }
];

let hazards = [
  { x: 550, y: 2335, width: 40, height: 15 },
  { x: 850, y: 1985, width: 40, height: 15 },
  { x: 700, y: 1585, width: 40, height: 15 },
  { x: 950, y: 1285, width: 40, height: 15 }
];

let collectibles = [
  { x: 220, y: 2170, width: 20, height: 20 },
  { x: 520, y: 2070, width: 20, height: 20 },
  { x: 820, y: 1970, width: 20, height: 20 },
  { x: 620, y: 1730, width: 20, height: 20 },
  { x: 1020, y: 1630, width: 20, height: 20 },
  { x: 720, y: 1380, width: 20, height: 20 },
  { x: 1120, y: 930, width: 20, height: 20 }
];

let checkpoints = [
  { x: 520, y: 2070, width: 24, height: 28 },
  { x: 620, y: 1730, width: 24, height: 28 },
  { x: 1020, y: 1630, width: 24, height: 28 },
  { x: 1120, y: 930, width: 24, height: 28 }
];

// Spawn marker (special, movable in create mode)
let spawnMarker = { x: 100, y: 2200, width: 28, height: 32 };

// End goal block at the top of the tower
let endGoal = { x: 1250, y: 500, width: 60, height: 60 };

// ==== DRAW FUNCTIONS (extended) ====
function drawPlatform(ctx, p) {
  ctx.save();
  const g = ctx.createLinearGradient(p.x, p.y, p.x, p.y + p.height);
  g.addColorStop(0, p.floor ? "#555" : "#6d7b91");
  g.addColorStop(1, p.floor ? "#333" : "#393f52");
  ctx.fillStyle = g;
  ctx.fillRect(p.x, p.y, p.width, p.height);
  ctx.strokeStyle = "#2a3147";
  ctx.strokeRect(p.x, p.y, p.width, p.height);
  ctx.restore();
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

function drawSpawnMarker(ctx, sp) {
  ctx.fillStyle = "#3aeeb4";
  ctx.fillRect(sp.x, sp.y, sp.width, sp.height);
  ctx.fillStyle = "#111";
  ctx.font = "bold 11px sans-serif";
  ctx.fillText("SP", sp.x + 4, sp.y + 20);
}

function drawEndGoal(ctx, eg) {
  ctx.fillStyle = "#ffd700";
  ctx.fillRect(eg.x, eg.y, eg.width, eg.height);
  ctx.fillStyle = "#000";
  ctx.font = "bold 14px sans-serif";
  ctx.fillText("END", eg.x + 8, eg.y + 35);
}

function drawWorld(ctx) {
  for (const plat of platforms) drawPlatform(ctx, plat);
  for (const hazard of hazards) drawHazard(ctx, hazard);
  for (const c of collectibles) drawCollectible(ctx, c);
  for (const cp of checkpoints) drawCheckpoint(ctx, cp);
  drawSpawnMarker(ctx, spawnMarker);
  drawEndGoal(ctx, endGoal);
}

// ==== CAMERA LOGIC ====
function getSplitMode(p1, p2) {
  const dx = (p1.x + p1.width/2) - (p2.x + p2.width/2);
  const dy = (p1.y + p1.height/2) - (p2.y + p2.height/2);
  const dist = Math.hypot(dx, dy);
  if (dist > SPLIT_DISTANCE) return "split";
  else if (dist < MERGE_DISTANCE) return "merged";
  else return currentSplitMode;
}
function calcCamera(px, py) {
  let camX = Math.round(px - VIEW_WIDTH/2);
  let camY = Math.round(py - VIEW_HEIGHT/2);
  camX = clamp(camX, 0, WORLD_WIDTH - VIEW_WIDTH);
  camY = clamp(camY, 0, WORLD_HEIGHT - VIEW_HEIGHT);
  return [camX, camY];
}
function calcMergedCamera(p1, p2) {
  let mx = (p1.x + p1.width/2 + p2.x + p2.width/2)/2;
  let my = (p1.y + p1.height/2 + p2.y + p2.height/2)/2;
  return calcCamera(mx, my);
}

// ==== PLAYERS ====
const player1 = new Player(100, 2200, "#23b4e4", { left:"KeyA", right:"KeyD", jump:"KeyW" });
const player2 = new Player(160, 2200, "#e4b423", { left:"ArrowLeft", right:"ArrowRight", jump:"ArrowUp" });

function resetPlayers() {
  player1.x = spawnMarker.x; player1.y = spawnMarker.y;
  player1.vx = player1.vy = 0; player1.dead = false;
  player2.x = spawnMarker.x+60; player2.y = spawnMarker.y;
  player2.vx = player2.vy = 0; player2.dead = false;
  player2.isBot = (playMode !== "multiplayer");
}

// ==== EDITOR INTERACTION ====
let dragObj = null;
canvas.addEventListener("mousedown", e=>{
  if(mode!=="create") return;
  const rect = canvas.getBoundingClientRect();
  const wx = e.clientX - rect.left;
  const wy = e.clientY - rect.top;
  // Check spawn marker first
  if(wx>=spawnMarker.x && wx<=spawnMarker.x+spawnMarker.width &&
     wy>=spawnMarker.y && wy<=spawnMarker.y+spawnMarker.height) {
    dragObj = spawnMarker;
    dragObj.offsetX = wx - spawnMarker.x;
    dragObj.offsetY = wy - spawnMarker.y;
    return;
  }
  // Otherwise place new
  if(currentTool==="platform") platforms.push({x:wx,y:wy,width:100,height:20});
  if(currentTool==="hazard") hazards.push({x:wx,y:wy,width:40,height:15});
  if(currentTool==="collectible") collectibles.push({x:wx,y:wy,width:20,height:20});
  if(currentTool==="checkpoint") checkpoints.push({x:wx,y:wy,width:24,height:28});
});
canvas.addEventListener("mousemove", e=>{
  if(mode!=="create"||!dragObj) return;
  const rect = canvas.getBoundingClientRect();
  dragObj.x = e.clientX - rect.left - dragObj.offsetX;
  dragObj.y = e.clientY - rect.top - dragObj.offsetY;
});
window.addEventListener("mouseup", ()=>{ dragObj=null; });

// ==== GAME LOOP ====
function gameLoop() {
  ctx.clearRect(0,0,CANVAS_WIDTH,CANVAS_HEIGHT);

  if(mode==="play") {
    if(player1.dead) resetPlayers();
    if(player2.dead && playMode==="multiplayer") resetPlayers();

    player1.update(platforms);
    if(playMode!=="multiplayer") player2.isBot=true;
    if(playMode!=="none") player2.update(platforms);

    currentSplitMode = (playMode==="multiplayer") ? getSplitMode(player1,player2) : "merged";

    if(playMode==="multiplayer" && currentSplitMode==="split") {
      // Left view
      ctx.save(); ctx.beginPath(); ctx.rect(0,0,VIEW_WIDTH,VIEW_HEIGHT); ctx.clip();
      let [cx1,cy1]=calcCamera(player1.x+player1.width/2,player1.y+player1.height/2);
      ctx.translate(-cx1,-cy1);
      drawWorld(ctx); player1.draw(ctx); player2.draw(ctx);
      ctx.restore();
      // Divider
      ctx.strokeStyle="#222"; ctx.lineWidth=4;
      ctx.beginPath(); ctx.moveTo(VIEW_WIDTH,0); ctx.lineTo(VIEW_WIDTH,VIEW_HEIGHT); ctx.stroke();
      // Right view
      ctx.save(); ctx.beginPath(); ctx.rect(VIEW_WIDTH,0,VIEW_WIDTH,VIEW_HEIGHT); ctx.clip();
      let [cx2,cy2]=calcCamera(player2.x+player2.width/2,player2.y+player2.height/2);
      ctx.translate(-cx2+VIEW_WIDTH,-cy2);
      drawWorld(ctx); player2.draw(ctx); player1.draw(ctx);
      ctx.restore();
    } else {
      ctx.save();
      let [cx,cy] = (playMode==="multiplayer") ? calcMergedCamera(player1,player2)
                                               : calcCamera(player1.x+player1.width/2,player1.y+player1.height/2);
      ctx.translate(-cx,-cy);
      drawWorld(ctx); player1.draw(ctx);
      if(playMode!=="none") player2.draw(ctx);
      ctx.restore();
    }
  }

  if(mode==="create") {
    drawWorld(ctx);
  }

  requestAnimationFrame(gameLoop);
}
gameLoop();
