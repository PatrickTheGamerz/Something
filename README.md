<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Tower Layout</title>
  <style>
    body { background: #223; margin: 0; overflow: hidden; }
    #towerCanvas { background: #444; display: block; margin: 0 auto; }
  </style>
</head>
<body>
<canvas id="towerCanvas" width="1024" height="480"></canvas>
<script>
// ==== MAP/LAYOUT DEFINITION ====

// Platforms (ground, steps, walls, tower structure)
const platforms = [
  { x: 0, y: 450, width: 2000, height: 50 },
  { x: 100, y: 370, width: 250, height: 20 },
  { x: 270, y: 300, width: 120, height: 16 },
  { x: 400, y: 250, width: 130, height: 16 },
  { x: 650, y: 320, width: 140, height: 18 },
  { x: 900, y: 390, width: 80, height: 18 },
  { x: 1100, y: 340, width: 150, height: 16 },
  { x: 1400, y: 290, width: 120, height: 18 },
  { x: 1500, y: 220, width: 100, height: 16 },
  { x: 650, y: 220, width: 18, height: 100 },
  { x: 1680, y: 100, width: 170, height: 18 },
];

// Hazards (spikes, traps)
const hazards = [
  { x: 500, y: 465, width: 36, height: 15 },
  { x: 1190, y: 465, width: 36, height: 15 },
  { x: 1520, y: 465, width: 40, height: 15 },
];

// Collectibles (coins, gems, etc.)
const collectibles = [
  { x: 170, y: 340, width: 20, height: 20 },
  { x: 450, y: 225, width: 20, height: 20 },
  { x: 680, y: 297, width: 20, height: 20 },
  { x: 1450, y: 267, width: 20, height: 20 },
  { x: 1710, y: 75, width: 20, height: 20 },
];

// Checkpoints (respawn flags)
const checkpoints = [
  { x: 330, y: 325, width: 24, height: 28 },
  { x: 1120, y: 308, width: 24, height: 28 },
  { x: 1515, y: 186, width: 24, height: 28 },
];

// ==== DRAW HELPERS ====
function drawPlatform(ctx, p) {
  ctx.fillStyle = "#666";
  ctx.fillRect(p.x, p.y, p.width, p.height);
  ctx.strokeStyle = "#333";
  ctx.strokeRect(p.x, p.y, p.width, p.height);
}

function drawHazard(ctx, h) {
  ctx.fillStyle = "#b43a3a";
  ctx.fillRect(h.x, h.y, h.width, h.height);
  ctx.strokeStyle = "#fff";
  ctx.beginPath();
  ctx.moveTo(h.x, h.y + h.height);
  ctx.lineTo(h.x + h.width/2, h.y);
  ctx.lineTo(h.x + h.width, h.y + h.height);
  ctx.stroke();
}

function drawCollectible(ctx, c) {
  ctx.beginPath();
  ctx.arc(c.x+c.width/2, c.y+c.height/2, 9, 0, 2*Math.PI);
  ctx.fillStyle = "#eeca3a";
  ctx.fill();
  ctx.strokeStyle = "#e5cc77";
  ctx.lineWidth = 2;
  ctx.stroke();
}

function drawCheckpoint(ctx, cp) {
  ctx.fillStyle = "#4ab3ed";
  ctx.fillRect(cp.x, cp.y, cp.width, cp.height);
  ctx.fillStyle = "#fff";
  ctx.font = "bold 11px sans-serif";
  ctx.fillText("✔", cp.x+5, cp.y+18);
}

// ==== RENDER LOOP ====
const canvas = document.getElementById("towerCanvas");
const ctx = canvas.getContext("2d");

function render() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  for (const plat of platforms) drawPlatform(ctx, plat);
  for (const hazard of hazards) drawHazard(ctx, hazard);
  for (const c of collectibles) drawCollectible(ctx, c);
  for (const cp of checkpoints) drawCheckpoint(ctx, cp);

  requestAnimationFrame(render);
}

render();
</script>
</body>
</html>
