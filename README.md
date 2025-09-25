<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>EToH Ring 1 Platformer–2P Split-Screen</title>
  <style>
    html, body {
      margin: 0;
      padding: 0;
      background: #1a1a1a;
      height: 100%;
      overflow: hidden;
    }
    body {
      width: 100vw;
      height: 100vh;
    }
    #gameCanvas {
      display: block;
      margin: 0 auto;
      background: #222;
      width: 100vw;
      height: 100vh;
      border: none;
    }
    #info {
      position: absolute;
      top: 10px; left: 0; right: 0;
      text-align: center;
      color: #eee;
      font-family: 'Segoe UI', Arial, sans-serif;
      font-size: 1.1em;
      z-index: 10;
      pointer-events: none;
      text-shadow: 0 2px 10px #000A;
    }
    #winnerBanner {
      position: absolute;
      left: 50%; top: 40%;
      transform: translate(-50%,-50%);
      color: #ffffcc; 
      font-size: 3em;
      font-family: 'Segoe UI', Arial, sans-serif;
      font-weight: bold;
      text-shadow: 0 4px 24px #202055cc, 2px 2px 0 #200;
      background: rgba(40,40,110,0.9);
      padding: 0.6em 2.2em;
      border-radius: 0.7em;
      box-shadow: 0 0 32px #1b1f3b80;
      pointer-events: none;
      opacity: 0.97;
      z-index: 100;
      display: none;
    }
    /* Split screen divider */
    #splitDiv {
      position: absolute;
      top: 0; bottom: 0;
      width: 8px;
      left: 50%;
      margin-left: -4px;
      background: linear-gradient(#fffff6a0, #414170cf 50%, #aaa1ff50);
      z-index: 20;
      border-radius: 3px;
      display: none;
      pointer-events: none;
    }
  </style>
