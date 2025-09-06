<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<title>Orange TD — v1.26 (dual-synced, waves, matchmaking)</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<style>
  :root {
    --bg: #0f1220; --panel:#171a2b; --accent:#58d68d; --warn:#fca5a5;
    --p1:#5dade2; --p2:#f5b041; --text:#eaeef6; --sub:#9aa4c7;
  }
  * { box-sizing: border-box; }
  body { margin:0; background:var(--bg); color:var(--text); font:14px/1.2 system-ui,Segoe UI,Roboto,Arial; }
  /* Matchmaking bar */
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
  #wrap { display:grid; grid-template-columns:240px 1fr 240px; height:calc(100vh - 56px - 52px); }
  .panel { background:var(--panel); border-right:1px solid #2a2f4a; padding:10px; overflow:auto; }
  .panel.right { border-right:none; border-left:1px solid #2a2f4a; }
  .panel h3 { margin:0 0 8px; font-size:13px; color:var(--sub); letter-spacing:.4px; text-transform:uppercase; }
  .btn { width:100%; margin:6px 0; padding:8px; border:1px solid #2a2f4a; background:#1b2038; color:var(--text); border-radius:6px; cursor:pointer; text-align:left; }
  .btn[disabled] { opacity:.55; cursor:not-allowed; border-color:#2a2f4a; background:#141933; color:#8390b8; }
  .btn:hover:not([disabled]) { background:#20264a; }
  .btn small { color:var(--sub); }
  .row { display:flex; gap:8px; }
  .hint { color:var(--sub); font-size:12px; margin-top:6px; }
  canvas { display:block; width:100%; height:100%; background:linear-gradient(#0f1220,#0d1122); }
  footer { position:fixed; left:0; right:0; bottom:0; padding:6px 10px; font-size:12px; color:var(--sub); text-align:center; background:rgba(15,18,32,.6); backdrop-filter: blur(4px); }
  .warn { color:var(--warn); }
  .sep { height:6px; }
</style>
</head>
<body>

<!-- Matchmaking bar -->
<div id="match">
  <input id="name" type="text" placeholder="Your name" />
  <button id="toggleReady" class="primary">Ready</button>
  <span id="status" class="pill">Status: <strong id="statusText">waiting</strong></span>
  <span class="pill">P1: <strong id="p1Name">—</strong> <span class="muted" id="p1Tag"></span></span>
  <span class="pill">P2: <strong id="p2Name">—</strong> <span class="muted" id="p2Tag"></span></span>
  <span class="pill">Version: <strong>1.26</strong></span>
  <div id="banner"></div>
</div>

<header>
  <div class="stats">
    <strong class="p1">Player 1</strong>
    <span class="badge">Gold: <span id="g1">0</span></span>
    <span class="badge">Base: <span id="h1">100</span> HP</span>
  </div>
  <div><strong id="waveHeader">Wave: 0</strong></div>
  <div class="stats" style="justify-content:flex-end;">
    <span class="badge">Gold: <span id="g2">0</span></span>
    <span class="badge">Base: <span id="h2">100</span> HP</span>
    <strong class="p2">Player 2</strong>
  </div>
</header>

<div id="wrap">
  <div class="panel">
    <h3>Build (P1)</h3>
    <button class="btn act-select" data-p="1" data-t="basic">Basic Tower — 50g<br><small>Range 120 • Dmg 1 • 1.1s</small></button>
    <button class="btn act-select" data-p="1" data-t="sniper">Sniper Tower — 100g<br><small>Range 200 • Dmg 4 • 4.0s</small></button>

    <div class="sep"></div>
    <h3>Send (P1)</h3>
    <button class="btn act-send" data-p="1" data-c="normal" id="send1-normal">Send Normal — 25g<br><small>CD 0.5s • Unlock W2</small></button>
    <button class="btn act-send" data-p="1" data-c="fast"   id="send1-fast">Send Fast — 30g<br><small>CD 0.85s • Unlock W3</small></button>
    <button class="btn act-send" data-p="1" data-c="slow"   id="send1-slow">Send Slow — 50g<br><small>CD 1.0s • Unlock W5</small></button>

    <div class="hint">Place on left half. Keys: [1]=Basic, [2]=Sniper, [Q]=Normal, [W]=Fast, [E]=Slow. Toggle Sell [A].</div>
    <button class="btn" id="sell1">Sell Mode: Off</button>
  </div>

  <canvas id="game" width="1200" height="700"></canvas>

  <div class="panel right">
    <h3>Build (P2)</h3>
    <button class="btn act-select" data-p="2" data-t="basic">Basic Tower — 50g<br><small>Range 120 • Dmg 1 • 1.1s</small></button>
    <button class="btn act-select" data-p="2" data-t="sniper">Sniper Tower — 100g<br><small>Range 200 • Dmg 4 • 4.0s</small></button>

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
  Open in two tabs. Click Ready in both. 60s build phase before wave 1. Waves auto-advance after 60s (30s if one presses Skip; instant if both). Gold only from kills.
</footer>

<script>
/* ================= Shared store (localStorage) ================= */
const KEY = 'orange_td_v126';

// Per-tab identity
const MY = {
  id: sessionStorage.getItem('td_id') || (() => {
    const id = (crypto.randomUUID ? crypto.randomUUID() : (Date.now().toString(36)+Math.random().toString(36).slice(2)));
    sessionStorage.setItem('td_id', id); return id;
  })(),
  role: sessionStorage.getItem('td_role') || null, // 'P1' | 'P2' | null
  name: sessionStorage.getItem('td_name') || '',
  ready: false
};

const TowerDefs = {
  basic:  { cost:50, range:120, dmg:1, fireRate:1.1, color:'#8fd3fe' },
  sniper: { cost:100, range:200, dmg:4, fireRate:4.0, color:'#b9ff8a' }
};
const CreepDefs = {
  normal: { cost:25, hp:6,  speed:40, bounty:10, color:'#ff6f61', unlockWave:2, cd:0.5 },
  fast:   { cost:30, hp:5,  speed:60, bounty:20, color:'#ffd061', unlockWave:3, cd:0.85 },
  slow:   { cost:50, hp:18, speed:25, bounty:35, color:'#9c7bff', unlockWave:5, cd:1.0 }
};

const initialShared = () => ({
  players: {
    P1:{ id:null, name:null, ready:false, skip:false },
    P2:{ id:null, name:null, ready:false, skip:false }
  },
  status: 'waiting',          // waiting | matching | build | wave | finished
  epochStart: null,           // ms timestamp when current round started (build phase)
  buildDuration: 60,          // seconds (pre-wave build time)
  waveNumber: 0,
  waveAutoDuration: 60,       // seconds (auto-advance wave after this time)
  seq: 0,                     // action sequence
  actions: [],                // {seq,t,from,type,...}
  soloSince: null             // ms: when only one seat remained during build/wave
});

const save = s => localStorage.setItem(KEY, JSON.stringify(s));
const load = () => { try { const raw = localStorage.getItem(KEY); return raw ? JSON.parse(raw) : null; } catch { return null; } };

let S = load() || initialShared();
if (!load()) save(S);

/* ================= Matchmaking + UI ================= */
const el = id => document.getElementById(id);
const nameInput = el('name');
const toggleReadyBtn = el('toggleReady');
const statusText = el('statusText');
const p1Name = el('p1Name');
const p2Name = el('p2Name');
const p1Tag = el('p1Tag');
const p2Tag = el('p2Tag');
const banner = el('banner');
const waveHeader = el('waveHeader');

function occupied(role){ return !!(S.players[role].id); }
function bothReady(){ return S.players.P1.ready && S.players.P2.ready; }
function skipCount(){ return (S.players.P1.skip?1:0) + (S.players.P2.skip?1:0); }

function assignRandomSeat(name){
  const roles = ['P1','P2'];
  const free = roles.filter(r => !occupied(r));
  if (free.length === 0) return false;
  const pick = free[Math.floor(Math.random()*free.length)];
  S.players[pick].id = MY.id;
  S.players[pick].name = name;
  S.players[pick].ready = true; // pressing Ready assigns & sets ready
  S.players[pick].skip = false;
  MY.role = pick;
  sessionStorage.setItem('td_role', pick);
  sessionStorage.setItem('td_name', name);
  save(S);
  return true;
}

function nowSimTime(){
  const s = load() || S;
  if (!s.epochStart) return 0;
  return Math.max(0, (Date.now() - s.epochStart)/1000);
}

function pushAction(a){
  const s = load() || S;
  s.seq = (s.seq||0) + 1;
  a.seq = s.seq;
  a.from = MY.id;
  a.t = nowSimTime();
  s.actions.push(a);
  save(s);
  S = s;
}

function startRound() {
  S.epochStart = Date.now();
  S.status = 'build';
  S.waveNumber = 0;
  S.seq = 0;
  S.actions = [];
  S.players.P1.skip = false; S.players.P2.skip = false;
  S.soloSince = null;
  save(S);
}

function toggleReady() {
  const name = (nameInput.value||'').trim() || 'Player';
  S = load() || S;

  // If no seat, try to claim one randomly
  if (!MY.role) {
    const ok = assignRandomSeat(name);
    if (!ok) {
      statusText.textContent = 'matching';
      return;
    }
  } else {
    const seat = S.players[MY.role];
    if (seat.id !== MY.id) { MY.role = null; save(S); renderMM(); return; }

    if (S.status === 'build') {
      seat.skip = !seat.skip;
      save(S);
      if (skipCount() === 2) pushAction({ type:'startWave' });
    } else if (S.status === 'wave') {
      seat.skip = !seat.skip;
      save(S);
    } else {
      seat.ready = !seat.ready;
      seat.skip = false;
      save(S);
    }
  }

  if ((S.status === 'waiting' || S.status === 'matching') && occupied('P1') && occupied('P2') && bothReady()) {
    startRound();
  } else {
    if (!occupied('P1') || !occupied('P2')) {
      S.status = 'matching';
      save(S);
    }
  }

  MY.ready = MY.role ? S.players[MY.role].ready : false;
  renderMM();
}

function renderMM(){
  S = load() || S;

  p1Name.textContent = S.players.P1.name || '—';
  p2Name.textContent = S.players.P2.name || '—';
  p1Tag.textContent = S.players.P1.ready ? '• ready' : '';
  p2Tag.textContent = S.players.P2.ready ? '• ready' : '';
  nameInput.value = MY.name || '';
  waveHeader.textContent = `Wave: ${S.waveNumber}`;

  if (S.status === 'waiting' || S.status === 'matching') {
    statusText.textContent = (!occupied('P1') || !occupied('P2')) ? 'matching' : 'waiting';
    toggleReadyBtn.textContent = MY.role ? (S.players[MY.role].ready ? 'Unready' : 'Ready') : 'Ready';
    toggleReadyBtn.disabled = false;
    banner.textContent = (!occupied('P1') || !occupied('P2')) ? 'Waiting for opponent…' : '';
  } else if (S.status === 'build') {
    statusText.textContent = 'build';
    const elapsed = nowSimTime();
    const baseRemain = Math.max(0, S.buildDuration - elapsed);
    const sc = skipCount();
    const effRemain = sc >= 1 ? baseRemain/2 : baseRemain;
    const remain = Math.max(0, effRemain).toFixed(0);
    banner.textContent = `Build phase: ${remain}s. One Skip halves timer; both Skips start instantly.`;
    const seat = MY.role ? S.players[MY.role] : null;
    toggleReadyBtn.textContent = seat && seat.skip ? 'Unskip' : 'Skip';
    toggleReadyBtn.disabled = !MY.role;
  } else if (S.status === 'wave') {
    statusText.textContent = 'wave';
    const canSkipNow = window.canSkipWave ? window.canSkipWave() : false;
    const seat = MY.role ? S.players[MY.role] : null;
    toggleReadyBtn.textContent = (!canSkipNow) ? 'Skip (locked)' : (seat && seat.skip ? 'Unskip wave' : 'Skip wave');
    toggleReadyBtn.disabled = !MY.role || !canSkipNow;
    banner.textContent = 'Wave in progress…';
  } else if (S.status === 'finished') {
    statusText.textContent = 'finished';
    toggleReadyBtn.textContent = 'Ready';
    toggleReadyBtn.disabled = false;
    banner.innerHTML = '<span class="warn">Game over. Press Ready to start again.</span>';
  }

  // Rejoin prompt (only once)
  if ((S.status === 'build' || S.status === 'wave') && !MY.role && !localStorage.getItem('td_rejoin_asked')) {
    localStorage.setItem('td_rejoin_asked', '1');
    const wants = confirm('A round is in progress. Do you want to join as the open seat?');
    if (wants) {
      const nm = (nameInput.value||'').trim() || 'Player';
      assignRandomSeat(nm);
      renderMM();
    }
  }

  if (window.lockSideControls) window.lockSideControls();
}

toggleReadyBtn.addEventListener('click', toggleReady);

// Sync across tabs
window.addEventListener('storage', (e) => {
  if (e.key === KEY) {
    S = load() || S;
    if (MY.role && S.players[MY.role].id === MY.id) {
      MY.ready = S.players[MY.role].ready;
    } else if (MY.role && S.players[MY.role].id !== MY.id) {
      MY.role = null; MY.ready = false;
      sessionStorage.removeItem('td_role');
    }
    renderMM();
  }
});

// Leave seat on close (soft)
window.addEventListener('beforeunload', () => {
  try {
    const s = load() || S;
    const r = MY.role;
    if (r && s.players[r].id === MY.id && (s.status==='waiting' || s.status==='matching')) {
      s.players[r] = { id:null, name:null, ready:false, skip:false };
      if (s.status!=='finished') s.status='matching';
      localStorage.setItem(KEY, JSON.stringify(s));
    }
  } catch {}
});

/* ================= Deterministic dual simulation (waves) ================= */
(() => {
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const ui = {
    g1: document.getElementById('g1'), g2: document.getElementById('g2'),
    h1: document.getElementById('h1'), h2: document.getElementById('h2'),
    sell1: document.getElementById('sell1'), sell2: document.getElementById('sell2'),
    send1: { normal: document.getElementById('send1-normal'), fast: document.getElementById('send1-fast'), slow: document.getElementById('send1-slow') },
    send2: { normal: document.getElementById('send2-normal'), fast: document.getElementById('send2-fast'), slow: document.getElementById('send2-slow') }
  };

  const WORLD = { w:1200, h:700, laneY:350, half:600 };

  // Local deterministic world state
  const L = {
    time: 0,
    over: false, winner: null,
    bases: { 1:{ x:80, y:350, hp:100 }, 2:{ x:1120, y:350, hp:100 } },
    gold: { 1:0, 2:0 }, // gold only from kills
    selected: { 1:'basic', 2:'basic' },
    selling: { 1:false, 2:false },
    towers: [], creeps: [], bullets: [],
    seenSeq: 0,
    lastSend: { 1:{}, 2:{} },
    // waves
    curWave: 0,
    waveStartSimT: null,   // sim time when current wave began
    waveSchedule: [],      // [{t, targetPlayer, type}]
    waveSpawnsDone: false,
    nextWaveSkipAllowedAt: null // sim time when skip can be pressed (after spawns done + 1s)
  };

  // Utility
  const clamp=(v,a,b)=>Math.max(a,Math.min(b,v));
  const dist=(a,b)=>Math.hypot(a.x-b.x,a.y-b.y);
  function randN(n){ let x=(n>>>0)^0x9e3779b9; x=(x^(x>>>16))>>>0; x=Math.imul(x,2246822519)>>>0; x=(x^(x>>>13))>>>0; x=Math.imul(x,3266489917)>>>0; x=(x^(x>>>16))>>>0; return (x>>>0)/4294967296; }

  function updateSellLabels(){
    ui.sell1.textContent = `Sell Mode: ${L.selling[1] ? 'On' : 'Off'}`;
    ui.sell2.textContent = `Sell Mode: ${L.selling[2] ? 'On' : 'Off'}`;
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
  window.lockSideControls = lockSideControls;

  function nowSimTime(){
    const s = load() || S;
    if (!s.epochStart) return 0;
    return Math.max(0, (Date.now() - s.epochStart)/1000);
  }
  function pushAction(a){
    const s = load() || S;
    if (!s.epochStart) return;
    s.seq = (s.seq || 0) + 1;
    a.seq = s.seq;
    a.from = MY.id;
    a.t = nowSimTime();
    s.actions.push(a);
    save(s);
    S = s;
  }

  // Build wave schedule deterministically
  function buildWaveSchedule(waveNum){
    const total = 8 + (waveNum-1)*4;            // more enemies each wave
    const gap = Math.max(0.25, 1.0 - waveNum*0.05); // spawn faster for higher waves
    const schedule = [];
    let t = 0;
    for (let i=0;i<total;i++){
      const targetPlayer = (i % 2) ? 1 : 2; // alternate lanes
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

  // Expose canSkipWave for UI disabling/labels
  window.canSkipWave = function canSkipWave(){
    if (S.status!=='wave') return false;
    return L.waveSpawnsDone && L.nextWaveSkipAllowedAt != null && L.time >= L.nextWaveSkipAllowedAt;
  };

  // Apply shared actions deterministically
  function applyPendingActions(){
    const s = load() || S; S = s;
    for (const a of s.actions){
      if (a.seq <= L.seenSeq) continue;
      if (a.t > L.time + 1e-6) continue;

      let role = null;
      if (a.p === 1) role = 'P1';
      else if (a.p === 2) role = 'P2';

      switch(a.type){
        case 'select': {
          if (S.status!=='build' && S.status!=='wave') break;
          if (role && s.players[role].id !== a.from) break;
          if (!TowerDefs[a.t]) break;
          L.selected[a.p] = a.t;
        } break;

        case 'toggleSell': {
          if (S.status!=='build' && S.status!=='wave') break;
          if (role && s.players[role].id !== a.from) break;
          L.selling[a.p] = !L.selling[a.p];
          updateSellLabels();
        } break;

        case 'send': {
          if (S.status!=='wave') break;
          if (role && s.players[role].id !== a.from) break;
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
          if (role && s.players[role].id !== a.from) break;
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
          if (role && s.players[role].id !== a.from) break;
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

  function beginWave(n){
    if (S.status==='finished') return;
    S.waveNumber = n;
    S.status = 'wave';
    save(S);
    // local init
    L.curWave = n;
    L.waveSchedule = buildWaveSchedule(n);
    L.waveStartSimT = L.time;
    L.waveSpawnsDone = false;
    L.nextWaveSkipAllowedAt = null;
    L.lastSend[1] = {}; L.lastSend[2] = {};
    const s = load() || S; s.players.P1.skip=false; s.players.P2.skip=false; save(s); S=s;
    renderMM();
  }

  function endWaveAndAdvance(){
    const next = S.waveNumber + 1;
    beginWave(next);
  }

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
      L.nextWaveSkipAllowedAt = L.time + 1; // allow skip 1s after last spawn
      const s = load() || S; s.players.P1.skip=false; s.players.P2.skip=false; save(s); S = s;
      renderMM();
    }
  }

  // One-player-left auto-win: 10s countdown
  function soloWinCheck(){
    const inRound = (S.status==='build' || S.status==='wave');
    const seats = (occupied('P1')?1:0) + (occupied('P2')?1:0);
    if (!inRound) { if (S.soloSince) { S.soloSince=null; save(S);} return; }
    if (seats === 1) {
      if (!S.soloSince) { S.soloSince = Date.now(); save(S); }
      const remain = Math.max(0, 10 - (Date.now() - S.soloSince)/1000);
      banner.innerHTML = `<span class="warn">Opponent disconnected. Awarding win in ${Math.ceil(remain)}s if they don’t return…</span>`;
      if (remain <= 0) {
        const winnerRole = occupied('P1') ? 'P1' : 'P2';
        finishMatch(winnerRole === 'P1' ? 1 : 2);
      }
    } else {
      if (S.soloSince) { S.soloSince=null; save(S); }
    }
  }

  function finishMatch(winnerP){
    L.over = true; L.winner = winnerP;
    const s = load() || S; s.status='finished'; save(s); S=s;
    renderMM();
  }

  // Auto-advance logic (with skip division)
  function waveAutoAdvanceCheck(){
    if (S.status!=='wave' || L.waveStartSimT==null) return;
    const elapsed = L.time - L.waveStartSimT;
    const sc = skipCount();
    const divisor = sc >= 1 ? 2 : 1; // halve timer if at least one player skips
    const autoDur = (S.waveAutoDuration || 60) / divisor;
    const canSkipNow = window.canSkipWave();

    // Auto-advance after autoDur seconds regardless of creeps
    if (elapsed >= autoDur) {
      endWaveAndAdvance();
      return;
    }
    // Manual skip: if both are skipping and skip window open, advance immediately
    if (canSkipNow && sc === 2) {
      endWaveAndAdvance();
    }
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
      btn.title = (!onWave ? `Unlocks on Wave ${def.unlockWave}` :
                   cdRemain>0 ? `Cooldown: ${cdRemain.toFixed(2)}s` :
                   !enoughGold ? `Need ${def.cost} gold` : '');
    }
    // P2
    for (const k of ['normal','fast','slow']){
      const btn = ui.send2[k];
      const def = CreepDefs[k];
      const onWave = w >= def.unlockWave && S.status==='wave';
      const cdRemain = Math.max(0, (L.lastSend[2][k]||-Infinity) + def.cd - now);
      const enoughGold = L.gold[2] >= def.cost;
      btn.disabled = !(onWave && cdRemain <= 0 && enoughGold);
      btn.title = (!onWave ? `Unlocks on Wave ${def.unlockWave}` :
                   cdRemain>0 ? `Cooldown: ${cdRemain.toFixed(2)}s` :
                   !enoughGold ? `Need ${def.cost} gold` : '');
    }
  }

  // UI events -> actions
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

  // Keyboard shortcuts
  window.addEventListener('keydown',(e)=>{
    if (e.repeat) return;
    const canBuild = (S.status==='build' || S.status==='wave') && !L.over;
    switch(e.key){
      // P1 build/select
      case '1': if (canBuild && MY.role==='P1') pushAction({ type:'select', p:1, t:'basic' }); break;
      case '2': if (canBuild && MY.role==='P1') pushAction({ type:'select', p:1, t:'sniper' }); break;
      case 'a': case 'A': if (canBuild && MY.role==='P1') pushAction({ type:'toggleSell', p:1 }); break;
      // P1 sends (during wave only)
      case 'q': case 'Q': if (S.status==='wave' && MY.role==='P1') pushAction({ type:'send', p:1, c:'normal' }); break;
      case 'w': case 'W': if (S.status==='wave' && MY.role==='P1') pushAction({ type:'send', p:1, c:'fast' }); break;
      case 'e': case 'E': if (S.status==='wave' && MY.role==='P1') pushAction({ type:'send', p:1, c:'slow' }); break;
      // P2 build/select
      case '9': if (canBuild && MY.role==='P2') pushAction({ type:'select', p:2, t:'basic' }); break;
      case '0': if (canBuild && MY.role==='P2') pushAction({ type:'select', p:2, t:'sniper' }); break;
      case 'l': case 'L': if (canBuild && MY.role==='P2') pushAction({ type:'toggleSell', p:2 }); break;
      // P2 sends
      case 'p': case 'P': if (S.status==='wave' && MY.role==='P2') pushAction({ type:'send', p:2, c:'normal' }); break;
      case 'o': case 'O': if (S.status==='wave' && MY.role==='P2') pushAction({ type:'send', p:2, c:'fast' }); break;
      case 'i': case 'I': if (S.status==='wave' && MY.role==='P2') pushAction({ type:'send', p:2, c:'slow' }); break;
    }
  });

  // Placement via canvas -> place or sell action (with coords)
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

  // Simulation step (no passive gold)
  function step(dt){
    if (L.over) return;

    // towers fire
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

    // creeps move
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

    // bullets fade
    for (let i=L.bullets.length-1;i>=0;i--){
      const b = L.bullets[i];
      b.life -= dt;
      if (b.life<=0) L.bullets.splice(i,1);
    }
  }

  // Draw
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
      ctx.fillText(`Player ${L.winner} wins!`, canvas.width/2, canvas.height/2 - 10);
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

  // Main loop with fixed timestep
  function loop(){
    // Solo win detection
    soloWinCheck();

    // Wave spawning
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
      if (skipCount() === 2) {
        if (S.waveNumber===0) beginWave(1);
      } else {
        const effRemain = (skipCount() >= 1) ? baseRemain/2 : baseRemain;
        if (effRemain <= 0 && S.waveNumber===0) beginWave(1);
      }
    }

    // Auto-advance waves and manual skip logic
    if (S.status==='wave') waveAutoAdvanceCheck();

    // Update header wave text and send locks
    waveHeader.textContent = `Wave: ${S.waveNumber}`;
    updateSendLocks();

    draw();
    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);

  // Initial UI
  updateSellLabels();
  renderMM();
})();
</script>
</body>
</html>

