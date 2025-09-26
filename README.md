<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Tower Platformer</title>
  <style>
    canvas {
      background: #222;
      display: block;
      margin: 0 auto;
    }
  </style>
</head>
<body>
  <canvas id="gameCanvas" width="800" height="600"></canvas>
  
  <!-- Import Player logic -->
  <script type="module">
    import { Player } from './Player.js';

    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');

    // Simple level floor
    const floorY = 550;

    // Create player
    const player = new Player(100, floorY - 50);

    // Input handling
    const keys = {};
    window.addEventListener('keydown', e => keys[e.code] = true);
    window.addEventListener('keyup', e => keys[e.code] = false);

    function gameLoop() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);

      // Update player
      player.update(keys, floorY);

      // Draw floor
      ctx.fillStyle = '#444';
      ctx.fillRect(0, floorY, canvas.width, canvas.height - floorY);

      // Draw player
      player.draw(ctx);

      requestAnimationFrame(gameLoop);
    }

    gameLoop();
  </script>
</body>
</html>


























<script>
class Player {
  constructor(x, y) {
    this.x = x;
    this.y = y;
    this.width = 40;
    this.height = 50;
    this.color = 'tomato';

    this.velX = 0;
    this.velY = 0;
    this.speed = 4;
    this.jumpStrength = 12;
    this.gravity = 0.6;
    this.grounded = false;
  }

  update(keys, floorY) {
    if (keys['ArrowLeft']) this.velX = -this.speed;
    else if (keys['ArrowRight']) this.velX = this.speed;
    else this.velX = 0;

    if (keys['Space'] && this.grounded) {
      this.velY = -this.jumpStrength;
      this.grounded = false;
    }

    this.velY += this.gravity;

    this.x += this.velX;
    this.y += this.velY;

    if (this.y + this.height >= floorY) {
      this.y = floorY - this.height;
      this.velY = 0;
      this.grounded = true;
    }
  }

  draw(ctx) {
    ctx.fillStyle = this.color;
    ctx.fillRect(this.x, this.y, this.width, this.height);
  }
}
</script>

