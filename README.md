// One-file, working online build. Save as server.js. Run: `npm i express ws && node server.js`
// Open: http://localhost:8080/?room=test
// Share that URL with another device to play together. Both start with 100 gold.

const express = require('express');
const { WebSocketServer } = require('ws');
const http = require('http');

const PORT = process.env.PORT || 8080;
const app = express();
const server = http.createServer(app);
const wss = new WebSocketServer({ server });

// In-memory rooms
const rooms = new Map();
const nowMs = () => Date.now();

function getOrCreateRoom(roomId){
  if (!rooms.has(roomId)) {
    rooms.set(roomId, {
      clients: new Map(), // clientId -> ws
      S: {
        roomId,
        status: 'waiting',      // 'waiting' | 'build' | 'wave' | 'finished'
        waveNumber: 0,
        seq: 0,
        actions: [],            // authoritative action log
        epochStart: null,       // ms timestamp
        buildDuration: 60,
        waveAutoDuration: 60,
        soloSince: null,
        players: {
          P1: { id: null, name: null, ready: false, skip: false },
          P2: { id: null, name: null, ready: false, skip: false }
        }
      }
    });
  }
  return rooms.get(roomId);
}

function chooseRole(S){
  if (!S.players.P1.id) return 'P1';
  if (!S.players.P2.id) return 'P2';
  return null;
}
function countOccupied(S){
  return (S.players.P1.id?1:0) + (S.players.P2.id?1:0);
}
function bothReady(S){
  return !!(S.players.P1.id && S.players.P2.id && S.players.P1.ready && S.players.P2.ready);
}
function startBuildPhase(room){
  const S = room.S;
  S.status = 'build';
  S.waveNumber = 0;
  S.seq = 0;
  S.actions = [];
  S.epochStart = nowMs();
  S.players.P1.skip = false;
  S.players.P2.skip = false;
}
function endWaveAndAdvance(room){
  const S = room.S;
  S.waveNumber += 1;
  S.status = 'wave';
  S.players.P1.skip = false;
  S.players.P2.skip = false;
  broadcast(room, { type:'state', S });
}
function finishMatch(room, winnerP){
  const S = room.S;
  S.status = 'finished';
  broadcast(room, { type:'state', S, winnerP });
}
function broadcast(room, msg){
  const data = JSON.stringify(msg);
  for (const ws of room.clients.values()){
    try { ws.send(data); } catch {}
  }
}

