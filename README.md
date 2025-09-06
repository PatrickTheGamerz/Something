const http = require('http');
const WebSocket = require('ws');

const PORT = process.env.PORT || 8080;
const TICK = 0.016;

function nowMs() { return Date.now(); }
function quantizeSec(sec) { return Math.round(sec / TICK) * TICK; }

const HTML = `<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<title>Orange TD — online v2.3</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<style>
  :root { --bg:#0f1220; --panel:#171a2b; --accent:#58d68d; --warn:#fca5a5; --p1:#5dade2; --p2:#f5b041; --text:#eaeef6; --sub:#9aa4c7; }
  * { box-sizing:border-box; }
  body { margin:0; background:var(--bg); color:var(--text); font:14px/1.2 system-ui,Segoe UI,Roboto,Arial; }
  #match { display:flex; gap:8px; align-items:center; flex-wrap:wrap; padding:10px; background:#101425; border-bottom:1px solid #2a2f4a; }
  #match input { padding:8px; border-radius:6px; border:1px solid #2a2f4a; background:#0f1220; color:var(--text); min-width:180px; }
  #match button { padding:8px 10px; border-radius:6px; border:1px solid #2a2f4a; background:#1b2038; color:var(--text); cursor:pointer; }
  #match button.primary { background:#15262a; border-color:#254a3d; color:var(--accent); }
  #match button[disabled] { opacity:.55; cursor:not-allowed; }
  #match .pill { padding:4px 8px; border-radius:999px; background:#222742; color:var(--sub); font-size:12px; }
  #match .muted { color:var(--sub); }
  #banner { margin-left:auto; color:var(--sub); }
  header { display:flex; justify-content:space-between; align-items:center; background:var(--panel); padding:8px 12px; border-bottom:1px solid #2a2f4a; }
  header .stats { display:flex; gap:16px; align-items:center; }
  .badge { padding:2px 8px; border-radius:12px; background:#222742; color:var(--sub); }
  .p1 { color:var(--p1); }
  .p2 { color:var(--p2); }
  #wrap { display:grid; grid-template-columns:220px 1fr 220px; height:calc(100vh - 56px - 52px); }
  .panel { background:var(--panel); border-right:1px solid #2a2f4a; padding:10px; overflow:auto; }
  .panel.right { border-right:none; border-left:1px solid #2a2f4a; }
  .panel h3 { margin:0 0 8px; font-size:13px; color:var(--sub); letter-spacing:.4px; text-transform:uppercase; }
  .btn { width:100%; margin:6px 0; padding:8px; border:1px solid #2a2f4a; background:#1b2038; color:#eaeef6; border-radius:6px; cursor:pointer; text-align:left; }
  .btn[disabled] { opacity:.55; cursor:not-allowed; border-color:#2a2f4a; background:#141933; color:#8390b8; }
  .btn:hover:not([disabled]) { background:#20264a; }
  .btn small { color:var(--sub); }
  .hint { color:var(--sub); font-size:12px; margin-top:6px; }
  canvas { display:block; width:100%; height:100%; background:linear-gradient(#0f1220,#0d1122); }
  footer { position:fixed; left:0; right:0; bottom:0; padding:6px 10px; font-size:12px; color:var(--sub); text-align:center; background:rgba(15,18,32,.6); backdrop-filter: blur(4px); }
  .warn { color:var(--warn); }
  .sep { height:6px; }
</style>
</head>
<body>
<div id="match">
  <input id="name" type="text" placeholder="Your name" />
  <button id="toggleReady" class="primary">Ready</button>
  <span class="pill">Status: <strong id="statusText">waiting</strong></span>
  <span class="pill">P1: <strong id="p1Name">—</strong> <span class="muted" id="p1Tag"></span></span>
  <span class="pill">P2: <strong id="p2Name">—</strong> <span class="muted" id="p2Tag"></span></span>
  <span class="pill">Version: <strong>2.3</strong></span>
  <div id="banner"></div>
</div>
<header>
  <div class="stats">
    <strong class="p1">Player 1</strong>
    <span class="badge">Gold: <span id="g1">100</span></span>
    <span class="badge">Base: <span id="h1">100</span> HP</span>
  </div>
  <div><strong id="waveHeader">Wave: 0</strong></div>
  <div class="stats" style="justify-content:flex-end;">
    <span class="badge">Gold: <span id="g2">100</span></span>
    <span class="badge">Base: <span id="h2">100</span> HP</span>
    <strong class="p2">Player 2</strong>
  </div>
</header>
<div id="wrap">
  <div class="panel">
    <h3>Build (P1)</h3>
    <button class="btn act-select" data-p="1" data-t="basic" id="build1-basic">Basic Tower — 50g<br><small>Range 120 • Dmg 1 • 1.1s</small></button>
    <button class="btn act-select" data-p="1" data-t="sniper" id="build1-sniper">Sniper Tower — 100g<br><small>Range 300 • Dmg 4 • 4.0s</small></button>
    <div class="sep"></div>
    <h3>Send (P1)</h3>
    <button class="btn act-send" data-p="1" data-c="normal" id="send1-normal">Send Normal — 25g<br><small>CD 0.5s • Unlock W2</small></button>
    <button class="btn act-send" data-p="1" data-c="fast"   id="send1-fast">Send Fast — 30g<br><small>CD 0.85s • Unlock W3</small></button>
    <button class="btn act-send" data-p="1" data-c="slow"   id="send1-slow">Send Slow — 50g<br><small>CD 1.0s • Unlock W5</small></button>
    <div class="hint">Place on left half. Keys: [1]=Basic, [2]=Sniper, [Q]=Normal, [W]=Fast, [E]=Slow. Toggle Sell [A].</div>
    <button class="btn" id="sell1">Sell Mode: Off</button>
  </div>
  <canvas id="game"></canvas>
  <div class="panel right">
    <h3>Build (P2)</h3>
    <button class="btn act-select" data-p="2" data-t="basic" id="build2-basic">Basic Tower — 50g<br><small>Range 120 • Dmg 1 • 1.1s</small></button>
    <button class="btn act-select" data-p="2" data-t="sniper" id="build2-sniper">Sniper Tower — 100g<br><small>Range 300 • Dmg 4 • 4.0s</small></button>
    <div class="sep"></div>
    <h3>Send (P2)</h3>
    <button class="btn act-send" data-p="2" data-c="normal" id="send2-normal">Send Normal — 25g<br><small>CD 0.5s • Unlock W2</small></button>
    <button class="btn act-send" data-p="2" data-c="fast"   id="send2-fast">Send Fast — 30g<br><small>CD 0.85s • Unlock W3</small></button>
    <button class="btn act-send" data-p="2" data-c="slow"   id="send2-slow">Send Slow — 50g<br><small>CD 1.0s • Unlock W5</small></button>
    <div class="hint">Place on right half. Keys: [9]=Basic, [0]=Sniper, [P]=Normal, [O]=Fast, [I]=Slow. Toggle Sell [L].</div>
    <button class="btn" id="sell2">Sell Mode: Off</button>
  </div>
</div>
<footer>
  Open this page on two devices to play. Click Ready in both. 60s build phase before wave 1. Waves auto-advance. Gold from kills and base hits.
</footer>
<script>
const TICK=0.016;
const TowerDefs={basic:{cost:50,range:120,dmg:1,fireRate:1.1,color:'#8fd3fe'},sniper:{cost:100,range:300,dmg:4,fireRate:4.0,color:'#b9ff8a'}};
const CreepDefs={normal:{cost:25,hp:6,speed:40,bounty:10,color:'#ff6f61',unlockWave:2,cd:0.5},fast:{cost:30,hp:5,speed:60,bounty:20,color:'#ffd061',unlockWave:3,cd:0.85},slow:{cost:50,hp:18,speed:25,bounty:35,color:'#9c7bff',unlockWave:5,cd:1.0}};
const UI={name:document.getElementById('name'),toggleReady:document.getElementById('toggleReady'),statusText:document.getElementById('statusText'),p1Name:document.getElementById('p1Name'),p2Name:document.getElementById('p2Name'),p1Tag:document.getElementById('p1Tag'),p2Tag:document.getElementById('p2Tag'),banner:document.getElementById('banner'),waveHeader:document.getElementById('waveHeader'),g1:document.getElementById('g1'),g2:document.getElementById('g2'),h1:document.getElementById('h1'),h2:document.getElementById('h2'),sell1:document.getElementById('sell1'),sell2:document.getElementById('sell2'),send1:{normal:document.getElementById('send1-normal'),fast:document.getElementById('send1-fast'),slow:document.getElementById('send1-slow')},send2:{normal:document.getElementById('send2-normal'),fast:document.getElementById('send2-fast'),slow:document.getElementById('send2-slow')},build1:{basic:document.getElementById('build1-basic'),sniper:document.getElementById('build1-sniper')},build2:{basic:document.getElementById('build2-basic'),sniper:document.getElementById('build2-sniper')}};
const canvas=document.getElementById('game');const ctx=canvas.getContext('2d');
function fitCanvas(){const side=220+220+20;const w=Math.max(1800,window.innerWidth-side);const h=Math.max(650,Math.min(860,window.innerHeight-56-52));canvas.width=w;canvas.height=h;}
window.addEventListener('resize',fitCanvas);fitCanvas();
function makeWorld(){return{w:canvas.width,h:canvas.height,laneY:Math.round(canvas.height/2),half:Math.round(canvas.width/2)}}
let WORLD=makeWorld();
const MY={id:sessionStorage.getItem('td_id')||(()=>{const id=(crypto.randomUUID?crypto.randomUUID():(Date.now().toString(36)+Math.random().toString(36).slice(2)));sessionStorage.setItem('td_id',id);return id;})(),role:null};
let S={status:'waiting',players:{P1:{id:null,name:null,ready:false,skip:false},P2:{id:null,name:null,ready:false,skip:false}},epochStart:null,buildDuration:60,waveAutoDuration:60,waveNumber:0,seq:0,actions:[],soloSince:null};
const L={time:0,over:false,winner:null,bases:{1:{x:80,y:0,hp:100},2:{x:0,y:0,hp:100}},gold:{1:100,2:100},selected:{1:'basic',2:'basic'},selling:{1:false,2:false},towers:[],creeps:[],bullets:[],seenSeq:0,lastSend:{1:{},2:{}},curWave:0,waveStartSimT:null,waveSchedule:[],waveSpawnsDone:false,nextWaveSkipAllowedAt:null};
const clamp=(v,a,b)=>Math.max(a,Math.min(b,v));const dist=(a,b)=>Math.hypot(a.x-b.x,a.y-b.y);
function randN(n){let x=(n>>>0)^0x9e3779b9;x=(x^(x>>>16))>>>0;x=Math.imul(x,2246822519)>>>0;x=(x^(x>>>13))>>>0;x=Math.imul(x,3266489917)>>>0;x=(x^(x>>>16))>>>0;return(x>>>0)/4294967296;}
function recalcWorldAnchors(){WORLD=makeWorld();L.bases[1].x=80;L.bases[1].y=WORLD.laneY;L.bases[2].x=WORLD.w-80;L.bases[2].y=WORLD.laneY;}
window.addEventListener('resize',recalcWorldAnchors);recalcWorldAnchors();
let ws=null;
function wsUrl(){const proto=location.protocol==='https:'?'wss':'ws';return proto+'://'+location.host;}
function connect(){ws=new WebSocket(wsUrl());ws.onopen=()=>{send({type:'hello',playerId:MY.id});const nm=(UI.name.value||'').trim();if(nm)send({type:'setName',name:nm});};ws.onmessage=ev=>{const msg=JSON.parse(ev.data);if(msg.type==='assigned'){MY.role=msg.role;renderMM();}else if(msg.type==='state'){S=msg.state;renderMM();}};ws.onclose=()=>{UI.banner.innerHTML='<span class="warn">Disconnected. Reconnecting in 2s…</span>';setTimeout(connect,2000);};}
function send(o){if(ws&&ws.readyState===1)ws.send(JSON.stringify(o));}
connect();
function occupied(role){return!!(S.players[role]?.id)}
function skipCount(){return(S.players.P1.skip?1:0)+(S.players.P2.skip?1:0)}
function nowSimTime(){if(!S.epochStart)return 0;return Math.max(0,(Date.now()-S.epochStart)/1000)}
function pushAction(a){if(!S.epochStart)return;send({type:'action',payload:a})}
function renderMM(){UI.p1Name.textContent=S.players.P1.name||'—';UI.p2Name.textContent=S.players.P2.name||'—';UI.p1Tag.textContent=S.players.P1.ready?'• ready':'';UI.p2Tag.textContent=S.players.P2.ready?'• ready':'';UI.waveHeader.textContent='Wave: '+S.waveNumber;if(S.status==='waiting'){UI.statusText.textContent=(!occupied('P1')||!occupied('P2'))?'matching':'waiting';UI.toggleReady.textContent=(MY.role&&S.players[MY.role].ready)?'Unready':'Ready';UI.banner.textContent=(!occupied('P1')||!occupied('P2'))?'Waiting for opponent…':''}else if(S.status==='build'){UI.statusText.textContent='build';const elapsed=nowSimTime();const baseRemain=Math.max(0,(S.buildDuration||60)-elapsed);const eff=skipCount()>=1?baseRemain/2:baseRemain;UI.banner.textContent='Build phase: '+Math.ceil(Math.max(0,eff))+'s. One Skip halves timer; both Skips start instantly.';const seat=MY.role?S.players[MY.role]:null;UI.toggleReady.textContent=seat&&seat.skip?'Unskip':'Skip';}else if(S.status==='wave'){UI.statusText.textContent='wave';const seat=MY.role?S.players[MY.role]:null;const ok=canSkipWave();UI.toggleReady.textContent=!ok?'Skip (locked)':(seat&&seat.skip?'Unskip wave':'Skip wave');UI.banner.textContent='';}else if(S.status==='finished'){UI.statusText.textContent='finished';UI.toggleReady.textContent='Ready';UI.banner.innerHTML='<span class="warn">Game over. Press Ready to start again.</span>'}lockSideControls();}
UI.toggleReady.addEventListener('click',()=>{const nm=(UI.name.value||'').trim();if(nm)send({type:'setName',name:nm});send({type:'toggleReady'})});
UI.name.addEventListener('change',()=>{const nm=(UI.name.value||'').trim();if(nm)send({type:'setName',name:nm})});
function updateSellLabels(){UI.sell1.textContent='Sell Mode: '+(L.selling[1]?'On':'Off');UI.sell2.textContent='Sell Mode: '+(L.selling[2]?'On':'Off')}
function lockSideControls(){const canAct=(S.status==='build'||S.status==='wave')&&!L.over&&occupied('P1')&&occupied('P2');const myP=MY.role==='P1'?1:MY.role==='P2'?2:null;document.querySelectorAll('.act-select[data-p="1"], .act-send[data-p="1"]').forEach(b=>b.disabled=!(canAct&&myP===1));document.querySelectorAll('.act-select[data-p="2"], .act-send[data-p="2"]').forEach(b=>b.disabled=!(canAct&&myP===2));UI.sell1.disabled=!(canAct&&myP===1);UI.sell2.disabled=!(canAct&&myP===2);updateBuildLocks();updateSendLocks();}
function buildWaveSchedule(n){const total=8+(n-1)*4;const gap=Math.max(0.25,1.0-n*0.05);const schedule=[];let t=0;for(let i=0;i<total;i++){const targetPlayer=(i%2)?1:2;let type='normal';if(n>=5&&i%5===0)type='slow';else if(n>=3&&i%3===0)type='fast';schedule.push({t,targetPlayer,type});t+=gap}return schedule}
function nearestOwnTowerIndex(p,x,y,rad){let best=-1,bd=rad;for(let i=0;i<L.towers.length;i++){const t=L.towers[i];if(t.p!==p)continue;const d=Math.hypot(t.x-x,t.y-y);if(d<bd){bd=d;best=i}}return best}
function spawnCreep(p,type,detIndex){const def=CreepDefs[type];if(!def)return;const baseX=p===1?L.bases[1].x:L.bases[2].x;const dir=p===1?1:-1;const y=WORLD.laneY+((randN(123456+detIndex)*120)-60);L.creeps.push({p,x:baseX+dir*30,y,dir,type,hp:def.hp,maxHp:def.hp,speed:def.speed,bounty:def.bounty,color:def.color,r:12})}
function beginWave(n){if(S.status==='finished')return;S.waveNumber=n;S.status='wave';L.curWave=n;L.waveSchedule=buildWaveSchedule(n);L.waveStartSimT=L.time;L.waveSpawnsDone=false;L.nextWaveSkipAllowedAt=null;L.lastSend[1]={};L.lastSend[2]={};renderMM();send({type:'setWave',waveNumber:n,status:'wave'})}
function endWaveAndAdvance(){beginWave(S.waveNumber+1)}
function emitScheduledSpawns(){if(S.status!=='wave'||L.waveStartSimT==null)return;const since=L.time-L.waveStartSimT;while(L.waveSchedule.length&&L.waveSchedule[0].t<=since+1e-6){const ev=L.waveSchedule.shift();const enemyP=ev.targetPlayer===1?2:1;const detIndex=S.waveNumber*10000+(L.waveSchedule.length);spawnCreep(enemyP,ev.type,detIndex)}if(!L.waveSpawnsDone&&L.waveSchedule.length===0){L.waveSpawnsDone=true;L.nextWaveSkipAllowedAt=L.time+1;renderMM()}}
function canSkipWave(){if(S.status!=='wave')return false;return L.waveSpawnsDone&&L.nextWaveSkipAllowedAt!=null&&L.time>=L.nextWaveSkipAllowedAt}
function waveAutoAdvanceCheck(){if(S.status!=='wave'||L.waveStartSimT==null)return;const elapsed=L.time-L.waveStartSimT;const divisor=skipCount()>=1?2:1;const autoDur=(S.waveAutoDuration||60)/divisor;const ok=canSkipWave();if(elapsed>=autoDur){endWaveAndAdvance();return}if(ok&&skipCount()===2)endWaveAndAdvance()}
function updateSendLocks(){const w=S.waveNumber;const now=L.time;for(const k of ['normal','fast','slow']){const d=CreepDefs[k];const cd1=Math.max(0,(L.lastSend[1][k]||-Infinity)+d.cd-now);const on1=w>=d.unlockWave&&S.status==='wave';const eg1=L.gold[1]>=d.cost;UI.send1[k].disabled=!(on1&&cd1<=0&&eg1&&MY.role==='P1'&&occupied('P1')&&occupied('P2'));UI.send1[k].title=!on1?('Unlocks on Wave '+d.unlockWave):cd1>0?('Cooldown: '+cd1.toFixed(2)+'s'):!eg1?('Need '+d.cost+' gold'):'';const cd2=Math.max(0,(L.lastSend[2][k]||-Infinity)+d.cd-now);const on2=w>=d.unlockWave&&S.status==='wave';const eg2=L.gold[2]>=d.cost;UI.send2[k].disabled=!(on2&&cd2<=0&&eg2&&MY.role==='P2'&&occupied('P1')&&occupied('P2'));UI.send2[k].title=!on2?('Unlocks on Wave '+d.unlockWave):cd2>0?('Cooldown: '+cd2.toFixed(2)+'s'):!eg2?('Need '+d.cost+' gold'):''}}
function updateBuildLocks(){const canBuild=(S.status==='build'||S.status==='wave')&&!L.over&&occupied('P1')&&occupied('P2');for(const t of ['basic','sniper']){const d1=TowerDefs[t],a1=L.gold[1]>=d1.cost;UI.build1[t].disabled=!(canBuild&&MY.role==='P1'&&a1);UI.build1[t].title=a1?'':('Need '+d1.cost+' gold');const d2=TowerDefs[t],a2=L.gold[2]>=d2.cost;UI.build2[t].disabled=!(canBuild&&MY.role==='P2'&&a2);UI.build2[t].title=a2?'':('Need '+d2.cost+' gold')}}
function applyPendingActions(){const acts=S.actions||[];for(const a of acts){if(a.seq<=L.seenSeq)continue;if(a.t>L.time+1e-6)continue;let role=null;if(a.p===1)role='P1';else if(a.p===2)role='P2';const fromMatches=!role||S.players[role].id===a.from;switch(a.type){case'select':{if((S.status!=='build'&&S.status!=='wave')||!fromMatches)break;if(!TowerDefs[a.t])break;L.selected[a.p]=a.t;}break;case'toggleSell':{if((S.status!=='build'&&S.status!=='wave')||!fromMatches)break;L.selling[a.p]=!L.selling[a.p];updateSellLabels();}break;case'send':{if(S.status!=='wave'||!fromMatches)break;const def=CreepDefs[a.c];if(!def)break;if(S.waveNumber<def.unlockWave)break;const last=L.lastSend[a.p][a.c]||-Infinity;if(a.t-last<def.cd-1e-6)break;if(L.gold[a.p]<def.cost)break;L.gold[a.p]-=def.cost;L.lastSend[a.p][a.c]=a.t;spawnCreep(a.p,a.c,a.seq);}break;case'place':{if((S.status!=='build'&&S.status!=='wave')||!fromMatches)break;const type=L.selected[a.p];const def=TowerDefs[type];if(!def)break;const x=clamp(a.x,a.p===1?40:WORLD.half+40,a.p===1?WORLD.half-40:WORLD.w-40);const y=clamp(a.y,WORLD.laneY-220,WORLD.laneY+220);let ok=true;for(const t of L.towers){if(t.p===a.p&&dist({x,y},t)<30){ok=false;break}}if(!ok)break;if(L.gold[a.p]<def.cost)break;L.gold[a.p]-=def.cost;L.towers.push({p:a.p,x,y,type,cooldown:0,...def});}break;case'sell':{if((S.status!=='build'&&S.status!=='wave')||!fromMatches)break;const idx=nearestOwnTowerIndex(a.p,a.x,a.y,24);if(idx!==-1){const t=L.towers[idx];const refund=Math.round(TowerDefs[t.type].cost*0.6);L.gold[a.p]+=refund;L.towers.splice(idx,1);}}break;case'startWave':{if(S.status!=='build')break;if(S.waveNumber===0)beginWave(1);}break;}L.seenSeq=Math.max(L.seenSeq,a.seq)}}
document.querySelectorAll('.act-select').forEach(btn=>{btn.addEventListener('click',()=>{if((S.status!=='build'&&S.status!=='wave')||L.over)return;const p=+btn.dataset.p;if((p===1&&MY.role!=='P1')||(p===2&&MY.role!=='P2'))return;const t=btn.dataset.t;pushAction({type:'select',p,t});});});
document.querySelectorAll('.act-send').forEach(btn=>{btn.addEventListener('click',()=>{if(S.status!=='wave'||L.over)return;const p=+btn.dataset.p;if((p===1&&MY.role!=='P1')||(p===2&&MY.role!=='P2'))return;const c=btn.dataset.c;pushAction({type:'send',p,c});});});
UI.sell1.onclick=()=>{if((S.status==='build'||S.status==='wave')&&MY.role==='P1')pushAction({type:'toggleSell',p:1})};
UI.sell2.onclick=()=>{if((S.status==='build'||S.status==='wave')&&MY.role==='P2')pushAction({type:'toggleSell',p:2})};
window.addEventListener('keydown',e=>{if(e.repeat)return;const canBuild=(S.status==='build'||S.status==='wave')&&!L.over;switch(e.key){case'1':if(canBuild&&MY.role==='P1')pushAction({type:'select',p:1,t:'basic'});break;case'2':if(canBuild&&MY.role==='P1')pushAction({type:'select',p:1,t:'sniper'});break;case'a':case'A':if(canBuild&&MY.role==='P1')pushAction({type:'toggleSell',p:1});break;case'q':case'Q':if(S.status==='wave'&&MY.role==='P1')pushAction({type:'send',p:1,c:'normal'});break;case'w':case'W':if(S.status==='wave'&&MY.role==='P1')pushAction({type:'send',p:1,c:'fast'});break;case'e':case'E':if(S.status==='wave'&&MY.role==='P1')pushAction({type:'send',p:1,c:'slow'});break;case'9':if(canBuild&&MY.role==='P2')pushAction({type:'select',p:2,t:'basic'});break;case'0':if(canBuild&&MY.role==='P2')pushAction({type:'select',p:2,t:'sniper'});break;case'l':case'L':if(canBuild&&MY.role==='P2')pushAction({type:'toggleSell',p:2});break;case'p':case'P':if(S.status==='wave'&&MY.role==='P2')pushAction({type:'send',p:2,c:'normal'});break;case'o':case'O':if(S.status==='wave'&&MY.role==='P2')pushAction({type:'send',p:2,c:'fast'});break;case'i':case'I':if(S.status==='wave'&&MY.role==='P2')pushAction({type:'send',p:2,c:'slow'});break;}});
canvas.addEventListener('click',e=>{if((S.status!=='build'&&S.status!=='wave')||L.over)return;const rect=canvas.getBoundingClientRect();const x=(e.clientX-rect.left)*(canvas.width/rect.width);const y=(e.clientY-rect.top)*(canvas.height/rect.height);const p=x<WORLD.half?1:2;if((p===1&&MY.role!=='P1')||(p===2&&MY.role!=='P2'))return;if(L.selling[p]){pushAction({type:'sell',p,x,y});return}pushAction({type:'place',p,x,y})});
function step(dt){if(L.over)return;for(const t of L.towers){t.cooldown=Math.max(0,t.cooldown-dt);if(t.cooldown>0)continue;let target=null,best=1e9;for(const c of L.creeps){if(c.p===t.p)continue;const d=dist(t,c);if(d<=t.range&&d<best){best=d;target=c}}if(target){target.hp-=t.dmg;L.bullets.push({x1:t.x,y1:t.y,x2:target.x,y2:target.y,life:0.07,color:t.color});t.cooldown=t.fireRate;if(target.hp<=0){const bounty=(CreepDefs[target.type]?.bounty||10);L.gold[t.p]+=bounty}}}
for(let i=L.creeps.length-1;i>=0;i--){const c=L.creeps[i];c.x+=c.dir*c.speed*dt;c.y+=Math.sign(WORLD.laneY-c.y)*20*dt;const enemy=c.p===1?2:1;const bx=enemy===1?L.bases[1].x:L.bases[2].x;const by=WORLD.laneY;if(Math.hypot(c.x-bx,c.y-by)<22){const damage=Math.max(0,Math.min(c.hp,L.bases[enemy].hp));L.bases[enemy].hp=clamp(L.bases[enemy].hp-damage,0,1000);const bounty=(CreepDefs[c.type]?.bounty||10);L.gold[enemy]+=bounty;L.creeps.splice(i,1);if(L.bases[enemy].hp<=0&&!L.over){L.over=true;L.winner=(enemy===1?2:1)}continue}if(c.hp<=0)L.creeps.splice(i,1)}
for(let i=L.bullets.length-1;i>=0;i--){const b=L.bullets[i];b.life-=dt;if(b.life<=0)L.bullets.splice(i,1)}}
function draw(){ctx.clearRect(0,0,canvas.width,canvas.height);ctx.strokeStyle='#2a2f4a';ctx.lineWidth=2;ctx.setLineDash([10,10]);ctx.beginPath();ctx.moveTo(WORLD.half,0);ctx.lineTo(WORLD.half,canvas.height);ctx.stroke();ctx.setLineDash([]);ctx.fillStyle='#111633';ctx.fillRect(0,WORLD.laneY-200,canvas.width,400);drawBase(1);drawBase(2);for(const t of L.towers){ctx.fillStyle=t.p===1?'#2e86de':'#f39c12';ctx.beginPath();ctx.arc(t.x,t.y,14,0,Math.PI*2);ctx.fill();ctx.strokeStyle='rgba(255,255,255,0.06)';ctx.lineWidth=1;ctx.beginPath();ctx.arc(t.x,t.y,t.range,0,Math.PI*2);ctx.stroke();ctx.fillStyle=t.color;ctx.beginPath();ctx.arc(t.x,t.y,6,0,Math.PI*2);ctx.fill()}for(const c of L.creeps){ctx.fillStyle=c.color;ctx.beginPath();ctx.arc(c.x,c.y,c.r,0,Math.PI*2);ctx.fill();const w=26,h=4;ctx.fillStyle='#00000080';ctx.fillRect(c.x-w/2,c.y-c.r-10,w,h);ctx.fillStyle='#6eff6e';ctx.fillRect(c.x-w/2,c.y-c.r-10,w*(c.hp/c.maxHp),h);ctx.strokeStyle='#00000040';ctx.strokeRect(c.x-w/2,c.y-c.r-10,w,h)}for(const b of L.bullets){ctx.strokeStyle=b.color;ctx.globalAlpha=Math.max(0,b.life/0.07);ctx.lineWidth=2;ctx.beginPath();ctx.moveTo(b.x1,b.y1);ctx.lineTo(b.x2,b.y2);ctx.stroke();ctx.globalAlpha=1}if(L.over){ctx.fillStyle='rgba(0,0,0,0.55)';ctx.fillRect(0,0,canvas.width,canvas.height);ctx.fillStyle='#fff';ctx.textAlign='center';ctx.font='bold 42px Segoe UI';ctx.fillText('Player '+L.winner+' wins!',canvas.width/2,canvas.height/2-10);ctx.font='16px Segoe UI';ctx.fillStyle='#cdd5f8';ctx.fillText('Press Ready to restart',canvas.width/2,canvas.height/2+24)}UI.g1.textContent=Math.floor(L.gold[1]);UI.g2.textContent=Math.floor(L.gold[2]);UI.h1.textContent=Math.floor(L.bases[1].hp);UI.h2.textContent=Math.floor(L.bases[2].hp)}
function drawBase(p){const b=L.bases[p];ctx.fillStyle=p===1?'#2e86de':'#f39c12';ctx.beginPath();ctx.arc(b.x,b.y,20,0,Math.PI*2);ctx.fill();ctx.strokeStyle='#ffffff18';ctx.lineWidth=3;ctx.beginPath();ctx.arc(b.x,b.y,24,0,Math.PI*2);ctx.stroke()}
function loop(){if(S.status==='wave')emitScheduledSpawns();const target=S.epochStart?nowSimTime():L.time;const maxStep=TICK;while(L.time+1e-6<target){const dt=Math.min(maxStep,target-L.time);step(dt);L.time+=dt;if(S.status==='wave')emitScheduledSpawns()}applyPendingActions();if(S.status==='build'){const elapsed=L.time;const baseRemain=(S.buildDuration||60)-elapsed;if(skipCount()===2){if(S.waveNumber===0)beginWave(1)}else{const eff=skipCount()>=1?baseRemain/2:baseRemain;if(eff<=0&&S.waveNumber===0)beginWave(1)}}if(S.status==='wave')waveAutoAdvanceCheck();document.getElementById('waveHeader').textContent='Wave: '+S.waveNumber;updateBuildLocks();updateSendLocks();draw();requestAnimationFrame(loop)}
requestAnimationFrame(loop);updateSellLabels();renderMM();
</script>
</body>
</html>`;

