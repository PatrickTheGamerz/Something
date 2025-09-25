<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>EToH Ring 1: 2-Player Platformer</title>
  <style>
    html, body {
      width: 100%;
      height: 100%;
      margin: 0;
      background: #181D2A;
      overflow: hidden;
    }
    body {
      font-family: 'Segoe UI', Arial, sans-serif;
      user-select: none;
    }
    #gameCanvas {
      position: absolute;
      left: 0; top: 0; right: 0; bottom: 0;
      width: 100vw; height: 100vh;
      background: #181D2A;
      display: block;
      outline: none;
    }
    #uiOverlay {
      position: absolute;
      top: 0; left: 0; width: 100%; z-index: 2;
      pointer-events: none;
      color: #FFF;
      font-size: 22px;
      text-align: center;
      text-shadow: 0 2px 6px rgba(0,0,0,0.7);
      transition: opacity 0.3s;
    }
    .button {
      background: #202A44;
      border: none;
      color: #fff;
      padding: 7px 18px;
      margin: 0 8px;
      border-radius: 7px;
      cursor: pointer;
      font-size: 18px;
      font-weight: 500;
      box-shadow: 0 4px 16px rgba(20,30,44,0.36);
      outline: none;
      transition: background 0.25s;
    }
    .button:active {
      background: #2355B1;
    }
    @media (max-width: 700px) {
      #gameCanvas {
        width: 100vw; height: 80vw;
        max-height: 60vh;
      }
    }
  </style>
</head>
<body>
  <canvas id="gameCanvas" tabindex="1"></canvas>
  <div id="uiOverlay"></div>
<script>
/************************************************
   EToH/JToH Inspired 2-Player HTML5 Platformer
   By: Research Synthesis (2025)
   See top of file for CSS and HTML structure.
************************************************/

/*------------------- Utility Functions -------------------*/
// Clamp value
function clamp(val, min, max) { return Math.max(min, Math.min(max, val)); }
// Colors for players
const PLAYER_COLORS = ["#4BC2FF", "#FF5999"];
const PLAYER_NAMES = ["Player 1", "Player 2"];

// Axis-aligned bounding box collision
function aabb(ax, ay, aw, ah, bx, by, bw, bh) {
  return ax < bx + bw && ax + aw > bx && ay < by + bh && ay + ah > by;
}

// Interpolate
function lerp(a, b, t) { return a + (b - a) * t; }

// Tweening for cutscenes
function easeInOutQuad(t){ return t<0.5 ? 2*t*t : 1-(-2*t+2)**2/2; }

/*------------------- Canvas Setup -------------------*/
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const uiOverlay = document.getElementById('uiOverlay');

let C_WIDTH = 980, C_HEIGHT = 690;
function resizeCanvas() {
  // Responsive resize keeping aspect ratio
  const ratio = C_WIDTH/C_HEIGHT;
  let w = window.innerWidth, h = window.innerHeight;
  if(w/h > ratio) w = h*ratio; else h = w/ratio;
  canvas.width = C_WIDTH; canvas.height = C_HEIGHT;
  canvas.style.width = w+"px"; canvas.style.height = h+"px";
}
resizeCanvas();
window.addEventListener('resize', resizeCanvas);

/*------------------- Game Constants -------------------*/
const GRAVITY = 0.74;
const MAX_FLOORS = 10; // Demo, varies per tower
const FLOOR_HEIGHT = 120; // Distance between floors
const PLAYER_WIDTH = 30, PLAYER_HEIGHT = 40;
const MOVE_SPD = 3.2;
const JUMP_PWR = 10.5;
const SPLIT_DIST = FLOOR_HEIGHT * 2.7; // Split when players are far vertically

// HP Mechanics
const PLAYER_MAX_HP = 100, HAZARD_DMG = 5, HP_REGEN = 7; // per sec

