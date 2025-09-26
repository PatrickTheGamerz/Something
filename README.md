<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>EToH Obby — WASD + Arrow Keys</title>
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
    <span class="pill">W/A/S/D + Arrow Keys • R to reset</span>
    <span id="status" class="pill">Checkpoint: 0 • Molecules: 0/5</span>
  </div>
  <canvas id="game"></canvas>

  <script>
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const DPR = Math.min(window.devicePixelRatio || 1, 2);
  function resize() {
    canvas.width = Math.floor(window.innerWidth * DPR);
    canvas.height = Math.floor(window.innerHeight * DPR);
  }
  window.addEventListener('resize', resize);
  resize();

  const keys = {};
  window.addEventListener('keydown', e => { keys[e.key.toLowerCase()] = true; if (['w','a','s','d','r','arrowup','arrowdown','arrowleft','arrowright'].includes(e.key.toLowerCase())) e.preventDefault(); });
  window.addEventListener('keyup', e => { keys[e.key.toLowerCase()] = false; });

  const cam = { x: 0, y: 0, lerp: 0.12 };
  const U = 32;

  function createPlayer(spawnX, spawnY, color) {
    return {
      x: spawnX, y: spawnY, w: 20, h: 28,
      vx: 0, vy: 0,
      speed: 160, jump: 360, air: 0.85,
      onGround: false, canDrop: false, facing: 1,
      checkpoint: { x: spawnX, y: spawnY },
      molecules: 0,
      color
    };
  }

  const player1 = createPlayer(2*U, 10.5*U, '#e86f6f'); // WASD
  const player2 = createPlayer(2*U, 10.5*U, '#6fe8e8'); // Arrow keys

  const level = {
    gravity: 900,
    friction: 0.84,
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
    thin: [
      { x: 14*U, y: 11.5*U, w: 4*U, h: 8 },
      { x: 36*U, y: 10.5*U, w: 5*U, h: 8 }
    ],
    moving: [
      { x: 40*U, y: 7.5*U, w: 4*U, h: 2*U, t: 0, amp: 5*U, speed: 1.2, axis: 'x' },
      { x: 46*U, y: 6*U, w: 4*U, h: 2*U, t: 0, amp: 2*U, speed: 1.6, axis: 'y' }
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

  function reset(hard = false) {
    [player1, player2].forEach(p => {
      p.x = (hard ? level.spawn.x : p.checkpoint.x) | 0;
      p.y = (hard ? level.spawn.y : p.checkpoint.y) | 0;
      p.vx = 0; p.vy = 0; p.onGround = false;
      if (hard) {
        p.molecules = 0;
        p.checkpoint = { x: level.spawn.x, y: level.spawn.y };
        level.molecules.forEach(m => m.collected = false);
      }
    });
  }

  reset(true);

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

  let last = performance.now();
  function loop(t) {
    const dt = Math.min((t - last) / 1000, 0.033);
    last = t;

    level.moving.forEach(m => {
      m.t += dt * m.speed;
      const offset = Math.sin(m.t) * m.amp;
      m._x = m.axis === 'x' ? m.x + offset : m.x;
      m._y = m.axis === 'y' ? m.y + offset : m.y;
    });

        : { left: keys.arrowleft, right: keys.arrowright, up: keys.arrowup, down: keys.arrowdown };

      const accel = player.onGround ? player.speed : player.speed * player.air;
      if (input.left) { player.vx -= accel * dt; player.facing = -1; }
      if (input.right) { player.vx += accel * dt; player.facing = 1; }

      // Apply friction when no input
      if (player.onGround && !input.left && !input.right) {
        player.vx *= 0.6; // stronger friction
        if (Math.abs(player.vx) < 1) player.vx = 0;
      }

      // Jump
      if (input.up && player.onGround) {
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

      // Collisions
      player.onGround = false;
      const solids = [...level.platforms, ...level.moving.map(m => ({ x:m._x, y:m._y, w:m.w, h:m.h }))];
      for (const p of solids) {
        const pb = { x: p.x, y: p.y, w: p.w, h: p.h };
        const ab = { x: player.x, y: player.y, w: player.w, h: player.h };
        if (!rectsOverlap(ab, pb)) continue;

        const dx1 = (pb.x + pb.w) - ab.x;
        const dx2 = (ab.x + ab.w) - pb.x;
        const dy1 = (pb.y + pb.h) - ab.y;
        const dy2 = (ab.y + ab.h) - pb.y;
        const pushX = Math.min(dx1, dx2);
        const pushY = Math.min(dy1, dy2);

        if (pushX < pushY) {
          if (dx1 < dx2) player.x = pb.x + pb.w;
          else player.x = pb.x - player.w;
          player.vx = 0;
        } else {
          if (dy1 < dy2) {
            player.y = pb.y + pb.h;
            player.vy = Math.max(player.vy, 0);
          } else {
            player.y = pb.y - player.h;
            player.vy = Math.min(player.vy, 0);
            player.onGround = true;
            player.canDrop = true;
            if ('_x' in p) {
              const carry = p.axis === 'x' ? Math.cos(p.t) * p.amp * p.speed * 0.03 : 0;
              player.x += carry;
            }
          }
        }
      }

      // Thin platforms
      for (const tp of level.thin) {
        const ab = { x: player.x, y: player.y, w: player.w, h: player.h };
        const pb = { x: tp.x, y: tp.y, w: tp.w, h: tp.h };
        const touchingFromAbove =
          ab.x < pb.x + pb.w && ab.x + ab.w > pb.x &&
          ab.y + ab.h > pb.y && ab.y + ab.h < pb.y + pb.h + 16 &&
          player.vy >= 0;
        if (touchingFromAbove && !(input.down && player.canDrop)) {
          player.y = pb.y - player.h;
          player.vy = 0;
          player.onGround = true;
        }
      }

      // Hazards
      for (const hz of level.hazards) {
        const hb = { x: hz.x, y: hz.y, w: hz.w, h: hz.h };
        const ab = { x: player.x, y: player.y, w: player.w, h: player.h };
        if (rectsOverlap(ab, hb)) {
          reset(false);
          break;
        }
      }

      // Molecules
      for (const m of level.molecules) {
        if (m.collected) continue;
        const dx = (player.x + player.w/2) - m.x;
        const dy = (player.y + player.h/2) - m.y;
        if (dx*dx + dy*dy < (18*18)) {
          m.collected = true;
          player.molecules++;
        }
      }

      // Checkpoint
      const checkpointZone = { x: 10*U, y: 10*U-2*U, w: 6*U, h: 2*U };
      if (rectsOverlap({ x: player.x, y: player.y, w: player.w, h: player.h }, checkpointZone)) {
        player.checkpoint = { x: player.x, y: player.y - 2 };
      }
    });

    // Status
    const totalCollected = player1.molecules + player2.molecules;
    const totalNeeded = level.molecules.length;
    const atGoal1 = rectsOverlap({ x: player1.x, y: player1.y, w: player1.w, h: player1.h }, level.goal);
    const atGoal2 = rectsOverlap({ x: player2.x, y: player2.y, w: player2.w, h: player2.h }, level.goal);
    if ((atGoal1 || atGoal2) && totalCollected >= totalNeeded) {
      document.getElementById('status').textContent = "Completed! Press R to replay";
    } else {
      document.getElementById('status').textContent =
        `Checkpoint: ${player1.checkpoint.x ? 1 : 0} • Molecules: ${totalCollected}/${totalNeeded}`;
    }

    if (keys.r) reset(true);

    // Camera
    const targetX = (player1.x + player2.x)/2 + player1.w/2 - canvas.width/2;
    const targetY = (player1.y + player2.y)/2 + player1.h/2 - canvas.height/2;
    cam.x += (targetX - cam.x) * cam.lerp;
    cam.y += (targetY - cam.y) * cam.lerp;

    // --- Render ---
    ctx.save();
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    const bg1 = ctx.createLinearGradient(0, 0, 0, canvas.height);
    bg1.addColorStop(0, '#10131a'); bg1.addColorStop(1, '#0a0d12');
    ctx.fillStyle = bg1;
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.translate(-cam.x, -cam.y);

    ctx.strokeStyle = 'rgba(255,255,255,0.05)'; ctx.lineWidth = 1;
    for (let gx = -100*U; gx < 100*U; gx += 2*U) {
      ctx.beginPath(); ctx.moveTo(gx, -100*U); ctx.lineTo(gx, 100*U); ctx.stroke();
    }
    for (let gy = -100*U; gy < 100*U; gy += 2*U) {
      ctx.beginPath(); ctx.moveTo(-100*U, gy); ctx.lineTo(100*U, gy); ctx.stroke();
    }

    ctx.fillStyle = '#a6a9b6';
    for (const p of level.platforms) drawRoundedRect(p.x, p.y, p.w, p.h, 6);
    ctx.fillStyle = '#c7cad6';
    for (const m of level.moving) drawRoundedRect(m._x, m._y, m.w, m.h, 6);
    ctx.fillStyle = '#9093a0';
    for (const tp of level.thin) drawRoundedRect(tp.x, tp.y, tp.w, tp.h, 6);

    for (const hz of level.hazards) {
      const grad = ctx.createLinearGradient(hz.x, hz.y, hz.x, hz.y + hz.h);
      grad.addColorStop(0, '#4db4ff'); grad.addColorStop(1, '#2766a3');
      ctx.fillStyle = grad;
      drawRoundedRect(hz.x, hz.y, hz.w, hz.h, 10);
      ctx.fillStyle = 'rgba(255,255,255,0.8)';
      ctx.font = `${14}px monospace`;
      ctx.fillText('EToH', hz.x + 6, hz.y + hz.h - 6);
    }

    ctx.fillStyle = '#3bd37f';
    drawRoundedRect(level.goal.x, level.goal.y, level.goal.w, level.goal.h, 6);
    ctx.fillStyle = '#0b2e1a';
    ctx.font = `${16}px monospace`;
    ctx.fillText('EXIT', level.goal.x + 6, level.goal.y + level.goal.h/2
        : { left: keys.arrowleft, right: keys.arrowright, up: keys.arrowup, down: keys.arrowdown };

      const accel = player.onGround ? player.speed : player.speed * player.air;
      if (input.left) { player.vx -= accel * dt; player.facing = -1; }
      if (input.right) { player.vx += accel * dt; player.facing = 1; }

      // Apply stronger friction when no input
      if (player.onGround && !input.left && !input.right) {
        player.vx *= 0.6;
        if (Math.abs(player.vx) < 1) player.vx = 0;
      }

      // Jump
      if (input.up && player.onGround) {
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

      // Collision detection and resolution (same as in previous block)
      // You can reuse the collision logic from earlier in the script
      // including solid platforms, thin platforms, hazards, molecules, and checkpoint logic
    });
