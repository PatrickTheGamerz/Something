<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>EToH Obby — 2‑Player Platformer</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <style>
    html, body { margin: 0; height: 100%; background: #0e0f14; font-family: system-ui, sans-serif; }
    #hud {
      position: absolute; inset: 0 0 auto 0; padding: 8px 12px; color: #e8e8f0; font-size: 14px;
      display: flex; gap: 16px; align-items: center; background: linear-gradient(to bottom, rgba(0,0,0,.35), rgba(0,0,0,0));
      user-select: none;
    }
    #game { display: block; width: 100vw; height: 100vh; }
    .pill { padding: 2px 8px; border-radius: 12px; background: rgba(255,255,255,.12); }
  </style>
</head>
<body>
  <div id="hud">
    <span class="pill">EToH Obby</span>
    <span class="pill">P1: WASD • P2: Arrows • R to reset</span>
    <span id="status" class="pill">Molecules: 0/5</span>
  </div>
  <canvas id="game"></canvas>

  <script>
  // --- Canvas setup ---
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const DPR = Math.min(window.devicePixelRatio || 1, 2);
  function resize() {
    canvas.width = Math.floor(window.innerWidth * DPR);
    canvas.height = Math.floor(window.innerHeight * DPR);
  }
  window.addEventListener('resize', resize);
  resize();

  // --- Input ---
  const keys = {};
  window.addEventListener('keydown', e => { keys[e.key.toLowerCase()] = true; if (["w","a","s","d","arrowup","arrowdown","arrowleft","arrowright","r"].includes(e.key.toLowerCase())) e.preventDefault(); });
  window.addEventListener('keyup', e => { keys[e.key.toLowerCase()] = false; });

  // --- Camera ---
  const cam = { x: 0, y: 0, lerp: 0.12 };

  // --- World units ---
  const U = 32;

  // --- Player factory ---
  function makePlayer(color, controls, spawn) {
    return {
      x: spawn.x, y: spawn.y, w: 20, h: 28,
      vx: 0, vy: 0,
      speed: 200, jump: 360,
      onGround: false, canDrop: false, facing: 1,
      molecules: 0,
      color,
      controls
    };
  }

  const level = {
    gravity: 900,
    goal: { x: 58*U, y: 6*U, w: 2*U, h: 2*U },
    platforms: [
      { x: 0*U,  y: 13*U, w: 40*U, h: 2*U },
      { x: 42*U, y: 13*U, w: 40*U, h: 2*U },
      { x: 10*U, y: 10*U, w: 6*U,  h: 2*U },
      { x: 18*U, y: 8*U,  w: 4*U,  h: 2*U },
      { x: 24*U, y: 6*U,  w: 4*U,  h: 2*U },
      { x: 30*U, y: 9*U,  w: 6*U,  h: 2*U },
      { x: 48*U, y: 10*U, w: 6*U,  h: 2*U },
      { x: 54*U, y: 8*U,  w: 4*U,  h: 2*U }
    ],
    hazards: [
      { x: 6*U, y: 13*U, w: 4*U, h: 1.5*U },
      { x: 22*U, y: 13*U, w: 3*U, h: 1.5*U },
      { x: 38*U, y: 13*U, w: 3*U, h: 1.5*U },
      { x: 50*U, y: 13*U, w: 6*U, h: 1.5*U }
    ],
    molecules: [
      { x: 12*U + U/2, y: 9.5*U },
      { x: 19.5*U, y: 7.5*U },
      { x: 25.5*U, y: 5.5*U },
      { x: 41*U, y: 6.8*U },
      { x: 55.5*U, y: 7.2*U }
    ],
    spawn: { x: 2*U, y: 10.5*U }
  };

  const player1 = makePlayer("#e86f6f", {left:"a", right:"d", up:"w"}, level.spawn);
  const player2 = makePlayer("#6fb3e8", {left:"arrowleft", right:"arrowright", up:"arrowup"}, {x: level.spawn.x+40, y: level.spawn.y});

  const players = [player1, player2];

  function rectsOverlap(a, b) {
    return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y;
  }
  function drawRoundedRect(x, y, w, h, r) {
    ctx.beginPath();
    ctx.moveTo(x+r, y);
    ctx.arcTo(x+w, y,   x+w, y+h, r);
    ctx.arcTo(x+w, y+h, x,   y+h, r);
    ctx.arcTo(x,   y+h, x,   y,   r);
    ctx.arcTo(x,   y,   x+w, y,   r);
    ctx.closePath();
    ctx.fill();
  }

  function reset() {
    players.forEach((p,i) => {
      p.x = level.spawn.x + i*40;
      p.y = level.spawn.y;
      p.vx = p.vy = 0;
      p.molecules = 0;
    });
    level.molecules.forEach(m => m.collected = false);
  }
  reset();

  // --- Update loop ---
  let last = performance.now();
  function loop(t) {
    const dt = Math.min((t - last) / 1000, 0.033);
    last = t;

    players.forEach(p => {
      // Input
      const left  = keys[p.controls.left];
      const right = keys[p.controls.right];
      const up    = keys[p.controls.up];

      // Stable movement: set velocity directly
      if (p.onGround) {
        if (left) { p.vx = -p.speed; p.facing=-1; }
        else if (right) { p.vx = p.speed; p.facing=1; }
        else p.vx = 0;
      } else {
        // air control
        if (left) { p.vx = -p.speed*0.7; p.facing=-1; }
        else if (right) { p.vx = p.speed*0.7; p.facing=1; }
      }

      // Jump
      if (up && p.onGround) {
        p.vy = -p.jump;
        p.onGround = false;
      }

      // Gravity
      p.vy += level.gravity * dt;
      if (p.vy > 800) p.vy = 800;

      // Integrate
      p.x += p.vx * dt;
      p.y += p.vy * dt;

      // Collisions with platforms
      p.onGround = false;
      for (const plat of level.platforms) {
        const pb = { x: plat.x, y: plat.y, w: plat.w, h: plat.h };
        const ab = { x: p.x, y: p.y, w: p.w, h: p.h };
        if (!rectsOverlap(ab, pb)) continue;

        // Overlap depths
        const dx1 = (pb.x + pb.w) - ab.x;
        const dx2 = (ab.x + ab.w) - pb.x;
        const dy1 = (pb.y + pb.h) - ab.y;
        const dy2 = (ab.y + ab.h) - pb.y;
        const pushX = Math.min(dx1, dx2);
        const pushY = Math.min(dy1, dy2);

        if (pushX < pushY) {
          // Horizontal resolution
          if (dx1 < dx2) p.x = pb.x + pb.w;
          else p.x = pb.x - p.w;
          p.vx = 0;
        } else {
          // Vertical resolution
          if (dy1 < dy2) {
            p.y = pb.y + pb.h;
            p.vy = Math.max(p.vy, 0);
          } else {
            p.y = pb.y - p.h;
            p.vy = 0;
            p.onGround = true;
          }
        }
      }

      // Hazards
      for (const hz of level.hazards) {
        const hb = { x: hz.x, y: hz.y, w: hz.w, h: hz.h };
        const ab = { x: p.x, y: p.y, w: p.w, h: p.h };
        if (rectsOverlap(ab, hb)) {
          // Respawn this player
          p.x = level.spawn.x;
          p.y = level.spawn.y;
          p.vx = p.vy = 0;
        }
      }

      // Collectibles
      for (const m of level.molecules) {
        if (m.collected) continue;
        const dx = (p.x + p.w/2) - m.x;
        const dy = (p.y + p.h/2) - m.y;
        if (dx*dx + dy*dy < (18*18)) {
          m.collected = true;
          p.molecules++;
        }
      }
    }); // end players loop

    // Camera follow (center between players)
    const midX = (player1.x + player2.x) / 2;
    const midY = (player1.y + player2.y) / 2;
    const targetX = midX - canvas.width/2;
    const targetY = midY - canvas.height/2;
    cam.x += (targetX - cam.x) * cam.lerp;
    cam.y += (targetY - cam.y) * cam.lerp;

    // --- Render ---
    ctx.save();
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // Background
    const bg1 = ctx.createLinearGradient(0, 0, 0, canvas.height);
    bg1.addColorStop(0, '#10131a'); bg1.addColorStop(1, '#0a0d12');
    ctx.fillStyle = bg1;
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    ctx.translate(-cam.x, -cam.y);

    // Platforms
    ctx.fillStyle = '#a6a9b6';
    for (const p of level.platforms) drawRoundedRect(p.x, p.y, p.w, p.h, 6);

    // Hazards
    for (const hz of level.hazards) {
      const grad = ctx.createLinearGradient(hz.x, hz.y, hz.x, hz.y + hz.h);
      grad.addColorStop(0, '#4db4ff'); grad.addColorStop(1, '#2766a3');
      ctx.fillStyle = grad;
      drawRoundedRect(hz.x, hz.y, hz.w, hz.h, 10);
      ctx.fillStyle = 'rgba(255,255,255,0.8)';
      ctx.font = '14px monospace';
      ctx.fillText('EToH', hz.x + 6, hz.y + hz.h - 6);
    }

    // Goal
    ctx.fillStyle = '#3bd37f';
    drawRoundedRect(level.goal.x, level.goal.y, level.goal.w, level.goal.h, 6);
    ctx.fillStyle = '#0b2e1a';
    ctx.font = '16px monospace';
    ctx.fillText('EXIT', level.goal.x + 6, level.goal.y + level.goal.h/2 + 5);

    // Molecules
    for (const m of level.molecules) {
      if (m.collected) continue;
      ctx.save();
      ctx.translate(m.x, m.y);
      ctx.fillStyle = '#ffe27a';
      ctx.beginPath(); ctx.arc(0, 0, 10, 0, Math.PI*2); ctx.fill();
      ctx.strokeStyle = '#ffd24f'; ctx.lineWidth = 3;
      ctx.beginPath(); ctx.moveTo(8, 0); ctx.lineTo(18, -6); ctx.stroke();
      ctx.restore();
    }

    // Players
    players.forEach(p => {
      ctx.fillStyle = p.color;
      drawRoundedRect(p.x, p.y, p.w, p.h, 6);
      ctx.fillStyle = '#2b2f45';
      drawRoundedRect(p.x + (p.facing===1 ? 8 : 2), p.y+8, 10, 6, 3);
    });

    ctx.restore();

    // HUD
    document.getElementById('status').textContent =
      `Molecules: ${players.reduce((a,b)=>a+b.molecules,0)}/${level.molecules.length}`;

    if (keys['r']) reset();

    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);
  </script>
</body>
</html>