/*------------------- Key Level Data (DEMO: ToAST, ToK, CoLS) -------------------*/
const TOWER_PRESETS = [
  // Tower of A Simple Time (ToAST): Easy
  {
    id: "ToAST", name: "Tower of A Simple Time",
    palette: ["#758DEB", "#b9daf2", "#dddceb", "#212A5C"],
    floors: [
      // Platforms: x, y, w, h
      [ 
        {x: 60, y: 0, w: 340, h: 18}, // F1 base
        {x: 140, y: 60, w: 260, h: 16}, // F2
        {x: 100, y: 120, w: 150, h: 16},
        {x: 260, y: 180, w: 150, h: 16},
        {x: 80, y: 240, w: 320, h: 16}, // F5
        {x: 180, y: 300, w: 220, h: 14},
        {x: 60, y: 360, w: 140, h: 16},
        {x: 340, y: 420, w: 110, h: 16},
        {x: 90, y: 480, w: 340, h: 14},
        {x: 180, y: 540, w: 220, h: 16}, // F10
      ],
      // Hazards: x, y, w, h; (floor #, y rel to base)
      [
        {x:132, y:120, w:41, h:7, f:2}, // F3
        {x:370, y:300, w:30, h:7, f:6},
      ]
    ],
    intro: "Beginner tower with simple jumps and a welcoming palette. Pure platform basics.",
    totalFloors: 10,
  },
  // Tower of Killjoys (ToK): Mid-level
  {
    id: "ToK", name: "Tower of Killjoys",
    palette: ["#e8905f","#ffe073","#f7c780","#44270a"],
    floors: [
      [
        {x:60, y:0, w:220, h:18},
        {x:320, y:60, w:180, h:16}, // F2
        {x:120, y:120, w:140, h:16},
        {x:340, y:180, w:110, h:14},
        {x:60, y:240, w:320, h:14}, // F5
        {x:210, y:300, w:180, h:14},
        {x:128, y:360, w:230, h:12},
        {x:60, y:420, w:100, h:16},
        {x:220, y:480, w:241, h:13},
        {x:70, y:540, w:340, h:16},
      ],
      [
        {x:210, y:480, w:100, h:7, f:9},
        {x:350, y:120, w:30, h:7, f:3}
      ]
    ],
    intro: "A wide range of platforms and a couple of nasty hazard patches with basic timing puzzles and a push block.",
    totalFloors: 10,
    pushBlocks: [
      {x:200, y:181, w:30, h:30, minX:70, maxX:370, f:4, targetBtn:0}
    ],
    buttons: [
      {x:150, y:306, f:6, target:0} // Triggers moving platform for a period
    ]
  },
  // Citadel of Laptop Splitting (CoLS): Long, advanced, multiple mechanics
  {
    id: "CoLS", name: "Citadel of Laptop Splitting",
    palette: ["#7FDDC9","#1A3944","#BFE6E4","#17392E"],
    floors: [
      // 15 floors (unrolled for brevity, demo 10)
      [
        {x:60, y:0, w:340, h:18},
        {x:60, y:60, w:150, h:16},
        {x:270, y:120, w:180, h:16},
        {x:100, y:180, w:140, h:16},
        {x:60, y:240, w:170, h:16},
        {x:240, y:300, w:160, h:15},
        {x:80, y:360, w:120, h:14},
        {x:320, y:420, w:110, h:13},
        {x:80, y:480, w:340, h:14},
        {x:190, y:540, w:210, h:13}
      ],
      [
        {x:250, y:60, w:40, h:7, f:2},
        {x:150, y:360, w:80, h:8, f:7}
      ]
    ],
    intro: "A daunting citadel with more floors, tricky moving objects, advanced puzzles and multi-part segments.",
    totalFloors: 15,
    movingPlatforms: [
      {from:{x:100,y:180}, to:{x:340,y:180}, w:60, h:16, spd:1.1, f:4, loop:true}
    ],
    spinners: [
      {cx:220, cy:360, r:50, len:90, speed:0.03, f:7}
    ],
    buttons: [
      {x:150, y:546, f:10, target:0}, // unlock upper push block
    ]
  }
];

/*------------------- Button/Control Bindings -------------------*/
// Player 1: WASD, Left Shift
const P1_KEYS = { left:'a', right:'d', jump:'w', down:'s', action:'Shift' };
// Player 2: Arrow keys, Right Shift
const P2_KEYS = { left:'ArrowLeft', right:'ArrowRight', jump:'ArrowUp', down:'ArrowDown', action:'/' };

/************************************************
|-------------- Classes & Structures ------------|
************************************************/

// Timed Effect Constants
const EFFECTS = {
  NONE: { speed: 1, jump: 1, canJump: true },
  SLOW: { speed: 0.6, jump: 0.75, canJump: true },
  HIJUMP: { speed: 1, jump: 1.5, canJump: true },
  NOJUMP: { speed: 1, jump: 0.5, canJump: false }
}

// Timer helper 
function Timer(duration) {
  let t = 0, running = false;
  return {
    start: () => { t = 0; running = true; },
    stop: () => { running = false; },
    tick: dt => { if(running) t += dt; },
    done: () => t >= duration,
    time: () => t
  }
}

