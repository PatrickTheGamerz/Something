<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>EToH Obby — WASD Platformer</title>
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
    <span class="pill">W/A/S/D to move • R to reset</span>
    <span id="status" class="pill">Checkpoint: 0 • Molecules: 0/5</span>
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

  // --- Input (WASD) ---
  const keys = { w:false, a:false, s:false, d:false, r:false };
  window.addEventListener('keydown', (e) => {
    if (['w','a','s','d','r'].includes(e.key.toLowerCase())) e.preventDefault();
    if (e.key === ' ') e.preventDefault();
    keys[e.key.toLowerCase()] = true;
  });
  window.addEventListener('keyup', (e) => keys[e.key.toLowerCase()] = false);

  // --- Camera ---
  const cam = { x: 0, y: 0, lerp: 0.12 };

  // --- World units ---
  const U = 32; // tile size in px (world units scaled by DPR at draw)

  // --- Player ---
  const player = {
    x: 0, y: 0, w: 20, h: 28,
    vx: 0, vy: 0,
    speed: 160, jump: 360, air: 0.85,
    onGround: false, canDrop: false, facing: 1,
    checkpoint: { x: 0, y: 0 },
    molecules: 0
  };

  // --- Level: platforms, thin platforms, hazards (EToH), moving, collectibles ---
  const level = {
    gravity: 900,
    friction: 0.84,
    goal: { x: 58*U, y: 6*U, w: 2*U, h: 2*U },
    platforms: [
      // Ground and benches
      { x: 0*U,  y: 13*U, w: 40*U, h: 2*U },
      { x: 42*U, y: 13*U, w: 40*U, h: 2*U },
      { x: 10*U, y: 10*U, w: 6*U,  h: 2*U },
      { x: 18*U, y: 8*U,  w: 4*U,  h: 2*U },
      { x: 24*U, y: 6*U,  w: 4*U,  h: 2*U },
      { x: 30*U, y: 9*U,  w: 6*U,  h: 2*U },
      { x: 48*U, y: 10*U, w: 6*U,  h: 2*U },
      { x: 54*U, y: 8*U,  w: 4*U,  h: 2*U }
    ],
    thin: [
      // Drop-through lab grates
      { x: 14*U, y: 11.5*U, w: 4*U, h: 8 }, // thin height is small
      { x: 36*U, y: 10.5*U, w: 5*U, h: 8 }
    ],
    moving: [
      // Oscillating platform (horizontal)
      { x: 40*U, y: 7.5*U, w: 4*U, h: 2*U, t: 0, amp: 5*U, speed: 1.2, axis: 'x' },
      // Oscillating platform (vertical)
      { x: 46*U, y: 6*U, w: 4*U, h: 2*U, t: 0, amp: 2*U, speed: 1.6, axis: 'y' }
    ],
    hazards: [
      // EToH pools
      { x: 6*U, y: 13*U, w: 4*U, h: 1.5*U },
      { x: 22*U, y: 13*U, w: 3*U, h: 1.5*U },
      { x: 38*U, y: 13*U, w: 3*U, h: 1.5*U },
      { x: 50*U, y: 13*U, w: 6*U, h: 1.5*U }
    ],
    molecules: [
      // Collectibles: ethanol molecules (grab them all)
      { x: 12*U + U/2, y: 9.5*U },
      { x: 19.5*U, y: 7.5*U },
      { x: 25.5*U, y: 5.5*U },
      { x: 41*U, y: 6.8*U },
      { x: 55.5*U, y: 7.2*U }
    ],
    spawn: { x: 2*U, y: 10.5*U }
  };

  function reset(hard = false) {
    player.x = (hard ? level.spawn.x : player.checkpoint.x) | 0;
    player.y = (hard ? level.spawn.y : player.checkpoint.y) | 0;
    player.vx = 0; player.vy = 0;
    player.onGround = false;
    if (hard) {
      player.molecules = 0;
      level.molecules.forEach(m => m.collected = false);
      player.checkpoint = { x: level.spawn.x, y: level.spawn.y };
    }
  }

  // Initialize
  player.checkpoint = { x: level.spawn.x, y: level.spawn.y };
  reset(true);

  // --- Helpers ---
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

  // --- Update loop ---
  let last = performance.now();
  function loop(t) {
    const dt = Math.min((t - last) / 1000, 0.033);
    last = t;

    // Input forces
    const left  = keys.a ? -1 : 0;
    const right = keys.d ?  1 : 0;
    const up    = keys.w;
    const down  = keys.s;

    // Horizontal movement
    const accel = player.onGround ? player.speed : player.speed * player.air;
    player.vx += (left + right) * accel * dt;
    if (left) player.facing = -1;
    else if (right) player.facing = 1;

    // Friction
    if (player.onGround && !left && !right) player.vx *= level.friction;

    // Jump
    if (up && player.onGround) {
      player.vy = -player.jump;
      player.onGround = false;
      player.canDrop = false;
    }

    // Gravity
    player.vy += level.gravity * dt;

    // Cap speeds
    const maxVX = 260;
    const maxVY = 800;
    player.vx = Math.max(Math.min(player.vx, maxVX), -maxVX);
    player.vy = Math.max(Math.min(player.vy, maxVY), -maxVY);

    // Integrate
    player.x += player.vx * dt;
    player.y += player.vy * dt;

    // Move platforms update
    level.moving.forEach(m => {
      m.t += dt * m.speed;
      const offset = Math.sin(m.t) * m.amp;
      if (m.axis === 'x') m._x = m.x + offset, m._y = m.y;
      else m._x = m.x, m._y = m.y + offset;
    });

    // Collisions: platforms (solid)
    player.onGround = false;
    const solids = [...level.platforms, ...level.moving.map(m => ({ x:m._x, y:m._y, w:m.w, h:m.h }))];
    for (const p of solids) {
      const pb = { x: p.x, y: p.y, w: p.w, h: p.h };
      const ab = { x: player.x, y: player.y, w: player.w, h: player.h };
      if (!rectsOverlap(ab, pb)) continue;

      // Resolve using minimal axis push
      const dx1 = (pb.x + pb.w) - ab.x;     // push left
      const dx2 = (ab.x + ab.w) - pb.x;     // push right
      const dy1 = (pb.y + pb.h) - ab.y;     // push up
      const dy2 = (ab.y + ab.h) - pb.y;     // push down
      const pushX = Math.min(dx1, dx2);
      const pushY = Math.min(dy1, dy2);

      if (pushX < pushY) {
        // Horizontal push
        if (dx1 < dx2) player.x = pb.x + pb.w; // push to right
        else player.x = pb.x - player.w;       // push to left
        player.vx = 0;
      } else {
        // Vertical push
        if (dy1 < dy2) {
          player.y = pb.y + pb.h; // push below platform
          player.vy = Math.max(player.vy, 0);
        } else {
          player.y = pb.y - player.h; // land on platform
          player.vy = Math.min(player.vy, 0);
          player.onGround = true;
          player.canDrop = true;
          // Carry with moving platform if standing on it
          if ('_x' in p) {
            const carry = p.axis === 'x' ? Math.cos(p.t) * p.amp * p.speed * 0.03 : 0;
            player.x += carry;
          }
        }
      }
    }

    // Thin platforms (drop-through with S)
    for (const tp of level.thin) {
      const ab = { x: player.x, y: player.y, w: player.w, h: player.h };
      const pb = { x: tp.x, y: tp.y, w: tp.w, h: tp.h };
      const touchingFromAbove =
        ab.x < pb.x + pb.w && ab.x + ab.w > pb.x &&
        ab.y + ab.h > pb.y && ab.y + ab.h < pb.y + pb.h + 16 &&
        player.vy >= 0;
      if (touchingFromAbove && !(down && player.canDrop)) {
        player.y = pb.y - player.h;
        player.vy = 0;
        player.onGround = true;
      }
    }

    // Hazards: EToH pools
    for (const hz of level.hazards) {
      const hb = { x: hz.x, y: hz.y, w: hz.w, h: hz.h };
      const ab = { x: player.x, y: player.y, w: player.w, h: player.h };
      if (rectsOverlap(ab, hb)) {
        reset(false); // respawn at checkpoint
        break;
      }
    }

    // Collect molecules
    for (const m of level.molecules) {
      if (m.collected) continue;
      const dx = (player.x + player.w/2) - m.x;
      const dy = (player.y + player.h/2) - m.y;
      if (dx*dx + dy*dy < (18*18)) {
        m.collected = true;
        player.molecules++;
      }
    }

    // Checkpoint: first bench top
    const checkpointZone = { x: 10*U, y: 10*U-2*U, w: 6*U, h: 2*U };
    if (rectsOverlap({ x: player.x, y: player.y, w: player.w, h: player.h }, checkpointZone)) {
      player.checkpoint = { x: player.x, y: player.y - 2 };
    }

    // Goal: require all molecules
    const atGoal = rectsOverlap({ x: player.x, y: player.y, w: player.w, h: player.h }, level.goal);
    if (atGoal && player.molecules >= level.molecules.length) {
      document.getElementById('status').textContent = "Completed! Press R to replay";
    } else {
      document.getElementById('status').textContent =
        `Checkpoint: ${player.checkpoint.x ? 1 : 0} • Molecules: ${player.molecules}/${level.molecules.length}`;
    }

    // Reset key
    if (keys.r) reset(true);

    // Camera follow
    const targetX = player.x + player.w/2 - canvas.width/2;
    const targetY = player.y + player.h/2 - canvas.height/2;
    cam.x += (targetX - cam.x) * cam.lerp;
    cam.y += (targetY - cam.y) * cam.lerp;

    // --- Render ---
    ctx.save();
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // Parallax lab background
    const bg1 = ctx.createLinearGradient(0, 0, 0, canvas.height);
    bg1.addColorStop(0, '#10131a'); bg1.addColorStop(1, '#0a0d12');
    ctx.fillStyle = bg1;
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    ctx.translate(-cam.x, -cam.y);

    // Grid backdrop
    ctx.strokeStyle = 'rgba(255,255,255,0.05)'; ctx.lineWidth = 1;
    for (let gx = -100*U; gx < 100*U; gx += 2*U) {
      ctx.beginPath(); ctx.moveTo(gx, -100*U); ctx.lineTo(gx, 100*U); ctx.stroke();
    }
    for (let gy = -100*U; gy < 100*U; gy += 2*U) {
      ctx.beginPath(); ctx.moveTo(-100*U, gy); ctx.lineTo(100*U, gy); ctx.stroke();
    }

    // Platforms
    ctx.fillStyle = '#a6a9b6';
    for (const p of level.platforms) drawRoundedRect(p.x, p.y, p.w, p.h, 6);

    // Moving platforms
    ctx.fillStyle = '#c7cad6';
    for (const m of level.moving) drawRoundedRect(m._x, m._y, m.w, m.h, 6);

    // Thin platforms
    ctx.fillStyle = '#9093a0';
    for (const tp of level.thin) drawRoundedRect(tp.x, tp.y, tp.w, tp.h, 6);

    // Hazards (EToH pools)
    for (const hz of level.hazards) {
      const grad = ctx.createLinearGradient(hz.x, hz.y, hz.x, hz.y + hz.h);
      grad.addColorStop(0, '#4db4ff'); grad.addColorStop(1, '#2766a3');
      ctx.fillStyle = grad;
      drawRoundedRect(hz.x, hz.y, hz.w, hz.h, 10);
      ctx.fillStyle = 'rgba(255,255,255,0.8)';
      ctx.font = `${14}px monospace`;
      ctx.fillText('EToH', hz.x + 6, hz.y + hz.h - 6);
    }

    // Goal (exit)
    ctx.fillStyle = '#3bd37f';
    drawRoundedRect(level.goal.x, level.goal.y, level.goal.w, level.goal.h, 6);
    ctx.fillStyle = '#0b2e1a';
    ctx.font = `${16}px monospace`;
    ctx.fillText('EXIT', level.goal.x + 6, level.goal.y + level.goal.h/2 + 5);

    // Molecules (collectibles)
    for (const m of level.molecules) {
      if (m.collected) continue;
      ctx.save();
      ctx.translate(m.x, m.y);
      ctx.scale(1, 1);
      // Simple ethanol icon: circle + tail
      ctx.fillStyle = '#ffe27a'; ctx.beginPath(); ctx.arc(0, 0, 10, 0, Math.PI*2); ctx.fill();
      ctx.strokeStyle = '#ffd24f'; ctx.lineWidth = 3;
      ctx.beginPath(); ctx.moveTo(8, 0); ctx.lineTo(18, -6); ctx.stroke();
      ctx.restore();
    }

    // Player
    ctx.fillStyle = '#e86f6f';
    drawRoundedRect(player.x, player.y, player.w, player.h, 6);
    // Visor
    ctx.fillStyle = '#2b2f45';
    drawRoundedRect(player.x + (player.facing === 1 ? 8 : 2), player.y + 8, 10, 6, 3);

    ctx.restore();

    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);
  </script>
</body>
</html>
