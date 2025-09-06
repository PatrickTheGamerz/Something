<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Tic Tac Toe Online</title>
  <link rel="stylesheet" href="style.css" />
</head>
<body>
  <main class="container">
    <h1>Tic Tac Toe Online</h1>

    <div class="controls">
      <button id="createBtn">Create game</button>
      <button id="joinBtn">Join game</button>
      <button id="copyBtn" disabled>Copy invite link</button>
      <button id="resetBtn" disabled>Reset</button>
    </div>

    <div id="roomInfo" class="room-info"></div>
    <div id="status" class="status">Create or join a game to start</div>

    <div id="board" class="board" aria-label="Tic Tac Toe board" role="grid">
      <!-- 9 cells -->
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

    <footer class="footer">
      <span id="youAre"></span>
    </footer>
  </main>

  <!-- Firebase SDKs (compat for simplicity) -->
  <script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-database-compat.js"></script>

  <script src="app.js" type="module"></script>
</body>
</html>
