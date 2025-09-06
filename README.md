// server.js
// Minimal WebSocket server for Orange TD two-device matchmaking and deterministic action sync.
// Run: npm i ws && node server.js

import { WebSocketServer } from 'ws';
import http from 'http';

const PORT = process.env.PORT || 8080;

const TOWER_DEFS = {
  basic:  { cost:50 },
  sniper: { cost:100 }
};
const CREEP_DEFS = {
  normal: { cost:25, unlockWave:2, cd:0.5 },
  fast:   { cost:30, unlockWave:3, cd:0.85 },
  slow:   { cost:50, unlockWave:5, cd:1.0 }
};

function nowMs() { return Date.now(); }

function initialShared() {
  return {
    players: {
      P1: { id:null, name:null, ready:false, skip:false },
      P2: { id:null, name:null, ready:false, skip:false }
    },
    status: 'waiting',          // waiting | matching | build | wave | finished
    epochStart: null,           // ms when build began
    buildDuration: 60,          // seconds
    waveNumber: 0,
    waveAutoDuration: 60,       // seconds
    seq: 0,                     // action sequence
    actions: [],                // authoritative action log
    soloSince: null,            // ms timestamp
    waveStartMs: null           // ms when current wave began (server-side marker)
  };
}

const S = initialShared();
const wss = new WebSocketServer({ noServer: true });

const clients = new Map(); // ws -> { id, role: 'P1'|'P2'|null, name, canSkip:false }

function broadcast(type, payload) {
  const msg = JSON.stringify({ type, ...payload });
  for (const ws of clients.keys()) {
    if (ws.readyState === ws.OPEN) ws.send(msg);
  }
}

function sendState(ws) {
  ws.send(JSON.stringify({ type:'state', S }));
}

function pushActionServer(a, fromId) {
  if (!S.epochStart) return; // ignore if no round
  S.seq = (S.seq || 0) + 1;
  a.seq = S.seq;
  a.from = fromId || 'server';
  a.t = Math.max(0, (nowMs() - S.epochStart) / 1000);
  S.actions.push(a);
  // trim actions if needed
  if (S.actions.length > 5000) S.actions.splice(0, S.actions.length - 2000);
  broadcast('action', { a });
}

function occupied(role){ return !!(S.players[role].id); }
function bothReady(){ return S.players.P1.ready && S.players.P2.ready; }
function skipCount(){ return (S.players.P1.skip?1:0) + (S.players.P2.skip?1:0); }
function bothCanSkipWave() {
  // both players connected and both reported canSkip true
  const need = ['P1','P2'];
  let ok = 0;
  for (const [ws, c] of clients) {
    if (c.role && need.includes(c.role) && c.canSkip) ok++;
  }
  return ok === 2;
}

function assignSeat(ws, id, name) {
  // First, if they already had a role, keep it if still free
  const c = clients.get(ws);
  if (c?.role && S.players[c.role].id === id) return c.role;

  if (!occupied('P1')) {
    S.players.P1 = { id, name, ready:true, skip:false };
    clients.get(ws).role = 'P1';
    return 'P1';
  }
  if (!occupied('P2')) {
    S.players.P2 = { id, name, ready:true, skip:false };
    clients.get(ws).role = 'P2';
    return 'P2';
  }
  return null;
}

function releaseSeatById(id) {
  for (const seat of ['P1','P2']) {
    if (S.players[seat].id === id) {
      S.players[seat] = { id:null, name:null, ready:false, skip:false };
    }
  }
}

function startRound() {
  S.epochStart = nowMs();
  S.status = 'build';
  S.waveNumber = 0;
  S.seq = 0;
  S.actions = [];
  S.players.P1.skip = false; S.players.P2.skip = false;
  S.soloSince = null;
  S.waveStartMs = null;
  broadcast('state', { S });
}

function beginWave(n) {
  if (S.status === 'finished') return;
  S.waveNumber = n;
  S.status = 'wave';
  S.waveStartMs = nowMs();
  S.players.P1.skip = false; S.players.P2.skip = false;
  // push deterministic action for clients to locally begin wave
  pushActionServer({ type:'startWave' }, 'server');
  broadcast('state', { S });
}

function endWaveAndAdvance() {
  const next = S.waveNumber + 1;
  beginWave(next);
}

function serverTick() {
  // Solo win detection (in build/wave only)
  const inRound = (S.status==='build' || S.status==='wave');
  const seats = (occupied('P1')?1:0) + (occupied('P2')?1:0);
  if (!inRound) {
    if (S.soloSince) { S.soloSince = null; }
  } else {
    if (seats === 1) {
      if (!S.soloSince) { S.soloSince = nowMs(); }
      const remain = Math.max(0, 10 - (nowMs() - S.soloSince)/1000);
      if (remain <= 0) {
        // award win to occupied seat
        const winnerRole = occupied('P1') ? 'P1' : 'P2';
        // mark finished; clients will render end state
        S.status = 'finished';
        broadcast('state', { S });
        return;
      }
    } else {
      if (S.soloSince) S.soloSince = null;
    }
  }

  // Waiting/matching -> start build when both seated and ready
  if ((S.status === 'waiting' || S.status === 'matching')) {
    if (occupied('P1') && occupied('P2') && bothReady()) {
      startRound();
      return;
    }
    if (!occupied('P1') || !occupied('P2')) {
      S.status = 'matching';
    }
    return;
  }

  // Build: auto-start wave 1 when timer ends; halve remaining if 1 skip; instant if both
  if (S.status === 'build' && S.epochStart) {
    const elapsed = (nowMs() - S.epochStart)/1000;
    const baseRemain = (S.buildDuration || 60) - elapsed;
    if (skipCount() === 2) {
      if (S.waveNumber===0) beginWave(1);
      return;
    }
    const effRemain = (skipCount() >= 1) ? baseRemain/2 : baseRemain;
    if (effRemain <= 0 && S.waveNumber===0) beginWave(1);
    return;
  }

  // Wave: auto-advance after duration; halve if at least one skip; or immediate if both skip and canSkipWindow open (clients report canSkip)
  if (S.status === 'wave' && S.waveStartMs) {
    const elapsed = (nowMs() - S.waveStartMs)/1000;
    const divisor = skipCount() >= 1 ? 2 : 1;
    const autoDur = (S.waveAutoDuration || 60) / divisor;
    if (elapsed >= autoDur) {
      endWaveAndAdvance();
      return;
    }
    if (skipCount() === 2 && bothCanSkipWave()) {
      endWaveAndAdvance();
      return;
    }
  }
}

