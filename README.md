<!--
HTML5 Canvas JToH/EToH Ring 1 Tower 2-Player Platformer
Full code in a single HTML file with all towers, split-screen, mechanics, and features.

Usage: Save as .html and open in any modern browser.

NOTE: The code is thoroughly commented and organized for clarity and extensibility.
-->
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Ring 1 Tower Platformer - 2 Player</title>
<style>
  html, body {
    margin: 0; padding: 0;
    background: #1a1a1a;
    overflow: hidden;
    height: 100%;
    width: 100%;
    user-select: none;
  }
  body { width: 100vw; height: 100vh; }
  canvas { display: block; background: #292929; margin: 0 auto; }
</style>
</head>
<body>
<canvas id="game"></canvas>
<script>
// =============================
// SECTION 1. CORE CONFIGURATION
// =============================

// General canvas/game settings
const BASE_WIDTH = 1500;       // Logical canvas width
const BASE_HEIGHT = 900;       // Logical canvas height
const PLAYER_COUNT = 2;

// Key control mapping for both players
const PLAYER_CONTROLS = [
  // Player 1: WASD and Shift/Ctrl/e for interact
  {
    left: ['KeyA','ArrowLeft'],
    right: ['KeyD','ArrowRight'],
    up: ['KeyW','ArrowUp'],
    down: ['KeyS','ArrowDown'],
    jump: ['Space','Numpad0'],
    interact: ['ShiftLeft','ShiftRight','KeyE']
  },
  // Player 2: Arrows and Enter/Slash/RightCtrl for interact
  {
    left: ['ArrowLeft','KeyJ'],
    right: ['ArrowRight','KeyL'],
    up: ['ArrowUp','KeyI'],
    down: ['ArrowDown','KeyK'],
    jump: ['Numpad0','Slash','Enter'],
    interact: ['ControlRight','Enter']
  }
];

// CAMERA: Split when players are far; merge when on same floor and close
const SPLIT_DIST = 300;                     // Distance threshold for split
const CAMERA_MARGIN = 50;                   // Margin for camera floor focusing

// Player/physics constants
const PLAYER_WIDTH = 36;
const PLAYER_HEIGHT = 52;
const PLAYER_SPEED = 4.0;
const PLAYER_JUMP = 12.4;
const GRAVITY = 0.52;
const MAX_VEL_X = 6.8;
const MAX_VEL_Y = 16.0;

const REGEN_TIME = 90;                      // Frames until regen (1.5s at 60 fps)
const REGEN_AMOUNT = 8;                     // HP per regen event
const LAVA_DAMAGE = 5;                      // Damage per lava touch

// EFFECTS
const EFFECTS = {
  none:       { label: '',                color:'#fff',   icon:'', apply: p=>{} },
  slow:       { label: 'SLOW',            color:'#9cf',   icon:'🐢', apply: p=>{p.speedMult=0.5;} },
  highjump:   { label: 'HIGH JUMP',       color:'#fc9',   icon:'🦘', apply: p=>{p.jumpMult=1.8;} },
  nojump:     { label: 'NO JUMP',         color:'#c30',   icon:'🚫', apply: p=>{p.canJump=false;} }
};

// Platform colors
const PLATFORM_COLORS = {
  default: '#bbbbbb',
  lava: '#ee4444',
  moving: '#fcf87c',
  spinning: '#7cffde',
  button: '#7caff6',
  pushable: '#ffb763',
  win: '#53f364'
};

// Font settings
const FONT_MAIN = '16px Arial, sans-serif';
const FONT_TITLE = '700 34px Arial, sans-serif';
const FONT_HUD = '700 20px Arial, sans-serif';

// Map scaling and floor settings
const MAP_FLOOR_HEIGHT = 160;               // Logical height per floor
const MAP_FLOOR_PAD = 28;                   // Padding above last platform and below first
const PLATFORM_SIZE = {
  w: 90, h: 22,
  smallW: 38, smallH: 15
};
const MIN_PLATFORMS_PER_FLOOR = 30;

const CUTSCENE_TIME = 140;                  // Frames for cutscene

// ==============================
// SECTION 2. DATA: TOWERS, FLOORS
// ==============================

// Define all towers: floor count, color per floor, basic layout seed/metadata
const TOWER_LIST = [
  {
    id: 'ToAST',
    name: 'Tower of A Simple Time',
    floors: 10,
    colors: ['#f3e6b7','#ffeedf','#fee3b8','#f6be9b','#d1e3e6','#bddddb',
      '#93abc7','#efedd9','#eebcbc','#fffbbe']
  },
  {
    id: 'ToA',
    name: 'Tower of Anger',
    floors: 10,
    colors: ['#f98634','#f9e01a','#f4953f','#f2661d','#c62828','#8b3529','#cf3404','#bb9c24','#f4c75b','#830a0a']
  },
  {
    id: 'ToM',
    name: 'Tower of Madness',
    floors: 10,
    colors: ['#47d066','#33c489','#28cbd0','#16aadb','#3997b8','#699be6','#346b93','#2b86a0','#2bae8e','#c5fff7']
  },
  {
    id: 'ToNI',
    name: 'Tower of Noticeable Infuriation',
    floors: 10,
    colors: ['#fc8add','#f99cac','#eb3dc2','#a158bb','#913967','#e66dd7','#be36b8','#d28add','#bb2d9b','#ecb1fc']
  },
  {
    id: 'ToH',
    name: 'Tower of Hecc',
    floors: 10,
    colors: ['#c44646','#ff9268','#ac3c1c','#db1b1b','#f4905b','#c62f2f','#ec653b','#feaa6d','#d42549','#bb3333']
  },
  {
    id: 'ToK',
    name: 'Tower of Keyboard Yeeting',
    floors: 10,
    colors: ['#a6e4ff','#eacfff','#fff77c','#f3ae91','#f1e2ff','#69c2ff','#ff6e66','#ffd6c2','#b6ebff','#c3ecfa']
  },
  {
    id: 'ToKY',
    name: 'Tower of Keyboard Yeeting',
    floors: 10,
    colors: ['#a6e4ff','#eacfff','#fff77c','#f3ae91','#f1e2ff','#69c2ff','#ff6e66','#ffd6c2','#b6ebff','#c3ecfa']
  },
  {
    id: 'ToS',
    name: 'Tower of Stress',
    floors: 10,
    colors: ['#a4a6eb','#747bce','#8090e6','#6c8593','#6370ab','#5161a0','#aab0cc','#7874c4','#c6d0ea','#9199cf']
  },
  {
    id: 'ToSP',
    name: 'Tower of Screen Punching',
    floors: 9,
    colors: ['#b5baff','#7dc4e6','#f7e9df','#beeab3','#b294e6','#e0e0e0','#c3e4f3','#97c3f3','#e7c4ef']
  },
  {
    id: 'ToR',
    name: 'Tower of Rage',
    floors: 10,
    colors: ['#da2022','#f2c311','#e26100','#e6973b','#eb4749','#c42424','#f95921','#8b1616','#fdca44','#ce292a']
  },
  {
    id: 'ToIE',
    name: 'Tower of Impossible Expectations',
    floors:10,
    colors: ['#323232','#4f524e','#6e6e6e','#222222','#cce7fa','#161616','#e2e2e2','#c7d3cc','#8cbac7','#404288']
  },
  {
    id: 'ToTS',
    name: 'Tower of True Skill',
    floors:10,
    colors: ['#fa2121','#ffbc11','#24ff39','#0fd8ff','#3311ff','#f311ea','#ff9320','#00fafc','#fd49a2','#7d7dfa']
  },
  {
    id: 'TT',
    name: 'Thanos Tower',
    floors:10,
    colors: ['#621ce9','#402582','#8372be','#1b193e','#7158be','#242348','#cfc9ef','#8732cf','#5033e6','#4092cf']
  },
  {
    id: 'CoLS',
    name: 'Citadel of Laptop Splitting',
    floors:15,
    colors:
        [
          '#FFFF00','#F5CD30','#A05F35','#12EED4','#27462D','#F8F8F8',
          '#FFC9C9','#04AFEC','#1B2A35','#FF00BF','#00FF00', '#FF0000',
          '#0010B0','#F8D96D','#000014'
        ]
  }
];

// ====================
// SECTION 3. UTILITIES
// ====================

// Clamp a value between min and max
function clamp(v, min, max) { return Math.max(min, Math.min(max, v)); }

// Random integer in [a, b)
function randInt(a, b) { return Math.floor(Math.random() * (b - a)) + a; }

// Ease in-out quadratic for cutscene transitions
function easeInOutQuad(t) {
  return t < 0.5 ? 2*t*t : -1+(4-2*t)*t;
}

// Color interpolation helper
function lerpColor(a, b, t) {
  const ah = /^#([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(a);
  const bh = /^#([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(b);
  if (!ah || !bh) return a;
  const [ar,ag,ab] = [parseInt(ah[1],16),parseInt(ah[2],16),parseInt(ah[3],16)];
  const [br,bg,bb] = [parseInt(bh[1],16),parseInt(bh[2],16),parseInt(bh[3],16)];
  const r = Math.round(ar + (br-ar)*t);
  const g = Math.round(ag + (bg-ag)*t);
  const b = Math.round(ab + (bb-ab)*t);
  return `rgb(${r},${g},${b})`;
}

// =========================
// SECTION 4. GAME ENTITIES
// =========================

// Classes for Platform, Special Objects, Button, etc.

class Platform {
  constructor(x, y, w, h, type='solid', meta={}) {
    this.x = x; this.y = y;
    this.w = w; this.h = h;
    this.type = type; // 'solid','lava','moving','spinning','win','button','pushable'
    this.meta = meta; // {dir, speed, ...}
    this.baseX = x;
    this.baseY = y;
  }
  // For moving platforms: update position
  update(tick) {
    if (this.type === 'moving') {
      // Looping 1D movement (horizontal/vertical)
      let range = this.meta.range || 100;
      let speed = this.meta.speed || 1.6;
      let axis = this.meta.axis || 'x';
      let phase = ((tick/60)*speed+this.meta.seed)%2;
      let disp = Math.sin(phase * Math.PI) * range;
      if (axis === 'x') this.x = this.baseX + disp;
      else this.y = this.baseY + disp;
    } else if (this.type === 'spinning') {
      // Rotate around center
      let r = this.meta.radius||58;
      let s = this.meta.speed||0.9;
      let ang = ((tick/60)*s+this.meta.seed) * Math.PI * 2;
      this.x = this.baseX + Math.cos(ang)*r;
      this.y = this.baseY + Math.sin(ang)*r;
    }
    // For pushable, handled elsewhere (by collisions)
  }
  // Render platform
  render(ctx, color) {
    ctx.save();
    ctx.fillStyle = color||PLATFORM_COLORS.default;
    ctx.strokeStyle = '#202028';
    ctx.globalAlpha = 1.0;
    if (this.type==='lava') ctx.globalAlpha = 0.85;
    ctx.fillRect(this.x, this.y, this.w, this.h);
    ctx.strokeRect(this.x, this.y, this.w, this.h);
    // Special indicators
    if (this.type==='moving') {
      ctx.globalAlpha = 0.6;
      ctx.fillStyle = '#787800';
      ctx.fillRect(this.x, this.y, this.w, 4);
    }
    if (this.type==='win') {
      ctx.globalAlpha = 0.55;
      ctx.fillStyle = '#b9ff6c';
      ctx.fillRect(this.x, this.y, this.w, this.h);
      ctx.globalAlpha = 1.0;
      ctx.strokeStyle = '#3d7f29';
      ctx.lineWidth = 3;
      ctx.strokeRect(this.x+2, this.y+2, this.w-4, this.h-4);
    }
    if (this.type==='button') {
      ctx.globalAlpha = 0.8;
      ctx.fillStyle = '#3cf';
      ctx.fillRect(this.x+this.w/4, this.y, this.w/2, this.h*0.7);
      ctx.globalAlpha = 1.0;
      ctx.fillStyle = '#fff';
      ctx.fillRect(this.x+this.w/2-6,this.y+5, 12,9);
    }
    if (this.type==='pushable') {
      ctx.globalAlpha = 0.82;
      ctx.fillStyle = '#b96e05';
      ctx.fillRect(this.x, this.y, this.w, this.h);
      ctx.globalAlpha = 1.0;
      ctx.strokeRect(this.x, this.y, this.w, this.h);
    }
    ctx.restore();
  }
}

// Effect timer for slow, high jump etc.
class TimedEffect {
  constructor(type, duration) {
    this.type = type; this.duration = duration;
  }
  tick() { this.duration--; }
  isActive() { return this.duration > 0; }
}

// Player class
class Player {
  constructor(id, name, color, spawn) {
    this.id = id;
    this.name = name;
    this.color = color;
    this.spawn = Object.assign({},spawn);

    this.x = spawn.x; this.y = spawn.y;
    this.vx = 0; this.vy = 0;
    this.dir = 1;
    this.width = PLAYER_WIDTH; this.height = PLAYER_HEIGHT;
    this.hp = 100;
    this.grounded = false;
    this.jumpCool = 0;
    this.alive = true;

    this.floor = 0;           // Updated each frame
    this.lastSafeY = spawn.y;
    this.regenTimer = 0;
    this.timer = 0;           // Timer: frames since round started
    this.hasWon = false;
    this.completionTime = null;

    this.effect = null;       // TimedEffect
    this.speedMult = 1.0;
    this.jumpMult = 1.0;
    this.canJump = true;
  }
  respawn() {
    Object.assign(this, {
      x:this.spawn.x, y:this.spawn.y,
      vx:0, vy:0, hp:100, alive:true, effect:null,
      speedMult:1.0, jumpMult:1.0, canJump:true,
      hasWon:false, completionTime:null
    });
  }
  applyEffect(effectType, duration) {
    this.effect = new TimedEffect(effectType, duration);
    // Reset multipliers and statuses; effect will be re-applied in update
    this.speedMult = 1.0; this.jumpMult = 1.0; this.canJump = true;
  }
  updateEffect() {
    this.speedMult = 1.0; this.jumpMult = 1.0; this.canJump = true;
    if (this.effect && this.effect.isActive()) {
      EFFECTS[this.effect.type].apply(this);
      this.effect.tick();
      if (!this.effect.isActive()) { this.effect = null; }
    }
  }
}

// =============================
// SECTION 5. MAP GENERATION LOGIC
// =============================

// Platform layout generator for arbitrary floor/platform counts
function genTowerLayout(towerInfo, seed) {
  // Deterministic PRNG for reproducibility (using seed)
  function seededRandom() {
    let t = Math.sin(seed++) * 10000;
    return t - Math.floor(t);
  }

  let floors = [];
  let totalFloors = towerInfo.floors;
  let towerW = BASE_WIDTH/2.7;
  let x0 = BASE_WIDTH/2 - towerW/2;
  let platH = PLATFORM_SIZE.h;
  let platW = PLATFORM_SIZE.w;
  let floorH = MAP_FLOOR_HEIGHT; // Height per floor
  let pad = MAP_FLOOR_PAD;

  // WIN platform at the very top of last floor
  let winpadY = pad + (totalFloors-1)*floorH - platH - 38;

  for (let f=0; f<totalFloors; ++f) {
    let py = BASE_HEIGHT - pad - (f+1) * floorH + 40;
    let color = towerInfo.colors[f % towerInfo.colors.length];
    let platforms=[];
    // Central large entry platform
    platforms.push(new Platform(x0 + towerW/2 - platW/2, py+floorH-platH-18, platW, platH,'solid'));
    // Scatter random & mechanic platforms using seed
    let n = Math.max(MIN_PLATFORMS_PER_FLOOR, 32 + Math.floor(8*seededRandom()));
    for (let i=0; i<n; ++i) {
      let t = (i+f*3)*1.13 + seededRandom();
      // Alternate between edge and center
      let bx = x0 + 44 + (towerW-88-platW) * seededRandom();
      // Stagger platform height
      let by = py+floorH-platH-30 - floorH * seededRandom() * 0.89;
      let type = 'solid';
      // Place obstacles and mechanics pseudo-randomly
      if (i%15===2 && f>0) type='lava';
      if ((i%22)===7 && f%3===0) type='moving';
      if ((i+f)%14===6) type='spinning';
      if ((i+f)%17===5 && f>1) type='pushable';
      if ((i+f)%25===3 && i>10 && i<24) type='button';
      // Buttons as powerups, not keys, on random platforms
      let meta={};
      if (type==='moving') {
        meta.axis = i%2===0?'x':'y';
        meta.range = 80+seededRandom()*70;
        meta.speed = 0.8+0.7*seededRandom();
        meta.seed = 0.7*f + i/23;
      }
      if (type==='spinning') {
        meta.radius = 32+seededRandom()*38;
        meta.speed = 0.4 + seededRandom();
        meta.seed = seed/743.2 + i/13 + f/1.3;
      }
      // Place win at top center
      if (f === totalFloors-1 && i===Math.floor(n/2)) {
        type = 'win';
        bx = x0 + towerW/2 - platW/2; by = winpadY;
      }
      // Prevent win/portal in low floors
      platforms.push(new Platform(bx, by, platW, platH, type, meta));
    }
    floors.push({color, platforms});
  }

  return {
    floors
  };
}
// Helper: get the floor index for a given y coordinate
function floorAtY(y, totalFloors, pad=MAP_FLOOR_PAD, floorH=MAP_FLOOR_HEIGHT) {
  let fy = BASE_HEIGHT - y - pad;
  let fi = Math.floor(fy / floorH);
  if (fi<0) fi=0; if (fi>=totalFloors) fi=totalFloors-1;
  return fi;
}

// ==============================
// SECTION 6. INPUT MANAGEMENT
// ==============================

const keyState = {};
window.addEventListener('keydown',e=>{
  keyState[e.code]=true;
});
window.addEventListener('keyup',e=>{
  keyState[e.code]=false;
});

// ==============================
// SECTION 7. GAME STATE MACHINE
// ==============================

// Game states: 'cutscene','playing','won','reset'
let gameState = {
  round:0,
  state:'cutscene',
  cutsceneT:0,
  tower:null,
  towerData:null,
  players:[],
  winnerId:null,
  winnerTime:null,
  split:true,
  lastSplitState:null
};

// Initialize or reset a new round
function newRound() {
  // Random tower
  let towerIx = randInt(0, TOWER_LIST.length);
  let tower = TOWER_LIST[towerIx];
  let seed = Math.floor(Math.random()*100000);
  let towerData = genTowerLayout(tower, seed);

  // Compute player spawns: first floor, left and right
  let platforms = towerData.floors[0].platforms;
  let leftSpawn=platforms[0];
  let rightSpawn=platforms[Math.floor(platforms.length/3)];
  let spawns = [
    {x:leftSpawn.x+leftSpawn.w/2-PLAYER_WIDTH, y:leftSpawn.y-PLAYER_HEIGHT-7},
    {x:rightSpawn.x+rightSpawn.w/2, y:rightSpawn.y-PLAYER_HEIGHT-7}
  ];

  // Create players
  let colors=['#39d27a','#25a1d2','#e239d2','#c97218'];
  let names=['PLAYER 1','PLAYER 2','PLAYER 3','PLAYER 4'];
  let players=[];
  for (let i=0; i<PLAYER_COUNT; ++i)
    players.push(new Player(i, names[i], colors[i%colors.length], spawns[i]));

  Object.assign(gameState, {
    round:gameState.round+1,
    state:'cutscene',
    cutsceneT:0,
    tower,
    towerIx,
    towerData,
    players,
    winnerId:null,
    winnerTime:null,
    split:true
  });
}
newRound();

// ==============================
// SECTION 8. CANVAS SETUP
// ==============================

const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
// Responsive scaling
function resizeCanvas() {
  const w = window.innerWidth, h = window.innerHeight;
  let scale = Math.min(w/BASE_WIDTH, h/BASE_HEIGHT);
  canvas.width=BASE_WIDTH*scale;
  canvas.height=BASE_HEIGHT*scale;
  canvas.style.width = w+'px';
  canvas.style.height = h+'px';
  ctx.setTransform(scale,0,0,scale,0,0);
}
window.addEventListener('resize',resizeCanvas);
resizeCanvas();

// ==============================
// SECTION 9. MAIN GAME LOOP
// ==============================

let tick = 0;
function mainLoop() {
  tick++;
  update();
  render();
  window.requestAnimationFrame(mainLoop);
}
mainLoop();

// =======================
// SECTION 10. UPDATE LOGIC
// =======================

function update() {
  let state=gameState;
  if (state.state==='cutscene') {
    state.cutsceneT++;
    if (state.cutsceneT > CUTSCENE_TIME)
      state.state='playing';
  } else if (state.state==='playing') {
    // Update each player with controls
    for (let i=0;i<PLAYER_COUNT;++i)
      updatePlayer(state.players[i], i, state);
    // Check win condition
    for (let i=0;i<PLAYER_COUNT;++i) {
      let p=state.players[i];
      if (!p.hasWon && checkPlayerWin(p,state.towerData)) {
        p.hasWon=true;
        p.completionTime=p.timer;
        if (!state.winnerId) {
          state.winnerId = p.id;
          state.winnerTime = p.completionTime;
          setTimeout(()=>{ state.state='won'; },600);
        }
      }
    }
    // Split or merge camera view
    let f0 = state.players[0].floor, f1 = state.players[1].floor;
    let y0 = state.players[0].y, y1 = state.players[1].y;
    let d = Math.abs(y0-y1)+Math.abs(f0-f1);
    state.lastSplitState = state.split;
    if (d<MAP_FLOOR_HEIGHT/2 && f0===f1)
      state.split = false;
    else
      state.split = true;
  } else if (state.state==='won') {
    // Wait for Enter to start new round; (or space)
    if (keyState['Enter']||keyState['Space']) {
      newRound();
    }
  }
}

// ===========================
// SECTION 11. PLAYER HANDLING
// ===========================

function updatePlayer(p, pid, state) {
  if (!p.alive) return;

  // Effect ticks
  p.updateEffect();

  // Increment timer if not won
  if (!p.hasWon) p.timer++;

  // Find current floor
  p.floor = floorAtY(p.y+PLAYER_HEIGHT/2, state.tower.floors);

  // Input: controls
  const ctl = PLAYER_CONTROLS[pid];
  let moveL=ctl.left.some(c=>keyState[c]);
  let moveR=ctl.right.some(c=>keyState[c]);
  let jump=ctl.jump.some(c=>keyState[c]);
  let interact=ctl.interact.some(c=>keyState[c]);

  let spMult=p.speedMult||1.0;
  let jMult=(p.canJump===false)?0:p.jumpMult||1.0;

  // Horizontal movement
  if (moveL && !moveR) { p.vx -= 0.6*spMult; p.dir = -1; }
  else if (moveR && !moveL) { p.vx += 0.6*spMult; p.dir = 1; }
  else { // Friction
    if (p.grounded) p.vx*=0.78;
    else p.vx*=0.92;
    if (Math.abs(p.vx)<0.14) p.vx=0;
  }
  p.vx = clamp(p.vx, -MAX_VEL_X*spMult, MAX_VEL_X*spMult);

  // Jump: only if grounded and jump enabled
  if (p.jumpCool>0) p.jumpCool--;
  if (jump && p.grounded && p.canJump!==false && p.jumpCool===0) {
    p.vy = -PLAYER_JUMP*jMult;
    p.grounded = false; p.jumpCool=14;
  }

  // Gravity
  p.vy += GRAVITY;
  p.vy = clamp(p.vy, -MAX_VEL_Y, MAX_VEL_Y);

  // Predicted movement; test collisions; do up to 2 steps for precise collision
  let nstep=2;
  for (let si=0;si<nstep;++si) {
    let dx=p.vx/nstep, dy=p.vy/nstep;
    let collided = false, collisionY=0;

    // "Hit platforms": all visible floors within 2
    let visFloors = [];
    let f = p.floor;
    let tdata = state.towerData.floors;
    for (let fi=Math.max(0,f-2);fi<=Math.min(tdata.length-1,f+2);++fi)
      visFloors = visFloors.concat(tdata[fi].platforms);
    // Move pushables before players
    let pushPlatforms = visFloors.filter(pl=>pl.type==='pushable');
    for (const plat of visFloors) plat.update(tick);

    // Axis-aligned bbox
    let nx=p.x+dx, ny=p.y+dy;
    let bbox = {x:nx,y:ny,w:p.width,h:p.height};
    p.grounded=false;

    // Lava, win, moving, spinning, button, solid, pushable all handled
    let onAny = false;
    for (const platform of visFloors) {
      if (AABB(bbox, platform)) {
        // Lava hurts
        if (platform.type==='lava') {
          p.hp = Math.max(0, p.hp-LAVA_DAMAGE);
          p.regenTimer = 0;
        }
        // Win pad: flag
        if (platform.type==='win' && !p.hasWon) {
          p.hasWon=true; p.completionTime=p.timer;
        }
        // Moving/spinning act as regular platforms
        if (['solid','moving','spinning','win','pushable','button'].includes(platform.type)) {
          // Check if standing on: land via Y axis
          if (p.y+ p.height <= platform.y + 8 && dy>0) {
            ny = platform.y - p.height;
            p.vy = 0;
            p.grounded = true;
            p.lastSafeY=platform.y - p.height;
            onAny = true;
          }
          // Hit head from below: kill upward velocity
          else if (p.y >= platform.y + platform.h - 9 && dy<0) {
            ny = platform.y + platform.h +0.2;
            p.vy = 0.4;
          }
          // Side collision: block horizontal
          else {
            if (dx>0) nx = platform.x - p.width-0.2;
            else if (dx<0) nx = platform.x + platform.w +0.2;
            p.vx = 0.1*p.vx;
          }
        }
        // Buttons: apply random effect (simulate powerups)
        if (platform.type==='button' && interact) {
          let effTypes = ['slow','highjump','nojump'];
          let e = effTypes[ (tick+Math.floor(nx+ny))%3 ];
          p.applyEffect(e, 180+Math.floor(50*Math.random()));
        }
        // Pushable: push if moving towards
        if (platform.type==='pushable' && Math.abs(dx)>0.1) {
          platform.x += dx*0.6;
        }
      }
    }
    p.x=nx; p.y=ny;
  }

  // Push player back if falling out of map
  if (p.y > BASE_HEIGHT+120) {
    // Respawn or drop to first platform?
    p.x = p.spawn.x; p.y = p.spawn.y; p.hp -= 20;
    p.vx=0; p.vy=-8; if (p.hp<1) {p.alive=false;}
  }

  // Cap within map
  p.x = clamp(p.x, 30, BASE_WIDTH-30-p.width);

  // Health regeneration
  p.regenTimer = p.regenTimer+1;
  if (p.regenTimer > REGEN_TIME && p.hp<100) {
    p.hp = clamp(p.hp+REGEN_AMOUNT,0,100); p.regenTimer=0;
  }
  // Death
  if (p.hp<=0) { p.alive=false; }
}

// AABB collision between entity and platform
function AABB(a, b) {
  return a.x+a.w > b.x && a.x < b.x+b.w && a.y+a.h > b.y && a.y < b.y+b.h;
}

function checkPlayerWin(player, towerData) {
  // On any win pad on top floor
  let plat = towerData.floors[towerData.floors.length-1].platforms;
  for (const p of plat) if (p.type==='win' && AABB(player, p))
    return true;
  return false;
}

// =========================
// SECTION 12. RENDERING
// =========================

function render() {
  ctx.clearRect(0,0,BASE_WIDTH,BASE_HEIGHT);

  // Cutscene camera: pan and zoom-in/out
  if (gameState.state==='cutscene') {
    renderCutscene();
    return;
  }

  // Playing: split or shared
  if (gameState.split) {
    // Split horizontally
    renderCameraHalf(gameState.players[0], 0, 0, BASE_WIDTH, BASE_HEIGHT/2, false, 0);
    renderCameraHalf(gameState.players[1], 0, BASE_HEIGHT/2, BASE_WIDTH, BASE_HEIGHT/2, false, 1);
    // Split line
    ctx.save();
    ctx.fillStyle='#222'; ctx.globalAlpha=0.7;
    ctx.fillRect(0,BASE_HEIGHT/2-2, BASE_WIDTH,4);
    ctx.globalAlpha=1.0; ctx.restore();
  } else {
    // Single merged camera—centered on mean Y/floor of both players
    renderCameraHalf(gameState.players[0],0,0,BASE_WIDTH,BASE_HEIGHT,false,0,gameState.players[1]);
  }

  // If won, overlay winner information
  if (gameState.state==='won')
    renderWinScreen();
}

// Draw one camera's view, following given player (optionally both for single camera)
function renderCameraHalf(player, x,y,w,h, full, pid,otherPlayer) {
  ctx.save();
  ctx.beginPath();
  ctx.rect(x,y,w,h); // Clip to half
  ctx.clip();

  // Center camera: focus on current floor, keep player visible
  let camY = clamp(player.y-90, 0, BASE_HEIGHT-MAP_FLOOR_HEIGHT*1.2);
  // If two players and single camera, center between
  if (otherPlayer) {
    camY = Math.min(player.y, otherPlayer.y)-90;
  }

  ctx.translate(-0, -camY + y);

  // Only draw relevant floor range (+/- 2 of current)
  let f = player.floor;
  for (let fi=Math.max(0,f-2);fi<=Math.min(gameState.tower.floors-1,f+3);++fi) {
    let floor = gameState.towerData.floors[fi];
    let c = floor.color;
    // Floor background
    ctx.save();
    ctx.fillStyle=c;
    ctx.globalAlpha=0.18;
    ctx.fillRect(0, BASE_HEIGHT-(fi+1)*MAP_FLOOR_HEIGHT, BASE_WIDTH, MAP_FLOOR_HEIGHT);
    ctx.globalAlpha=1.0;
    ctx.restore();
    // Platforms
    for (const plat of floor.platforms) {
      let pc = PLATFORM_COLORS[plat.type] || PLATFORM_COLORS.default;
      plat.render(ctx, pc);
    }
  }
  // Win pad sparkles
  if (player.hasWon||player.completionTime)
    drawWinSparkles(ctx, player);

  // Player and co
  drawPlayer(ctx,player,0);
  if (otherPlayer && otherPlayer.id!=player.id)
    drawPlayer(ctx,otherPlayer,0.6);

  // HUD
  ctx.save();
  // Health bar
  ctx.globalAlpha=0.95;
  let hpw = 170, hph = 20;
  ctx.fillStyle='#1c2224';
  ctx.fillRect(12, camY+10, hpw, hph);
  ctx.fillStyle='#39e35a';
  ctx.fillRect(12, camY+10, hpw*(player.hp/100), hph);
  ctx.strokeStyle='#cdd'; ctx.strokeRect(12, camY+10, hpw, hph);
  ctx.font=FONT_HUD; ctx.fillStyle='#fff';
  ctx.fillText(player.name+': '+Math.round(player.hp),12,camY+28+8);
  // Timer
  let t = (player.timer||0)/60;
  ctx.fillText('Time: '+t.toFixed(2)+'s', 16,camY+60);
  // Floor
  ctx.fillText('Floor: ' + (player.floor+1), 16, camY+92);
  // Effect
  if (player.effect) {
    ctx.fillStyle=EFFECTS[player.effect.type].color||'#fff';
    ctx.fillText(EFFECTS[player.effect.type].label+' '+EFFECTS[player.effect.type].icon,16,camY+124);
  }
  ctx.restore();

  ctx.restore();
}

// Cutscene camera: pan from tower name, zoom into win pad, then out
function renderCutscene() {
  // Phases: [0-0.33] tower name fade in; [0.33-0.55] zoom to win pad; [0.55-1.0] zoom out to players
  let t = clamp(gameState.cutsceneT/CUTSCENE_TIME,0,1);
  let phase=0, phaseT=0;
  if (t < 0.33) { phase=0; phaseT=t/0.33; }
  else if (t < 0.55) { phase=1; phaseT=(t-0.33)/0.22; }
  else { phase=2; phaseT=(t-0.55)/0.45; }

  ctx.save();
  // Camera trajectories
  if (phase===0) {
    // Tower name fade in
    ctx.fillStyle='#1a1729'; ctx.fillRect(0,0,BASE_WIDTH,BASE_HEIGHT);
    ctx.font = FONT_TITLE;
    ctx.fillStyle='#fff';
    ctx.globalAlpha=phaseT;
    ctx.fillText(gameState.tower.name, BASE_WIDTH/2-180, 120+phaseT*18);
    ctx.globalAlpha=1.0;
  } else if (phase===1) {
    // Zoom in to win pad
    let wpad = findWinPad(gameState.towerData);
    let focusY = wpad?(wpad.y-170):80;
    let scale = 1.5-easeInOutQuad(phaseT)*1.05;
    ctx.save();
    ctx.translate(BASE_WIDTH/2, BASE_HEIGHT/2);
    ctx.scale(scale,scale);
    ctx.translate(-BASE_WIDTH/2, -focusY);
    renderCameraHalf(gameState.players[0],0,0,BASE_WIDTH,BASE_HEIGHT,false,0);
    ctx.restore();
    ctx.font=FONT_TITLE;
    ctx.fillStyle='#ffc15f'; ctx.globalAlpha=(1-phaseT)*0.85;
    ctx.fillText(gameState.tower.name, BASE_WIDTH/2-180,120+13*(1-phaseT));
    ctx.globalAlpha=1.0;
  } else {
    // Zoom out to players
    let y0 = gameState.players[0].y, y1 = gameState.players[1].y;
    let focusY = Math.min(y0,y1)-100;
    let scale = 0.89+easeInOutQuad(phaseT)*0.18;
    ctx.save();
    ctx.translate(BASE_WIDTH/2, BASE_HEIGHT/2);
    ctx.scale(scale,scale);
    ctx.translate(-BASE_WIDTH/2, -focusY);
    renderCameraHalf(gameState.players[0],0,0,BASE_WIDTH,BASE_HEIGHT,false,0,gameState.players[1]);
    ctx.restore();
    ctx.font=FONT_TITLE;
    ctx.globalAlpha=(1-phaseT)*0.8;
    ctx.fillStyle='#fff';
    ctx.fillText('Ready?', BASE_WIDTH/2-78, BASE_HEIGHT/2-124);
    ctx.globalAlpha=1.0;
  }
  ctx.restore();
}

// Draw player sprite (stick figure)
function drawPlayer(ctx,p,ghostAlpha=0) {
  ctx.save();
  ctx.globalAlpha= p.alive?(1-ghostAlpha):0.33;
  // Body
  ctx.fillStyle=p.color;
  ctx.fillRect(p.x, p.y, p.width, p.height);
  // Eyes
  ctx.fillStyle='#fff';
  ctx.fillRect(p.x+7, p.y+15, 7,7);
  ctx.fillRect(p.x+p.width-14, p.y+15, 7,7);
  ctx.fillStyle='#222';
  ctx.fillRect(p.x+9, p.y+18, 3,3);
  ctx.fillRect(p.x+p.width-12, p.y+18, 3,3);
  // Nameplate
  ctx.font='bold 12px Arial';
  ctx.globalAlpha=0.55; ctx.fillStyle='#232233';
  ctx.fillRect(p.x, p.y-20, p.width,13);
  ctx.globalAlpha=1.0; ctx.fillStyle='#fff';
  ctx.fillText(p.name, p.x+4, p.y-8);
  ctx.restore();
}

// Find win pad for cutscene
function findWinPad(towerData) {
  let floor = towerData.floors[towerData.floors.length-1];
  for (let p of floor.platforms)
    if (p.type==='win') return p;
  return floor.platforms[0];
}

function drawWinSparkles(ctx, player) {
  if (!player.hasWon && !player.completionTime) return;
  let t=player.completionTime||player.timer;
  for (let i=0;i<18;++i) {
    let angle=(t/19+i*2.7)%6.28;
    let rx = player.x+PLAYER_WIDTH/2+Math.cos(angle)*30;
    let ry = player.y-18+Math.sin(angle)*22;
    ctx.save();
    ctx.globalAlpha=0.5+0.5*Math.sin(t/12+i);
    ctx.fillStyle='#f4ff93';
    ctx.beginPath();
    ctx.arc(rx,ry,5+3*Math.sin(t/17+i),0,6.29); ctx.fill();
    ctx.restore();
  }
}

function renderWinScreen() {
  ctx.save();
  // Overlay
  ctx.globalAlpha=0.82;
  ctx.fillStyle='#181847';
  ctx.fillRect(0,0,BASE_WIDTH,BASE_HEIGHT);
  ctx.globalAlpha=1.0;

  ctx.font='bold 48px Arial, sans-serif';
  ctx.fillStyle='#ffc31c';
  ctx.fillText('WINNER!', BASE_WIDTH/2-128, BASE_HEIGHT/2-60);

  let p=gameState.players[gameState.winnerId];
  ctx.font='bold 32px Arial, sans-serif';
  ctx.fillStyle=p.color||'#fff';
  ctx.fillText(p.name+'', BASE_WIDTH/2-80, BASE_HEIGHT/2);

  ctx.font='24px Arial, sans-serif';
  ctx.fillStyle='#fff';
  let t = (gameState.winnerTime/60).toFixed(2);
  ctx.fillText('Completion Time: '+t+'s', BASE_WIDTH/2-90, BASE_HEIGHT/2+50);

  ctx.fillStyle='#aaff66';
  ctx.font='18px Arial, sans-serif';
  ctx.fillText('Press ENTER or SPACE to play again.', BASE_WIDTH/2-120,BASE_HEIGHT/2+120);
  ctx.restore();
}

</script>
</body>
</html>