// Player Structure
class Player {
  constructor(idx, spawn) {
    this.idx = idx;
    this.x = spawn.x;
    this.y = spawn.y;
    this.vx = 0;
    this.vy = 0;
    this.hp = PLAYER_MAX_HP;
    this.maxHp = PLAYER_MAX_HP;
    this.regenTimer = 0;
    this.onGround = false;
    this.effect = {...EFFECTS.NONE};
    this.effectTimer = 0;
    this.color = PLAYER_COLORS[idx];
    this.lastSafe = {...spawn}; // checkpoint
    this.name = PLAYER_NAMES[idx];
    this.controls = idx === 0 ? P1_KEYS : P2_KEYS;
    this.held = {}; // input keys
    this.score = 0;
    this.winState = false;
  }
  applyEffect(e, dur) {
    this.effect = {...e};
    this.effectTimer = dur || 0;
  }
  updateEffect(dt) {
    if(this.effectTimer > 0){
      this.effectTimer -= dt;
      if(this.effectTimer <= 0) this.effect = {...EFFECTS.NONE};
    }
  }
  tickRegen(dt) {
    if(this.hp < this.maxHp && this.regenTimer <= 0){
      this.hp = clamp(this.hp + HP_REGEN * dt, 0, this.maxHp);
    }
    if(this.regenTimer > 0) this.regenTimer -= dt;
  }
  hurt(amt) {
    if(this.hp > 0) {
      this.hp = clamp(this.hp - amt, 0, this.maxHp);
      this.regenTimer = 1.75; // 1.75s before start regen
    }
  }
  reset(spawn) {
    this.x = spawn.x; this.y = spawn.y;
    this.vx = this.vy = 0;
    this.hp = this.maxHp;
    this.effect = {...EFFECTS.NONE};
    this.winState = false;
    this.lastSafe = {...spawn};
  }
}

/*------------------- Game Entities -------------------*/
// Platform, Hazard, MovingPlatform, Button, Spinner, PushableBlock structs
class Platform {
  constructor(x, y, w, h) { this.x = x; this.y = y; this.w = w; this.h = h; }
}
class Hazard {
  constructor(x, y, w, h) { this.x = x; this.y = y; this.w = w; this.h = h; }
}
class ButtonEntity {
  constructor(x, y, floorIdx, targetIdx) {
    this.x = x; this.y = y; this.floorIdx = floorIdx;
    this.targetIdx = targetIdx; // index in linked platforms
    this.pressed = false; this.timer = 0;
  }
}
class MovingPlatform {
  constructor(from, to, w, h, spd, floorIdx, loop) {
    this.from = {...from}; this.to = {...to}; this.w = w; this.h = h;
    this.spd = spd; this.t = 0; this.dir = 1; this.loop = loop; this.floorIdx = floorIdx;
    this.active = true;
  }
  get pos(){
    return {
      x: lerp(this.from.x, this.to.x, this.t),
      y: this.from.y,
      w: this.w, h: this.h
    };
  }
  update(dt) {
    if(!this.active) return;
    this.t += dt * this.spd * this.dir;
    if(this.t > 1) { if(this.loop) {this.t = 1; this.dir = -1;} else {this.t = 1; this.active = false;}}
    if(this.t < 0) { if(this.loop) {this.t = 0; this.dir = 1;} else {this.t = 0; this.active = false;}}
  }
  reset() { this.t = 0; this.dir = 1; this.active = true; }
}
class Spinner {
  constructor(cx, cy, radius, len, speed, floorIdx) {
    this.cx = cx; this.cy = cy; this.r = radius; this.len = len; this.spd = speed;
    this.floorIdx = floorIdx;
    this.ang = 0;
  }
  update(dt) { this.ang += this.spd*dt*60; }
}
class PushBlock {
  constructor(x, y, w, h, minX, maxX, floorIdx, targetBtn) {
    this.x = x; this.y = y; this.w = w; this.h = h;
    this.minX = minX; this.maxX = maxX; this.floorIdx = floorIdx; this.targetBtn = targetBtn;
    this.vx = 0;
  }
  update(dt){
    if(Math.abs(this.vx) > 0.01) this.x += this.vx*dt*60, this.vx *= 0.92;
  }
}
class TimedEffectTrigger {
  constructor(x, y, w, h, floorIdx, effect, duration){
    this.x = x; this.y = y; this.w = w; this.h = h;
    this.floorIdx = floorIdx; this.effect = effect;
    this.duration = duration;
  }
}