</head>
<body>
  <div id="info">
    <b>[EToH Ring 1 Split-Screen Platformer]</b>
    <br>
    P1: <span style="color:#f66">A/D = Move, W = Jump</span>
    &nbsp;&nbsp; | &nbsp;&nbsp;
    P2: <span style="color:#6af">←/→ = Move, ↑ = Jump</span>
    &nbsp;&nbsp; | &nbsp;&nbsp;
    First to reach the win pad at the top wins!
    <br>
    Press R to restart after a win.
  </div>
  <canvas id="gameCanvas" width="1280" height="720"></canvas>
  <div id="winnerBanner"></div>
  <div id="splitDiv"></div>
  
  <script>
  // ==== CONFIGURATION ====

  const CANVAS_WIDTH = 1280;
  const CANVAS_HEIGHT = 720;
  const VIEWPORT_W = CANVAS_WIDTH / 2;
  const VIEWPORT_H = CANVAS_HEIGHT;
  const GRAVITY = 0.95;
  const MOVE_SPEED = 4.2;
  const JUMP_VEL = -13.5;
  const PLAYER_W = 37, PLAYER_H = 54;
  const SPLIT_DISTANCE = 380; // Minimum distance before splitting view

  // ==== COLOR PALETTES ====
  const towerColors = {
    ToAST: {
      bg: '#bceeff', floor: '#90e8a8', plat: '#7ce87e', hazard: '#e87171'
    },
    ToA:   {
      bg: '#ffd9b2', floor: '#ffb24a', plat: '#ff8444', hazard: '#db328b'
    },
    ToM:   {
      bg: '#f6e9ff', floor: '#87bdff', plat: '#24aafe', hazard: '#7c60c0'
    },
    ToNI:  {
      bg: '#ececff', floor: '#bfa8ff', plat: '#745ebd', hazard: '#de43c6'
    },
  };
  const PLAYER1_COLOR = '#ff6675', PLAYER2_COLOR = '#58aaff';

  // ==== DATA STRUCTURES ====
  // Each tower as {id, name, floors: [ { platforms:[], hazards:[], ... } ]}
  // Each platform/hazard: { x,y,w,h,type }
  // All Y coordinates: y=0 is top of the tower, increase downward

  // Helper for making floor layout concise
  function floorY(floorIdx, floorH) { return floorIdx * floorH; }

  // Tower of A Simple Time: tutorial, wide easy floors
  const ToAST = {
    id: 'ToAST',
    name: 'Tower of A Simple Time',
    floorH: 110,
    floors: [
      // Floor 1 (bottom)
      { platforms: [
          { x:70, y:780, w:320, h:30, type:'plat' },
          { x:425, y:730, w:105, h:25, type:'plat' },
        ], hazards: [] },
      // Floor 2
      { platforms: [
          { x:175, y:650, w:280, h:24, type:'plat' },
          { x:355, y:595, w:70, h:16, type:'plat' },
        ], hazards: [ { x:125, y:670, w:34, h:12, type:'spike' } ] },
      // Floor 3
      { platforms: [
          { x:495, y:520, w:240, h:23, type:'plat' },
          { x:363, y:470, w:40, h:15, type:'plat' },
        ], hazards: [ { x:590, y:530, w:27, h:13, type:'spike' } ] },
      // Floor 4
      { platforms: [
          { x:650, y:410, w:230, h:22, type:'plat' },
          { x:872, y:355, w:105, h:20, type:'plat' },
        ], hazards: [ { x:725, y:425, w:37, h:15, type:'spike' } ] },
      // Floor 5 (upper)
      { platforms: [
          { x:1050, y:290, w:190, h:24, type:'plat' },
          { x:1220, y:210, w:35, h:12, type:'plat' },
        ], hazards: [ { x:1190, y:300, w:38, h:12, type:'spike' } ] },
      // Floor 6
      { platforms: [
          { x:900, y:150, w:165, h:19, type:'plat' },
          { x:1050, y:110, w:110, h:22, type:'plat' }
        ], hazards: [ { x:950, y:160, w:27, h:14, type:'spike' } ] },
      // Floor 7
      { platforms: [
          { x:485, y:65, w:220, h:17, type:'plat' },
        ], hazards: [ { x:610, y:70, w:30, h:9, type:'spike' } ] },
      // Floor 8 (top, win pad)
      { platforms: [
          { x:300, y:0, w:150, h:22, type:'win' }
        ], hazards: [] }
    ]
  };

  // Tower of Anger: denser, thinner platforms, orange, slightly harder
  const ToA = {
    id: 'ToA',
    name: 'Tower of Anger',
    floorH: 100,
    floors: [
      { platforms: [
          { x:40, y:760, w:200, h:20, type:'plat' },
          { x:320, y:705, w:80, h:15, type:'plat' }
        ], hazards: [ { x:60, y:770, w:22, h:12, type:'spike' } ] },
      { platforms: [
          { x:200, y:670, w:110, h:22, type:'plat' },
          { x:395, y:640, w:90, h:17, type:'plat' }
        ], hazards: [ { x:220, y:680, w:40, h:14, type:'spike' } ] },
      { platforms: [
          { x:620, y:585, w:150, h:22, type:'plat' },
        ], hazards: [ { x:670, y:595, w:25, h:12, type:'spike' } ] },
      { platforms: [
          { x:780, y:500, w:77, h:15, type:'plat' },
          { x:860, y:480, w:100, h:20, type:'plat' }
        ], hazards: [ { x:810, y:505, w:29, h:13, type:'spike' } ] },
      { platforms: [
          { x:1020, y:415, w:140, h:18, type:'plat' }
        ], hazards: [ { x:1050, y:425, w:45, h:16, type:'spike' } ] },
      { platforms: [
          { x:1180, y:345, w:95, h:15, type:'plat' }
        ], hazards: [ { x:1199, y:352, w:50, h:12, type:'spike' } ] },
      { platforms: [
          { x:750, y:245, w:230, h:18, type:'plat' },
        ], hazards: [ { x:900, y:255, w:35, h:12, type:'spike' } ] },
      { platforms: [
          { x:600, y:130, w:120, h:22, type:'plat' },
        ], hazards: [] },
      { platforms: [
          { x:425, y:40, w:100, h:18, type:'plat' },
        ], hazards: [ { x:460, y:46, w:25, h:12, type:'spike' } ] },
      { platforms: [
          { x:280, y:0, w:155, h:25, type:'win' }
        ], hazards: [] }
    ]
  };

  // Tower of Madness: playful, jumps, moving items (shown static), slightly harder
  const ToM = {
    id: 'ToM',
    name: 'Tower of Madness',
    floorH: 110,
    floors: [
      { platforms: [
          { x:50, y:795, w:160, h:27, type:'plat' },
          { x:285, y:730, w:90, h:20, type:'plat' }
        ], hazards: [ { x:74, y:800, w:38, h:10, type:'spike' } ] },
      { platforms: [
          { x:180, y:670, w:115, h:23, type:'plat' },
          { x:340, y:610, w:60, h:15, type:'plat' }
        ], hazards: [ { x:195, y:684, w:32, h:9, type:'spike' } ] },
      { platforms: [
          { x:480, y:530, w:135, h:17, type:'plat' },
          { x:630, y:500, w:95, h:15, type:'plat' }
        ], hazards: [ { x:540, y:545, w:23, h:11, type:'spike' } ] },
      { platforms: [
          { x:730, y:447, w:145, h:25, type:'plat' }
        ], hazards: [ { x:770, y:460, w:40, h:12, type:'spike' } ] },
      { platforms: [
          { x:870, y:386, w:85, h:15, type:'plat' },
          { x:1040, y:340, w:89, h:20, type:'plat' }
        ], hazards: [ { x:895, y:390, w:52, h:16, type:'spike' } ] },
      { platforms: [
          { x:1120, y:290, w:89, h:20, type:'plat' }
        ], hazards: [ { x:1135, y:295, w:32, h:14, type:'spike' } ] },
      { platforms: [
          { x:920, y:220, w:162, h:19, type:'plat' }
        ], hazards: [ { x:950, y:230, w:52, h:14, type:'spike' } ] },
      { platforms: [
          { x:590, y:135, w:190, h:20, type:'plat' }
        ], hazards: [ { x:670, y:140, w:43, h:14, type:'spike' } ] },
      { platforms: [
          { x:285, y:55, w:120, h:21, type:'plat' },
        ], hazards: [ { x:320, y:62, w:41, h:13, type:'spike' } ] },
      { platforms: [
          { x:170, y:0, w:150, h:23, type:'win' }
        ], hazards: [] }
    ]
  };

  // Tower of Noticeable Infuriation: many hazards, trial look, thin ledges
  const ToNI = {
    id: 'ToNI',
    name: 'Tower of Noticeable Infuriation',
    floorH: 120,
    floors: [
      { platforms: [
          { x:60, y:770, w:102, h:15, type:'plat' },
          { x:222, y:730, w:80, h:11, type:'plat' }
        ], hazards: [ { x:93, y:788, w:31, h:12, type:'spike' }, { x:137, y:785, w:41, h:14, type:'spike' } ] },
      { platforms: [
          { x:145, y:685, w:140, h:12, type:'plat' },
          { x:335, y:650, w:70, h:11, type:'plat' }
        ], hazards: [ { x:165, y:698, w:60, h:13, type:'spike' } ] },
      { platforms: [
          { x:520, y:590, w:90, h:11, type:'plat' },
          { x:652, y:570, w:54, h:10, type:'plat' }
        ], hazards: [ { x:547, y:599, w:27, h:9, type:'spike' } ] },
      { platforms: [
          { x:770, y:520, w:108, h:8, type:'plat' }
        ], hazards: [ { x:790, y:527, w:44, h:9, type:'spike' } ] },
      { platforms: [
          { x:920, y:470, w:82, h:8, type:'plat' }
        ], hazards: [ { x:926, y:478, w:33, h:9, type:'spike' } ] },
      { platforms: [
          { x:1050, y:410, w:77, h:11, type:'plat' }
        ], hazards: [ { x:1075, y:417, w:47, h:10, type:'spike' } ] },
      { platforms: [
          { x:875, y:325, w:189, h:10, type:'plat' }
        ], hazards: [ { x:900, y:335, w:62, h:9, type:'spike' } ] },
      { platforms: [
          { x:460, y:235, w:130, h:14, type:'plat' },
          { x:395, y:200, w:77, h:9, type:'plat' }
        ], hazards: [ { x:470, y:255, w:28, h:10, type:'spike' } ] },
      { platforms: [
          { x:210, y:125, w:93, h:12, type:'plat' }
        ], hazards: [ { x:265, y:130, w:27, h:10, type:'spike' } ] },
      { platforms: [
          { x:120, y:0, w:118, h:23, type:'win' }
        ], hazards: [] }
    ]
  };

  // All towers data
  const towers = [ToAST, ToA, ToM, ToNI];

  // ---- UTILITY ----

  function randInt(a,b){ return Math.floor(Math.random()*(b-a))+a; }

  // ==== GAME STATE ====
  let currentTower;
  let towerColor;
  let floorH = 100, totalFloors;
  let players = [];
  let winPad = null;
  let gameState = 'playing'; // 'win', 'playing'
  let winner = null;
  let winnerBanner = document.getElementById('winnerBanner');
  let splitDiv = document.getElementById('splitDiv');

  // Player class
  class Player {
    constructor(id, color, spawnX, spawnY, controls) {
      this.id = id;
      this.x = spawnX;
      this.y = spawnY;
      this.vx = 0;
      this.vy = 0;
      this.color = color;
      this.onGround = false;
      this.controls = controls;
      this.keyState = {};
      this.won = false;
    }
    get rect(){ return {x:this.x, y:this.y, w:PLAYER_W, h:PLAYER_H}; }
    reset(x,y){
      this.x = x; this.y = y; this.vx=0; this.vy=0; this.onGround=false; this.won=false;
    }
  }

  // ==== Input HANDLING ====
  // Controls: P1: A/D/W; P2: Left/Right/Up
  function setupPlayersForTower() {
    // First platform of first floor as spawn
    let p1 = currentTower.floors[0].platforms[0];
    let px1 = p1.x+8, py1 = p1.y-PLAYER_H-2;
    let px2 = px1+60, py2 = py1;

    players = [
      new Player(1, PLAYER1_COLOR, px1, py1, {
        left: 'KeyA', right: 'KeyD', jump: 'KeyW'
      }),
      new Player(2, PLAYER2_COLOR, px2, py2, {
        left: 'ArrowLeft', right: 'ArrowRight', jump: 'ArrowUp'
      })
    ];
  }
  // Keyboard events
  window.addEventListener('keydown', (e)=>{
    if (e.repeat) return;
    // For both players
    for (let pl of players) pl.keyState[e.code] = true;
    // Restart
    if ((e.code==='KeyR') && gameState==='win') restartGame();
  });
  window.addEventListener('keyup', (e)=>{
    for (let pl of players) pl.keyState[e.code] = false;
  });

  // ==== RANDOM TOWER SELECTION ====
  function selectRandomTower() {
    let idx = randInt(0, towers.length);
    currentTower = towers[idx];
    towerColor = towerColors[currentTower.id];
    floorH = currentTower.floorH;
    totalFloors = currentTower.floors.length;
  }

  // ==== GAME LOGIC ====
  function resetForNewTower() {
    selectRandomTower();
    setupPlayersForTower();
    winPad = null;
    // Find win pad
    let winFloor = currentTower.floors[ currentTower.floors.length-1 ];
    for (let p of winFloor.platforms) if (p.type==='win') winPad = p;
    gameState = 'playing'; winner = null;
    winnerBanner.style.display = 'none';
    splitDiv.style.display = 'none';
  }
  resetForNewTower();

  // ==== PHYSICS and MOVEMENT ====

  function updatePlayers() {
    for (let plIdx=0; plIdx<players.length; ++plIdx){
      let p = players[plIdx];
      if (gameState!=='playing') continue;
      // Horizontal movement
      let left = !!p.keyState[p.controls.left];
      let right = !!p.keyState[p.controls.right];
      let jump = !!p.keyState[p.controls.jump];
      if (left && !right) p.vx = -MOVE_SPEED;
      else if (right && !left) p.vx = MOVE_SPEED;
      else p.vx = 0;
      // Apply gravity
      p.vy += GRAVITY;
      if (p.vy > 13) p.vy = 13;
      // Jump
      if (jump && p.onGround) {
        p.vy = JUMP_VEL;
        p.onGround = false;
      }
      // Move
      let origX=p.x, origY=p.y;
      p.x += p.vx;
      // X Collisions (platforms + hazard boundaries)
      let collidedX = false;
      for (let fl of currentTower.floors){
        for (let plat of fl.platforms){
          if (rectsOverlap(p.x, p.y,PLAYER_W,PLAYER_H, plat.x, plat.y,plat.w,plat.h) 
            && plat.type!=='win') {
            // Wall collisions (ignore win pad)
            if (p.vx<0) p.x = plat.x+plat.w+0.1;
            else if (p.vx>0) p.x = plat.x-PLAYER_W-0.1;
            collidedX = true;
          }
        }
      }
      // Revert X if collided
      if (collidedX) p.x=origX;
      // Y movement
      p.y += p.vy;
      // Y Collisions (reset onGround if landed)
      p.onGround = false;
      // Ground/platforms
      for (let fl of currentTower.floors){
        for (let plat of fl.platforms){
          if (rectsOverlap(p.x, p.y,PLAYER_W,PLAYER_H, plat.x, plat.y,plat.w,plat.h)) {
            // For each platform, separate as needed
            if (p.vy>0 && (origY+PLAYER_H)<=plat.y+7 && plat.type!=='win'){
                p.y = plat.y-PLAYER_H-0.1;
                p.vy=0; p.onGround=true;
            } else if (p.vy<0 && (origY)>=plat.y+plat.h-7) {
                // Hitting from below
                p.y = plat.y+plat.h+0.1; p.vy=0;
            } else if (plat.type==='win'){
              // Win pad
              onWin(plIdx);
              return;
            }
          }
        }
      }
      // Out of bounds
      if (p.y > 900){
        // Respawn
        let sp = currentTower.floors[0].platforms[0];
        p.reset(sp.x+10+plIdx*80, sp.y-PLAYER_H-2);
      }
      // Hazards: spikes kill//respawn
      for (let fl of currentTower.floors) for (let hz of fl.hazards)
        if (rectsOverlap(p.x, p.y,PLAYER_W,PLAYER_H, hz.x,hz.y,hz.w,hz.h)){
          // Respawn
          let sp = currentTower.floors[0].platforms[0];
          p.reset(sp.x+12+plIdx*85, sp.y-PLAYER_H-2);
        }
    }
  }

  function onWin(playerIdx){
    gameState = 'win';
    winner = players[playerIdx];
    winner.won = true;
    winnerBanner.innerHTML = `🏆 Player ${winner.id} Wins! <br><span style="font-size:60%;">(${currentTower.name})</span>`;
    winnerBanner.style.display = '';
    setTimeout(()=>{ winnerBanner.style.opacity='1'; }, 10);
  }
  function restartGame(){
    resetForNewTower();
  }

  // ==== COLLISION UTILITY ====
  function rectsOverlap(ax,ay,aw,ah, bx,by,bw,bh){
    return (ax < bx+bw && ax+aw > bx && ay < by+bh && ay+ah > by);
  }

  // ==== CAMERA AND SPLIT-SCREEN ====
  function getViewports(){
    // Calculate world distance between players
    let cx1 = players[0].x+PLAYER_W/2, cy1 = players[0].y+PLAYER_H/2;
    let cx2 = players[1].x+PLAYER_W/2, cy2 = players[1].y+PLAYER_H/2;
    let dist = Math.sqrt( (cx1-cx2)**2 + (cy1-cy2)**2 );
    if (dist < SPLIT_DISTANCE){
      // Shared viewport: includes both players, zoom as needed
      let midX = (cx1+cx2)/2, midY = (cy1+cy2)/2;
      return [ 
        {x: midX-CANVAS_WIDTH/4, y: midY-CANVAS_HEIGHT/2, w:CANVAS_WIDTH/2, h:CANVAS_HEIGHT},
        {x: midX-CANVAS_WIDTH/4, y: midY-CANVAS_HEIGHT/2, w:CANVAS_WIDTH/2, h:CANVAS_HEIGHT},
        true // merged
      ];
    } else {
      // Split: each player gets own
      let vps = [];
      for (let pl of players){
        let camX = pl.x + PLAYER_W/2 - VIEWPORT_W/2;
        let camY = pl.y + PLAYER_H/2 - VIEWPORT_H/2;
        // Clamp so view not outside tower area
        camX = Math.max(0, Math.min(camX, 1280-VIEWPORT_W) );
        camY = Math.max(-70, Math.min(camY, 810-VIEWPORT_H) );
        vps.push( { x: camX, y: camY, w:VIEWPORT_W, h:VIEWPORT_H });
      }
      return [ vps[0], vps[1], false ];
    }
  }

  // ==== DRAWING ====
  const canv = document.getElementById('gameCanvas');
  const ctx = canv.getContext('2d');

  function draw() {
    ctx.clearRect(0,0,CANVAS_WIDTH,CANVAS_HEIGHT);

    let [vp1, vp2, merged] = getViewports();

    // Split line
    if (!merged){
      splitDiv.style.display = '';
      // Fill vertical bar on split
      ctx.save();
      ctx.globalAlpha = 0.7;
      ctx.fillStyle = 'rgba(120,120,185,0.14)';
      ctx.fillRect(CANVAS_WIDTH/2-4,0,8,CANVAS_HEIGHT);
      ctx.restore();
    } else {
      splitDiv.style.display = 'none';
    }

    // Viewport draws
    for(let vi=0; vi<2; ++vi){
      // If merged, draw only once
      if (merged && vi==1) continue;
      let vp = vi===0?vp1:vp2;
      ctx.save();
      // Set viewport
      ctx.beginPath();
      ctx.rect(vi*VIEWPORT_W,0,VIEWPORT_W,VIEWPORT_H);
      ctx.clip();
      // Translate to camera
      ctx.translate(vi*VIEWPORT_W-vp.x, -vp.y);

      drawTower(vi, vp);

      // Draw other player only if in view
      for (let pi=0; pi<2; ++pi){
        if (merged || pi===vi){
          drawPlayer(players[pi], pi+1);
        }
        else if (rectsOverlap(players[pi].x,players[pi].y, PLAYER_W,PLAYER_H,
          vp.x, vp.y, vp.w, vp.h )){
          drawPlayer(players[pi], pi+1);
        }
      }
      // Own player always drawn
      ctx.restore();
    }
    // Frame and player labels
    if (!merged){
      ctx.save();
      ctx.fillStyle = '#221cee';
      ctx.globalAlpha = 0.10;
      ctx.fillRect(0,0,VIEWPORT_W,CANVAS_HEIGHT);
      ctx.fillStyle = '#1cbcd1';
      ctx.fillRect(VIEWPORT_W,0,VIEWPORT_W,CANVAS_HEIGHT);
      ctx.restore();
    }
  }

  function drawTower(viewIdx, vp){
    // Draw each floor background
    for (let floor=0; floor<currentTower.floors.length; ++floor){
      let fy = 800-floor*floorH-10;
      ctx.save();
      ctx.fillStyle = towerColor.bg;
      ctx.globalAlpha = 0.24;
      ctx.fillRect(0, fy, 1280, floorH+15 );
      ctx.restore();
    }
    // Draw each floor separator
    for (let floor=0; floor<currentTower.floors.length; ++floor){
      let fy = 800-floor*floorH-5;
      ctx.save();
      ctx.strokeStyle = towerColor.floor;
      ctx.lineWidth = 3.1;
      ctx.globalAlpha = 0.98;
      ctx.beginPath();
      ctx.moveTo(0, fy);
      ctx.lineTo(1280, fy);
      ctx.stroke();
      ctx.restore();
      // Floor number
      ctx.save();
      ctx.font = '19px Segoe UI, Arial';
      ctx.fillStyle = '#333';
      ctx.fillText('F' + (floor+1), 8, fy+23);
      ctx.restore();
    }
    // Platforms
    for (let fl of currentTower.floors){
      for (let plat of fl.platforms){
        if (plat.type==='win')
          drawWinPad(plat.x, plat.y, plat.w, plat.h);
        else
          drawPlatform(plat.x, plat.y, plat.w, plat.h);
      }
    }
    // Hazards
    for (let fl of currentTower.floors) for (let hz of fl.hazards){
      drawHazard(hz.x, hz.y, hz.w, hz.h);
    }
  }

  function drawPlatform(x, y, w, h){
    ctx.save();
    ctx.globalAlpha = 0.94;
    ctx.fillStyle = towerColor.plat;
    ctx.strokeStyle = "#eeeeee";
    ctx.lineWidth = 2.5;
    ctx.beginPath();
    ctx.roundRect(x, y, w, h, 7.5);
    ctx.fill();
    ctx.stroke();
    ctx.restore();
  }
  function drawWinPad(x, y, w, h){
    ctx.save();
    // Pad with rainbow gradient
    let grd = ctx.createLinearGradient(x, y, x+w, y+h);
    grd.addColorStop(0, "#ffe367");
    grd.addColorStop(0.31,"#9ae7ae");
    grd.addColorStop(0.64,"#52afff");
    grd.addColorStop(1,"#cac6ee");
    ctx.fillStyle = grd;
    ctx.strokeStyle = "#aaaaee";
    ctx.lineWidth = 5;
    ctx.beginPath();
    ctx.roundRect(x, y, w, h, 10);
    ctx.fill();
    ctx.globalAlpha = 0.86;
    ctx.stroke();
    ctx.font = 'bold 19px Segoe UI, Arial';
    ctx.fillStyle = '#3850e8';
    ctx.fillText('WIN!', x+Math.max(12, (w-50)/2), y+Math.max(23, h-6)/2);
    ctx.restore();
  }
  function drawHazard(x,y,w,h){
    ctx.save();
    ctx.globalAlpha = 0.90;
    ctx.fillStyle = towerColor.hazard;
    ctx.beginPath();
    // Draw triangle (spike)
    ctx.moveTo(x, y+h);
    ctx.lineTo(x+w/2, y-7);
    ctx.lineTo(x+w, y+h);
    ctx.closePath();
    ctx.fill();
    ctx.restore();
  }
  function drawPlayer(pl, n){
    ctx.save();
    ctx.translate(pl.x, pl.y);
    ctx.globalAlpha = pl.won ? 0.65 : 1.0;
    // Body
    ctx.fillStyle = pl.id==1 ? PLAYER1_COLOR : PLAYER2_COLOR;
    ctx.strokeStyle = pl.won ? "#fffef0" : "#16161d";
    ctx.lineWidth = 2.3;
    ctx.beginPath();
    ctx.roundRect(0, 0, PLAYER_W, PLAYER_H, 9.2 );
    ctx.fill();
    ctx.stroke();
    // Face
    ctx.fillStyle = '#fff'; ctx.beginPath();
    ctx.ellipse(PLAYER_W/2,PLAYER_H/2-7,11,12,0,0,Math.PI*2);
    ctx.fill();
    ctx.fillStyle = '#444'; // Eyes
    ctx.beginPath();
    ctx.arc(PLAYER_W/2-5,PLAYER_H/2-9,2,0,2*Math.PI);
    ctx.arc(PLAYER_W/2+5,PLAYER_H/2-9,2,0,2*Math.PI);
    ctx.fill();
    // ID label
    ctx.font = 'bold 17.5px Segoe UI, Arial';
    ctx.fillStyle = pl.id==1 ? '#ffbbba' : '#b7e1ff';
    ctx.fillText(`P${pl.id}`, 4, PLAYER_H-10);
    ctx.restore();
  }

  // ==== MAIN GAME LOOP ====
  function gameloop(){
    if (gameState==='playing')
      updatePlayers();
    draw();
    requestAnimationFrame(gameloop);
  }
  gameloop();

  // ==== GAME RESTART HANDLING ====
  // Press R to restart.

  // ==== RESIZING ====
  // Optionally, resize canvas to window, but world coords remain fixed.
  window.addEventListener('resize', ()=>{
    canv.width = CANVAS_WIDTH;
    canv.height = CANVAS_HEIGHT;
  });

  </script>
</body>
</html>