setInterval(serverTick, 100);

// Basic HTTP upgrade to WS (no static hosting, just WS)
const server = http.createServer((req, res) => {
  res.writeHead(200);
  res.end('Orange TD WebSocket server is running.\n');
});
server.on('upgrade', (request, socket, head) => {
  wss.handleUpgrade(request, socket, head, function done(ws) {
    wss.emit('connection', ws, request);
  });
});
server.listen(PORT, () => {
  console.log(`Server listening on :${PORT}`);
});

wss.on('connection', (ws) => {
  clients.set(ws, { id:null, role:null, name:null, canSkip:false });

  ws.on('message', (raw) => {
    try {
      const msg = JSON.parse(raw.toString());
      switch (msg.type) {
        case 'join': {
          const id = msg.id || String(Math.random());
          const name = (msg.name || 'Player').slice(0, 24);
          clients.get(ws).id = id;
          clients.get(ws).name = name;

          if (!occupied('P1') || !occupied('P2')) {
            assignSeat(ws, id, name);
            // if one seat free, status is matching
            if (!occupied('P1') || !occupied('P2')) S.status = (S.status==='waiting'?'matching':S.status);
          }
          sendState(ws);
          broadcast('state', { S });
        } break;

        case 'setName': {
          const c = clients.get(ws);
          if (!c) break;
          c.name = (msg.name || 'Player').slice(0, 24);
          if (c.role && S.players[c.role]) {
            S.players[c.role].name = c.name;
            broadcast('state', { S });
          }
        } break;

        case 'toggleReady': {
          const c = clients.get(ws); if (!c) break;
          // claim seat if none
          if (!c.role) {
            assignSeat(ws, c.id || String(Math.random()), c.name || 'Player');
          } else {
            const seat = S.players[c.role];
            if (seat && seat.id === c.id) {
              if (S.status === 'build') {
                seat.skip = !seat.skip; // skip in build
              } else if (S.status === 'wave') {
                seat.skip = !seat.skip; // skip in wave
              } else {
                seat.ready = !seat.ready;
                seat.skip = false;
              }
            }
          }
          // Update matching status if seats not full
          if ((S.status === 'waiting' || S.status === 'matching') && (!occupied('P1') || !occupied('P2'))) {
            S.status = 'matching';
          }
          broadcast('state', { S });
        } break;

        case 'waveCanSkip': {
          const c = clients.get(ws); if (!c) break;
          c.canSkip = !!msg.can;
          // no broadcast needed; serverTick reads this
        } break;

        case 'action': {
          const c = clients.get(ws); if (!c) break;
          const a = msg.a || {};
          // validate basic ownership and context; server only sequences+stamps and rebroadcasts
          // Ownership validation: match p to role
          if (a.p === 1 && c.role !== 'P1') break;
          if (a.p === 2 && c.role !== 'P2') break;

          // Gate by status
          if (a.type === 'send' && S.status !== 'wave') break;
          if (['select','toggleSell','place','sell'].includes(a.type) && !(S.status==='build' || S.status==='wave')) break;

          // Optional server-side rule checks (lightweight):
          if (a.type === 'send') {
            const def = CREEP_DEFS[a.c]; if (!def) break;
            if (S.waveNumber < def.unlockWave) break;
          }
          if (a.type === 'place') {
            const type = a.t || 'basic';
            if (!TOWER_DEFS[type]) break;
          }

          pushActionServer(a, c.id);
        } break;

        case 'claimSeat': {
          const c = clients.get(ws);
          if (!c) break;
          const seat = assignSeat(ws, c.id || String(Math.random()), c.name || 'Player');
          sendState(ws);
          broadcast('state', { S });
        } break;

        case 'leaveSeat': {
          const c = clients.get(ws); if (!c) break;
          if (c.role) {
            const role = c.role;
            releaseSeatById(c.id);
            c.role = null;
            if (S.status !== 'finished' && (S.status==='waiting' || S.status==='matching')) {
              S.status = 'matching';
            }
            broadcast('state', { S });
          }
        } break;

        default: break;
      }
    } catch (e) {
      // ignore malformed messages
    }
  });

  ws.on('close', () => {
    const c = clients.get(ws);
    if (c) {
      // Soft release only outside finished; keep solo win logic in round
      if (c.role && (S.status==='waiting' || S.status==='matching')) {
        releaseSeatById(c.id);
        if (S.status!=='finished') S.status='matching';
        broadcast('state', { S });
      }
    }
    clients.delete(ws);
  });

  // Send initial state
  sendState(ws);
});
