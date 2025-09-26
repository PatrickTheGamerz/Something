<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>EToH Tower</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <style>
    html, body { margin: 0; height: 100%; background: #0e0f14; overflow: hidden; }
    canvas { display: block; width: 100vw; height: 100vh; background: #1a1c2c; }
  </style>
</head>
<body>
  <canvas id="game"></canvas>
  <script type="module">
    import { setupPlayers, updatePlayers, drawPlayers } from './Players.js';

    const canvas = document.getElementById('game');
    const ctx = canvas.getContext('2d');
    const DPR = window.devicePixelRatio || 1;
    canvas.width = window.innerWidth * DPR;
    canvas.height = window.innerHeight * DPR;

    const U = 32;
    const level = {
      gravity: 900,
      platforms: [
        { x: 0, y: 13*U, w: 40*U, h: 2*U },
        { x: 10*U, y: 10*U, w: 6*U, h: 2*U },
        { x: 18*U, y: 8*U, w: 4*U, h: 2*U },
        { x: 24*U, y: 6*U, w: 4*U, h: 2*U },
        { x: 30*U, y: 9*U, w: 6*U, h: 2*U }
      ],
      spawn: { x: 2*U, y: 10*U }
    };

    const players = setupPlayers(level.spawn.x, level.spawn.y);

    let last = performance.now();
    function loop(t) {
      const dt = Math.min((t - last) / 1000, 0.033);
      last = t;

      updatePlayers(players, level, dt);

      ctx.clearRect(0, 0, canvas.width, canvas.height);

      // Background gradient
      const bg = ctx.createLinearGradient(0, 0, 0, canvas.height);
      bg.addColorStop(0, '#2a2d3a');
      bg.addColorStop(1, '#0e0f14');
      ctx.fillStyle = bg;
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      // Platforms
      ctx.fillStyle = '#a6a9b6';
      for (const p of level.platforms) {
        ctx.fillRect(p.x, p.y, p.w, p.h);
      }

      drawPlayers(ctx, players);

      requestAnimationFrame(loop);
    }

    requestAnimationFrame(loop);
  </script>
</body>
</html>

































export function setupPlayers(spawnX, spawnY) {
  const createPlayer = (color, keys) => ({
    x: spawnX, y: spawnY, w: 20, h: 28,
    vx: 0, vy: 0,
    speed: 160, jump: 360, onGround: false,
    color, keys
  });

  const players = [
    createPlayer('#e86f6f', { left: 'a', right: 'd', up: 'w' }),
    createPlayer('#6fe8e8', { left: 'arrowleft', right: 'arrowright', up: 'arrowup' })
  ];

  window.addEventListener('keydown', e => {
    players.forEach(p => p[e.key.toLowerCase()] = true);
  });
  window.addEventListener('keyup', e => {
    players.forEach(p => p[e.key.toLowerCase()] = false);
  });

  return players;
}

export function updatePlayers(players, level, dt) {
  for (const p of players) {
    const left = p[p.keys.left];
    const right = p[p.keys.right];
    const up = p[p.keys.up];

    const accel = p.onGround ? p.speed : p.speed * 0.85;
    if (left) p.vx -= accel * dt;
    if (right) p.vx += accel * dt;

    if (p.onGround && !left && !right) {
      p.vx *= 0.6;
      if (Math.abs(p.vx) < 1) p.vx = 0;
    }

    if (up && p.onGround) {
      p.vy = -p.jump;
      p.onGround = false;
    }

    p.vy += level.gravity * dt;

    p.vx = Math.max(Math.min(p.vx, 260), -260);
    p.vy = Math.max(Math.min(p.vy, 800), -800);

    p.x += p.vx * dt;
    p.y += p.vy * dt;

    p.onGround = false;
    for (const plat of level.platforms) {
      const ab = { x: p.x, y: p.y, w: p.w, h: p.h };
      const pb = plat;
      if (
        ab.x < pb.x + pb.w && ab.x + ab.w > pb.x &&
        ab.y < pb.y + pb.h && ab.y + ab.h > pb.y
      ) {
        const dy = (pb.y - ab.y - ab.h);
        if (dy >= -10 && p.vy >= 0) {
          p.y = pb.y - ab.h;
          p.vy = 0;
          p.onGround = true;
        }
      }
    }
  }
}

export function drawPlayers(ctx, players) {
  for (const p of players) {
    ctx.fillStyle = p.color;
    ctx.fillRect(p.x, p.y, p.w, p.h);
    ctx.fillStyle = '#2b2f45';
    ctx.fillRect(p.x + (p.vx >= 0 ? 8 : 2), p.y + 8, 10, 6);
  }
}





