<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Last Breath Sans Fight</title>
<meta name="viewport" content="width=device-width, initial-scale=1">
<style>
  body,html {margin:0;height:100%;background:#000;color:#fff;font-family:sans-serif;overflow:hidden}
  canvas {position:absolute;inset:0}
  #dialogueBox {
    position:absolute;bottom:40px;left:50%;transform:translateX(-50%);
    background:rgba(0,0,0,0.7);border:2px solid #fff;border-radius:8px;
    padding:12px 18px;max-width:80%;font-size:18px;display:none;
  }
  #phaseOverlay {
    position:absolute;inset:0;display:flex;align-items:center;justify-content:center;
    font-size:40px;font-weight:bold;color:white;text-shadow:0 0 20px #fff;
    background:rgba(0,0,0,0.8);display:none;
  }
</style>
</head>
<body>
<canvas id="game"></canvas>
<div id="dialogueBox"></div>
<div id="phaseOverlay"></div>
<script>
const canvas=document.getElementById("game"),ctx=canvas.getContext("2d");
function resize(){canvas.width=window.innerWidth;canvas.height=window.innerHeight;}
window.addEventListener("resize",resize);resize();

let phase=1,phaseText=["Not a slacker anymore","The slaughter continues","An enigmatic encounter"];
let dialogueBox=document.getElementById("dialogueBox");
let phaseOverlay=document.getElementById("phaseOverlay");

function showDialogue(text,cb){
  dialogueBox.style.display="block";dialogueBox.textContent="";
  let i=0;let interval=setInterval(()=>{
    dialogueBox.textContent=text.slice(0,i++);
    if(i>text.length){clearInterval(interval);setTimeout(()=>{dialogueBox.style.display="none";if(cb)cb();},1500);}
  },40);
}
function showPhaseOverlay(text,cb){
  phaseOverlay.style.display="flex";phaseOverlay.textContent=text;
  setTimeout(()=>{phaseOverlay.style.display="none";if(cb)cb();},2000);
}

// Simple player objects
class Player{
  constructor(x,color){this.x=x;this.y=canvas.height-80;this.color=color;this.hp=100;this.radius=20;}
  draw(){ctx.fillStyle=this.color;ctx.beginPath();ctx.arc(this.x,this.y,this.radius,0,Math.PI*2);ctx.fill();}
}
let p1=new Player(canvas.width*0.3,"cyan");
let p2=new Player(canvas.width*0.7,"magenta");

// Projectiles
let projectiles=[];
function spawnBone(x,y,vx,vy,owner){projectiles.push({type:"bone",x,y,vx,vy,r:6,owner,life:2});}
function spawnBlasterCircle(target,count=8){
  for(let i=0;i<count;i++){
    let angle=(i/count)*Math.PI*2;
    projectiles.push({type:"blaster",x:target.x+Math.cos(angle)*120,y:target.y+Math.sin(angle)*120,dirX:-Math.cos(angle),dirY:-Math.sin(angle),tele:0.6,fire:0.4});
  }
}

// Phase attacks
function phaseAttacks(){
  if(phase===1 && Math.random()<0.01)spawnBlasterCircle(p2);
  if(phase===2 && Math.random()<0.008)spawnBlasterCircle(p2,12);
  if(phase===3 && Math.random()<0.005){
    // sweeping mega blaster
    projectiles.push({type:"mega",x:0,y:canvas.height/2,dir:1,tele:0.8,fire:0.6});
  }
}

// Update/draw loop
let last=performance.now();
function loop(now){
  let dt=(now-last)/1000;last=now;
  ctx.fillStyle="black";ctx.fillRect(0,0,canvas.width,canvas.height);
  // Draw players
  p1.draw();p2.draw();
  // Update projectiles
  for(let i=projectiles.length-1;i>=0;i--){
    let p=projectiles[i];
    if(p.type==="bone"){
      p.x+=p.vx*dt;p.y+=p.vy*dt;p.life-=dt;
      ctx.fillStyle="white";ctx.beginPath();ctx.arc(p.x,p.y,p.r,0,Math.PI*2);ctx.fill();
      if(p.life<=0)projectiles.splice(i,1);
    }else if(p.type==="blaster"){
      if(p.tele>0){p.tele-=dt;ctx.strokeStyle="cyan";ctx.beginPath();ctx.moveTo(p.x,p.y);ctx.lineTo(p.x+p.dirX*150,p.y+p.dirY*150);ctx.stroke();}
      else if(p.fire>0){p.fire-=dt;ctx.strokeStyle="white";ctx.lineWidth=8;ctx.beginPath();ctx.moveTo(p.x,p.y);ctx.lineTo(p.x+p.dirX*200,p.y+p.dirY*200);ctx.stroke();ctx.lineWidth=1;}
      else projectiles.splice(i,1);
    }else if(p.type==="mega"){
      if(p.tele>0){p.tele-=dt;ctx.strokeStyle="yellow";ctx.strokeRect(0,p.y-20,canvas.width,40);}
      else if(p.fire>0){p.fire-=dt;ctx.fillStyle="white";ctx.fillRect(0,p.y-30,canvas.width,60);}
      else projectiles.splice(i,1);
    }
  }
  phaseAttacks();
  requestAnimationFrame(loop);
}
requestAnimationFrame(loop);