/*------------------- Level Loader and Logic -------------------*/
class Level {
  constructor(preset){
    this.id = preset.id; this.name = preset.name;
    this.palette = preset.palette;
    this.intro = preset.intro;
    this.totalFloors = preset.totalFloors;
    this.platforms = []; this.hazards = [];
    this.buttons = [], this.movingPlatforms = [];
    this.spinners = [], this.pushBlocks = [];
    this.effectTriggers = [];
    
    // Load platforms and hazards, offset for floor stacking
    let [plats, hazards] = preset.floors;
    for(let i=0; i < plats.length; ++i){
      let pf = plats[i];
      let fy = FLOOR_HEIGHT * i;
      this.platforms.push(new Platform(pf.x, fy + pf.y, pf.w, pf.h));
    }
    if(hazards) for(const hz of hazards) {
      let fy = hz.f !== undefined ? FLOOR_HEIGHT * (hz.f-1) : 0;
      this.hazards.push(new Hazard(hz.x, fy + hz.y, hz.w, hz.h));
    }
    // Optional: moving platforms
    if(preset.movingPlatforms){
      for(const mp of preset.movingPlatforms){
        let fy = mp.f !== undefined ? FLOOR_HEIGHT*(mp.f-1) : 0;
        this.movingPlatforms.push(new MovingPlatform(mp.from, mp.to, mp.w, mp.h, mp.spd, mp.f, mp.loop));
      }
    }
    if(preset.spinners){
      for(const sp of preset.spinners){
        let fy = sp.f !== undefined ? FLOOR_HEIGHT*(sp.f-1) : 0;
        this.spinners.push(new Spinner(sp.cx, fy + sp.cy, sp.r, sp.len, sp.speed, sp.f));
      }
    }
    if(preset.buttons){
      for(const btn of preset.buttons){
        let fy = btn.f !== undefined ? FLOOR_HEIGHT*(btn.f-1) : 0;
        this.buttons.push(new ButtonEntity(btn.x, fy + btn.y, btn.f, btn.target));
      }
    }
    if(preset.pushBlocks){
      for(const pb of preset.pushBlocks){
        let fy = pb.f !== undefined ? FLOOR_HEIGHT*(pb.f-1) : 0;
        this.pushBlocks.push(new PushBlock(pb.x, fy + pb.y, pb.w, pb.h, pb.minX, pb.maxX, pb.f, pb.targetBtn));
      }
    }
    // For demonstration: install a random effect zone
    this.effectTriggers.push(new TimedEffectTrigger(90, 420, 42, 20, 8, EFFECTS.SLOW, 2));
    // Win Pad at top
    const lastPlat = this.platforms[this.platforms.length-1];
    this.winPad = {x: lastPlat.x + lastPlat.w - 54, y: lastPlat.y - 30, w:42, h:16};
    // Spawn locations per player
    this.p1Spawn = {x: 75, y: 8}; this.p2Spawn = {x: 105, y: 8};
    this.offsetY = FLOOR_HEIGHT * (this.platforms.length-1);
  }
}

/*------------------- Camera Management -------------------*/
class Camera {
  constructor(){ this.x = 0; this.y = 0; }
  set(x, y){ this.x = x; this.y = y; }
  lerpTo(x, y, amt){
    this.x = lerp(this.x, x, amt);
    this.y = lerp(this.y, y, amt);
  }
  // Camera restricts over tower bounds
  static clampY(y){
    return clamp(y, 0, (FLOOR_HEIGHT*MAX_FLOORS)-(C_HEIGHT-160));
  }
}