const INDEX_HTML = `<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1"/>
<title>Dual TD Online</title>
<style>
* { box-sizing: border-box; }
body { margin: 0; background: #0e1330; color: #e6ecff; font-family: Segoe UI, system-ui, sans-serif; }
.topbar, .footer { background: #12193f; padding: 8px 12px; display: flex; align-items: center; justify-content: space-between; }
.topbar .right span { margin-left: 12px; }
.sep { display: inline-block; width: 1px; height: 16px; background: #2a2f4a; margin: 0 8px; }
.layout { display: grid; grid-template-columns: 260px 1fr 260px; gap: 10px; padding: 10px; }
aside { background: #151a46; padding: 10px; border: 1px solid #253067; border-radius: 6px; }
.group { margin-bottom: 12px; }
button { background: #223071; color: #e6ecff; border: 1px solid #304198; padding: 6px 10px; margin: 4px 2px; border-radius: 4px; cursor: pointer; }
button:disabled { opacity: 0.5; cursor: default; }
.stage { position: relative; background: #0e1438; border: 1px solid #253067; border-radius: 6px; padding: 8px; display: flex; align-items: center; justify-content: center; }
#game { display: block; width: 100%; height: auto; max-width: 1200px; border: 1px solid #2a2f4a; background: #0e1438; }
.banner { position: absolute; top: 8px; left: 50%; transform: translateX(-50%); padding: 6px 10px; background: #1a2158; border: 1px solid #2d3aa3; border-radius: 4px; color: #cdd5f8; min-width: 260px; text-align: center; }
.footer .mm > * { margin-right: 8px; }
.warn { color: #ffb86b; }
</style>
</head>
<body>
  <header class="topbar">
    <div class="left">
      <span id="waveHeader">Wave: 0</span>
      <span id="statusHeader"></span>
    </div>
    <div class="right">
      <span>P1 Gold: <b id="g1">100</b></span>
      <span>P1 HP: <b id="h1">100</b></span>
      <span class="sep"></span>
      <span>P2 Gold: <b id="g2">100</b></span>
      <span>P2 HP: <b id="h2">100</b></span>
    </div>
  </header>

  <main class="layout">
    <aside>
      <h3>Player 1</h3>
      <div class="group">
        <button class="act-select" data-p="1" data-t="basic">Select Basic</button>
        <button class="act-select" data-p="1" data-t="sniper">Select Sniper</button>
        <button id="sell1">Sell Mode: Off</button>
      </div>
      <div class="group">
        <h4>Send</h4>
        <button class="act-send" id="send1-normal" data-p="1" data-c="normal">Normal</button>
        <button class="act-send" id="send1-fast" data-p="1" data-c="fast">Fast</button>
        <button class="act-send" id="send1-slow" data-p="1" data-c="slow">Slow</button>
      </div>
    </aside>

    <section class="stage">
      <canvas id="game" width="1200" height="700"></canvas>
      <div id="banner" class="banner"></div>
    </section>

    <aside>
      <h3>Player 2</h3>
      <div class="group">
        <button class="act-select" data-p="2" data-t="basic">Select Basic</button>
        <button class="act-select" data-p="2" data-t="sniper">Select Sniper</button>
        <button id="sell2">Sell Mode: Off</button>
      </div>
      <div class="group">
        <h4>Send</h4>
        <button class="act-send" id="send2-normal" data-p="2" data-c="normal">Normal</button>
        <button class="act-send" id="send2-fast" data-p="2" data-c="fast">Fast</button>
        <button class="act-send" id="send2-slow" data-p="2" data-c="slow">Slow</button>
      </div>
    </aside>
  </main>

  <footer class="footer">
    <div class="mm">
      <span id="mmSeatP1">P1: —</span>
      <span id="mmSeatP2">P2: —</span>
      <button id="readyBtn">Ready</button>
      <button id="skipBtn" disabled>Skip</button>
      <span id="roomInfo"></span>
    </div>
  </footer>

<script>
(() => {
  // URL params
  const params = new URLSearchParams(location.search);
  const ROOM = params.get('room') || 'default';
  const NAME = params.get('name') || \`Player\${Math.floor(Math.random()*1000)}\`;

  // DOM
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const waveHeader = document.getElementById('waveHeader');
  const statusHeader = document.getElementById('statusHeader');
  const banner = document.getElementById('banner');
  const ui = {
    g1: document.getElementById('g1'), g2: document.getElementById('g2'),
    h1: document.getElementById('h1'), h2: document.getElementById('h2'),
    sell1: document.getElementById('sell1'), sell2: document.getElementById('sell2'),
    send1: { normal: document.getElementById('send1-normal'), fast: document.getElementById('send1-fast'), slow: document.getElementById('send1-slow') },
    send2: { normal: document.getElementById('send2-normal'), fast: document.getElementById('send2-fast'), slow: document.getElementById('send2-slow') },
    mmSeatP1: document.getElementById('mmSeatP1'),
    mmSeatP2: document.getElementById('mmSeatP2'),
    readyBtn: document.getElementById('readyBtn'),
    skipBtn: document.getElementById('skipBtn'),
    roomInfo: document.getElementById('roomInfo')
  };

  // World constants
  const WORLD = { w:1200, h:700, laneY:350, half:600 };

  // Server-shared state snapshot
  let S = {
    roomId: ROOM,
    status: 'waiting',
    waveNumber: 0,
    seq: 0,
    actions: [],
    epochStart: null,
    buildDuration: 60,
    waveAutoDuration: 60,
    soloSince: null,
    players: { P1:{ id:null, name:null, ready:false, skip:false }, P2:{ id:null, name:null, ready:false, skip:false } }
  };

  // Client identity
  let MY = { id: null, role: null };

  // Local deterministic state
  const L = {
    time: 0,
    over: false, winner: null,
    bases: { 1:{ x:80, y:350, hp:100 }, 2:{ x:1120, y:350, hp:100 } },
    gold: { 1:100, 2:100 }, // start with 100
    selected: { 1:'basic', 2:'basic' },
    selling: { 1:false, 2:false },
    towers: [], creeps: [], bullets: [],
    seenSeq: 0,
    lastSend: { 1:{}, 2:{} },
    // waves
    curWave: 0,
    waveStartSimT: null,
    waveSchedule: [],
    waveSpawnsDone: false,
    nextWaveSkipAllowedAt: null
  };

  // Defs
  const CreepDefs = {
    normal: { hp: 30, speed: 60, bounty: 6, color: '#a3ff9b', r:12, cost: 8, cd: 1.0, unlockWave: 1 },
    fast:   { hp: 20, speed: 100, bounty: 7, color: '#9bd6ff', r:11, cost: 10, cd: 1.2, unlockWave: 3 },
    slow:   { hp: 60, speed: 45,  bounty: 12, color: '#ffda85', r:13, cost: 18, cd: 1.8, unlockWave: 5 }
  };
  const TowerDefs = {
    basic:  { cost: 20, dmg: 10, fireRate: 0.6, range: 140, color: '#ff6e6e' },
    sniper: { cost: 35, dmg: 18, fireRate: 1.0, range: 220, color: '#9a6eff' }
  };

  // Utils
  const clamp=(v,a,b)=>Math.max(a,Math.min(b,v));
  const dist=(a,b)=>Math.hypot(a.x-b.x,a.y-b.y);
  function randN(n){ let x=(n>>>0)^0x9e3779b9; x=(x^(x>>>16))>>>0; x=Math.imul(x,2246822519)>>>0; x=(x^(x>>>13))>>>0; x=Math.imul(x,3266489917)>>>0; x=(x^(x>>>16))>>>0; return (x>>>0)/4294967296; }

  // UI helpers
  function occupied(role){ return !!(S.players && S.players[role] && S.players[role].id); }
  function skipCount(){ return (S.players.P1.skip?1:0)+(S.players.P2.skip?1:0); }
  function updateSellLabels(){
    ui.sell1.textContent = \`Sell Mode: \${L.selling[1] ? 'On' : 'Off'}\`;
    ui.sell2.textContent = \`Sell Mode: \${L.selling[2] ? 'On' : 'Off'}\`;
  }
  function lockSideControls() {
    const canAct = (S.status==='build' || S.status==='wave') && !L.over && occupied('P1') && occupied('P2');
    const myP = MY.role === 'P1' ? 1 : (MY.role === 'P2' ? 2 : null);

    document.querySelectorAll('.act-select[data-p="1"], .act-send[data-p="1"]').forEach(btn=>{
      btn.disabled = !(canAct && myP === 1);
    });
    ui.sell1.disabled = !(canAct && myP === 1);

    document.querySelectorAll('.act-select[data-p="2"], .act-send[data-p="2"]').forEach(btn=>{
      btn.disabled = !(canAct && myP === 2);
    });
    ui.sell2.disabled = !(canAct && myP === 2);

    updateSendLocks();
  }
  function nowSimTime(){ return S.epochStart ? Math.max(0, (Date.now() - S.epochStart)/1000) : 0; }
  function wsSend(obj){ if (socket && socket.readyState === 1) socket.send(JSON.stringify(obj)); }
  function pushAction(a){
    if (!S.epochStart) return;
    a.clientId = MY.id;
    wsSend({ type:'action', clientId: MY.id, action: a });
  }

  // Wave schedule
  function buildWaveSchedule(waveNum){
    const total = 8 + (waveNum-1)*4;
    const gap = Math.max(0.25, 1.0 - waveNum*0.05);
    const schedule = [];
    let t = 0;
    for (let i=0;i<total;i++){
      const targetPlayer = (i % 2) ? 1 : 2;
      let type = 'normal';
      if (waveNum >= 5 && (i % 5 === 0)) type = 'slow';
      else if (waveNum >= 3 && (i % 3 === 0)) type = 'fast';
      schedule.push({ t, targetPlayer, type });
      t += gap;
    }
    return schedule;
  }
  function spawnCreep(p, type, detIndex){
    const def = CreepDefs[type]; if (!def) return;
    const baseX = p===1 ? 80 : 1120;
    const dir = (p===1)? 1 : -1;
    const y = WORLD.laneY + ( (randN(123456 + detIndex) * 120) - 60 );
    L.creeps.push({
      p, x: baseX + dir*30, y,
      dir, type, hp:def.hp, maxHp:def.hp, speed:def.speed, bounty:def.bounty, color:def.color, r:12
    });
  }
  function nearestOwnTowerIndex(p, x, y, rad){
    let best=-1, bd=rad;
    for (let i=0;i<L.towers.length;i++){
      const t=L.towers[i]; if (t.p!==p) continue;
      const d=Math.hypot(t.x-x,t.y-y);
      if (d<bd){bd=d; best=i;}
    }
    return best;
  }
  window.canSkipWave = function canSkipWave(){
    if (S.status!=='wave') return false;
    return L.waveSpawnsDone && L.nextWaveSkipAllowedAt != null && L.time >= L.nextWaveSkipAllowedAt;
  };

  // Actions application (deterministic)
  function applyPendingActions(){
    for (const a of S.actions || []){
      if (a.seq <= L.seenSeq) continue;
      if (a.t > L.time + 1e-6) continue;

      switch(a.type){
        case 'select': {
          if (S.status!=='build' && S.status!=='wave') break;
          if (!TowerDefs[a.t]) break;
          L.selected[a.p] = a.t;
        } break;

        case 'toggleSell': {
          if (S.status!=='build' && S.status!=='wave') break;
          L.selling[a.p] = !L.selling[a.p];
          updateSellLabels();
        } break;

        case 'send': {
          if (S.status!=='wave') break;
          const def = CreepDefs[a.c]; if (!def) break;
          if (S.waveNumber < def.unlockWave) break;
          const last = L.lastSend[a.p][a.c] || -Infinity;
          if (a.t - last < def.cd - 1e-6) break;
          if (L.gold[a.p] < def.cost) break;
          L.gold[a.p] -= def.cost;
          L.lastSend[a.p][a.c] = a.t;
          spawnCreep(a.p, a.c, a.seq);
        } break;

        case 'place': {
          if (S.status!=='build' && S.status!=='wave') break;
          const type = L.selected[a.p];
          const def = TowerDefs[type]; if (!def) break;
          const xClamped = clamp(a.x, a.p===1?40:WORLD.half+40, a.p===1?WORLD.half-40:WORLD.w-40);
          const yClamped = clamp(a.y, WORLD.laneY-180, WORLD.laneY+180);
          let ok = true;
          for (const t of L.towers) { if (t.p===a.p && dist({x:xClamped,y:yClamped}, t) < 30) { ok=false; break; } }
          if (!ok) break;
          if (L.gold[a.p] < def.cost) break;
          L.gold[a.p] -= def.cost;
          L.towers.push({ p:a.p, x:xClamped, y:yClamped, type, cooldown:0, ...def });
        } break;

        case 'sell': {
          if (S.status!=='build' && S.status!=='wave') break;
          const idx = nearestOwnTowerIndex(a.p, a.x, a.y, 24);
          if (idx !== -1) {
            const t=L.towers[idx];
            const refund = Math.round(TowerDefs[t.type].cost*0.6);
            L.gold[a.p]+=refund;
            L.towers.splice(idx,1);
          }
        } break;

        case 'startWave': {
          if (S.status!=='build') break;
          beginWave(1);
        } break;
      }

      L.seenSeq = Math.max(L.seenSeq, a.seq);
    }
  }

  // Wave control (client-local)
  function beginWave(n){
    if (S.status==='finished') return;
    S.waveNumber = n;
    S.status = 'wave';
    L.curWave = n;
    L.waveSchedule = buildWaveSchedule(n);
    L.waveStartSimT = L.time;
    L.waveSpawnsDone = false;
    L.nextWaveSkipAllowedAt = null;
    L.lastSend[1] = {}; L.lastSend[2] = {};
    renderMM();
  }
  function endWaveAndAdvance(){ wsSend({ type:'advance', clientId: MY.id }); }

  function emitScheduledSpawns(){
    if (S.status !== 'wave' || L.waveStartSimT == null) return;
    const sinceStart = L.time - L.waveStartSimT;
    while (L.waveSchedule.length && L.waveSchedule[0].t <= sinceStart + 1e-6){
      const ev = L.waveSchedule.shift();
      const enemyP = ev.targetPlayer === 1 ? 2 : 1;
      const detIndex = S.waveNumber*10000 + (L.waveSchedule.length);
      spawnCreep(enemyP, ev.type, detIndex);
    }
    if (!L.waveSpawnsDone && L.waveSchedule.length === 0) {
      L.waveSpawnsDone = true;
      L.nextWaveSkipAllowedAt = L.time + 1;
      renderMM();
    }
  }

  function soloWinCheck(){
    const inRound = (S.status==='build' || S.status==='wave');
    const seats = (occupied('P1')?1:0) + (occupied('P2')?1:0);
    if (!inRound) { banner.innerHTML = ''; return; }
    if (seats === 1) {
      banner.innerHTML = \`<span class="warn">Opponent disconnected. Waiting to award win…</span>\`;
    } else {
      banner.innerHTML = '';
    }
  }
  function finishMatch(winnerP){
    L.over = true; L.winner = winnerP;
    renderMM();
  }
  function waveAutoAdvanceCheck(){
    if (S.status!=='wave' || L.waveStartSimT==null) return;
    const elapsed = L.time - L.waveStartSimT;
    const sc = skipCount();
    const divisor = sc >= 1 ? 2 : 1;
    const autoDur = (S.waveAutoDuration || 60) / divisor;
    const canSkipNow = window.canSkipWave();
    if (elapsed >= autoDur) { endWaveAndAdvance(); return; }
    if (canSkipNow && sc === 2) { endWaveAndAdvance(); }
  }

  function updateSendLocks(){
    const w = S.waveNumber;
    const now = L.time;
    // P1
    for (const k of ['normal','fast','slow']){
      const btn = ui.send1[k];
      const def = CreepDefs[k];
      const onWave = w >= def.unlockWave && S.status==='wave';
      const cdRemain = Math.max(0, (L.lastSend[1][k]||-Infinity) + def.cd - now);
      const enoughGold = L.gold[1] >= def.cost;
      btn.disabled = !(onWave && cdRemain <= 0 && enoughGold);
      btn.title = (!onWave ? \`Unlocks on Wave \${def.unlockWave}\` :
                   cdRemain>0 ? \`Cooldown: \${cdRemain.toFixed(2)}s\` :
                   !enoughGold ? \`Need \${def.cost} gold\` : '');
    }
    // P2
    for (const k of ['normal','fast','slow']){
      const btn = ui.send2[k];
      const def = CreepDefs[k];
      const onWave = w >= def.unlockWave && S.status==='wave';
      const cdRemain = Math.max(0, (L.lastSend[2][k]||-Infinity) + def.cd - now);
      const enoughGold = L.gold[2] >= def.cost;
      btn.disabled = !(onWave && cdRemain <= 0 && enoughGold);
      btn.title = (!onWave ? \`Unlocks on Wave \${def.unlockWave}\` :
                   cdRemain>0 ? \`Cooldown: \${cdRemain.toFixed(2)}s\` :
                   !enoughGold ? \`Need \${def.cost} gold\` : '');
    }
  }

  // Input
  document.querySelectorAll('.act-select').forEach(btn=>{
    btn.addEventListener('click',()=>{
      if ((S.status!=='build' && S.status!=='wave') || L.over) return;
      const p = +btn.dataset.p;
      if ((p===1 && MY.role!=='P1') || (p===2 && MY.role!=='P2')) return;
      const t = btn.dataset.t;
      pushAction({ type:'select', p, t });
    });
  });
  document.querySelectorAll('.act-send').forEach(btn=>{
    btn.addEventListener('click',()=>{
      if (S.status!=='wave' || L.over) return;
      const p = +btn.dataset.p;
      if ((p===1 && MY.role!=='P1') || (p===2 && MY.role!=='P2')) return;
      const c = btn.dataset.c;
      pushAction({ type:'send', p, c });
    });
  });
  ui.sell1.onclick=()=>{ if ((S.status==='build' || S.status==='wave') && MY.role==='P1') pushAction({ type:'toggleSell', p:1 }); };
  ui.sell2.onclick=()=>{ if ((S.status==='build' || S.status==='wave') && MY.role==='P2') pushAction({ type:'toggleSell', p:2 }); };
  window.addEventListener('keydown',(e)=>{
    if (e.repeat) return;
    const canBuild = (S.status==='build' || S.status==='wave') && !L.over;
    switch(e.key){
      // P1
      case '1': if (canBuild && MY.role==='P1') pushAction({ type:'select', p:1, t:'basic' }); break;
      case '2': if (canBuild && MY.role==='P1') pushAction({ type:'select', p:1, t:'sniper' }); break;
      case 'a': case 'A': if (canBuild && MY.role==='P1') pushAction({ type:'toggleSell', p:1 }); break;
      case 'q': case 'Q': if (S.status==='wave' && MY.role==='P1') pushAction({ type:'send', p:1, c:'normal' }); break;
      case 'w': case 'W': if (S.status==='wave' && MY.role==='P1') pushAction({ type:'send', p:1, c:'fast' }); break;
      case 'e': case 'E': if (S.status==='wave' && MY.role==='P1') pushAction({ type:'send', p:1, c:'slow' }); break;
      // P2
      case '9': if (canBuild && MY.role==='P2') pushAction({ type:'select', p:2, t:'basic' }); break;
      case '0': if (canBuild && MY.role==='P2') pushAction({ type:'select', p:2, t:'sniper' }); break;
      case 'l': case 'L': if (canBuild && MY.role==='P2') pushAction({ type:'toggleSell', p:2 }); break;
      case 'p': case 'P': if (S.status==='wave' && MY.role==='P2') pushAction({ type:'send', p:2, c:'normal' }); break;
      case 'o': case 'O': if (S.status==='wave' && MY.role==='P2') pushAction({ type:'send', p:2, c:'fast' }); break;
      case 'i': case 'I': if (S.status==='wave' && MY.role==='P2') pushAction({ type:'send', p:2, c:'slow' }); break;
    }
  });
  canvas.addEventListener('click', (e)=>{
    if ((S.status!=='build' && S.status!=='wave') || L.over) return;
    const rect = canvas.getBoundingClientRect();
    const x = (e.clientX - rect.left) * (WORLD.w/rect.width);
    const y = (e.clientY - rect.top)  * (WORLD.h/rect.height);
    const p = x < WORLD.half ? 1 : 2;
    if ((p===1 && MY.role!=='P1') || (p===2 && MY.role!=='P2')) return;
    if (L.selling[p]) { pushAction({ type:'sell', p, x, y }); return; }
    pushAction({ type:'place', p, x, y });
  });

  // Sim
  function step(dt){
    if (L.over) return;

    // towers
    for (const t of L.towers){
      t.cooldown = Math.max(0, t.cooldown - dt);
      if (t.cooldown>0) continue;
      let target=null, best=1e9;
      for (const c of L.creeps){
        if (c.p===t.p) continue;
        const d = dist(t,c);
        if (d<=t.range && d<best){ best=d; target=c; }
      }
      if (target){
        target.hp -= t.dmg;
        L.bullets.push({ x1:t.x, y1:t.y, x2:target.x, y2:target.y, life:0.07, color:t.color });
        t.cooldown = t.fireRate;
        if (target.hp<=0){
          const bounty = (CreepDefs[target.type]?.bounty || 10);
          L.gold[t.p]+= bounty;
        }
      }
    }

    // creeps
    for (let i=L.creeps.length-1;i>=0;i--){
      const c = L.creeps[i];
      c.x += c.dir * c.speed * dt;
      c.y += Math.sign(WORLD.laneY - c.y) * 20 * dt;
      const enemy = c.p===1? 2:1;
      const bx = enemy===1? 80 : 1120, by = 350;
      if (Math.hypot(c.x-bx, c.y-by) < 22){
        L.bases[enemy].hp = clamp(L.bases[enemy].hp - 10, 0, 1000);
        L.creeps.splice(i,1);
        if (L.bases[enemy].hp<=0 && !L.over){
          finishMatch(enemy===1?2:1);
        }
        continue;
      }
      if (c.hp<=0) L.creeps.splice(i,1);
    }

    // bullets
    for (let i=L.bullets.length-1;i>=0;i--){
      const b = L.bullets[i];
      b.life -= dt;
      if (b.life<=0) L.bullets.splice(i,1);
    }
  }

  function draw(){
    if (canvas.width !== WORLD.w || canvas.height !== WORLD.h) {
      canvas.width = WORLD.w; canvas.height = WORLD.h;
    }
    ctx.clearRect(0,0,canvas.width,canvas.height);

    // midline
    ctx.strokeStyle = '#2a2f4a';
    ctx.lineWidth = 2;
    ctx.setLineDash([10,10]);
    ctx.beginPath();
    ctx.moveTo(WORLD.half, 0); ctx.lineTo(WORLD.half, canvas.height); ctx.stroke();
    ctx.setLineDash([]);

    // lane band
    ctx.fillStyle = '#111633';
    ctx.fillRect(0, WORLD.laneY-200, canvas.width, 400);

    // bases
    drawBase(1); drawBase(2);

    // towers
    for (const t of L.towers){
      ctx.fillStyle = t.p===1 ? '#2e86de' : '#f39c12';
      ctx.beginPath(); ctx.arc(t.x,t.y,14,0,Math.PI*2); ctx.fill();
      ctx.strokeStyle = 'rgba(255,255,255,0.06)'; ctx.lineWidth=1;
      ctx.beginPath(); ctx.arc(t.x,t.y,t.range,0,Math.PI*2); ctx.stroke();
      ctx.fillStyle = t.color;
      ctx.beginPath(); ctx.arc(t.x,t.y,6,0,Math.PI*2); ctx.fill();
    }

    // creeps
    for (const c of L.creeps){
      ctx.fillStyle = c.color;
      ctx.beginPath(); ctx.arc(c.x,c.y,c.r,0,Math.PI*2); ctx.fill();
      // hp bar
      const w = 26, h = 4;
      ctx.fillStyle = '#00000080'; ctx.fillRect(c.x-w/2, c.y-c.r-10, w, h);
      ctx.fillStyle = '#6eff6e'; ctx.fillRect(c.x-w/2, c.y-c.r-10, w*(c.hp/c.maxHp), h);
      ctx.strokeStyle = '#00000040'; ctx.strokeRect(c.x-w/2, c.y-c.r-10, w, h);
    }

    // bullets
    for (const b of L.bullets){
      ctx.strokeStyle = b.color; ctx.globalAlpha = Math.max(0, b.life/0.07);
      ctx.lineWidth = 2;
      ctx.beginPath(); ctx.moveTo(b.x1,b.y1); ctx.lineTo(b.x2,b.y2); ctx.stroke();
      ctx.globalAlpha = 1;
    }

    // overlay
    if (L.over){
      ctx.fillStyle = 'rgba(0,0,0,0.55)';
      ctx.fillRect(0,0,canvas.width,canvas.height);
      ctx.fillStyle = '#fff';
      ctx.textAlign='center';
      ctx.font = 'bold 42px Segoe UI';
      ctx.fillText(\`Player \${L.winner} wins!\`, canvas.width/2, canvas.height/2 - 10);
      ctx.font = '16px Segoe UI';
      ctx.fillStyle = '#cdd5f8';
      ctx.fillText('Press Ready to restart', canvas.width/2, canvas.height/2 + 24);
    }

    // HUD
    ui.g1.textContent = Math.floor(L.gold[1]);
    ui.g2.textContent = Math.floor(L.gold[2]);
    ui.h1.textContent = Math.floor(L.bases[1].hp);
    ui.h2.textContent = Math.floor(L.bases[2].hp);
  }

  function drawBase(p){
    const b = L.bases[p];
    ctx.fillStyle = p===1? '#2e86de' : '#f39c12';
    ctx.beginPath(); ctx.arc(b.x,b.y,20,0,Math.PI*2); ctx.fill();
    ctx.strokeStyle = '#ffffff18'; ctx.lineWidth = 3;
    ctx.beginPath(); ctx.arc(b.x,b.y,24,0,Math.PI*2); ctx.stroke();
  }

  function renderMM(){
    const p1 = S.players.P1, p2 = S.players.P2;
    ui.mmSeatP1.textContent = \`P1: \${p1.name ? \`\${p1.name}\${p1.ready?' (ready)':''}\` : '—'}\`;
    ui.mmSeatP2.textContent = \`P2: \${p2.name ? \`\${p2.name}\${p2.ready?' (ready)':''}\` : '—'}\`;
    ui.roomInfo.textContent = \`Room: \${ROOM}\`;
    const meReady = MY.role ? S.players[MY.role].ready : false;
    ui.readyBtn.textContent = meReady ? 'Unready' : 'Ready';
    ui.skipBtn.disabled = !window.canSkipWave();
    lockSideControls();
  }

  // Main loop
  function loop(){
    // Solo indicator (server handles awarding)
    soloWinCheck();

    // Spawns
    if (S.status==='wave') emitScheduledSpawns();

    // Integrate to shared time
    const target = (S.epochStart ? nowSimTime() : L.time);
    const maxStep = 0.016;
    while (L.time + 1e-6 < target) {
      const dt = Math.min(maxStep, target - L.time);
      step(dt);
      L.time += dt;
      if (S.status==='wave') emitScheduledSpawns();
    }

    // Apply actions up to L.time
    applyPendingActions();

    // Build phase: auto-start wave 1 when timer ends; halve remaining if 1 skip; instant if both
    if (S.status==='build') {
      const elapsed = L.time;
      const baseRemain = (S.buildDuration || 60) - elapsed;
      statusHeader.textContent = \` | Build: \${Math.max(0, Math.ceil(baseRemain))}s\`;
      if (skipCount() === 2) {
        if (S.waveNumber===0) beginWave(1);
      } else {
        const effRemain = (skipCount() >= 1) ? baseRemain/2 : baseRemain;
        if (effRemain <= 0 && S.waveNumber===0) beginWave(1);
      }
    }

    // Auto-advance waves and manual skip logic
    if (S.status==='wave') {
      const sinceStart = L.time - (L.waveStartSimT||L.time);
      const sc = skipCount();
      const divisor = sc >= 1 ? 2 : 1;
      const autoDur = (S.waveAutoDuration || 60) / divisor;
      statusHeader.textContent = \` | Wave time: \${Math.max(0, Math.ceil(autoDur - sinceStart))}s\`;
      waveAutoAdvanceCheck();
    }

    waveHeader.textContent = \`Wave: \${S.waveNumber}\`;
    updateSendLocks();

    draw();
    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);

  // Initial UI
  updateSellLabels();
  renderMM();

  // Ready/Skip
  ui.readyBtn.onclick = () => {
    if (!MY.role) return;
    const val = !S.players[MY.role].ready;
    wsSend({ type:'ready', clientId: MY.id, value: val });
  };
  ui.skipBtn.onclick = () => {
    if (!MY.role) return;
    const val = !S.players[MY.role].skip;
    wsSend({ type:'skip', clientId: MY.id, value: val });
  };

  // WebSocket
  let socket;
  function connectWS(){
    const proto = location.protocol === 'https:' ? 'wss' : 'ws';
    const url = \`\${proto}://\${location.host}/ws?room=\${encodeURIComponent(ROOM)}&name=\${encodeURIComponent(NAME)}\`;
    socket = new WebSocket(url);

    socket.onmessage = (ev) => {
      const msg = JSON.parse(ev.data);
      if (msg.type === 'hello') {
        MY.id = msg.clientId;
        MY.role = msg.role;
        S = msg.S || S;
        // Reset local sim on room join
        L.time = 0;
        L.over = false; L.winner = null;
        L.towers = []; L.creeps = []; L.bullets = [];
        L.gold = {1:100,2:100};
        L.seenSeq = 0;
        renderMM();
      } else if (msg.type === 'state') {
        const prevStatus = S.status;
        const prevWave = S.waveNumber;
        S = msg.S || S;
        // Start wave locally when server goes to wave or wave number changes
        if ((prevStatus !== 'wave' && S.status === 'wave') || (prevWave !== S.waveNumber)) {
          beginWave(S.waveNumber);
        }
        if (msg.winnerP) finishMatch(msg.winnerP);
        renderMM();
      } else if (msg.type === 'action') {
        // CRITICAL FIX: append incoming authoritative action to our S.actions
        if (!Array.isArray(S.actions)) S.actions = [];
        S.actions.push(msg.action);
      }
    };
    socket.onclose = () => setTimeout(connectWS, 1000);
  }
  connectWS();
})();
</script>
</body>
</html>`;

