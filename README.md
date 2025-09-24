<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no" />
  <title>Space Shooter Bonito — Planetas & Lua</title>
  <style>
    :root { --ui-size: 100px; }
    html, body { margin: 0; height: 100%; background: #000; color: #fff; font-family: Arial, Helvetica, sans-serif; }
    canvas { display: block; width: 100vw; height: 100vh; }

    /* Joystick */
    #joystick {
      position: absolute; bottom: 24px; left: 24px;
      width: var(--ui-size); height: var(--ui-size);
      background: radial-gradient( circle at 30% 30%, rgba(255,255,255,.18), rgba(255,255,255,.06) );
      border: 2px solid rgba(255,255,255,.25); border-radius: 50%;
      backdrop-filter: blur(2px); -webkit-backdrop-filter: blur(2px);
      touch-action: none; user-select: none;
    }
    #stick {
      position: absolute; width: calc(var(--ui-size) * .5); height: calc(var(--ui-size) * .5);
      top: 25%; left: 25%; border-radius: 50%;
      background: radial-gradient(circle at 30% 30%, rgba(255,255,255,.5), rgba(255,255,255,.25));
      box-shadow: 0 8px 20px rgba(0,0,0,.35) inset, 0 2px 12px rgba(0,0,0,.35);
      pointer-events: none; transform: translate(0,0);
    }

    /* Botão de tiro */
    #shootBtn {
      position: absolute; bottom: 36px; right: 28px; width: 88px; height: 88px; border-radius: 50%;
      border: 2px solid rgba(255, 230, 0, .7); color: #fff; font-size: 20px; font-weight: 700;
      background: radial-gradient(circle at 35% 35%, rgba(255,220,0,.45), rgba(255,150,0,.25));
      box-shadow: 0 8px 24px rgba(255,180,0,.25), inset 0 0 12px rgba(255,220,0,.35);
      touch-action: manipulation;
    }

    /* HUD */
    #hud { position: absolute; top: 12px; left: 14px; display: flex; gap: 16px; align-items: center; }
    #score { font-size: 18px; text-shadow: 0 2px 6px rgba(0,255,255,.35); }
    #difficulty { font-size: 14px; opacity: .85; }
  </style>
