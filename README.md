<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Game Menu</title>
  <style>
    body {
      margin: 0;
      height: 100vh;
      display: flex;
      justify-content: center;
      align-items: center;
      background: #0e0f14;
      font-family: system-ui, sans-serif;
      color: #e8e8f0;
    }
    #menu {
      display: flex;
      flex-direction: column;
      gap: 16px;
      text-align: center;
    }
    button {
      padding: 12px 24px;
      font-size: 18px;
      border: none;
      border-radius: 8px;
      background: #2b2f45;
      color: #fff;
      cursor: pointer;
      transition: background 0.2s;
    }
    button:hover {
      background: #3d4a6b;
    }
    h1 {
      margin-bottom: 20px;
      font-size: 28px;
      color: #7adfff;
    }
  </style>
</head>
<body>
  <div id="menu">
    <h1>My Game Menu</h1>
    <button onclick="startSection('roll')">Roll</button>
    <button onclick="startSection('tower')">Tower</button>
    <button onclick="startSection('player1')">Player 1</button>
    <button onclick="startSection('player2')">Player 2</button>
  </div>

  <script>
    function startSection(section) {
      alert("Starting section: " + section);
      // Later you’ll replace this alert with actual game logic
    }
  </script>




  
</body>
</html>