app.get('/', (req, res) => {
  res.setHeader('Content-Type', 'text/html; charset=utf-8');
  res.end(INDEX_HTML);
});

app.get('/ws', (req, res) => {
  res.statusCode = 426; // Upgrade Required
  res.end('WebSocket endpoint');
});

// WebSocket handling
wss.on('connection', (ws, req) => {
  try {
    const url = new URL(req.url, `http://${req.headers.host}`);
    const roomId = url.searchParams.get('room') || 'default';
    const name = url.searchParams.get('name') || `Guest${Math.floor(Math.random()*1000)}`;
    const clientId = `${Math.random().toString(36).slice(2)}-${Date.now()}`;

    const room = getOrCreateRoom(roomId);
    const S = room.S;
    room.clients.set(clientId, ws);

    // Seat assignment
    const role = chooseRole(S);
    if (role) {
      S.players[role].id = clientId;
      S.players[role].name = name;
      S.players[role].ready = false;
      S.players[role].skip = false;
    }

    // Hello snapshot
    ws.send(JSON.stringify({ type:'hello', clientId, role: role || null, S }));

    // Notify others
    broadcast(room, { type:'state', S });

    ws.on('message', (raw) => {
      let msg;
      try { msg = JSON.parse(raw.toString()); } catch { return; }

      const S = room.S;
      const myRole = (S.players.P1.id === msg.clientId) ? 'P1' :
                     (S.players.P2.id === msg.clientId) ? 'P2' : null;

      switch (msg.type) {
        case 'ready': {
          if (!myRole) return;
          S.players[myRole].ready = !!msg.value;
          if (bothReady(S) && S.status === 'waiting') {
            startBuildPhase(room);
          }
          broadcast(room, { type:'state', S });
        } break;

        case 'skip': {
          if (!myRole) return;
          S.players[myRole].skip = !!msg.value;
          broadcast(room, { type:'state', S });
        } break;

        case 'action': {
          if (!myRole) return;
          const a = msg.action || {};
          // Side-guard
          if (typeof a.p === 'number') {
            if ((a.p === 1 && myRole !== 'P1') || (a.p === 2 && myRole !== 'P2')) return;
          }
          if (!S.epochStart) return; // Not started yet
          S.seq += 1;
          a.seq = S.seq;
          a.from = msg.clientId;
          a.t = (nowMs() - S.epochStart)/1000;
          S.actions.push(a);
          // Broadcast minimal action update (clients append)
          broadcast(room, { type:'action', action:a, Sseq:S.seq });
        } break;

        case 'advance': {
          endWaveAndAdvance(room);
        } break;
      }

      // Server-side solo-win check
      const inRound = (S.status==='build' || S.status==='wave');
      const seats = countOccupied(S);
      if (!inRound) { S.soloSince = null; }
      else if (seats === 1) {
        if (!S.soloSince) S.soloSince = nowMs();
        const remain = Math.max(0, 10 - (nowMs() - S.soloSince)/1000);
        if (remain <= 0) {
          const winnerRole = S.players.P1.id ? 'P1' : 'P2';
          finishMatch(room, winnerRole === 'P1' ? 1 : 2);
        }
      } else {
        S.soloSince = null;
      }
    });

    ws.on('close', () => {
      const S = room.S;
      room.clients.delete(clientId);
      if (S.players.P1.id === clientId) {
        S.players.P1 = { id: null, name: null, ready: false, skip: false };
      } else if (S.players.P2.id === clientId) {
        S.players.P2 = { id: null, name: null, ready: false, skip: false };
      }

      // Cleanup empty rooms
      if (room.clients.size === 0 && !S.players.P1.id && !S.players.P2.id) {
        rooms.delete(roomId);
        return;
      }
      broadcast(room, { type:'state', S });
    });
  } catch (e) {
    // ignore
  }
});

server.listen(PORT, () => {
  console.log(\`Server running at http://localhost:\${PORT}\`);
});