function newRoom() {
  return {
    players: { P1: { ws: null, id: null, name: null, ready: false, skip: false }, P2: { ws: null, id: null, name: null, ready: false, skip: false } },
    status: 'waiting',
    epochStart: null,
    buildDuration: 60,
    waveAutoDuration: 60,
    waveNumber: 0,
    seq: 0,
    actions: [],
    soloSince: null
  };
}

const ROOM = newRoom();

function bothReady() { return ROOM.players.P1.ready && ROOM.players.P2.ready; }
function skipCount() { return (ROOM.players.P1.skip ? 1 : 0) + (ROOM.players.P2.skip ? 1 : 0); }

function startRound() {
  ROOM.epochStart = nowMs();
  ROOM.status = 'build';
  ROOM.waveNumber = 0;
  ROOM.seq = 0;
  ROOM.actions = [];
  ROOM.players.P1.skip = false;
  ROOM.players.P2.skip = false;
  ROOM.soloSince = null;
  broadcastState();
}

function pushAction(fromId, a) {
  if (!ROOM.epochStart) return;
  ROOM.seq = (ROOM.seq || 0) + 1;
  a.seq = ROOM.seq;
  a.from = fromId;
  const tSec = quantizeSec((nowMs() - ROOM.epochStart) / 1000);
  a.t = Math.max(0, tSec);
  ROOM.actions.push(a);
  broadcastState();
}