/*------------------- Main Game Controller -------------------*/
class Game {
  constructor(){
    this.players = [];
    this.level = null;
    this.state = "intro"; // "cutscene", "play", "win"
    this.towerList = [];
    this.selectedTowerIdx = 0;
    this.timer = 0;
    this.cameras = [new Camera(), new Camera()];
    this.splitMode = false;
    this.splitTransition = 0; // 0=none, 1=full split
    this.winnerIdx = -1;
  }
  /*----- Initialization -----*/
  init(){
    // Build list, randomize tower
    this.towerList = [...TOWER_PRESETS];
    this.selectedTowerIdx = Math.floor(Math.random()*this.towerList.length);
    this.level = new Level(this.towerList[this.selectedTowerIdx]);
    // Build players
    this.players = [
      new Player(0, this.level.p1Spawn),
      new Player(1, this.level.p2Spawn)
    ];
    // Place camera at base for intro
    this.cameras[0].set(0, this.level.platforms[0].y - 50);
    this.cameras[1].set(0, this.level.platforms[0].y - 50);
    this.timer = 0;
    this.state = "cutscene";
    this.winnerIdx = -1;
    uiOverlay.innerHTML = "";
    // Listen for key events
    this.clearInputs();
  }
  clearInputs(){
    for(const p of this.players) p.held = {};
  }
  /*----- Main Game Loop -----*/
  update(dt){
    if(this.state === "cutscene"){
      this.cutsceneAnim(dt);
      return;
    }
    if(this.state === "win") return;
    // Move platforms
    for(let mp of this.level.movingPlatforms) mp.update(dt);
    for(let sp of this.level.spinners) sp.update(dt);
    for(let pb of this.level.pushBlocks) pb.update(dt);

    // Player physics & input
    for(let i=0; i<2; ++i){
      this.updatePlayer(i, dt);
    }
    // Check for split-screen
    let y0 = this.players[0].y, y1 = this.players[1].y;
    if(Math.abs(y0 - y1) > SPLIT_DIST) this.splitMode = true;
    else this.splitMode = false;
    // Win Pad Detection
    for(let i=0; i<2; ++i){
      const p = this.players[i], pad = this.level.winPad;
      if(aabb(p.x, p.y, PLAYER_WIDTH, PLAYER_HEIGHT, pad.x, pad.y, pad.w, pad.h)){
        this.state = "win"; this.winnerIdx = i;
        setTimeout(()=>this.showWinner(i), 400);
        break;
      }
    }
  }
  /*----- Player Update (Physics, Collisions, Triggers) -----*/
  updatePlayer(idx, dt){
    const p = this.players[idx];
    p.updateEffect(dt);
    // Input movement
    let mv = 0;
    if(p.held[p.controls.left]) mv -= 1;
    if(p.held[p.controls.right]) mv += 1;
    p.vx = mv * MOVE_SPD * (p.effect.speed || 1);
    // Jump
    if(!p.jumpLock && p.held[p.controls.jump] && p.onGround && (p.effect.canJump !== false)){
      p.vy = -JUMP_PWR * (p.effect.jump || 1);
      p.onGround = false;
      p.jumpLock = true;
    }
    if(!p.held[p.controls.jump]) p.jumpLock = false;
    // Gravity
    p.vy += GRAVITY;
    // Next position
    let nx = p.x + p.vx; let ny = p.y + p.vy;
    // Collision with moving platforms (if standing), add platform velocity
    let bent = null;
    for(const mp of this.level.movingPlatforms){
      let mpp = mp.pos;
      if(aabb(p.x, p.y+PLAYER_HEIGHT, PLAYER_WIDTH, 4, mpp.x, mpp.y, mpp.w, mpp.h)){
        if(p.vy >= 0) bent = mp;
        break;
      }
    }
    if(bent) nx += (bent.pos.x - bent.from.x), ny += (bent.pos.y - bent.from.y);
    // Platform collision
    p.onGround = false;
    for(const plat of this.level.platforms.concat(this.level.movingPlatforms.map(mp=>mp.pos))){
      // Downwards land
      if(p.vy >= 0 && aabb(nx, ny+PLAYER_HEIGHT, PLAYER_WIDTH, 2, plat.x, plat.y, plat.w, plat.h)){
        // Landed
        ny = plat.y - PLAYER_HEIGHT;
        p.vy = 0;
        p.onGround = true;
      }
      // Hit head
      else if(p.vy < 0 && aabb(nx, ny, PLAYER_WIDTH, 2, plat.x, plat.y+plat.h-2, plat.w, 2)){
        p.vy = 0;
      }
    }
    // Stop at sides of platforms (basic, can enhance to AABB moving-collision if needed)
    for(const plat of this.level.platforms.concat(this.level.movingPlatforms.map(mp=>mp.pos))){
      if(aabb(nx, ny, PLAYER_WIDTH, PLAYER_HEIGHT, plat.x-2, plat.y, 2, plat.h)){
        nx = plat.x - PLAYER_WIDTH;
      }
      if(aabb(nx, ny, PLAYER_WIDTH, PLAYER_HEIGHT, plat.x+plat.w, plat.y, 2, plat.h)){
        nx = plat.x + plat.w + 1;
      }
    }
    // Clamp world bounds
    nx = clamp(nx, 0, C_WIDTH-PLAYER_WIDTH-12);
    // Respawn if below world
    if(ny > (FLOOR_HEIGHT*this.level.platforms.length+150)){
      nx = this.level.p1Spawn.x + 10*idx; ny = this.level.p1Spawn.y;
    }
    // Hazards
    for(const hz of this.level.hazards){
      if(aabb(nx, ny, PLAYER_WIDTH, PLAYER_HEIGHT, hz.x, hz.y, hz.w, hz.h)){
        p.hurt(HAZARD_DMG);
      }
    }
    p.x = nx; p.y = ny;
    // Pick up safe point (platform ground)
    for(const plat of this.level.platforms){
      if(aabb(p.x, p.y+PLAYER_HEIGHT+1, PLAYER_WIDTH, 1, plat.x, plat.y, plat.w, plat.h)){
        p.lastSafe = {x: p.x, y: plat.y - PLAYER_HEIGHT};
      }
    }
    // Buttons (activate when colliding)
    for(const btn of this.level.buttons){
      if(!btn.pressed && btn.floorIdx && aabb(p.x, p.y, PLAYER_WIDTH, PLAYER_HEIGHT, btn.x, btn.y, 20, 18)){
        btn.pressed = true; btn.timer = 1.5;
        // Activate assigned moving platform or trigger
        let tgt = this.level.movingPlatforms[btn.target];
        if(tgt) tgt.reset();
      }
    }
    for(const btn of this.level.buttons){
      if(btn.pressed) { btn.timer -= dt; if(btn.timer <= 0) btn.pressed = false; }
    }
    // Push blocks
    for(const b of this.level.pushBlocks){
      if(aabb(p.x, p.y, PLAYER_WIDTH, PLAYER_HEIGHT, b.x-3, b.y, b.w+6, b.h)){
        let dir = (p.x + PLAYER_WIDTH/2 > b.x + b.w/2) ? 1 : -1;
        b.vx = dir*1.3;
        b.x = clamp(b.x + dir*2, b.minX, b.maxX);
      }
    }
    // Effect triggers (apply if collision)
    for(const ef of this.level.effectTriggers){
      if(aabb(p.x, p.y, PLAYER_WIDTH, PLAYER_HEIGHT, ef.x, ef.y, ef.w, ef.h)){
        p.applyEffect(ef.effect, ef.duration);
      }
    }
    // Spinners (hurt on touch, knockback)
    for(const sp of this.level.spinners){
      let sx = sp.cx + sp.r*Math.cos(sp.ang), sy = sp.cy + sp.r*Math.sin(sp.ang);
      let ax = sx, ay = sy, aw = sp.len, ah = 8;
      if(aabb(p.x, p.y, PLAYER_WIDTH, PLAYER_HEIGHT, ax, ay, aw, ah)){
        p.hurt(HAZARD_DMG);
        p.vx -= Math.cos(sp.ang)*3;
        p.vy -= Math.sin(sp.ang)*2.2;
      }
    }
    // Timed status effect decay & hp regen
    p.updateEffect(dt);
    p.tickRegen(dt);
  }
  /*----- Cutscene Animation -----*/
  cutsceneAnim(dt){
    // Camera pans base to top, then slides title, waits, shows intro, fades to play
    this.timer += dt;
    let t = clamp(this.timer/3, 0, 1);
    let baseY = this.level.platforms[0].y - 50, topY = this.level.platforms[this.level.platforms.length-1].y - 60;
    let targetY = lerp(baseY, topY, easeInOutQuad(t));
    for(let cam of this.cameras) cam.y = targetY;
    // Fade in tower name
    ctx.globalAlpha = Math.min(t*1.2,1);
    ctx.fillStyle = this.level.palette[0] || "#FFF";
    ctx.font = "58px Segoe UI, Arial";
    ctx.fillText(this.level.name, C_WIDTH/2-ctx.measureText(this.level.name).width/2, C_HEIGHT/2-68);
    ctx.font = "22px Segoe UI";
    ctx.globalAlpha = Math.min((t-0.6)*2.5,1);
    ctx.fillStyle = "#FFF";
    ctx.fillText(this.level.intro, C_WIDTH/2-ctx.measureText(this.level.intro).width/2, C_HEIGHT/2-24);
    ctx.globalAlpha = 1;
    // Wait, then start gameplay
    if(this.timer > 3.6){
      this.state = "play";
      for(let i=0; i<2; ++i) this.cameras[i].set(this.players[i].x-100, Camera.clampY(this.players[i].y-240));
      setTimeout(()=> {uiOverlay.innerHTML=
        "Controls: <b>A/D/W</b> + Shift (P1), <b>←/→/↑</b> + / (P2).<br> Touch the Win Pad at the very top to win!";}, 300);
    }
  }
  /*----- Drawing Logic -----*/
  draw(){
    // Render game. If split, render 2 viewports; else, single median camera
    ctx.clearRect(0,0,canvas.width,canvas.height);
    // Set sky gradient
    let g = ctx.createLinearGradient(0,0,0,C_HEIGHT);
    g.addColorStop(0, this.level.palette[0]); g.addColorStop(1, this.level.palette[1]);
    ctx.fillStyle = g; ctx.fillRect(0,0,C_WIDTH,C_HEIGHT);
    // Single or Split?
    if(this.splitMode){
      // Top: Player 1; Bottom: Player 2
      for(let i=0; i<2; ++i){
        let cam = this.cameras[i];
        ctx.save();
        // Mask half canvas
        ctx.beginPath();
        if(i==0) ctx.rect(0,0,C_WIDTH,C_HEIGHT/2+2);
        else ctx.rect(0,C_HEIGHT/2-2,C_WIDTH,C_HEIGHT/2+4);
        ctx.clip();
        ctx.translate(-cam.x, -cam.y+(i==0?0:-C_HEIGHT/2));
        this.drawLevel(cam);
        for(let j=0; j<2; ++j){
          if(j==i) this.drawPlayer(j);
        }
        ctx.restore();
        // Divider
        ctx.strokeStyle = "#DEF";
        ctx.lineWidth = 3;
        ctx.beginPath();
        ctx.moveTo(0, C_HEIGHT/2); ctx.lineTo(C_WIDTH, C_HEIGHT/2);
        ctx.stroke();
      }
    } else {
      // Both players in same camera
      let mx = (this.players[0].x + this.players[1].x)/2 - C_WIDTH/2 + 80;
      let my = (this.players[0].y + this.players[1].y)/2 - C_HEIGHT/2 + 120;
      mx = clamp(mx, 0, C_WIDTH-480); my = Camera.clampY(my);
      ctx.save();
      ctx.translate(-mx, -my);
      this.drawLevel({x:mx, y:my});
      for(let i=0; i<2; ++i) this.drawPlayer(i);
      ctx.restore();
    }
    this.drawUI();
  }
 
