<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Live Online Users</title>
  <style>
    html,body { font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif; margin:0; padding:0; background:#0f172a; color:#e2e8f0; }
    .wrap { max-width: 720px; margin: 8vh auto; padding: 24px; }
    .card { background:#111827; border:1px solid #1f2937; border-radius:12px; padding:24px; }
    h1 { margin: 0 0 12px; font-weight: 700; font-size: 28px; color:#f8fafc; }
    .count { font-size: 56px; font-weight: 800; line-height: 1; letter-spacing: -0.02em; color:#22d3ee; }
    .muted { color:#94a3b8; font-size:14px; margin-top:10px; }
    .dot { display:inline-block; width:10px; height:10px; border-radius:50%; background:#22d3ee; margin-right:8px; animation:pulse 1.5s infinite; vertical-align: middle; }
    @keyframes pulse { 0%{opacity:.5} 50%{opacity:1} 100%{opacity:.5} }
    a { color:#93c5fd; text-decoration: none; }
    a:hover { text-decoration: underline; }
  </style>
</head>
<body>
  <div class="wrap">
    <div class="card">
      <h1><span class="dot"></span> People online right now</h1>
      <div id="onlineCount" class="count">0</div>
      <div class="muted">This number updates live as people open or leave this page.</div>
    </div>
    <p class="muted" style="margin-top:16px;">
      Privacy note: this demo stores a random, temporary ID to count presence. No personal info is collected.
    </p>
  </div>

  <!-- Firebase (v10+ modular SDK) -->
  <script type="module">
    // 1) Import Firebase modules from CDN
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.4/firebase-app.js";
    import { getDatabase, ref, onValue, onDisconnect, serverTimestamp, set, update, remove } from "https://www.gstatic.com/firebasejs/10.12.4/firebase-database.js";

    // 2) Paste your Firebase config here
    const firebaseConfig = {
      apiKey: "YOUR_API_KEY",
      authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
      databaseURL: "https://YOUR_PROJECT_ID-default-rtdb.europe-west1.firebasedatabase.app",
      projectId: "YOUR_PROJECT_ID",
      storageBucket: "YOUR_PROJECT_ID.appspot.com",
      messagingSenderId: "YOUR_SENDER_ID",
      appId: "YOUR_APP_ID"
    };

    // 3) Init
    const app = initializeApp(firebaseConfig);
    const db = getDatabase(app);

    // 4) Presence: create a unique ephemeral ID for this visitor
    const sessionId = crypto.randomUUID ? crypto.randomUUID() : Math.random().toString(36).slice(2);

    // Paths:
    // - /presence/sessions/{sessionId}: a heartbeat record for each online visitor
    // - /presence/summary/onlineCount: aggregated count (optional—computed client-side here)
    const sessionRef = ref(db, `presence/sessions/${sessionId}`);

    // 5) Mark this session as online and ensure cleanup on disconnect
    // Store minimal info: when the session started and a lastSeen timestamp
    await set(sessionRef, {
      startedAt: serverTimestamp(),
      lastSeen: serverTimestamp()
    });
    onDisconnect(sessionRef).remove().catch(() => {});

    // 6) Lightweight heartbeat to keep lastSeen fresh (helps in rare disconnect edge cases)
    const heartbeat = () => update(sessionRef, { lastSeen: serverTimestamp() }).catch(() => {});
    const HEARTBEAT_MS = 25000; // 25s
    const beat = setInterval(heartbeat, HEARTBEAT_MS);
    window.addEventListener("visibilitychange", () => { if (!document.hidden) heartbeat(); });
    window.addEventListener("beforeunload", () => {
      clearInterval(beat);
      // Best-effort synchronous cleanup
      navigator.sendBeacon?.(new URL(`presence/sessions/${sessionId}.json`, new URL(firebaseConfig.databaseURL)).toString(), JSON.stringify(null));
    });

    // 7) Subscribe to presence and display the count
    const onlineCountEl = document.getElementById("onlineCount");
    const sessionsRef = ref(db, "presence/sessions");
    onValue(sessionsRef, (snap) => {
      const data = snap.val() || {};
      // Count keys = people online
      const count = Object.keys(data).length;
      onlineCountEl.textContent = String(count);
    }, (err) => {
      console.error(err);
      onlineCountEl.textContent = "—";
    });
  </script>
</body>
</html>