function broadcast(ws, obj) { try { if (ws && ws.readyState === 1) ws.send(JSON.stringify(obj)); } catch {} }

function broadcastState() {
  const state = {
    status: ROOM.status,
    players: {
      P1: { id: ROOM.players.P1.id, name: ROOM.players.P1.name, ready: ROOM.players.P1.ready, skip: ROOM.players.P1.skip },
      P2: { id: ROOM.players.P2.id, name: ROOM.players.P2.name, ready: ROOM.players.P2.ready, skip: ROOM.players.P2.skip }
    },
    epochStart: ROOM.epochStart,
    buildDuration: ROOM.buildDuration,
    waveAutoDuration: ROOM.waveAutoDuration,
    waveNumber: ROOM.waveNumber,
    seq: ROOM.seq,
    actions: ROOM.actions,
    soloSince: ROOM.soloSince
  };
  const payload = { type: 'state', state };
  broadcast(ROOM.players.P1.ws, payload);
  broadcast(ROOM.players.P2.ws, payload);
}

function assignSeat(ws, playerId) {
  let role = null;
  if (!ROOM.players.P1.ws) role = 'P1';
  else if (!ROOM.players.P2.ws) role = 'P2';
  else role = 'P1';
  ROOM.players[role].ws = ws;
  ROOM.players[role].id = playerId;
  ROOM.players[role].ready = false;
  ROOM.players[role].skip = false;
  ws.role = role;
  ws.playerId = playerId;
  broadcast(ws, { type: 'assigned', role });
  broadcastState();
}

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
  res.end(HTML);
});

