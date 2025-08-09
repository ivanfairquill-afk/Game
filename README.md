# Game
Swing
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Spider Swing — Simple Browser Game</title>
  <style>
    :root{background:#111}
    html,body{height:100%;margin:0}
    canvas{display:block;background:linear-gradient(#87CEEB,#78A0D2 60%,#5c6c88);width:100%;height:100vh}
    #ui{position:fixed;left:16px;top:12px;color:#fff;font-family:system-ui,Arial;background:rgba(0,0,0,0.25);padding:8px 12px;border-radius:8px}
    #hint{position:fixed;right:16px;top:12px;color:#fff;font-family:system-ui,Arial;background:rgba(0,0,0,0.25);padding:8px 12px;border-radius:8px}
    button{margin-top:6px}
  </style>
</head>
<body>
  <canvas id="c"></canvas>
  <div id="ui">Score: <span id="score">0</span><br>Best: <span id="best">0</span><br><button id="restart">Restart</button></div>
  <div id="hint">Click / Tap to shoot web. Click again to release.</div>

<script>
// Simple Spider Swing — single file
(() => {
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d');
  let W = canvas.width = innerWidth;
  let H = canvas.height = innerHeight;

  window.addEventListener('resize', ()=>{W=canvas.width=innerWidth; H=canvas.height=innerHeight; generateBuildings();});

  // Game variables
  let buildings = [];
  let player = {x:150,y:200,vx:0,vy:0,r:12,attached:false,ax:0,ay:0,ropeLen:0};
  const gravity = 0.9;
  let mouse = {x:0,y:0};
  let score = 0, best = localStorage.getItem('spider_best')||0;
  document.getElementById('best').innerText = best;

  function rand(a,b){return Math.random()*(b-a)+a}

  function generateBuildings(){
    buildings = [];
    let x = -200;
    while(x < W*2){
      const w = rand(80,220);
      const h = rand(H*0.15, H*0.5);
      buildings.push({x, w, h});
      x += w + rand(60,220);
    }
  }

  function reset(){
    player.x = 150; player.y = 200; player.vx = 0; player.vy = 0; player.attached=false;
    score = 0; document.getElementById('score').innerText = score;
    generateBuildings();
  }

  generateBuildings();

  // Input
  canvas.addEventListener('mousemove', e=>{mouse.x=e.clientX; mouse.y=e.clientY});
  function pointerDown(e){
    const x = (e.touches ? e.touches[0].clientX : e.clientX);
    const y = (e.touches ? e.touches[0].clientY : e.clientY);
    mouse.x = x; mouse.y = y;
    if(!player.attached){
      // shoot web: check if we hit a rooftop (top edge of any building)
      for(let b of buildings){
        const bx = b.x - camX;
        if(mouse.x >= bx && mouse.x <= bx + b.w){
          const rooftopY = H - b.h;
          if(mouse.y <= rooftopY + 20){
            player.attached = true;
            player.ax = mouse.x + camX;
            player.ay = rooftopY;
            player.ropeLen = Math.hypot(player.x - player.ax, player.y - player.ay);
            return;
          }
        }
      }
    } else {
      // release web -> give a little momentum outward
      player.attached = false;
      // optional small impulse away from anchor
      const dx = player.x - player.ax;
      const dy = player.y - player.ay;
      const mag = Math.hypot(dx,dy)||1;
      player.vx += (dx/mag)*6;
      player.vy += (dy/mag)*2;
    }
  }
  canvas.addEventListener('mousedown', pointerDown);
  canvas.addEventListener('touchstart', pointerDown);

  // Camera
  let camX = 0;

  function update(){
    // physics
    if(!player.attached){
      player.vy += gravity*0.6;
      player.x += player.vx;
      player.y += player.vy;
      // air drag
      player.vx *= 0.995;
      // ground collision with buildings
      for(let b of buildings){
        const bx = b.x - camX;
        const rooftopY = H - b.h;
        if(player.x + player.r > b.x - camX && player.x - player.r < b.x - camX + b.w){
          if(player.y + player.r > rooftopY && player.y + player.r < rooftopY + 30){
            player.y = rooftopY - player.r;
            player.vy = 0;
            player.vx *= 0.6;
          }
        }
      }
    } else {
      // attached: simple pendulum constraint
      // position relative to anchor
      const dx = player.x - player.ax;
      const dy = player.y - player.ay;
      const dist = Math.hypot(dx,dy);
      // correct position to rope length
      const diff = (dist - player.ropeLen)/dist;
      player.x -= dx * diff;
      player.y -= dy * diff;
      // simple velocity from constraint
      player.vx += ( -dx * 0.02 );
      player.vy += ( -dy * 0.02 ) + gravity*0.05;
      player.x += player.vx; player.y += player.vy;
    }

    // update camera to follow player, with bounds
    camX = Math.max(0, player.x - 200);

    // scoring: passed building index
    for(let b of buildings){
      if(!b.passed && (player.x - camX) > b.x - camX + b.w){ b.passed = true; score++; document.getElementById('score').innerText = score; if(score>best){best=score; localStorage.setItem('spider_best',best); document.getElementById('best').innerText = best}} }

    // if fall below screen -> reset
    if(player.y > H + 200){
      // game over
      reset();
    }
  }

  function draw(){
    // sky is canvas background via CSS; draw ground gradient
    ctx.clearRect(0,0,W,H);

    // draw buildings
    for(let b of buildings){
      const bx = Math.floor(b.x - camX);
      const by = Math.floor(H - b.h);
      // building body
      ctx.fillStyle = '#222';
      ctx.fillRect(bx, by, Math.floor(b.w), Math.floor(b.h));
      // rooftop highlight
      ctx.fillStyle = '#333';
      ctx.fillRect(bx, by, Math.floor(b.w), 8);
      // windows
      ctx.fillStyle = 'rgba(255,230,100,0.08)';
      for(let i=10;i<b.w-10;i+=24){
        for(let j=12;j<b.h-24;j+=28){
          if(Math.random()>0.88) ctx.fillRect(bx+i, by+j, 10, 14);
        }
      }
    }

    // draw rope
    if(player.attached){
      ctx.beginPath();
      ctx.moveTo(player.ax - camX, player.ay);
      ctx.lineTo(player.x - camX, player.y);
      ctx.strokeStyle = 'rgba(240,240,240,0.9)';
      ctx.lineWidth = 2.5;
      ctx.stroke();
    }

    // draw spider (circle body + eyes)
    ctx.save();
    ctx.translate(player.x - camX, player.y);
    // legs
    ctx.strokeStyle = '#111'; ctx.lineWidth = 2;
    for(let i=0;i<6;i++){
      const ang = Math.PI/6 + (i-2.5)*0.25;
      ctx.beginPath(); ctx.moveTo(0,4); ctx.lineTo(Math.cos(ang)*18, Math.sin(ang)*18); ctx.stroke();
    }
    // body
    ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(0,0,12,0,Math.PI*2); ctx.fill();
    ctx.fillStyle = '#222'; ctx.beginPath(); ctx.arc(0,4,7,0,Math.PI*2); ctx.fill();
    // eyes
    ctx.fillStyle = '#fff'; ctx.beginPath(); ctx.arc(-4,-2,3,0,Math.PI*2); ctx.arc(4,-2,3,0,Math.PI*2); ctx.fill();
    ctx.fillStyle = '#000'; ctx.beginPath(); ctx.arc(-3,-2,1.2,0,Math.PI*2); ctx.arc(5,-2,1.2,0,Math.PI*2); ctx.fill();
    ctx.restore();

    // HUD
    ctx.fillStyle = 'rgba(0,0,0,0.2)'; ctx.fillRect(12, H-54, 220, 42);
    ctx.fillStyle = '#fff'; ctx.font = '18px system-ui'; ctx.fillText('Click/tap: attach / release', 20, H-30);
  }

  // main loop
  function loop(){ update(); draw(); requestAnimationFrame(loop); }
  loop();

  // restart button
  document.getElementById('restart').addEventListener('click', ()=>{reset();});

  // initial friendly note: a tiny tutorial
  canvas.addEventListener('mousemove', e=>{});

  // Start with reset to position everything clean
  reset();
})();
</script>
</body>
</html>

