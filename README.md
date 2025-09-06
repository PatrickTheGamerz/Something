<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Tic Tac Toe — Online</title>
  <meta name="description" content="Play a polished online Tic Tac Toe with a friend. Create a room, share the link, and play in real-time." />
  <link rel="preconnect" href="https://www.gstatic.com" crossorigin>

  <style>
    :root {
      --bg: #0b0f1a;
      --bg-2: #0e1222;
      --card: #131a2f;
      --card-2: #182040;
      --text: #e8ecff;
      --muted: #a6b0c3;
      --accent: #6c8cff;
      --accent-2: #9a7bff;
      --good: #59d98f;
      --warn: #ffb86c;
      --bad: #ff6b6b;
      --grid: rgba(255,255,255,0.08);
      --shadow: 0 10px 30px rgba(0,0,0,0.4);
    }

    * { box-sizing: border-box; }
    html, body { height: 100%; }
    body {
      margin: 0;
      color: var(--text);
      font-family: Inter, ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
      background:
        radial-gradient(1200px 800px at 20% -10%, #1b2350, var(--bg)) no-repeat,
        linear-gradient(160deg, var(--bg-2), var(--bg));
      overflow: hidden;
    }

    #bg { position: fixed; inset: 0; width: 100%; height: 100%; z-index: 0; }

    .frame {
      position: relative;
      z-index: 1;
      min-height: 100%;
      display: grid;
      grid-template-rows: auto 1fr auto;
      padding: clamp(16px, 3vw, 24px);
      max-width: 900px;
      margin: 0 auto;
    }

    .topbar {
      display: flex; align-items: center; justify-content: space-between;
      gap: 12px; margin-bottom: 16px;
    }
    .topbar h1 { margin: 0; font-size: clamp(22px, 3.2vw, 28px); letter-spacing: 0.3px; }
    .right { display: flex; gap: 8px; }
    .icon-btn {
      background: transparent; border: 1px solid var(--grid); color: var(--text);
      padding: 8px 10px; border-radius: 10px; cursor: pointer;
      transition: background .2s ease, transform .06s ease, border-color .2s;
    }
    .icon-btn:hover { background: rgba(255,255,255,0.06); }
    .icon-btn:active { transform: translateY(1px); }

    .panel {
      background: linear-gradient(180deg, rgba(255,255,255,.04), rgba(255,255,255,.02));
      border: 1px solid rgba(255,255,255,.12);
      border-radius: 16px;
      box-shadow: var(--shadow);
      padding: clamp(14px, 2.6vw, 22px);
      display: grid;
      gap: 14px;
      align-content: start;
    }

    .controls { display: flex; gap: 10px; flex-wrap: wrap; }
    button {
      background: #1a2142; color: var(--text);
      border: 1px solid rgba(255,255,255,0.14);
      padding: 10px 14px; border-radius: 12px; cursor: pointer;
      transition: background .2s, transform .06s, border-color .2s;
    }
    button:hover { background: #222a56; }
    button:active { transform: translateY(1px); }
    button:disabled { opacity: .55; cursor: not-allowed; }
    .primary {
      background: linear-gradient(180deg, #3a4fb8, #2d3e90);
      border-color: rgba(255,255,255,0.2);
    }
    .primary:hover { background: linear-gradient(180deg, #465ddd, #3449aa); }

    .info { display: flex; flex-wrap: wrap; gap: 8px; }
    .tag {
      background: rgba(255,255,255,0.06);
      border: 1px solid rgba(255,255,255,0.12);
      border-radius: 999px;
      padding: 6px 10px;
      color: var(--muted);
    }

    .status {
      padding: 10px 12px;
      background: rgba(255,255,255,0.04);
      border: 1px solid rgba(255,255,255,0.12);
      border-radius: 12px;
    }
    .status strong { color: var(--text); }

    .layout {
      display: grid;
      grid-template-columns: 1fr 280px;
      gap: 16px;
      align-items: start;
    }

    .seat-row {
      display: grid;
      grid-template-columns: repeat(2, minmax(100px, 1fr));
      gap: 10px;
    }
    .seat { background: #1b2344; }
    .seat.taken { background: #1a1f35; opacity: .7; }
    .seat.mine { outline: 2px solid var(--accent); }

    .board-wrap {
      display: grid;
      place-items: center;
      padding: 8px 0;
    }
    .board {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      gap: 8px;
      width: 100%;
      max-width: min(88vw, 340px);
      margin: 0 auto;
    }
    .cell {
      aspect-ratio: 1 / 1;
      font-size: clamp(26px, 8vw, 44px);
      font-weight: 800;
      letter-spacing: 1px;
      color: var(--text);
      background: linear-gradient(180deg, var(--card-2), var(--card));
      border: 1px solid var(--grid);
      border-radius: 12px;
      text-shadow: 0 6px 22px rgba(0,0,0,0.5);
      box-shadow: inset 0 0 0 1px rgba(255,255,255,0.02);
      position: relative;
      overflow: hidden;
    }
    .cell::after {
      content: "";
      position: absolute; inset: 0;
      background: radial-gradient(220px 220px at var(--mx,50%) var(--my,50%), rgba(108,140,255,0.12), transparent 60%);
      pointer-events: none;
    }
    .cell:hover { border-color: rgba(255,255,255,0.22); }
    .cell.win {
      outline: 3px solid var(--good);
      background:
        linear-gradient(180deg, rgba(89,217,143,0.16), transparent 60%),
        linear-gradient(180deg, var(--card-2), var(--card));
    }

    .meta {
      display: grid;
      grid-template-columns: 1fr;
      gap: 12px;
    }
    .score, .presence {
      background: rgba(255,255,255,0.04);
      border: 1px solid rgba(255,255,255,0.12);
      border-radius: 12px;
      padding: 10px;
    }
    .score span, .presence span { color: var(--muted); display: block; margin-bottom: 6px; }
    .score-box { display: flex; gap: 16px; }
    .pills { display: flex; flex-wrap: wrap; gap: 6px; }
    .pills .pill {
      background: rgba(255,255,255,0.06);
      border: 1px solid rgba(255,255,255,0.12);
      border-radius: 999px;
      padding: 4px 8px;
      color: var(--muted);
      font-size: 12px;
    }

    .foot { margin-top: 14px; color: var(--muted); text-align: center; }

    /* Light theme */
    .light {
      --bg: #f7f8fe;
      --bg-2: #eef1ff;
      --card: #ffffff;
      --card-2: #f9fbff;
      --text: #0c132b;
      --muted: #5b6680;
      --grid: rgba(0,0,0,0.08);
      --shadow: 0 10px 30px rgba(0,0,0,0.08);
    }
    .light body { background: linear-gradient(160deg, var(--bg-2), var(--bg)); }
    .light .panel { background: #ffffff; }

    @media (max-width: 840px) {
      .layout { grid-template-columns: 1fr; }
      .board { max-width: min(92vw, 340px); }
    }
    @media (max-width: 520px) {
      .meta { grid-template-columns: 1fr; }
    }
  </style>
</head>
<body>
  <canvas id="bg" aria-hidden="true"></canvas>

  <main class="frame">
    <header class="topbar">
      <h1>Tic Tac Toe — Online</h1>
      <div class="right">
        <button id="soundBtn" class="icon-btn" aria-label="Toggle sound">🔊</button>
        <button id="themeBtn" class="icon-btn" aria-label="Toggle theme">🌗</button>
      </div>
    </header>

    <section class="panel">
      <div class="controls">
        <button id="createBtn" class="primary">Create room</button>
        <button id="joinBtn">Join room</button>
        <button id="copyBtn" disabled>Copy invite link</button>
        <button id="leaveBtn" disabled>Leave seat</button>
        <button id="resetBtn" disabled>Reset game</button>
      </div>

      <div class="info">
        <div id="roomInfo" class="tag"></div>
        <div id="youAre" class="tag"></div>
        <div id="opponent" class="tag"></div>
      </div>

      <div id="status" class="status">Create or join a room to start</div>

      <div class="layout">
        <div>
          <div class="seat-row">
            <button id="takeXBtn" class="seat">Take X</button>
            <button id="takeOBtn" class="seat">Take O</button>
          </div>

          <div class="board-wrap">
            <div id="board" class="board" aria-label="Tic Tac Toe board" role="grid">
              <button class="cell" data-i="0" role="gridcell" aria-label="Cell 1"></button>
              <button class="cell" data-i="1" role="gridcell" aria-label="Cell 2"></button>
              <button class="cell" data-i="2" role="gridcell" aria-label="Cell 3"></button>
              <button class="cell" data-i="3" role="gridcell" aria-label="Cell 4"></button>
              <button class="cell" data-i="4" role="gridcell" aria-label="Cell 5"></button>
              <button class="cell" data-i="5" role="gridcell" aria-label="Cell 6"></button>
              <button class="cell" data-i="6" role="gridcell" aria-label="Cell 7"></button>
              <button class="cell" data-i="7" role="gridcell" aria-label="Cell 8"></button>
              <button class="cell" data-i="8" role="gridcell" aria-label="Cell 9"></button>
            </div>
          </div>
        </div>

        <div class="meta">
          <div class="score">
            <span>Score</span>
            <div class="score-box">
              <div>X: <strong id="scoreX">0</strong></div>
              <div>O: <strong id="scoreO">0</strong></div>
            </div>
          </div>
          <div class="presence">
            <span>Players online</span>
            <div id="presenceList" class="pills"></div>
          </div>
        </div>
      </div>
    </section>

    <footer class="foot">
      <small>Share your room link to play live. Spectators can watch in real time.</small>
    </footer>
  </main>

  <!-- Firebase SDKs (compat builds keep it simple for static hosting) -->
  <script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-database-compat.js"></script>

  <script>
    // ---------- 1) Firebase config (REPLACE placeholders with your project's values) ----------
    const firebaseConfig = {
      apiKey: "YOUR_API_KEY",
      authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
      databaseURL: "https://YOUR_PROJECT_ID-default-rtdb.europe-west1.firebasedatabase.app",
      projectId: "YOUR_PROJECT_ID",
      storageBucket: "YOUR_PROJECT_ID.appspot.com",
      messagingSenderId: "XXXXXXX",
      appId: "1:XXXXXXX:web:XXXXXXXX"
    };

    // ---------- 2) Firebase init ----------
    firebase.initializeApp(firebaseConfig);
    const db = firebase.database();

    // ---------- 3) Client identity ----------
    const clientId = (() => {
      const k = "ttt_client_id_v2";
      let id = localStorage.getItem(k);
      if (!id) { id = (crypto && crypto.randomUUID) ? crypto.randomUUID() : String(Math.random()).slice(2); localStorage.setItem(k, id); }
      return id;
    })();
    const now = () => Date.now();

    // ---------- 4) DOM ----------
    const bg = document.getElementById("bg");
    const boardEl = document.getElementById("board");
    const statusEl = document.getElementById("status");
    const roomInfoEl = document.getElementById("roomInfo");
    const youAreEl = document.getElementById("youAre");
    const opponentEl = document.getElementById("opponent");
    const presenceListEl = document.getElementById("presenceList");
    const scoreXEl = document.getElementById("scoreX");
    const scoreOEl = document.getElementById("scoreO");

    const createBtn = document.getElementById("createBtn");
    const joinBtn = document.getElementById("joinBtn");
    const copyBtn = document.getElementById("copyBtn");
    const leaveBtn = document.getElementById("leaveBtn");
    const resetBtn = document.getElementById("resetBtn");
    const takeXBtn = document.getElementById("takeXBtn");
    const takeOBtn = document.getElementById("takeOBtn");
    const soundBtn = document.getElementById("soundBtn");
    const themeBtn = document.getElementById("themeBtn");

    // ---------- 5) State ----------
    let roomId = getRoomFromURL();
    let myMark = null;
    let unsubRoom = null;
    let unsubPresence = null;
    let lifecycleUnsub = null;
    let soundsOn = true;

    // ---------- 6) Constants ----------
    const WIN_LINES = [
      [0,1,2],[3,4,5],[6,7,8],
      [0,3,6],[1,4,7],[2,5,8],
      [0,4,8],[2,4,6]
    ];

    // ---------- 7) Background animations (dark: stars, light: bubbles) ----------
    const ctx = bg.getContext("2d");
    let rafId = null;
    let animState = null;

    function resizeCanvas() {
      bg.width = window.innerWidth;
      bg.height = window.innerHeight;
      initAnimation();
    }
    window.addEventListener("resize", resizeCanvas);

    function initAnimation() {
      cancelAnimationFrame(rafId);
      const isLight = document.documentElement.classList.contains("light");
      if (isLight) {
        animState = { type: "bubbles", bubbles: Array.from({ length: 36 }, () => makeBubble()) };
      } else {
        animState = { type: "stars", stars: Array.from({ length: 100 }, () => makeStar()) };
      }
      drawBg();
    }

    function makeStar() {
      return { x: Math.random() * bg.width, y: Math.random() * bg.height, r: Math.random() * 1.6 + 0.4, s: Math.random() * 0.6 + 0.2, p: Math.random() * Math.PI * 2 };
    }
    function makeBubble() {
      const palette = [
        "rgba(108,140,255,0.18)","rgba(154,123,255,0.16)","rgba(89,217,143,0.16)","rgba(255,184,108,0.14)","rgba(255,107,107,0.14)"
      ];
      return { x: Math.random() * bg.width, y: bg.height + Math.random() * 200, r: Math.random() * 36 + 14, vx: (Math.random()-0.5)*0.3, vy: -(Math.random()*0.6+0.2), c: palette[(Math.random()*palette.length)|0], wob: Math.random()*Math.PI*2 };
    }
    function drawBg() {
      ctx.clearRect(0,0,bg.width,bg.height);
      if (!animState) { rafId = requestAnimationFrame(drawBg); return; }
      if (animState.type === "stars") {
        for (const st of animState.stars) {
          ctx.beginPath(); ctx.arc(st.x, st.y, st.r, 0, Math.PI*2);
          const tw = 0.35 + 0.65*Math.sin((Date.now()/700)*st.s + st.p);
          ctx.fillStyle = `rgba(140,160,255,${tw})`; ctx.fill();
          st.x += st.s*0.05; if (st.x > bg.width+5) st.x = -5;
        }
      } else { // bubbles
        for (const b of animState.bubbles) {
          ctx.beginPath();
          const grd = ctx.createRadialGradient(b.x-b.r*0.4, b.y-b.r*0.4, b.r*0.2, b.x, b.y, b.r);
          grd.addColorStop(0, "rgba(255,255,255,0.35)");
          grd.addColorStop(0.4, b.c);
          grd.addColorStop(1, "rgba(255,255,255,0.02)");
          ctx.fillStyle = grd;
          ctx.arc(b.x, b.y, b.r, 0, Math.PI*2); ctx.fill();
          b.wob += 0.01; b.x += b.vx + Math.sin(b.wob)*0.15; b.y += b.vy;
          if (b.y + b.r < -10 || b.x < -50 || b.x > bg.width + 50) Object.assign(b, makeBubble(), { y: bg.height + b.r, x: Math.random() * bg.width });
        }
      }
      rafId = requestAnimationFrame(drawBg);
    }
    resizeCanvas();

    // ---------- 8) Events ----------
    createBtn.addEventListener("click", onCreate);
    joinBtn.addEventListener("click", onJoinPrompt);
    copyBtn.addEventListener("click", onCopy);
    leaveBtn.addEventListener("click", onLeaveSeat);
    resetBtn.addEventListener("click", onReset);
    takeXBtn.addEventListener("click", () => claimSeat("X"));
    takeOBtn.addEventListener("click", () => claimSeat("O"));
    soundBtn.addEventListener("click", toggleSound);
    themeBtn.addEventListener("click", toggleTheme);

    boardEl.addEventListener("mousemove", (e) => {
      const rect = boardEl.getBoundingClientRect();
      const mx = ((e.clientX - rect.left) / rect.width) * 100 + "%";
      const my = ((e.clientY - rect.top) / rect.height) * 100 + "%";
      for (const btn of boardEl.querySelectorAll(".cell")) {
        btn.style.setProperty("--mx", mx);
        btn.style.setProperty("--my", my);
      }
    });
    boardEl.addEventListener("click", onCellClick);

    // ---------- 9) Auto-join via URL (do NOT auto-create) ----------
    if (roomId) joinRoom(roomId, { allowCreateIfMissing: false });

    // ---------- 10) Presence ----------
    let roomPresenceRef = null;
    function startPresence(room) {
      stopPresence();
      roomPresenceRef = db.ref(`rooms/${room}/presence/${clientId}`);
      const connRef = db.ref(".info/connected");
      connRef.on("value", (snap) => {
        if (snap.val() === true) {
          roomPresenceRef.onDisconnect().remove().then(() => {
            roomPresenceRef.set({ at: now() });
          });
        }
      });
      db.ref(`rooms/${room}/presence`).on("value", (s) => {
        renderPresence(Object.keys(s.val() || {}));
      });
      unsubPresence = () => {
        connRef.off();
        db.ref(`rooms/${room}/presence`).off();
      };
    }
    function stopPresence() { unsubPresence && unsubPresence(); unsubPresence = null; }

    // ---------- 11) Room lifecycle (auto-delete when empty) ----------
    function startLifecycleWatcher(room) {
      stopLifecycleWatcher();
      const presRef = db.ref(`rooms/${room}/presence`);
      const roomRef = db.ref(`rooms/${room}`);
      let deleteTimer = null;

      const scheduleCheck = async () => {
        const snap = await roomRef.get();
        const state = snap.val();
        if (!state) return;
        const presenceSnap = await presRef.get();
        const presenceEmpty = !presenceSnap.exists() || Object.keys(presenceSnap.val() || {}).length === 0;
        if (presenceEmpty) {
          if (!deleteTimer) {
            // Grace period to avoid flapping (5s)
            deleteTimer = setTimeout(async () => {
              const latest = (await roomRef.get()).val();
              const presLatest = (await presRef.get()).val();
              const stillEmpty = !presLatest || Object.keys(presLatest).length === 0;
              if (stillEmpty) {
                await roomRef.remove();
              }
              deleteTimer = null;
            }, 5000);
          }
        } else if (deleteTimer) {
          clearTimeout(deleteTimer);
          deleteTimer = null;
        }
      };

      presRef.on("value", scheduleCheck);
      roomRef.on("value", (s) => { if (!s.exists()) stopLifecycleWatcher(); });
      lifecycleUnsub = () => {
        presRef.off();
        roomRef.off();
      };
    }
    function stopLifecycleWatcher() { lifecycleUnsub && lifecycleUnsub(); lifecycleUnsub = null; }

    // ---------- 12) Actions ----------
    async function onCreate() {
      const id = await createUniqueRoomId();
      const ref = db.ref(`rooms/${id}`);
      const initial = {
        board: Array(9).fill(""),
        current: "X",
        players: { X: "", O: "" },
        status: "playing",
        winnerLine: [-1, -1, -1],
        score: { X: 0, O: 0 },
        createdAt: now(),
        updatedAt: now()
      };
      await ref.set(initial);
      setRoomInURL(id);
      joinRoom(id, { allowCreateIfMissing: false });
    }

    async function onJoinPrompt() {
      const input = prompt("Enter room code (e.g., ABC123):");
      if (!input) return;
      const id = input.trim().toUpperCase();
      const snap = await db.ref(`rooms/${id}`).get();
      if (!snap.exists()) { alert("Room not found. Check the code or ask your friend to share the link."); return; }
      joinRoom(id, { allowCreateIfMissing: false });
    }

    async function onCopy() {
      if (!roomId) return;
      await navigator.clipboard.writeText(window.location.href);
      copyBtn.textContent = "Copied!";
      setTimeout(() => (copyBtn.textContent = "Copy invite link"), 1200);
    }

    async function onLeaveSeat() {
      if (!roomId || !myMark) return;
      const ref = db.ref(`rooms/${roomId}/players/${myMark}`);
      await ref.transaction((seat) => seat === clientId ? "" : seat);
    }

    async function onReset() {
      if (!roomId) return;
      const ref = db.ref(`rooms/${roomId}`);
      await ref.transaction((state) => {
        if (state == null) return state;
        if (!canPlay(state)) return state;
        state.board = Array(9).fill("");
        state.current = "X";
        state.status = "playing";
        state.winnerLine = [-1,-1,-1];
        state.updatedAt = firebase.database.ServerValue.TIMESTAMP;
        return state;
      });
    }

    // ---------- 13) Join room ----------
    function joinRoom(id, opts = { allowCreateIfMissing: false }) {
      cleanupRoom();
      roomId = id;
      setRoomInURL(id);
      setControls(true);
      roomInfoEl.textContent = `Room: ${id} — share this link`;
      copyBtn.disabled = false;

      const ref = db.ref(`rooms/${id}`);

      if (opts.allowCreateIfMissing) {
        ref.once("value").then((snap) => {
          if (!snap.exists()) {
            ref.set({
              board: Array(9).fill(""),
              current: "X",
              players: { X: "", O: "" },
              status: "playing",
              winnerLine: [-1, -1, -1],
              score: { X: 0, O: 0 },
              createdAt: now(),
              updatedAt: now()
            });
          }
        });
      }

      ref.on("value", async (snap) => {
        const state = snap.val();
        if (!state) {
          setStatus("Room not found. Create a new room.");
          clearBoard();
          setYouAre(null);
          return;
        }

        const currentMark = state.players?.X === clientId ? "X"
                           : state.players?.O === clientId ? "O" : null;

        // Auto-assign random seat if available and user has none
        if (!currentMark) {
          const freeSeats = [];
          if (!state.players?.X) freeSeats.push("X");
          if (!state.players?.O) freeSeats.push("O");
          if (freeSeats.length) {
            const randomSeat = freeSeats[(Math.random() * freeSeats.length) | 0];
            await tryAutoSeat(id, randomSeat);
          }
        }

        // Update local mark after potential auto-seat
        const latest = (await ref.get()).val() || state;
        const myNow = latest.players?.X === clientId ? "X"
                     : latest.players?.O === clientId ? "O" : null;
        setYouAre(myNow);

        updateSeatButtons(latest);
        renderBoard(latest.board, latest.winnerLine);
        renderStatus(latest);
        renderScore(latest.score);

        copyBtn.disabled = false;
        resetBtn.disabled = false;
        leaveBtn.disabled = !myNow;

        startPresence(id);
        startLifecycleWatcher(id);
      });

      unsubRoom = () => ref.off();
    }

    async function tryAutoSeat(room, seat) {
      const ref = db.ref(`rooms/${room}`);
      await ref.transaction((state) => {
        if (!state) return state;
        if (!state.players) state.players = { X: "", O: "" };
        if (state.players.X === clientId || state.players.O === clientId) return state;
        if (!state.players[seat]) {
          state.players[seat] = clientId;
          state.updatedAt = firebase.database.ServerValue.TIMESTAMP;
        }
        return state;
      });
    }

    // ---------- 14) Seats ----------
    async function claimSeat(seat) {
      if (!roomId) return;
      if (seat !== "X" && seat !== "O") return;
      const ref = db.ref(`rooms/${roomId}`);
      await ref.transaction((state) => {
        if (state == null) return state;
        if (!state.players) state.players = { X: "", O: "" };
        const already = state.players.X === clientId ? "X" : (state.players.O === clientId ? "O" : "");
        if (already && already !== seat) state.players[already] = "";
        if (!state.players[seat] || state.players[seat] === clientId) {
          state.players[seat] = clientId;
        }
        state.updatedAt = firebase.database.ServerValue.TIMESTAMP;
        return state;
      });
    }

    // ---------- 15) Moves ----------
    async function onCellClick(e) {
      const cell = e.target.closest(".cell");
      if (!cell || !roomId) return;
      const idx = Number(cell.dataset.i);

      const ref = db.ref(`rooms/${roomId}`);
      await ref.transaction((state) => {
        if (state == null) return state;
        if (state.status !== "playing") return state;

        const mark = (state.players?.X === clientId) ? "X" : (state.players?.O === clientId) ? "O" : null;
        if (!mark) return state;
        if (state.current !== mark) return state;
        if (!Array.isArray(state.board)) state.board = Array(9).fill("");
        if (state.board[idx]) return state;

        state.board[idx] = mark;
        const ev = evaluate(state.board);
        if (ev.winner) {
          state.status = `${ev.winner} won`;
          state.winnerLine = ev.line;
          if (!state.score) state.score = { X: 0, O: 0 };
          state.score[ev.winner] = (state.score[ev.winner] || 0) + 1;
        } else if (ev.draw) {
          state.status = "draw";
          state.winnerLine = [-1,-1,-1];
        } else {
          state.status = "playing";
          state.winnerLine = [-1,-1,-1];
          state.current = (mark === "X") ? "O" : "X";
        }

        state.updatedAt = firebase.database.ServerValue.TIMESTAMP;
        return state;
      });
    }

    // ---------- 16) Logic ----------
    function evaluate(board) {
      for (const line of WIN_LINES) {
        const [a,b,c] = line;
        if (board[a] && board[a] === board[b] && board[a] === board[c]) {
          ding();
          return { winner: board[a], draw: false, line };
        }
      }
      const draw = board.every(Boolean);
      if (draw) ding();
      return { winner: null, draw, line: draw ? [-1,-1,-1] : null };
    }
    function canPlay(state) { return state.players?.X === clientId || state.players?.O === clientId; }

    // ---------- 17) Render ----------
    function renderBoard(board, winnerLine = []) {
      const lineSet = new Set(winnerLine || []);
      for (const btn of boardEl.querySelectorAll(".cell")) {
        const i = Number(btn.dataset.i);
        const v = board?.[i] || "";
        btn.textContent = v;
        btn.classList.toggle("win", lineSet.has(i));
      }
    }
    function renderStatus(state) {
      const players = state.players || { X: "", O: "" };
      takeXBtn.classList.toggle("taken", !!players.X && players.X !== clientId);
      takeOBtn.classList.toggle("taken", !!players.O && players.O !== clientId);
      takeXBtn.classList.toggle("mine", players.X === clientId);
      takeOBtn.classList.toggle("mine", players.O === clientId);

      const both = players.X && players.O;
      if (!both) {
        setStatus("Waiting for players… Claim X or O and share the link.");
        opponentEl.textContent = "Waiting for opponent…";
        return;
      }

      if (state.status === "playing") {
        const mine = (state.current === myMark);
        setStatus(`Turn: <strong>${state.current}</strong> ${mine ? "(your move)" : ""}`);
      } else if (state.status === "draw") {
        setStatus(`It's a draw. Reset to play again.`);
      } else if (state.status.endsWith("won")) {
        const w = state.status[0];
        const you = (w === myMark);
        setStatus(`Winner: <strong>${w}</strong> ${you ? "🎉" : ""} Reset to play again.`);
      } else {
        setStatus(state.status);
      }

      const opp =
        myMark === "X" ? (players.O ? "Opponent seated" : "Waiting for O…") :
        myMark === "O" ? (players.X ? "Opponent seated" : "Waiting for X…") :
        (players.X && players.O ? "Spectating a live match" : "Waiting for players…");
      opponentEl.textContent = opp;
    }
    function renderScore(score = { X: 0, O: 0 }) {
      scoreXEl.textContent = score.X ?? 0;
      scoreOEl.textContent = score.O ?? 0;
    }
    function renderPresence(ids) {
      presenceListEl.innerHTML = "";
      ids.forEach((id) => {
        const el = document.createElement("span");
        el.className = "pill";
        el.textContent = id === clientId ? "You" : id.slice(0, 8);
        presenceListEl.appendChild(el);
      });
    }

    // ---------- 18) UI helpers ----------
    function setStatus(html) { statusEl.innerHTML = html; }
    function setYouAre(mark) {
      myMark = mark;
      youAreEl.textContent = mark ? `You are: ${mark}` : `You are a spectator`;
      leaveBtn.disabled = !mark;
    }
    function setControls(inRoom) {
      copyBtn.disabled = !inRoom;
      resetBtn.disabled = !inRoom;
    }
    function updateSeatButtons(state) {
      const p = state.players || { X: "", O: "" };
      takeXBtn.disabled = !!p.X && p.X !== clientId;
      takeOBtn.disabled = !!p.O && p.O !== clientId;
    }
    function clearBoard() { renderBoard(Array(9).fill(""), []); }

    // ---------- 19) Sound & theme ----------
    function ding() {
      if (!soundsOn) return;
      try {
        const ctx = new (window.AudioContext || window.webkitAudioContext)();
        const o = ctx.createOscillator();
        const g = ctx.createGain();
        o.type = "sine";
        o.frequency.value = 880;
        o.connect(g); g.connect(ctx.destination);
        g.gain.setValueAtTime(0.001, ctx.currentTime);
        g.gain.exponentialRampToValueAtTime(0.15, ctx.currentTime + 0.01);
        g.gain.exponentialRampToValueAtTime(0.0001, ctx.currentTime + 0.2);
        o.start(); o.stop(ctx.currentTime + 0.21);
      } catch {}
    }

    (function initTheme() {
      const k = "ttt_theme";
      const saved = localStorage.getItem(k);
      if (saved === "light") document.documentElement.classList.add("light");
      themeBtn.textContent = document.documentElement.classList.contains("light") ? "🌞" : "🌗";
      initAnimation();
    })();

    function toggleTheme() {
      const k = "ttt_theme";
      document.documentElement.classList.toggle("light");
      const isLight = document.documentElement.classList.contains("light");
      localStorage.setItem(k, isLight ? "light" : "dark");
      themeBtn.textContent = isLight ? "🌞" : "🌗";
      initAnimation();
    }
    function toggleSound() { soundsOn = !soundsOn; soundBtn.textContent = soundsOn ? "🔊" : "🔈"; }

    // ---------- 20) Cleanup ----------
    function cleanupRoom() {
      unsubRoom && unsubRoom(); unsubRoom = null;
      stopPresence();
      stopLifecycleWatcher();
    }
    window.addEventListener("beforeunload", () => { /* presence cleaned via onDisconnect */ });

    // ---------- 21) URL & ID helpers ----------
    function generateRoomId() {
      const alphabet = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789";
      let s = "";
      for (let i = 0; i < 6; i++) s += alphabet[(Math.random() * alphabet.length) | 0];
      return s;
    }
    async function createUniqueRoomId(maxAttempts = 20) {
      for (let i = 0; i < maxAttempts; i++) {
        const id = generateRoomId();
        const exists = (await db.ref(`rooms/${id}`).get()).exists();
        if (!exists) return id;
      }
      // Fallback with longer id if collisions happen absurdly
      let id;
      do {
        id = generateRoomId() + generateRoomId();
      } while ((await db.ref(`rooms/${id}`).get()).exists());
      return id;
    }
    function getRoomFromURL() {
      const url = new URL(window.location.href);
      return url.searchParams.get("room")?.toUpperCase() || null;
    }
    function setRoomInURL(id) {
      const url = new URL(window.location.href);
      url.searchParams.set("room", id);
      history.replaceState(null, "", url);
    }

    // ---------- 22) Keyboard shortcuts ----------
    document.addEventListener("keydown", (e) => {
      if (!roomId) return;
      const map = { "1":0,"2":1,"3":2, "4":3,"5":4,"6":5, "7":6,"8":7,"9":8 };
      if (map[e.key] != null) {
        const cell = boardEl.querySelector(`.cell[data-i="${map[e.key]}"]`);
        if (cell) cell.click();
      }
    });
  </script>
</body>
</html>
```