</head>
<body>
  <canvas id="gameCanvas"></canvas>
  <div id="joystick"><div id="stick"></div></div>
  <button id="shootBtn">🔥</button>
  <div id="hud">
    <div id="score">Pontos: 0</div>
    <div id="difficulty">Dificuldade x1.0</div>
  </div>

  <script>
  // ===== Canvas HiDPI =====
  const canvas = document.getElementById('gameCanvas');
  const ctx = canvas.getContext('2d');
  let DPR = Math.max(1, Math.floor(window.devicePixelRatio || 1));
  let W = 0, H = 0;
  function resize(){
    DPR = Math.max(1, Math.floor(window.devicePixelRatio || 1));
    W = Math.floor(window.innerWidth * DPR);
    H = Math.floor(window.innerHeight * DPR);
    canvas.width = W; canvas.height = H;
    canvas.style.width = (W / DPR) + 'px';
    canvas.style.height = (H / DPR) + 'px';
  }
  window.addEventListener('resize', resize, {passive:true});
  resize();

  // ===== HUD =====
  const scoreEl = document.getElementById('score');
  const diffEl = document.getElementById('difficulty');

  // ===== Estado =====
  const state = {
    score: 0,
    diffMul: 1.0, // aumenta com pontuação
    time: 0,
    lastShot: 0
  };

  // ===== Jogador =====
  const player = { x: W*0.5, y: H*0.82, w: 46*DPR, h: 56*DPR, speed: 0.55*DPR };

  // ===== Controles =====
  const keys = {};
  window.addEventListener('keydown', (e)=>{ keys[e.key] = true; if(e.key === ' ') e.preventDefault(); });
  window.addEventListener('keyup', (e)=>{ keys[e.key] = false; });

  // Joystick
  const joyBase = document.getElementById('joystick');
  const joyStick = document.getElementById('stick');
  let joy = {active:false, dx:0, dy:0};
  function joyPos(e){ const t = e.touches? e.touches[0]: e; const r = joyBase.getBoundingClientRect(); return { x: t.clientX - (r.left + r.width/2), y: t.clientY - (r.top + r.height/2), r } }
  const JOY_DEAD = 0.12;
  function joyStart(e){ joy.active = true; moveJoy(e); e.preventDefault(); }
  function joyMove(e){ if(!joy.active) return; moveJoy(e); e.preventDefault(); }
  function moveJoy(e){ const p = joyPos(e); const max = p.r.width/2; const dist = Math.min(Math.hypot(p.x,p.y), max); const ang = Math.atan2(p.y,p.x); const dx = Math.cos(ang)*dist; const dy = Math.sin(ang)*dist; joyStick.style.transform = `translate(${dx}px, ${dy}px)`; joy.dx = (dist/max) * Math.cos(ang); joy.dy = (dist/max) * Math.sin(ang); if (Math.hypot(joy.dx, joy.dy) < JOY_DEAD) { joy.dx = 0; joy.dy = 0; joyStick.style.transform = 'translate(0,0)'; } }
  function joyEnd(){ joy.active=false; joy.dx=joy.dy=0; joyStick.style.transform='translate(0,0)'; }
  joyBase.addEventListener('touchstart', joyStart, {passive:false});
  joyBase.addEventListener('touchmove', joyMove, {passive:false});
  joyBase.addEventListener('touchend', joyEnd, {passive:true});

  // Botão de tiro
  document.getElementById('shootBtn').addEventListener('touchstart', ()=>shoot());

  // ===== Entidades =====
  const bullets = [];
  const enemies = [];
  const particles = [];

  // ===== Fundo (estrelas/paralaxe, planetas, lua) =====
  const layers = [
    { count: 140, speed: 0.010, size: [0.7, 1.6], alpha: [0.25, 0.7], stars: [] },
    { count: 90, speed: 0.030, size: [1.0, 2.4], alpha: [0.25, 0.8], stars: [] },
    { count: 50, speed: 0.060, size: [1.2, 2.8], alpha: [0.35, 0.9], stars: [] },
  ];
  function initStars(){
    layers.forEach(L=>{
      L.stars = Array.from({length: L.count}, ()=>({
        x: Math.random()*W, y: Math.random()*H, r: (L.size[0] + Math.random()*(L.size[1]-L.size[0]))*DPR,
        tw: Math.random()*Math.PI*2
      }));
    });
  }
  initStars();

  // Planetas estáticos bonitos
  const planets = [
    // Planeta com anéis
    { x: W*0.18, y: H*0.28, r: 82*DPR, hue: 28, sat: 70, light: 55, ring: { tilt: -18*Math.PI/180, inner: 1.2, outer: 1.9, alpha: 0.28 } },
    // Gasoso azulado com atmosfera
    { x: W*0.78, y: H*0.18, r: 66*DPR, hue: 205, sat: 70, light: 60, atmosphere: 1 },
  ];
  // Lua orbitando o planeta 0
  const moon = { around: 0, angle: Math.random()*Math.PI*2, dist: 160*DPR, r: 22*DPR, speed: 0.00025 };

  function drawStarfield(dt){
    layers.forEach(L=>{
      L.stars.forEach(s=>{
        s.x -= L.speed * dt * DPR; if (s.x < -6*DPR) s.x = W + 6*DPR;
        const a = 0.55 + 0.45 * Math.sin(s.tw + state.time*0.0025);
        ctx.globalAlpha = a * (L.alpha[0] + Math.random()*(L.alpha[1]-L.alpha[0]));
        ctx.fillStyle = '#cfefff';
        ctx.beginPath(); ctx.arc(s.x, s.y, s.r, 0, Math.PI*2); ctx.fill();
      });
    });
    ctx.globalAlpha = 1;
  }

  function drawPlanet(p){
    // Sombra/iluminação
    const grad = ctx.createRadialGradient(p.x - p.r*0.5, p.y - p.r*0.6, p.r*0.1, p.x, p.y, p.r*1.15);
    grad.addColorStop(0, `hsl(${p.hue} ${p.sat}% ${Math.max(20,p.light-15)}%)`);
    grad.addColorStop(0.6, `hsl(${p.hue} ${p.sat}% ${p.light}%)`);
    grad.addColorStop(1, `hsla(${p.hue} ${p.sat}% ${Math.max(10,p.light-35)}% / .9)`);
    ctx.fillStyle = grad;
    ctx.beginPath(); ctx.arc(p.x, p.y, p.r, 0, Math.PI*2); ctx.fill();

    // Anéis
    if (p.ring){
      ctx.save();
      ctx.translate(p.x, p.y);
      ctx.rotate(p.ring.tilt);
      const rg = ctx.createRadialGradient(0,0,p.r*p.ring.inner, 0,0,p.r*p.ring.outer);
      rg.addColorStop(0, `hsla(${p.hue} 80% 80% / 0)`);
      rg.addColorStop(0.35, `hsla(${p.hue} 85% 85% / ${p.ring.alpha})`);
      rg.addColorStop(1, `hsla(${p.hue} 80% 80% / 0)`);
      ctx.fillStyle = rg;
      ctx.beginPath(); ctx.ellipse(0,0,p.r*p.ring.outer, p.r*p.ring.outer*0.35, 0, 0, Math.PI*2); ctx.fill();
      ctx.restore();
    }

    // Atmosfera sutil
    if (p.atmosphere){
      const ag = ctx.createRadialGradient(p.x, p.y, p.r*0.9, p.x, p.y, p.r*1.25);
      ag.addColorStop(0, 'rgba(120,200,255,0)');
      ag.addColorStop(1, 'rgba(120,200,255,0.20)');
      ctx.fillStyle = ag; ctx.beginPath(); ctx.arc(p.x, p.y, p.r*1.25, 0, Math.PI*2); ctx.fill();
    }
  }

  function drawMoon(dt){
    const p = planets[moon.around];
    moon.angle += moon.speed * dt;
    const mx = p.x + Math.cos(moon.angle) * moon.dist;
    const my = p.y + Math.sin(moon.angle) * moon.dist * 0.92; // pequena elipse

    // Corpo lunar com crateras
    const g = ctx.createRadialGradient(mx - moon.r*0.4, my - moon.r*0.4, moon.r*0.2, mx, my, moon.r*1.1);
    g.addColorStop(0, '#eee'); g.addColorStop(1, '#9aa0a6');
    ctx.fillStyle = g; ctx.beginPath(); ctx.arc(mx, my, moon.r, 0, Math.PI*2); ctx.fill();

    // Crateras
    ctx.fillStyle = 'rgba(120,130,140,.55)';
    for(let i=0;i<6;i++){ const a = Math.random()*Math.PI*2; const r = Math.random()*moon.r*0.55; const cx = mx + Math.cos(a)*r*0.6; const cy = my + Math.sin(a)*r*0.6; const cr = Math.random()*moon.r*0.22 + moon.r*0.08; ctx.beginPath(); ctx.arc(cx, cy, cr, 0, Math.PI*2); ctx.fill(); }
  }

  // ===== Util =====
  function roundedRect(x,y,w,h,r){ ctx.beginPath(); ctx.moveTo(x+r,y); ctx.arcTo(x+w,y, x+w,y+h, r); ctx.arcTo(x+w,y+h, x,y+h, r); ctx.arcTo(x,y+h, x,y, r); ctx.arcTo(x,y, x+w,y, r); ctx.closePath(); }

  // ===== Desenho da Nave (mais realista + brilho) =====
  function drawShip(px, py, size){
    ctx.save(); ctx.translate(px, py);
    const bodyW = size*0.58, bodyH = size*1.10; // corpo
    // corpo metálico
    const grad = ctx.createLinearGradient(-bodyW/2, -bodyH/2, bodyW/2, bodyH/2);
    grad.addColorStop(0, '#b7c0c8'); grad.addColorStop(0.5, '#e2e6ea'); grad.addColorStop(1, '#a7b0b8');
    ctx.fillStyle = grad; roundedRect(-bodyW/2, -bodyH/2, bodyW, bodyH, 8*DPR); ctx.fill();
    // cockpit de vidro
    ctx.fillStyle = 'rgba(0,170,255,.85)'; ctx.beginPath(); ctx.ellipse(0, -bodyH*0.22, bodyW*0.18, bodyH*0.14, 0, 0, Math.PI*2); ctx.fill();
    // asas
    ctx.fillStyle = '#7f8790'; ctx.beginPath(); ctx.moveTo(-bodyW*0.65, 0); ctx.lineTo(-bodyW*0.28, bodyH*0.45); ctx.lineTo(-bodyW*0.20, bodyH*0.18); ctx.closePath(); ctx.fill();
    ctx.beginPath(); ctx.moveTo(bodyW*0.65, 0); ctx.lineTo(bodyW*0.28, bodyH*0.45); ctx.lineTo(bodyW*0.20, bodyH*0.18); ctx.closePath(); ctx.fill();
    // motor
    ctx.fillStyle = '#8a9096'; ctx.fillRect(-bodyW*0.16, bodyH*0.45, bodyW*0.32, 10*DPR);
    // chama animada
    const flam = size*0.28 + Math.sin(state.time*0.02)*size*0.05;
    const fireGrad = ctx.createLinearGradient(0, bodyH*0.5, 0, bodyH*0.5 + flam);
    fireGrad.addColorStop(0, 'rgba(255,255,180,.9)'); fireGrad.addColorStop(0.5, 'rgba(255,160,0,.85)'); fireGrad.addColorStop(1, 'rgba(255,60,0,.0)');
    ctx.fillStyle = fireGrad; ctx.beginPath(); ctx.moveTo(-bodyW*0.10, bodyH*0.50); ctx.lineTo(bodyW*0.10, bodyH*0.50); ctx.lineTo(0, bodyH*0.50 + flam); ctx.closePath(); ctx.fill();
    ctx.restore();
  }

  // ===== Inimigos (discos) =====
  function drawAlien(e){
    ctx.save(); ctx.translate(e.x, e.y);
    const s = e.size;
    // cúpula
    ctx.fillStyle = 'rgba(120,255,160,.9)'; ctx.beginPath(); ctx.ellipse(0, 0, s*0.45, s*0.28, 0, 0, Math.PI*2); ctx.fill();
    // base metálica
    const g = ctx.createLinearGradient(-s*0.72, s*0.10, s*0.72, s*0.10);
    g.addColorStop(0,'#8d8f95'); g.addColorStop(0.5,'#d0d4d9'); g.addColorStop(1,'#8d8f95');
    ctx.fillStyle = g; ctx.beginPath(); ctx.ellipse(0, s*0.18, s*0.78, s*0.30, 0, 0, Math.PI*2); ctx.fill();
    // luzes
    ctx.fillStyle = 'rgba(255,255,180,.8)'; for(let i=0;i<5;i++){ const a = (-Math.PI/2) + i*(Math.PI/4); const lx = Math.cos(a)*s*0.55; const ly = s*0.18 + Math.sin(a)*s*0.08; ctx.beginPath(); ctx.arc(lx, ly, 3*DPR, 0, Math.PI*2); ctx.fill(); }
    ctx.restore();
  }

  // ===== Spawns =====
  function spawnEnemy(){
    const s = (28 + Math.random()*18) * DPR;
    enemies.push({ x: Math.random()*(W*0.9) + W*0.05, y: -s, size: s, vy: (0.10 + Math.random()*0.10) * state.diffMul * DPR, vx: (Math.random()<0.5?-1:1) * (0.04 + Math.random()*0.06) * DPR });
  }
  let spawnTimer = 0;

  // ===== Tiro =====
  function shoot(){
    const now = performance.now();
    if (now - state.lastShot < 170) return;
    state.lastShot = now;
    bullets.push({ x: player.x, y: player.y - player.h*0.5, vy: -0.9*DPR, w: 6*DPR, h: 14*DPR });
  }
  window.addEventListener('keydown', (e)=>{ if(e.code === 'Space') shoot(); });

  // ===== Partículas =====
  function explode(x,y,color='#ffa800'){
    for(let i=0;i<18;i++){
      const sp = 0.10 + Math.random()*0.30;
      const a = (i/18)*Math.PI*2;
      particles.push({ x, y, vx: Math.cos(a)*sp*DPR, vy: Math.sin(a)*sp*DPR, r: (1+Math.random()*2)*DPR, life: 380 + Math.random()*260, color });
    }
  }

  // ===== Update & Draw =====
  let last = performance.now();
  function loop(now){
    const dt = now - last; last = now; state.time += dt;
    update(dt); draw(dt);
    requestAnimationFrame(loop);
  }

  function update(dt){
    // Dificuldade progressiva: a cada 5 pts, +10%
    const targetMul = 1 + Math.floor(state.score/5)*0.10;
    state.diffMul += (targetMul - state.diffMul) * 0.02; // interpola suave
    diffEl.textContent = `Dificuldade x${state.diffMul.toFixed(1)}`;

    // Movimento jogador (teclado + joystick)
    const kdx = (keys['ArrowRight']||keys['d']?1:0) - (keys['ArrowLeft']||keys['a']?1:0);
    const kdy = (keys['ArrowDown']||keys['s']?1:0) - (keys['ArrowUp']||keys['w']?1:0);
    const dx = (kdx + joy.dx) * player.speed * dt * 1.8;
    const dy = (kdy + joy.dy) * player.speed * dt * 1.8;
    player.x = Math.max(player.w*0.4, Math.min(W - player.w*0.4, player.x + dx));
    player.y = Math.max(player.h*0.4, Math.min(H - player.h*0.4, player.y + dy));

    // Disparo contínuo pelo mouse/tecla
    if (keys[' '] || keys['Spacebar']) shoot();

    // Balas
    for(let i=bullets.length-1;i>=0;i--){ const b = bullets[i]; b.y += b.vy * dt; if(b.y < -20*DPR) bullets.splice(i,1); }

    // Spawns
    spawnTimer += dt; const every = Math.max(420 - state.score*6, 180); // acelera com a pontuação
    if (spawnTimer > every){ spawnTimer = 0; spawnEnemy(); }

    // Inimigos
    for(let i=enemies.length-1;i>=0;i--){ const e = enemies[i]; e.y += e.vy * dt; e.x += e.vx * dt; if (e.x < e.size*0.5 || e.x > W - e.size*0.5) e.vx *= -1; if (e.y - e.size*0.5 > H + 40*DPR) enemies.splice(i,1); }

    // Colisão tiros x inimigos
    for(let i=enemies.length-1;i>=0;i--){ const e = enemies[i];
      for(let j=bullets.length-1;j>=0;j--){ const b = bullets[j];
        if ( Math.abs(b.x - e.x) < (e.size*0.55) && Math.abs(b.y - e.y) < (e.size*0.35) ){
          enemies.splice(i,1); bullets.splice(j,1); explode(e.x, e.y, '#ffb84a'); state.score++; scoreEl.textContent = 'Pontos: ' + state.score; break;
        }
      }
    }

    // Partículas
    for(let i=particles.length-1;i>=0;i--){ const p = particles[i]; p.x += p.vx * dt; p.y += p.vy * dt; p.life -= dt; p.vy += 0.00012 * dt; if (p.life <= 0) particles.splice(i,1); }
  }

  function draw(dt){
    // Fundo
    ctx.clearRect(0,0,W,H);
    drawStarfield(dt);
    planets.forEach(drawPlanet);
    drawMoon(dt);

    // Entidades
    drawShip(player.x, player.y, Math.max(player.w, 44*DPR));

    // Balas
    ctx.fillStyle = '#ffee88'; bullets.forEach(b=>{ roundedRect(b.x - b.w/2, b.y - b.h/2, b.w, b.h, 2*DPR); ctx.fill(); });

    // Inimigos
    enemies.forEach(drawAlien);

    // Partículas
    particles.forEach(p=>{ ctx.globalAlpha = Math.max(0, p.life/480); ctx.fillStyle = p.color; ctx.beginPath(); ctx.arc(p.x, p.y, p.r, 0, Math.PI*2); ctx.fill(); ctx.globalAlpha = 1; });
  }

  requestAnimationFrame(loop);
  </script>
</body>
</html>