// Phase transitions by HP thresholds
function checkPhase(){
  if(p2.hp<70 && phase===1){phase=2;showPhaseOverlay(phaseText[1],()=>showDialogue("heh. guess i gotta try harder."));}
  if(p2.hp<40 && phase===2){phase=3;showPhaseOverlay(phaseText[2],()=>showDialogue("...this is where it gets serious."));}
}
setInterval(checkPhase,500);

// Start with intro
showPhaseOverlay(phaseText[0],()=>showDialogue("you're gonna have a bad time."));
</script>
</body>
</html>


















<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Players – Sans Duel Setup</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <style>
    body {
      margin: 0;
      font-family: sans-serif;
      background: radial-gradient(circle at 50% 20%, #1a1a2e, #0f0f1a);
      color: #eee;
    }
    header { text-align: center; padding: 20px; }
    h1 { margin: 0; }
    .grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
      gap: 20px;
      padding: 20px;
      max-width: 1000px;
      margin: auto;
    }
    .card {
      background: #222;
      border-radius: 10px;
      padding: 16px;
      border: 1px solid #444;
    }
    label { display: block; margin-top: 10px; font-size: 0.9rem; }
    input, select {
      width: 100%;
      padding: 8px;
      margin-top: 4px;
      border-radius: 6px;
      border: 1px solid #555;
      background: #111;
      color: #eee;
    }
    input[type="color"] { padding: 0; height: 40px; }
    .actions { text-align: center; margin: 20px; }
    button {
      padding: 12px 20px;
      border: none;
      border-radius: 8px;
      background: #3a7bff;
      color: white;
      font-weight: bold;
      cursor: pointer;
      margin: 0 10px;
    }
    button.secondary { background: #555; }
  </style>
</head>
<body>
  <header>
    <h1>Sans Fight Setup</h1>
    <p>Configure players and choose game mode.</p>
  </header>

  <section class="grid">
    <div class="card">
      <h2>Player One</h2>
      <label>Name <input type="text" id="p1name" value="Player One"></label>
      <label>Color <input type="color" id="p1color" value="#7cf0ff"></label>
    </div>

    <div class="card">
      <h2>Player Two</h2>
      <label>Name <input type="text" id="p2name" value="Player Two"></label>
      <label>Color <input type="color" id="p2color" value="#e9a6ff"></label>
    </div>

    <div class="card">
      <h2>Options</h2>
      <label>Round Time (seconds) <input type="number" id="roundTime" value="90" min="30" max="300"></label>
      <label>Damage Scale <input type="number" id="damageScale" value="1.0" step="0.1" min="0.5" max="2"></label>
      <label>Game Mode
        <select id="gameMode">
          <option value="duel">2‑Player Duel</option>
          <option value="lastbreath">Last Breath Sans (3 Phases)</option>
        </select>
      </label>
    </div>
  </section>

  <div class="actions">
    <button class="secondary" id="resetBtn">Reset</button>
    <button id="startBtn">Start Fight</button>
  </div>

  <script>
    const p1name=document.getElementById("p1name");
    const p2name=document.getElementById("p2name");
    const p1color=document.getElementById("p1color");
    const p2color=document.getElementById("p2color");
    const roundTime=document.getElementById("roundTime");
    const damageScale=document.getElementById("damageScale");
    const gameMode=document.getElementById("gameMode");

    document.getElementById("resetBtn").onclick=()=>{
      p1name.value="Player One";p2name.value="Player Two";
      p1color.value="#7cf0ff";p2color.value="#e9a6ff";
      roundTime.value=90;damageScale.value=1.0;
      gameMode.value="duel";
      localStorage.removeItem("sans_duel_settings");
    };

    document.getElementById("startBtn").onclick=()=>{
      const settings={
        p1:{name:p1name.value||"Player One",color:p1color.value},
        p2:{name:p2name.value||"Player Two",color:p2color.value},
        opts:{
          roundTime:Number(roundTime.value),
          damageScale:Number(damageScale.value),
          mode:gameMode.value
        }
      };
      localStorage.setItem("sans_duel_settings",JSON.stringify(settings));
      window.location.href="Tower.html";
    };
  </script>
</body>
</html>


