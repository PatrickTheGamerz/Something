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
























<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Tower Section</title>
  <style>
    body { margin:0; background:#111; color:#eee; font-family:sans-serif; }
    #game { display:block; width:100vw; height:100vh; }
  </style>
</head>
<body>
  <canvas id="game"></canvas>
  <script>
    const canvas = document.getElementById('game');
    const ctx = canvas.getContext('2d');
    function resize(){ canvas.width=innerWidth; canvas.height=innerHeight; }
    window.onresize=resize; resize();

    function drawTowers(){
      ctx.clearRect(0,0,canvas.width,canvas.height);
      ctx.fillStyle="#444";
      for(let i=0;i<5;i++){
        ctx.fillRect(100+i*120, canvas.height-200, 80, 200);
      }
    }
    drawTowers();
  </script>
</body>
</html>





















<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Player 1 Section</title>
  <style>
    body { margin:0; background:#111; }
    #game { display:block; width:100vw; height:100vh; background:#1a1a1a; }
  </style>
</head>
<body>
  <canvas id="game"></canvas>
  <script>
    const canvas=document.getElementById('game');
    const ctx=canvas.getContext('2d');
    function resize(){ canvas.width=innerWidth; canvas.height=innerHeight; }
    window.onresize=resize; resize();

    const player={x:100,y:100,w:40,h:40,color:"#e86f6f",speed:200};
    const keys={};
    window.addEventListener('keydown',e=>keys[e.key]=true);
    window.addEventListener('keyup',e=>keys[e.key]=false);

    let last=performance.now();
    function loop(t){
      const dt=(t-last)/1000; last=t;
      if(keys['w']) player.y-=player.speed*dt;
      if(keys['s']) player.y+=player.speed*dt;
      if(keys['a']) player.x-=player.speed*dt;
      if(keys['d']) player.x+=player.speed*dt;

      ctx.clearRect(0,0,canvas.width,canvas.height);
      ctx.fillStyle=player.color;
      ctx.fillRect(player.x,player.y,player.w,player.h);
      requestAnimationFrame(loop);
    }
    requestAnimationFrame(loop);
  </script>
</body>
</html>


























<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Player 2 Section</title>
  <style>
    body { margin:0; background:#111; }
    #game { display:block; width:100vw; height:100vh; background:#1a1a1a; }
  </style>
</head>
<body>
  <canvas id="game"></canvas>
  <script>
    const canvas=document.getElementById('game');
    const ctx=canvas.getContext('2d');
    function resize(){ canvas.width=innerWidth; canvas.height=innerHeight; }
    window.onresize=resize; resize();

    const player={x:200,y:200,w:40,h:40,color:"#6fb3e8",speed:200};
    const keys={};
    window.addEventListener('keydown',e=>keys[e.key]=true);
    window.addEventListener('keyup',e=>keys[e.key]=false);

    let last=performance.now();
    function loop(t){
      const dt=(t-last)/1000; last=t;
      if(keys['ArrowUp']) player.y-=player.speed*dt;
      if(keys['ArrowDown']) player.y+=player.speed*dt;
      if(keys['ArrowLeft']) player.x-=player.speed*dt;
      if(keys['ArrowRight']) player.x+=player.speed*dt;

      ctx.clearRect(0,0,canvas.width,canvas.height);
      ctx.fillStyle=player.color;
      ctx.fillRect(player.x,player.y,player.w,player.h);
      requestAnimationFrame(loop);
    }
    requestAnimationFrame(loop);
  </script>
</body>
</html>




  













































<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Game Sections</title>
  <style>
    body { margin:0; background:#0e0f14; color:#e8e8f0; font-family:sans-serif; }
    #menu { display:flex; flex-direction:column; align-items:center; justify-content:center; height:100vh; gap:16px; }
    button { padding:12px 24px; font-size:18px; border:none; border-radius:8px; background:#2b2f45; color:#fff; cursor:pointer; }
    button:hover { background:#3d4a6b; }
    .section { display:none; width:100vw; height:100vh; }
    canvas { display:block; width:100%; height:100%; }
  </style>
</head>
<body>
  <div id="menu">
    <h1>Game Menu</h1>
    <button onclick="showSection('tower')">Tower</button>
    <button onclick="showSection('player1')">Player 1</button>
    <button onclick="showSection('player2')">Player 2</button>
  </div>
  <div id="tower" class="section"><canvas id="towerCanvas"></canvas></div>
  <div id="player1" class="section"><canvas id="p1Canvas"></canvas></div>
  <div id="player2" class="section"><canvas id="p2Canvas"></canvas></div>
  <script>
    function showSection(id){
      document.getElementById('menu').style.display='none'
      document.querySelectorAll('.section').forEach(s=>s.style.display='none')
      document.getElementById(id).style.display='block'
      if(id==='tower') startTower()
      if(id==='player1') startP1()
      if(id==='player2') startP2()
    }
    function startTower(){
      const c=document.getElementById('towerCanvas'),ctx=c.getContext('2d')
      c.width=innerWidth;c.height=innerHeight
      ctx.fillStyle='#444'
      for(let i=0;i<5;i++){ctx.fillRect(100+i*120,c.height-200,80,200)}
    }
    function startP1(){
      const c=document.getElementById('p1Canvas'),ctx=c.getContext('2d')
      c.width=innerWidth;c.height=innerHeight
      const p={x:100,y:100,w:40,h:40,color:'#e86f6f',speed:200}
      const keys={}
      onkeydown=e=>keys[e.key]=true
      onkeyup=e=>keys[e.key]=false
      let last=performance.now()
      function loop(t){
        const dt=(t-last)/1000;last=t
        if(keys['w'])p.y-=p.speed*dt
        if(keys['s'])p.y+=p.speed*dt
        if(keys['a'])p.x-=p.speed*dt
        if(keys['d'])p.x+=p.speed*dt
        ctx.clearRect(0,0,c.width,c.height)
        ctx.fillStyle=p.color
        ctx.fillRect(p.x,p.y,p.w,p.h)
        requestAnimationFrame(loop)
      }
      requestAnimationFrame(loop)
    }
    function startP2(){
      const c=document.getElementById('p2Canvas'),ctx=c.getContext('2d')
      c.width=innerWidth;c.height=innerHeight
      const p={x:200,y:200,w:40,h:40,color:'#6fb3e8',speed:200}
      const keys={}
      onkeydown=e=>keys[e.key]=true
      onkeyup=e=>keys[e.key]=false
      let last=performance.now()
      function loop(t){
        const dt=(t-last)/1000;last=t
        if(keys['ArrowUp'])p.y-=p.speed*dt
        if(keys['ArrowDown'])p.y+=p.speed*dt
        if(keys['ArrowLeft'])p.x-=p.speed*dt
        if(keys['ArrowRight'])p.x+=p.speed*dt
        ctx.clearRect(0,0,c.width,c.height)
        ctx.fillStyle=p.color
        ctx.fillRect(p.x,p.y,p.w,p.h)
        requestAnimationFrame(loop)
      }
      requestAnimationFrame(loop)
    }
  </script>
</body>
</html>