  drawLevel(cam){
    // Draw platforms
    for(const plat of this.level.platforms){
      ctx.fillStyle = this.level.palette[2];
      ctx.fillRect(plat.x, plat.y, plat.w, plat.h);
    }
    // Hazards
    for(const hz of this.level.hazards){
      ctx.fillStyle = "#FF3678";
      ctx.fillRect(hz.x, hz.y, hz.w, hz.h);
    }
    // Win Pad
    const pad = this.level.winPad;
    ctx.fillStyle = "#6AFFA1";
    ctx.fillRect(pad.x, pad.y, pad.w, pad.h);
    ctx.font = "bold 15px Segoe UI"; ctx.fillStyle = "#234";
    ctx.fillText("WIN", pad.x+pad.w/2-14, pad.y+13);
    // Buttons
    for(const btn of this.level.buttons){
      ctx.fillStyle = btn.pressed ? "#FFD94B" : "#FEE568";
      ctx.fillRect(btn.x, btn.y, 20, 18);
      ctx.font="11px Segoe UI"; ctx.fillStyle="#234";
      ctx.fillText("BTN", btn.x+2, btn.y+12);
    }
    // Moving platforms
    for(const mp of this.level.movingPlatforms){
      let pos = mp.pos;
      ctx.fillStyle = "#FAFD94"; ctx.fillRect(pos.x, pos.y, pos.w, pos.h);
    }
    // Spinners
    for(const sp of this.level.spinners){
      let theta = sp.ang, sx = sp.cx + sp.r*Math.cos(theta), sy = sp.cy + sp.r*Math.sin(theta);
      ctx.save();
      ctx.strokeStyle = "#86d"; ctx.lineWidth = 5.5;
      ctx.beginPath(); ctx.moveTo(sp.cx, sp.cy); ctx.lineTo(sx + sp.len, sy); ctx.stroke();
      ctx.beginPath(); ctx.arc(sp.cx, sp.cy, 12, 0, Math.PI*2); ctx.stroke();
      ctx.restore();
    }
    // Push blocks
    for(const b of this.level.pushBlocks){
      ctx.fillStyle = "#B86E2C"; ctx.fillRect(b.x, b.y, b.w, b.h);
    }
    // Effect triggers
    for(const ef of this.level.effectTriggers){
      ctx.globalAlpha = 0.3;
      ctx.fillStyle = "#5AFECD"; ctx.fillRect(ef.x, ef.y, ef.w, ef.h);
      ctx.globalAlpha = 1;
    }
    // Floor number marker
    let y0 = cam.y - 80; let topY = cam.y + C_HEIGHT + 80;
    for(let i=0; i < this.level.platforms.length; ++i){
      let pf = this.level.platforms[i];
      if(pf.y > y0 && pf.y < topY){
        ctx.font = "bold 17px Segoe UI"; ctx.fillStyle = "#234";
        ctx.fillText("F"+(i+1), pf.x-50, pf.y+12);
      }
    }
  }
  drawPlayer(idx){
    // Draw player rectangle, face, and info
    let p = this.players[idx];
    ctx.save();
    ctx.fillStyle = p.color;
    ctx.fillRect(p.x, p.y, PLAYER_WIDTH, PLAYER_HEIGHT);
    // Face
    ctx.fillStyle = "#fff"; ctx.beginPath();
    ctx.arc(p.x+PLAYER_WIDTH/2, p.y+PLAYER_HEIGHT/2, 8, 0, Math.PI*2);
    ctx.fill();
    ctx.restore();
    // Username, HP bar
    ctx.font = "15px Segoe UI";
    ctx.fillStyle = p.color;
    ctx.fillText(p.name, p.x-2, p.y-7);
    ctx.strokeStyle="#234";
    ctx.strokeRect(p.x-2, p.y+PLAYER_HEIGHT+3, PLAYER_WIDTH+4, 9);
    ctx.fillStyle="#2ED3B8";
    ctx.fillRect(p.x-2, p.y+PLAYER_HEIGHT+3, (PLAYER_WIDTH+4)*p.hp/p.maxHp, 9);
  }
  drawUI(){
    // HP bars, effects, floor, winner overlay
    let html = "";
    for(let i=0; i<2; ++i){
      let p=this.players[i];
      html += `<span style='color:${p.color}'>${p.name}: ${Math.round(p.hp)} HP</span>`;
      if(p.effect !== EFFECTS.NONE) html += ` <span style="color:#AFC">${Object.keys(EFFECTS).find(key=>EFFECTS[key]==p.effect)}</span>`;
      html += " | ";
    }
    html += `<span style="color:#BFD">Tower: <b>${this.level.name}</b></span>`;
    if(this.state === "win" && this.winnerIdx !== -1){
      html += `<div style="font-size:47px;color:#6AFFA1;text-shadow:1px 1px 12px #234">🏆 ${PLAYER_NAMES[this.winnerIdx]} WINS! 🏆</div>`;
      html += `<button class="button" onclick="location.reload()">Play again</button>`;
    }
    uiOverlay.innerHTML = html;
  }
  /*----- Winner Logic -----*/
  showWinner(i){
    uiOverlay.innerHTML += `<br><span style="font-size:24px;">Press <b>Play again</b> to start a new round.</span>`;
  }
}

/************************************************
|-------------- Input System ------------|
************************************************/
const keyState = {};
window.addEventListener('keydown', e=>{
  keyState[e.key] = true;
});
window.addEventListener('keyup', e=>{
  keyState[e.key] = false;
});
canvas.addEventListener("blur", ()=>Object.keys(keyState).forEach(k=>delete keyState[k]));

/*----- Assign to player held keys in updateInputs -----*/
function updateInputs(game){
  for(let i=0;i<2;++i){
    let p = game.players[i];
    let ctl = p.controls;
    p.held = {
      [ctl.left]: keyState[ctl.left],
      [ctl.right]: keyState[ctl.right],
      [ctl.jump]: keyState[ctl.jump],
      [ctl.down]: keyState[ctl.down],
      [ctl.action]: keyState[ctl.action]
    };
  }
}

/************************************************
|-------------- Main Game Loop ------------|
************************************************/
const game = new Game();
game.init();

let lastTime = performance.now();
function mainLoop(now){
  const dt = (now-lastTime)/1000;
  lastTime = now;
  updateInputs(game);
  game.update(dt);
  game.draw();
  requestAnimationFrame(mainLoop);
}
requestAnimationFrame(mainLoop);

</script>
</body>
</html>
