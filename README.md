<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>JToH/EToH Tower Platformer</title>
  <style>
    body { background: #191a1c; color: #eee; margin: 0; overflow: hidden; }
    canvas { background: #191a1c; display: block; margin: 0 auto; }
    #game-ui { position:absolute; top:0; left:0; width:100%; color:#fff; font-family:monospace; font-size:1.2em; text-shadow:2px 2px 4px #000;}
    .centered { position: absolute; left: 50%; top: 25%; transform: translate(-50%, -25%); }
  </style>
</head>
<body>
<div id="game-ui"></div>
<canvas id="game" width="1280" height="720"></canvas>
<script>
// ==== SETTINGS & DATA ====
// Tower metadata and simplified layouts for demonstration (expand for full auth.)
const TOWER_LIST = [
  { 
    id: "ToAST", name: "Tower of Annoyingly Simple Trials", floors: 10, difficulty: "Effortless", 
    floorColors: ["#ee2222","#ee9922","#eec822","#56c822","#09d787","#49c7dc","#3885e6","#b546d6","#ff76b9","#f7f7f7"],
    layout: [], // Will fill per-floor
  },
  { id: "ToA", name: "Tower of Anger", floors: 10, difficulty: "Medium",
    floorColors: ["#e86929","#f7b439","#e0dd28","#a4d91c","#46ca7d","#39c8ba","#2a7be3","#7d3fd1","#fa32a4","#eee"],
    layout: [],
  },
  { id: "ToM", name: "Tower of Madness", floors: 10, difficulty: "Difficult",
    floorColors: ["#a736b5","#b546d6","#495dda","#508be6","#39c8ba","#21c27b","#46ca7d","#a4d91c","#e0dd28","#eee"],
    layout: [],
  },
  { id: "ToNI", name: "Tower of Noticeable Infuriation", floors: 10, difficulty: "Medium",
    floorColors: ["#f09819","#ed6ea0","#fcb045","#ef9d43","#fe9c00","#f95800","#f1004b","#c800ee","#007afc","#fff"],
    layout: [],
  },
  { id: "ToH", name: "Tower of Hecc", floors: 10, difficulty: "Hard",
    floorColors: ["#aa007f","#7112a6","#591f7d","#391e47","#684455","#bc5779","#ec7d9f","#e6b1d0","#f8d4e4","#f7f7f7"],
    layout: [],
  },
  { id: "ToK", name: "Tower of Killjoys", floors: 10, difficulty: "Challenging",
    floorColors: ["#2be0d5","#07e166","#2ad145","#70d81e","#c9e221","#e6e23f","#e6b01f","#e67a1f","#e63f57","#c3002f"],
    layout: [],
  },
  { id: "ToKY", name: "Tower of Keyboard Yeeting", floors: 10, difficulty: "Difficult",
    floorColors: ["#58bdad","#39c8ba","#46ca7d","#7de63f","#bedc2d","#fff93f","#e6b01f","#f9b331","#b9832c","#45331e"],
    layout: [],
  },
  { id: "ToS", name: "Tower of Stress", floors: 10, difficulty: "Remorseless",
    floorColors: ["#df2b2b","#e6453f","#e67a1f","#e6b01f","#ecdc2a","#dcf829","#81e258","#39e384","#28d5a6","#4d8ae9"],
    layout: [],
  },
  { id: "ToSP", name: "Tower of Screen Punching", floors: 9, difficulty: "Intense",
    floorColors: ["#803432","#db363d","#b8001a","#ff1d58","#f75990","#fa9dbd","#e2cfbc","#b3b2b7","#ab9bbb"],
    layout: [],
  },
  { id: "ToR", name: "Tower of Rage", floors: 10, difficulty: "Remorseless",
    floorColors: ["#d93737","#f27630","#fbc84d","#79c267","#3256a8","#7e65b5","#745d92","#b76daf","#ef4271","#eeeeee"],
    layout: [],
  },
  { id: "ToIE", name: "Tower of Impossible Expectations", floors: 10, difficulty: "Intense",
    floorColors: ["#012459","#005985","#02a8d3","#00cfff","#00e7fe","#15b1da","#0089ba","#005985","#012459","#f7f7f7"],
    layout: [],
  },
  { id: "ToTS", name: "Tower of True Skill", floors: 10, difficulty: "Insane (Soul Crush)",
    floorColors: ["#ffda3e","#f6c62c","#f7b42c","#ff9e25","#ff7727","#f04900","#ed3652","#8c2158","#6d2c70","#f7f7f7"],
    layout: [],
  },
  { id: "TT", name: "Thanos Tower", floors: 10, difficulty: "Extreme (Soul Crush)",
    floorColors: ["#810f6f","#8e2ba7","#5c2483","#322d78","#2e4bad","#544cb7","#7597c9","#b9f2ff","#decbe4","#fff"],
    layout: [],
  },
  { id: "CoLS", name: "Citadel of Laptop Splitting", floors: 15, difficulty: "Intense",
    floorColors: ["#2e5266","#6e8898","#9fb1bc","#ced3dc","#b5c6b8","#6e8898","#4f5d75","#616283","#c2b9b0","#e5e5e5","#a3bbc8","#212b36","#324a5f","#457eac","#f7f7f7"],
    layout: [],
  },
];

// Utility for picking a random tower
function randomTower() {
  const idx = Math.floor(Math.random() * TOWER_LIST.length);
  return JSON.parse(JSON.stringify(TOWER_LIST[idx])); // Deep copy
}

// *** Constants ***
const CANVAS_W = 1280, CANVAS_H = 720;
const BASE_FLOOR_HEIGHT = 140; // px per floor in world units
const PLATFORM_COUNT_MIN = 30, PLATFORM_COUNT_MAX = 40; // per floor

// Player setup
const PLAYER_TEMPLATE = [
  { nickname: "Player 1", controls: { left: "a", right: "d", jump: "w" }, color: "#f53153" },
  { nickname: "Player 2", controls: { left: "ArrowLeft", right: "ArrowRight", jump: "ArrowUp" }, color: "#419bf9" }
];

// Game state
let currentTower;
let gameState = "cutscene"; // cutscene | play | win
let timerStart = null, completionTimes = [null, null];
let splitScreen = false;
let platformer;
let cutsceneZoom = 1.5, cutsceneTimer = 0;
let winPads = [];
let winningPlayers = [false, false];
let healthUI = [100, 100];

// *** Helper: Color mapping ***

function lerp(a,b,t) { return a + (b - a) * t; }
function clamp(v,a,b) { return v < a ? a : v > b ? b : v; }
function choose(arr) { return arr[Math.floor(Math.random()*arr.length)]; }
function colorHexLerp(color1, color2, t) {
  const c1 = parseInt(color1.slice(1),16);
  const c2 = parseInt(color2.slice(1),16);
  const r1=(c1>>16)&0xFF, g1=(c1>>8)&0xFF, b1=c1&0xFF;
  const r2=(c2>>16)&0xFF, g2=(c2>>8)&0xFF, b2=c2&0xFF;
  const r=lerp(r1,r2,t)|0,g=lerp(g1,g2,t)|0,b=lerp(b1,b2,t)|0;
  return "#"+((1<<24)|(r<<16)|(g<<8)|b).toString(16).slice(1);
}

// *** Tower generator: authentic structure w/ color and mechanics ***
function generateTowerLayout(tower, platformCount=PLATFORM_COUNT_MIN) {
  // For demo, use simple platforms, moving, lava, buttons and special per-floor effect.
  for (let f=0; f<tower.floors; ++f) {
    let floor = [];
    const color = tower.floorColors[f % tower.floorColors.length] || "#bbb";
    // Static platforms
    for (let i=0; i<platformCount; ++i) {
      // Each platform spans variable width, height aligned for variety
      let width = 60 + Math.random() * 60, height = 15 + Math.random()*10;
      let x = 60 + Math.random() * (CANVAS_W-180-width);
      let y = f * BASE_FLOOR_HEIGHT + lerp(10, BASE_FLOOR_HEIGHT-30, i/platformCount);
      // Floor boundaries: always add platforms to sides
      if(i<2) x = i==0?30:CANVAS_W-width-30;
      let kind = "platform";
      // 1 in 8: moving platform
      if (Math.random() < 0.12) {
        kind = "moving";
      }
      // 1 in 14: spinner
      else if (Math.random() < 0.07) {
        kind = "spinner";
      }
      else if (Math.random() < 0.07) {
        kind = "lava";
        width = 45+(Math.random()*30);
        height = 13+(Math.random()*5);
      }
      // 1 in 20: button
      else if (Math.random() < 0.05) {
        kind = "button";
        width = 25; height = 12;
      }
      // 1 in 16: timed effect block
      else if (Math.random() < 0.07) {
        kind = choose(['nojump','hijump','slow']);
        width = 32; height = 16;
      }
      floor.push({ x,y,width,height,kind,color });
    }
    // Extra: Main up-vertical platform (so all jumps are possible)
    floor.push({ x: CANVAS_W/2-40, y:f*BASE_FLOOR_HEIGHT+BASE_FLOOR_HEIGHT-30, width:80, height:16, kind:"platform", color });
    tower.layout.push(floor);
  }
  // Add win pad at the top
  tower.winPad = { x: CANVAS_W/2-45, y: (tower.floors)*BASE_FLOOR_HEIGHT-60, width: 90, height:24, color: "#fffa23" };
  return tower;
}


function platformColorForKind(kind, baseColor) {
  if(kind=='lava') return '#e24c3b';
  if(kind=='moving') return colorHexLerp(baseColor,'#3be2be',0.3);
  if(kind=='spinner') return colorHexLerp(baseColor,'#7351a6',0.3);
  if(kind=='button') return '#24b5ff';
  if(kind=='nojump') return '#2a485e';
  if(kind=='hijump') return '#cdea14';
  if(kind=='slow') return '#9458e8';
  return baseColor;
}


// *** Platformer core logic ***
class PlatformerGame {
  constructor(towerDesc) {
    this.tower = towerDesc;
    this.players = [
      this.createPlayer(0,this.spawnPoint(0)),
      this.createPlayer(1,this.spawnPoint(1))
    ];
    this.keys = {};
    this.cameras = [{},{},{},{}]; // Support up to 2 cameras if splitscreen
    this.won = [false,false];
    this.platformEffect = [{},{},{}]; // per player timed effect
    this.lastLavaHit = [0,0];
    this.lastButton = [null,null];
  }
  createPlayer(n,pos) {
    return {
      ix: n,
      x: pos.x, y: pos.y,
      vx:0,vy:0, onGround:false,
      width:30, height:46,
      color: PLAYER_TEMPLATE[n].color,
      hp: 100,
      maxHp: 100,
      regen: 12, // hp per 2.5s
      lastRegen: 0,
      dead: false,
      eff: {}, // 'nojump', 'hijump', 'slow' timed (effects)
      timeEff: {},
      winTime: null,
      floor: 0, // last touched floor
    };
  }
  spawnPoint(n) {
    return { x: 100+380*n, y: (this.tower.floors-1)*BASE_FLOOR_HEIGHT-60};
  }

  // Returns player's tower floor index (0 at base, increases up)
  playerFloor(player) {
    let f = Math.floor((player.y+player.height/2)/BASE_FLOOR_HEIGHT);
    return clamp(f,0,this.tower.floors-1);
  }

  // Updates logic and applies controls, physics, mechanics per frame
  update(dt,now) {
    for (let p=0; p<2; ++p) {
      let player = this.players[p];
      if (this.won[p] || player.dead) continue;
      // Regen timer
      if(now - player.lastRegen > 1200 && player.hp < player.maxHp) {
        player.hp = clamp(player.hp+player.regen,0,player.maxHp);
        player.lastRegen = now;
      }
      // Platformer movement
      let eff = player.eff;
      let moveSpeed = eff.slow ? 2.6 : 4.6;
      let jumpVel = eff.hijump ? -14.2 : -11.5;

      let left = this.keys[PLAYER_TEMPLATE[p].controls.left], right = this.keys[PLAYER_TEMPLATE[p].controls.right];
      let jump = this.keys[PLAYER_TEMPLATE[p].controls.jump];
      // Horizontal move
      if (left && !right) player.vx = -moveSpeed;
      else if (right && !left) player.vx = moveSpeed;
      else player.vx *= 0.6;
      // Gravity
      player.vy += 0.7;
      // Jump
      if (jump && player.onGround && !eff.nojump) {
        player.vy = jumpVel; player.onGround = false;
      }
      // Clamp velocities
      player.vx = clamp(player.vx,-9,9);
      player.vy = clamp(player.vy,-20,14);

      // Detect platforms below (per current floor only for perf/scaling)
      let floorNo = this.playerFloor(player);
      player.floor = floorNo;
      let platforms = this.tower.layout[floorNo] || [];

      let grounded = false, c = 0;

      // Horizontal move/collision
      player.x += player.vx;
      player.x = clamp(player.x,0, CANVAS_W-player.width);
      // Vertical move/collision
      let oldY = player.y;
      player.y += player.vy;
      // If going below 0 (fall), reset to base
      if (player.y > this.tower.floors*BASE_FLOOR_HEIGHT+50) {
        Object.assign(player, this.createPlayer(p,this.spawnPoint(p)));
      }
      // Collision with platforms
      for (let plat of platforms) {
        if (rectHit(player, plat)) {
          // Lava: damage and knock away
          if (plat.kind=='lava') {
            if(now-this.lastLavaHit[p]>580) {
              player.hp -= 5;
              this.lastLavaHit[p] = now;
              // Knock up
              player.vy = -5;
            }
          } 
          // Button triggers
          else if (plat.kind=='button') {
            if(this.lastButton[p]!==plat) {
              // Spawn temp platform nearby for demo
              this.tower.layout[floorNo].push({
                x: plat.x+90, y: plat.y-30, width: 86, height:12, kind:"platform", color: "#54f3e2", exptime: now+4800 });
              this.lastButton[p]=plat;
            }
          }
          // Effects: timed
          else if (['nojump','hijump','slow'].includes(plat.kind)) {
            player.eff[plat.kind]=true; 
            player.timeEff[plat.kind]=now+2900;
          }
          // Ride on top
          else if (player.vy>=0 && oldY+player.height<=plat.y+4) {
            player.y = plat.y-player.height; 
            player.vy = 0;
            grounded = true; 
          }
          // Bump head
          else if (player.vy<0 && oldY>plat.y+plat.height-8) {
            player.y = plat.y+plat.height+0.1; player.vy = 0.4;
          }
        }
      }
      // Timed removal for temp platforms
      for(let i=platforms.length-1;i>=0;--i){
        if(platforms[i].exptime && platforms[i].exptime<now) platforms.splice(i,1);
      }
      player.onGround = grounded;
      // Floor boundaries (no falling through base/top)
      player.y = clamp(player.y, -10, this.tower.floors*BASE_FLOOR_HEIGHT-12);
      // Remove expired effects
      for (let key in player.timeEff) {
        if (player.timeEff[key] && player.timeEff[key]<now) {
          delete player.eff[key]; delete player.timeEff[key];
        }
      }
      // Check for win pad
      if (
        player.y+player.height>this.tower.winPad.y &&
        player.y< this.tower.winPad.y+this.tower.winPad.height &&
        player.x+player.width>this.tower.winPad.x &&
        player.x< this.tower.winPad.x+this.tower.winPad.width
      ) {
        // Win!
        this.won[p]=true; player.winTime=Date.now()-timerStart; winningPlayers[p]=true;
        completionTimes[p]=player.winTime;
      }
      // Death check
      if(player.hp <= 0) {
        player.dead=true;
      }

    }
  }
}

// Rectangular collision check
function rectHit(a,b) {
  return a.x+0.1<a.width+b.x &&
         a.x+b.width>a.x &&
         a.y+0.1<a.height+b.y &&
         a.y+b.height>a.y;
}

function drawPlayer(ctx,p,n,camera) {
  ctx.save();
  ctx.fillStyle=p.color;
  ctx.fillRect(p.x-camera.x,p.y-camera.y,p.width,p.height);
  // Health bar
  ctx.fillStyle="#2e212f";
  ctx.fillRect(p.x-camera.x,p.y-camera.y-13,p.width,10);
  ctx.fillStyle=p.hp>60?'#38e41c':(p.hp>24?'#f2b90a':'#e13b3b');
  ctx.fillRect(p.x-camera.x,p.y-camera.y-13,p.width*p.hp/p.maxHp,10);
  ctx.font="bold 16px monospace"; ctx.fillStyle="#fff";
  ctx.textAlign="center";
  ctx.fillText(PLAYER_TEMPLATE[n].nickname, p.x-camera.x+p.width/2, p.y-camera.y-23);
  ctx.restore();
}

// Platform drawing
function drawPlatform(ctx,plat,camera,spinAngle){
  ctx.save();
  ctx.translate(plat.x-camera.x+plat.width/2, plat.y-camera.y+plat.height/2);
  if(plat.kind==='spinner' && spinAngle) ctx.rotate(spinAngle);
  ctx.fillStyle=platformColorForKind(plat.kind, plat.color);
  ctx.fillRect(-plat.width/2,-plat.height/2,plat.width,plat.height);
  if(plat.kind=='lava') {
    ctx.strokeStyle='#ff7b69'; ctx.lineWidth=2;
    ctx.beginPath();
    ctx.moveTo(-plat.width/2,-plat.height/3);
    for(var i=0; i<plat.width; i+=8){
      ctx.lineTo(i-plat.width/2, Math.random()*8-plat.height/3 + plat.height/2);
    }
    ctx.stroke();
  }
  else if(['nojump','slow','hijump'].includes(plat.kind)){
    ctx.font="bold 16px monospace";
    ctx.fillStyle="#323";
    ctx.textAlign="center";
    ctx.fillText({nojump:"NO JUMP",slow:"SLOW",hijump:"HIGH JUMP"}[plat.kind],0,5);
  }
  else if(plat.kind=='button'){
    ctx.beginPath();
    ctx.arc(0,plat.height/5,plat.width/4,0,2*Math.PI);
    ctx.fillStyle="#0fc";
    ctx.fill();
    ctx.strokeStyle="#001660";
    ctx.stroke();

  }
  ctx.restore();
}

// Camera rectangles: auto split if on diff floors, else single
function computeCameras(platformer) {
  const [p1,p2] = platformer.players;
  if(Math.abs(platformer.playerFloor(p1)-platformer.playerFloor(p2))>0 ||
     Math.abs(p1.x-p2.x)>650
  ) {
    splitScreen=true;
    // Two mini-cameras: top/bottom
    return [
      { x: clamp(p1.x-CANVAS_W/2.5,0,CANVAS_W- CANVAS_W/2), y: clamp(p1.y-170,0, platformer.tower.floors*BASE_FLOOR_HEIGHT-340) },
      { x: clamp(p2.x-CANVAS_W/2.5,0,CANVAS_W- CANVAS_W/2), y: clamp(p2.y-170,0, platformer.tower.floors*BASE_FLOOR_HEIGHT-340) }
    ];
  } else {
    splitScreen=false;
    // Shared camera centered on mid of both
    const cx = (p1.x+p2.x)/2, cy=(p1.y+p2.y)/2;
    return [
      {
        x:clamp(cx-CANVAS_W/2,0,CANVAS_W- CANVAS_W), 
        y:clamp(cy- CANVAS_H/2,0, platformer.tower.floors*BASE_FLOOR_HEIGHT- CANVAS_H)
      }
    ];
  }
}

// *** Main Game Render Loop ***
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

function renderGame(now) {
  ctx.clearRect(0,0,canvas.width,canvas.height);
  let camers = computeCameras(platformer);
  let time = timerStart ? (Date.now()-timerStart)/1000 : 0;
  let spinAngle = (now/700)% (2*Math.PI);
  // For split-screen
  if(splitScreen) {
    for(let scr=0;scr<2;++scr){
      ctx.save();
      // Clip to half screen
      ctx.beginPath();
      ctx.rect(scr*CANVAS_W/2,0,CANVAS_W/2, CANVAS_H);
      ctx.clip();
      // Draw background gradient per floor
      let cam = camers[scr];
      drawTowerBackground(ctx, platformer.tower,cam);
      drawFloorMarkers(ctx,platformer.tower,cam);
      // Draw platforms
      let startF = Math.max(0, Math.floor(cam.y/BASE_FLOOR_HEIGHT)-1),
          endF = Math.min(platformer.tower.floors-1, Math.floor((cam.y+CANVAS_H)/BASE_FLOOR_HEIGHT)+2);
      for(let f=startF;f<=endF;++f){
        for(let plat of platformer.tower.layout[f]) {
          drawPlatform(ctx,plat, cam, plat.kind=="spinner"?spinAngle:0);
        }
      }
      // Draw win pad
      let wp= platformer.tower.winPad;
      ctx.save();
      ctx.fillStyle="#fffa23";
      ctx.globalAlpha=0.85;
      ctx.fillRect(wp.x-cam.x,wp.y-cam.y,wp.width,wp.height);
      ctx.font="bold 18px monospace";
      ctx.fillStyle="#242e33";
      ctx.globalAlpha=1.0;
      ctx.fillText("WIN PAD",wp.x-cam.x+wp.width/2,wp.y-cam.y+wp.height/2+5);
      ctx.restore();
      // Draw player
      let cc = platformer.players[scr];
      drawPlayer(ctx,cc,scr,cam);
      // Health/Timer overlay
      ctx.font="16px monospace";
      ctx.fillStyle="#fff";
      ctx.fillText(`Time: ${ (completionTimes[scr]) ? (completionTimes[scr]/1000).toFixed(2) : time.toFixed(2)}s`, scr*CANVAS_W/2+24,37);
      ctx.restore();
    }
  } else {
    // Single camera
    let cam = camers[0];
    drawTowerBackground(ctx,platformer.tower,cam);
    drawFloorMarkers(ctx,platformer.tower,cam);
    for (let f=0;f<platformer.tower.floors;++f) {
      for (let plat of platformer.tower.layout[f]) {
        drawPlatform(ctx,plat,cam,plat.kind=="spinner"?spinAngle:0);
      }
    }
    // Win pad
    let wp= platformer.tower.winPad;
    ctx.save();
    ctx.fillStyle="#fffa23";
    ctx.globalAlpha=0.82;
    ctx.fillRect(wp.x-cam.x,wp.y-cam.y,wp.width,wp.height);
    ctx.font="bold 18px monospace";
    ctx.fillStyle="#242e33";
    ctx.globalAlpha=1.0;
    ctx.fillText("WIN PAD",wp.x-cam.x+wp.width/2,wp.y-cam.y+wp.height/2+5);
    ctx.restore();
    // Both players
    for(let p=0;p<2;++p) {
      drawPlayer(ctx,platformer.players[p],p,cam);
      ctx.font="17px monospace";
      ctx.fillStyle="#fff";
      ctx.fillText(`HP: ${platformer.players[p].hp}`,32,35+30*p);
      ctx.fillText(`Time: ${ (completionTimes[p]) ? (completionTimes[p]/1000).toFixed(2) : time.toFixed(2)}s`, 150,35+30*p);
    }
  }
}

function drawTowerBackground(ctx, tower, cam){
  ctx.save();
  let f0 = Math.floor(cam.y/BASE_FLOOR_HEIGHT);
  let colorA = tower.floorColors[f0%tower.floorColors.length],
      colorB = tower.floorColors[(f0+1)%tower.floorColors.length];
  // Vertical gradient per visible window
  let grad = ctx.createLinearGradient(0,cam.y,0,cam.y+CANVAS_H);
  grad.addColorStop(0,colorA); grad.addColorStop(1,colorB);
  ctx.fillStyle=grad;
  ctx.fillRect(0,0,CANVAS_W,CANVAS_H);
  ctx.restore();
}
function drawFloorMarkers(ctx, tower, cam){
  ctx.save();
  for(let f=0;f<tower.floors;++f){
    let y=f*BASE_FLOOR_HEIGHT-cam.y;
    ctx.strokeStyle="#292f39";
    ctx.beginPath();
    ctx.moveTo(0,y); ctx.lineTo(CANVAS_W,y);
    ctx.stroke();
    ctx.font="bold 22px monospace";
    ctx.fillStyle="#fff";
    ctx.globalAlpha=0.21;
    ctx.fillText(`F${tower.floors-f}`,CANVAS_W-58,y+22);
    ctx.globalAlpha=1.0;
  }
  ctx.restore();
}

// *** Cutscene ***
function renderCutscene(now){
  ctx.save();
  // Animated zoom
  let z = lerp(1.5,1.0,Math.min(cutsceneTimer/1.85,1));
  ctx.translate(CANVAS_W/2, CANVAS_H/2);
  ctx.scale(z,z);
  ctx.translate(-CANVAS_W/2, -CANVAS_H/2);
  // Background
  let colA=currentTower.floorColors[0], colB=currentTower.floorColors[1];
  let grad = ctx.createLinearGradient(0,0,0,CANVAS_H);
  grad.addColorStop(0,colA); grad.addColorStop(1,colB);
  ctx.fillStyle=grad; ctx.fillRect(0,0,CANVAS_W,CANVAS_H);
  ctx.font="bold 84px monospace";
  ctx.textAlign="center"; ctx.fillStyle="#fff";
  ctx.globalAlpha=lerp(1.8,0.1,cutsceneTimer/2.7);
  ctx.fillText(currentTower.name,CANVAS_W/2,CANVAS_H/2-32);
  ctx.font="bold 30px monospace";
  ctx.globalAlpha=lerp(1.7,0.05,cutsceneTimer/2.4);
  ctx.fillText(`[${currentTower.difficulty}]`, CANVAS_W/2,CANVAS_H/2+52);
  ctx.restore();
}

function renderWinScreen(){
  ctx.save();
  ctx.fillStyle="#1b2b2c"; ctx.globalAlpha=0.95;
  ctx.fillRect(0,0,canvas.width,canvas.height);
  ctx.globalAlpha=1.0;
  ctx.font="bold 54px monospace";
  ctx.textAlign="center";
  ctx.fillStyle="#f7db36";
  ctx.fillText("Congratulations!",CANVAS_W/2,210);
  ctx.font="bold 32px monospace";
  ctx.fillStyle="#fff";
  for(let p=0;m=completionTimes.length,p<2;++p){
    ctx.fillText(`${PLAYER_TEMPLATE[p].nickname}:`,CANVAS_W/2, 282+p*57);
    ctx.fillStyle="#44e433";
    ctx.fillText(`Time: ${completionTimes[p]? (completionTimes[p]/1000).toFixed(2)+"s" : "--"}`,CANVAS_W/2, 314+ p*57);
    ctx.fillStyle="#fff";
  }
  ctx.font="24px monospace";
  ctx.fillStyle="#f49";
  ctx.fillText("Press [R] to replay another random tower...", CANVAS_W/2,CANVAS_H-55);
  ctx.restore();
}

// *** MAIN LOOP ***
function mainLoop() {
  let now = Date.now();
  if(gameState=='cutscene'){
    renderCutscene(now);
    cutsceneTimer += 1/60;
    if(cutsceneTimer>2.15) {
      gameState='play'; timerStart=Date.now();
    }
  }
  else if(gameState=='play'){
    // Update logic
    platformer.update(1/60,now);
    renderGame(now);
    // Win?
    if(winningPlayers[0]&&winningPlayers[1]) {
        gameState='win';
    }
  }
  else if(gameState=='win'){
    renderWinScreen();
  }
  requestAnimationFrame(mainLoop);
}

// KEYS
document.addEventListener('keydown', function(e){
  for(var p=0;p<2;++p){
    let k = PLAYER_TEMPLATE[p].controls;
    if(e.key.toLowerCase()==k.left) platformer.keys[k.left]=1;
    if(e.key.toLowerCase()==k.right) platformer.keys[k.right]=1;
    if(e.key.toLowerCase()==k.jump) platformer.keys[k.jump]=1;
  }
  // Replay on win screen
  if(gameState=='win'&&e.key.toLowerCase()=='r') setupGame();
});
document.addEventListener('keyup', function(e){
  for(var p=0;p<2;++p){
    let k = PLAYER_TEMPLATE[p].controls;
    if(e.key.toLowerCase()==k.left) platformer.keys[k.left]=0;
    if(e.key.toLowerCase()==k.right) platformer.keys[k.right]=0;
    if(e.key.toLowerCase()==k.jump) platformer.keys[k.jump]=0;
  }
});

// Launch a random tower game
function setupGame(){
  cutsceneTimer=0; timerStart=null;
  currentTower = generateTowerLayout(randomTower(), PLATFORM_COUNT_MIN + Math.floor(Math.random()*(PLATFORM_COUNT_MAX-PLATFORM_COUNT_MIN+1)));
  platformer = new PlatformerGame(currentTower);
  completionTimes=[null,null]; winningPlayers=[false,false];
  healthUI=[100,100];
  gameState="cutscene";
}

setupGame();
mainLoop();

</script>
</body>
</html>
