<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Live Online Players</title>
  <style>
    body { font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif; background:#0f172a; color:#e2e8f0; margin:0; }
    .wrap { max-width: 760px; margin: 8vh auto; padding: 24px; }
    .card { background:#111827; border:1px solid #1f2937; border-radius:12px; padding:24px; }
    h1 { margin:0 0 12px; font-size:28px; color:#f8fafc; }
    .count { font-size:56px; font-weight:800; color:#22d3ee; }
    .muted { color:#94a3b8; font-size:14px; margin-top:10px; }
    .err { color:#fca5a5; white-space:pre-wrap; }
    .ok { color:#86efac; }
    code { background:#0b1220; border:1px solid #1f2937; padding:2px 6px; border-radius:6px; }
  </style>
</head>
<body>
  <div class="wrap">
    <div class="card">
      <h1>Players on this page</h1>
      <div id="onlineCount" class="count">0</div>
      <div id="status" class="muted">Starting…</div>
      <div id="details" class="muted"></div>
      <div id="error" class="muted err"></div>
    </div>
    <div class="muted" style="margin-top:16px">
      If the count stays 0, check the messages above. Most common issues: wrong <code>databaseURL</code>, database not enabled, or rules blocking writes.
    </div>
  </div>

  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.4/firebase-app.js";
    import {
      getDatabase, ref, onValue, onDisconnect, serverTimestamp, set, update
    } from "https://www.gstatic.com/firebasejs/10.12.4/firebase-database.js";

    const ui = {
      count: document.getElementById("onlineCount"),
      status: document.getElementById("status"),
      details: document.getElementById("details"),
      error: document.getElementById("error"),
    };

    // 🔹 Replace with your Firebase config
    const firebaseConfig = {
      apiKey: "YOUR_API_KEY",
      authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
      databaseURL: "https://YOUR_INSTANCE_ID.europe-west1.firebasedatabase.app",
      projectId: "YOUR_PROJECT_ID",
      storageBucket: "YOUR_PROJECT_ID.appspot.com",
      messagingSenderId: "YOUR_SENDER_ID",
      appId: "YOUR_APP_ID"
    };

    const log = (...args) => console.log("[presence]", ...args);
    const showStatus = (text, ok = false) => {
      ui.status.textContent = text;
      ui.status.className = "muted" + (ok ? " ok" : "");
    };
    const showDetails = (text) => ui.details.textContent = text;
    const showError = (e) => ui.error.textContent = (typeof e === "string") ? e : (e?.message || String(e));

    try {
      // 1) Init Firebase
      const app = initializeApp(firebaseConfig);
      const db = getDatabase(app);
      showStatus("Initialized Firebase…");

      // 2) Identify this page (could also be a game room ID)
      const pageId = encodeURIComponent(window.location.pathname);

      // 3) Monitor connection state
      const connectedRef = ref(db, ".info/connected");
      let registered = false;
      let sessionRef = null;
      let beatTimer = null;

      // 4) Listen for sessions only on this page
      const sessionsRef = ref(db, `presence/pages/${pageId}/sessions`);
      onValue(sessionsRef, (snap) => {
        const data = snap.val() || {};
        const count = Object.keys(data).length;
        ui.count.textContent = String(count);
        showDetails(`Players on this page: ${count}`);
      }, (err) => {
        showError("Failed to listen to sessions: " + err.message);
      });

      // 5) Register presence when connected
      onValue(connectedRef, async (snap) => {
        const isConnected = snap.val() === true;
        log("connected =", isConnected);
        if (!isConnected) {
          showStatus("Connecting to Realtime Database…");
          return;
        }

        showStatus("Connected to Realtime Database", true);

        if (!registered) {
          const sessionId = (crypto.randomUUID?.() || Math.random().toString(36).slice(2)) + "-" + Date.now();
          sessionRef = ref(db, `presence/pages/${pageId}/sessions/${sessionId}`);
          log("Registering session", sessionId);

          try {
            await set(sessionRef, {
              startedAt: serverTimestamp(),
              lastSeen: serverTimestamp()
            });
            onDisconnect(sessionRef).remove().catch((e) => log("onDisconnect remove failed", e));
            registered = true;
            showDetails((ui.details.textContent || "") + "\nRegistered presence.");
          } catch (e) {
            showError("Could not write presence (check databaseURL and rules): " + e.message);
            return;
          }

          // Heartbeat every 25s
          const beat = () => update(sessionRef, { lastSeen: serverTimestamp() }).catch((e) => log("heartbeat failed", e));
          beatTimer = setInterval(beat, 25000);
          document.addEventListener("visibilitychange", () => { if (!document.hidden) beat(); });
          window.addEventListener("beforeunload", () => { clearInterval(beatTimer); });
        }
      }, (err) => {
        showError("Failed to read connection state: " + err.message);
      });

    } catch (e) {
      showError("Initialization error: " + e.message);
    }
  </script>
</body>
</html>