const wss = new WebSocket.Server({ server });

wss.on('connection', (ws) => {
  ws.isAlive = true;
  ws.on('pong', () => { ws.isAlive = true; });
  ws.on('message', (data) => {
    let msg = null;
    try { msg = JSON.parse(data); } catch { return; }
    if (msg.type === 'hello') {
      const pid = msg.playerId || Math.random().toString(36).slice(2);
      assignSeat(ws, pid);
      return;
    }
    if (!ws.role) return;
    switch (msg.type) {
      case 'setName': {
        const name = (msg.name || 'Player').slice(0, 32);
        ROOM.players[ws.role].name = name;
        broadcastState();
      } break;
      case 'toggleReady': {
        const seat = ROOM.players[ws.role];
        if (ROOM.status === 'waiting') {
          seat.ready = !seat.ready;
          ROOM.players.P1.skip = false; ROOM.players.P2.skip = false;
          if (bothReady()) startRound(); else broadcastState();
        } else if (ROOM.status === 'build') {
          seat.skip = !seat.skip; broadcastState();
          if (skipCount() === 2) pushAction(ws.playerId, { type: 'startWave' });
        } else if (ROOM.status === 'wave') {
          seat.skip = !seat.skip; broadcastState();
        } else if (ROOM.status === 'finished') {
          ROOM.status = 'waiting';
          ROOM.players.P1.ready = false; ROOM.players.P2.ready = false;
          ROOM.actions = []; ROOM.epochStart = null; ROOM.waveNumber = 0;
          broadcastState();
        }
      } break;
      case 'action': {
        pushAction(ws.playerId, msg.payload);
      } break;
      case 'setWave': {
        ROOM.waveNumber = msg.waveNumber | 0;
        ROOM.status = msg.status || ROOM.status;
        broadcastState();
      } break;
    }
  });
  ws.on('close', () => {
    if (!ws.role) return;
    ROOM.players[ws.role].ws = null;
    if (ROOM.status === 'waiting') {
      ROOM.players[ws.role] = { ws: null, id: null, name: null, ready: false, skip: false };
    }
    broadcastState();
  });
});

setInterval(() => {
  wss.clients.forEach(ws => {
    if (!ws.isAlive) return ws.terminate();
    ws.isAlive = false;
    try { ws.ping(); } catch {}
  });
}, 20000);

server.listen(PORT, () => {
  console.log(`Server running at http://localhost:${PORT}`);
});
