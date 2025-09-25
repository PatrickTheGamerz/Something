<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>2-Player EToH Tower Obby</title>
<style>
  body { margin:0; background:#111; color:#eee; font-family:sans-serif; }
  #menu, #game { width:100vw; height:100vh; display:flex; align-items:center; justify-content:center; }
  #menu { flex-direction:column; gap:20px; }
  #game { display:none; }
  canvas { background:#0e0f14; display:block; }
  button { padding:10px 20px; font-size:18px; cursor:pointer; }
</style>
</head>
<body>

<!-- /menu section -->
<div id="menu">
  <h1>EToH Tower Obby</h1>
  <p>Roll for tower difficulty:</p>
  <button onclick="startGame('easy')">Easy</button>
  <button onclick="startGame('medium')">Medium</button>
  <button onclick="startGame('hard')">Hard</button>
</div>

<!-- /tower section -->
<div id="game">
  <canvas id="canvas"></canvas>
</div>

<script>
// ===== /player1 section & /player2 section =====

// Canvas setup
const canvas = document.getElementById("canvas");
const ctx = canvas.getContext("2d");
function resize(){ canvas.width=window.innerWidth; canvas.height=window.innerHeight; }
window.addEventListener("resize",resize); resize();

// Input
const keys={};
window.addEventListener("keydown",e=>keys[e.key]=true);
window.addEventListener("keyup",e=>keys[e.key]=false);

// Player class
class Player {
  constructor(x,y,color,controls){
    this.x=x; this.y=y; this.w=30; this.h=40;
    this.vx=0; this.vy=0;
    this.color=color;
    this.controls=controls;
    this.onGround=false;
  }
  update(dt,platforms){
    const accel=300, maxSpeed=200, jump=400, gravity=900;
    // Horizontal
    if(keys[this.controls.left]) this.vx=-accel*dt;
    else if(keys[this.controls.right]) this.vx=accel*dt;
    else this.vx=0;
    // Jump
    if(keys[this.controls.up] && this.onGround){
      this.vy=-jump; this.onGround=false;
    }
    // Gravity
    this.vy+=gravity*dt;
    // Move
    this.x+=this.vx; this.y+=this.vy*dt;
    // Collisions
    this.onGround=false;
    for(const p of platforms){
      if(this.x<this.x+this.w && this.x+this.w>p.x &&
         this.y+this.h>p.y && this.y<this.y+p.h){
        // Simple ground collision
        if(this.vy>0 && this.y+this.h-p.y<20){
          this.y=p.y-this.h; this.vy=0; this.onGround=true;
        }
      }
    }
  }
  draw(){
    ctx.fillStyle=this.color;
    ctx.fillRect(this.x,this.y,this.w,this.h);
  }
}

// Tower generator
let platforms=[];
function generateTower(diff){
  platforms=[];
  let step=diff==="easy"?120:diff==="medium"?90:70;
  for(let i=0;i<15;i++){
    let px=Math.random()*(canvas.width-100);
    let py=canvas.height-(i*step+50);
    platforms.push({x:px,y:py,w:100,h:20});
  }
}

// Game state
let players=[];
let last=0;
function loop(ts){
  const dt=(ts-last)/1000; last=ts;
  ctx.clearRect(0,0,canvas.width,canvas.height);
  // Update/draw platforms
  ctx.fillStyle="#888";
  for(const p of platforms) ctx.fillRect(p.x,p.y,p.w,p.h);
  // Update/draw players
  for(const pl of players){ pl.update(dt,platforms); pl.draw(); }
  requestAnimationFrame(loop);
}

// Start game
function startGame(diff){
  document.getElementById("menu").style.display="none";
  document.getElementById("game").style.display="flex";
  generateTower(diff);
  players=[
    new Player(100,canvas.height-100,"#e86f6f",{left:"a",right:"d",up:"w"}),
    new Player(200,canvas.height-100,"#6f9fe8",{left:"ArrowLeft",right:"ArrowRight",up:"ArrowUp"})
  ];
  last=performance.now();
  requestAnimationFrame(loop);
}
</script>
</body>
</html>
