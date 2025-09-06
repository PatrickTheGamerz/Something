<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Live Online Users</title>
  <style>
    body {
      font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif;
      background: #0f172a;
      color: #e2e8f0;
      margin: 0;
    }
    .wrap {
      max-width: 720px;
      margin: 10vh auto;
      padding: 24px;
    }
    .card {
      background: #111827;
      border: 1px solid #1f2937;
      border-radius: 12px;
      padding: 24px;
    }
    h1 {
      margin: 0 0 12px;
      font-size: 28px;
      color: #f8fafc;
    }
    .count {
      font-size: 56px;
      font-weight: 800;
      color: #22d3ee;
    }
    .muted {
      color: #94a3b8;
      font-size: 14px;
      margin-top: 10px;
    }
    .err {
      color: #fca5a5;
    }
  </style>
</head>
<body>
  <div class="wrap">
    <div class="card">
      <h1>People online right now</h1>
      <div id="onlineCount" class="count">0</div>
      <div id="status" class="muted">Connecting…</div>
      <div id="error" class="muted err"></div>
    </div>
  </div>

  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.4/firebase-app.js";
    import {
      getDatabase,
      ref,
      onValue,
      onDisconnect,
      serverTimestamp,
      set,
      update
    } from "https://www.gstatic.com/firebasejs/10.12.4/firebase-database.js";

    const onlineCountEl = document.getElementById("onlineCount");
    const statusEl = document.getElementById("status");
    const errorEl = document.getElementById("error");

    // 1) Replace with your Firebase config
    const firebaseConfig = {
      apiKey: "YOUR_API_KEY",
      authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
      databaseURL: "https://YOUR_PROJECT_ID-default-rtdb.REGION.firebasedatabase.app",
      projectId: "YOUR_PROJECT_ID",
      storageBucket: "YOUR_PROJECT_ID.appspot.com",
      messagingSenderId: "YOUR_SENDER_ID",
      appId: "YOUR_APP_ID"
    };

    try {
      // 2) Init Firebase
      const app = initializeApp(firebaseConfig);
      const db = getDatabase(app);

      // 3) Create a unique session ID
      const sessionId = crypto.randomUUID
        ? crypto.randomUUID()
        : Math.random().toString(36).slice(2);

      const sessionRef = ref(db, `presence/sessions/${sessionId}`);

      // 4) Mark this session as online
      await set(sessionRef, {
        startedAt: serverTimestamp(),
        lastSeen: serverTimestamp()
      });

      // 5) Remove when disconnected
      onDisconnect(sessionRef).remove().catch(console.error);

      // 6) Heartbeat to keep lastSeen fresh
      const heartbeat = () =>
        update(sessionRef, { lastSeen: serverTimestamp() }).catch(console.error);
      const HEARTBEAT_MS = 25000;
      const beat = setInterval(heartbeat, HEARTBEAT_MS);

      window.addEventListener("visibilitychange", () => {
        if (!document.hidden) heartbeat();
      });
      window.addEventListener("beforeunload", () => {
        clearInterval(beat);
      });

      // 7) Listen for changes in presence
      const sessionsRef = ref(db, "presence/sessions");
      onValue(
        sessionsRef,
        (snap) => {
          const data = snap.val() || {};
          const count = Object.keys(data).length;
          onlineCountEl.textContent = String(count);
          statusEl.textContent = "Connected";
        },
        (err) => {
          errorEl.textContent = "Error: " + err.message;
          statusEl.textContent = "Error";
        }
      );
    } catch (err) {
      errorEl.textContent = "Init error: " + err.message;
      statusEl.textContent = "Error";
    }
  </script>
</body>
</html>
