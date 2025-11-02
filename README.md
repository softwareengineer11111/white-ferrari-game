<!doctype html>
<html lang="ar">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>White Ferrari — فيراري بيضاء (Easy + Character)</title>
<style>
  :root{ --ui-bg: rgba(0,0,0,0.55); --accent: #ffffff; --accent-2: #ffd700; }
  html,body{ height:100%; margin:0; font-family:"Segoe UI", Roboto, "Helvetica Neue", Arial, "Noto Naskh Arabic", sans-serif; background: linear-gradient(180deg,#0b0b0d,#161616); color:var(--accent); -webkit-tap-highlight-color: transparent; }
  #game{ position:relative; width:100%; height:100vh; overflow:hidden; touch-action:none; }
  canvas{ display:block; width:100%; height:100%; background:#0b0b0d; }
  #hud{ position:absolute; top:12px; left:50%; transform:translateX(-50%); display:flex; gap:12px; align-items:center; background:var(--ui-bg); padding:8px 14px; border-radius:10px; z-index:10; }
  .title{ font-size:20px; font-weight:700; text-align:center; line-height:1; }
  .score{ font-size:16px; font-weight:600; color:var(--accent-2); }
  #controls{ position:absolute; right:14px; bottom:14px; display:flex; gap:8px; flex-direction:column; z-index:11; }
  .touch-row{ display:flex; gap:8px; }
  .touch-btn{ width:64px; height:64px; border-radius:10px; background: linear-gradient(180deg, rgba(255,255,255,0.06), rgba(255,255,255,0.02)); border:1px solid rgba(255,255,255,0.06); display:flex; align-items:center; justify-content:center; color:var(--accent); font-weight:700; user-select:none; -webkit-user-select:none; touch-action:none; }
  #overlay{ position:absolute; inset:0; display:flex; align-items:center; justify-content:center; z-index:20; pointer-events:none; }
  .panel{ pointer-events:auto; background:linear-gradient(180deg, rgba(0,0,0,0.75), rgba(255,255,255,0.03)); padding:22px; border-radius:14px; text-align:center; color:var(--accent); box-shadow: 0 10px 30px rgba(0,0,0,0.7); max-width:90%; }
  .btn{ display:inline-block; margin-top:12px; padding:10px 16px; border-radius:10px; background:var(--accent); color:#000; font-weight:700; cursor:pointer; border:none; }
  .small{ font-size:13px; opacity:0.9; }
  /* watermark */
  #watermark { position: absolute; left: 12px; bottom: 12px; z-index: 12; font-size:13px; color: rgba(255,255,255,0.9); background: rgba(0,0,0,0.45); padding:6px 10px; border-radius:8px; pointer-events:none; }
</style>
</head>
<body>
<div id="game" role="application" aria-label="White Ferrari game / لعبة فيراري بيضاء">
  <canvas id="canvas"></canvas>

  <div id="hud" aria-hidden="false">
    <div class="title">White Ferrari — فيراري بيضاء</div>
    <div style="width:12px"></div>
    <div class="score" id="scoreText">Score: 0 ● النتيجة: 0</div>
    <div style="width:12px"></div>
    <div class="small" id="bestText">Best: 0 ● الأفضل: 0</div>
  </div>

  <div id="controls" aria-hidden="false">
    <div class="touch-row">
      <div class="touch-btn" id="btnLeft" data-dir="left">◀</div>
      <div class="touch-btn" id="btnUp" data-dir="up">▲</div>
      <div class="touch-btn" id="btnDown" data-dir="down">▼</div>
      <div class="touch-btn" id="btnRight" data-dir="right">▶</div>
    </div>
    <div class="small" style="text-align:center; opacity:0.85;">Touch / لمس — Hold to move</div>
  </div>

  <div id="overlay">
    <div class="panel" id="startPanel" role="dialog" aria-modal="true">
      <h2>White Ferrari — فيراري بيضاء</h2>
      <p class="small">Easy mode enabled. Use arrow keys or touch to move. Avoid obstacles to score points.<br>تم اختيار مستوى سهل. استخدم مفاتيح الأسهم أو اللمس للحركة وتجنّب العقبات لكسب النقاط.</p>
      <button class="btn" id="startBtn">Start / ابدأ</button>
      <div class="small" style="margin-top:8px; opacity:0.9;">Best stored locally — الأفضل مخزّن محلياً</div>
    </div>

    <div class="panel" id="gameOverPanel" style="display:none;" role="dialog" aria-modal="true">
      <h2 id="goTitle">Game Over ● انتهت اللعبة</h2>
      <p id="goScore">Score: 0 — النتيجة: 0</p>
      <button class="btn" id="restartBtn">Play Again / العب مجدداً</button>
    </div>
  </div>

  <div id="watermark">Made by White Ferrari — صنع بواسطة White Ferrari</div>
</div>

<script>
/*
  Easy difficulty + character sprite (green hair, emo clothes).
  - Collision hitbox is slightly forgiving.
  - Obstacles spawn less frequently and move slower.
  - Player is drawn as a stylized "emo" character (canvas drawing), green hair.
  - Audio retained (engine/point/collision). UI bilingual.
*/

/* ====== Canvas setup ====== */
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d', { alpha: false });
let W=0, H=0;
function resize(){
  W = window.innerWidth; H = window.innerHeight;
  const ratio = window.devicePixelRatio || 1;
  canvas.width = Math.round(W * ratio);
  canvas.height = Math.round(H * ratio);
  canvas.style.width = W + 'px';
  canvas.style.height = H + 'px';
  ctx.setTransform(ratio,0,0,ratio,0,0);
}
window.addEventListener('resize', resize);
resize();

/* ====== HUD & state ====== */
const scoreText = document.getElementById('scoreText');
const bestText = document.getElementById('bestText');
let score = 0;
let best = Number(localStorage.getItem('whiteFerrariBest') || 0);
function updateHUD(){ scoreText.textContent = `Score: ${Math.floor(score)} ● النتيجة: ${Math.floor(score)}`; bestText.textContent = `Best: ${Math.floor(best)} ● الأفضل: ${Math.floor(best)}`; }
updateHUD();

let running=false, lastTime=0;

/* ====== Player (character) ====== */
const player = {
  x:0, y:0, w:90, h:110, // character taller than car
  vx:0, vy:0,
  accel: 1600, // a bit higher for responsive control
  friction: 12,
  maxSpeed: 520
};

function resetPlayer(){
  player.w = Math.max(56, Math.min(140, W * 0.12));
  player.h = player.w * 1.25;
  player.x = W/2 - player.w/2;
  player.y = H * 0.72 - player.h/2;
  player.vx = 0; player.vy = 0;
}

/* hitbox forgiveness factor: shrink collision box */
const HITBOX_SHRINK = 0.62; // 62% of visual size -> more forgiving on easy

/* ====== Input Handling ====== */
const keys = { left:false, right:false, up:false, down:false };
window.addEventListener('keydown', e => {
  if(e.repeat) return;
  if(e.key === 'ArrowLeft' || e.key === 'a') keys.left = true;
  if(e.key === 'ArrowRight' || e.key === 'd') keys.right = true;
  if(e.key === 'ArrowUp' || e.key === 'w') keys.up = true;
  if(e.key === 'ArrowDown' || e.key === 's') keys.down = true;
});
window.addEventListener('keyup', e => {
  if(e.key === 'ArrowLeft' || e.key === 'a') keys.left = false;
  if(e.key === 'ArrowRight' || e.key === 'd') keys.right = false;
  if(e.key === 'ArrowUp' || e.key === 'w') keys.up = false;
  if(e.key === 'ArrowDown' || e.key === 's') keys.down = false;
});

// touch buttons
const btnLeft = document.getElementById('btnLeft');
const btnRight = document.getElementById('btnRight');
const btnUp = document.getElementById('btnUp');
const btnDown = document.getElementById('btnDown');
[btnLeft,btnRight,btnUp,btnDown].forEach(btn => {
  btn.addEventListener('pointerdown', e => {
    e.preventDefault();
    const d = btn.dataset.dir;
    if(d === 'left') keys.left = true;
    if(d === 'right') keys.right = true;
    if(d === 'up') keys.up = true;
    if(d === 'down') keys.down = true;
    btn.setPointerCapture(e.pointerId);
    btn.style.transform = 'translateY(2px) scale(0.985)';
  });
  btn.addEventListener('pointerup', e => {
    e.preventDefault();
    const d = btn.dataset.dir;
    if(d === 'left') keys.left = false;
    if(d === 'right') keys.right = false;
    if(d === 'up') keys.up = false;
    if(d === 'down') keys.down = false;
    try{ btn.releasePointerCapture(e.pointerId); } catch(e){}
    btn.style.transform = '';
  });
  btn.addEventListener('pointercancel', e => {
    const d = btn.dataset.dir;
    if(d === 'left') keys.left = false;
    if(d === 'right') keys.right = false;
    if(d === 'up') keys.up = false;
    if(d === 'down') keys.down = false;
    btn.style.transform = '';
  });
});

// drag-to-move
let dragging=false, dragId=null, lastPointer=null;
canvas.addEventListener('pointerdown', (e) => { dragging=true; dragId=e.pointerId; lastPointer={x:e.clientX,y:e.clientY}; canvas.setPointerCapture(e.pointerId); });
canvas.addEventListener('pointermove', (e) => {
  if(!dragging || e.pointerId !== dragId) return;
  const dx = e.clientX - lastPointer.x;
  const dy = e.clientY - lastPointer.y;
  player.x += dx; player.y += dy;
  lastPointer = { x: e.clientX, y: e.clientY };
  clampPlayer();
});
canvas.addEventListener('pointerup', (e) => { if(e.pointerId !== dragId) return; dragging=false; dragId=null; canvas.releasePointerCapture(e.pointerId); });
canvas.addEventListener('pointercancel', (e) => { dragging=false; dragId=null; });

/* ====== Obstacles (easier settings) ====== */
const obstacles = [];
let spawnTimer = 0;
// Easy mode parameters
const EASY_SPAWN_BASE = 1.6;    // seconds between spawns (base)
const EASY_SPEED_MIN = 90;      // slower obstacles
const EASY_SPEED_MAX = 180;     // slower max
const EASY_OBSTACLE_SCALE = 0.82; // slightly smaller obstacles

function spawnObstacle(){
  const ow = Math.max(44, Math.min(180, W * (0.06 + Math.random()*0.10))) * EASY_OBSTACLE_SCALE;
  const oh = ow * (0.45 + Math.random()*0.6);
  const ox = Math.random() * (W - ow);
  // speed scaled with score but gentler
  const speed = EASY_SPEED_MIN + Math.random()*(EASY_SPEED_MAX - EASY_SPEED_MIN) + Math.min(200, score*0.3);
  obstacles.push({ x:ox, y:-oh-20, w:ow, h:oh, speed, color: `hsl(${Math.random()*360} 60% 45% / 0.95)` });
}

/* ====== Helpers ====== */
function clampPlayer(){
  player.x = Math.max(0, Math.min(W - player.w, player.x));
  player.y = Math.max(0, Math.min(H - player.h, player.y));
}
function rectsOverlap(a,b){
  return !(a.x + a.w < b.x || a.x > b.x + b.w || a.y + a.h < b.y || a.y > b.y + b.h);
}

/* ====== Audio (same as before) ====== */
let audioCtx = null;
let engineOsc = null;
let engineGain = null;
let masterGain = null;

function ensureAudio(){
  if(audioCtx) return;
  audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  masterGain = audioCtx.createGain();
  masterGain.gain.value = 0.9;
  masterGain.connect(audioCtx.destination);
  engineOsc = audioCtx.createOscillator();
  engineGain = audioCtx.createGain();
  engineOsc.type = 'sawtooth';
  engineOsc.frequency.value = 140;
  engineGain.gain.value = 0;
  engineOsc.connect(engineGain);
  engineGain.connect(masterGain);
  engineOsc.start();
}
function setEngineLevel(level){
  if(!audioCtx) return;
  engineGain.gain.cancelScheduledValues(audioCtx.currentTime);
  engineGain.gain.linearRampToValueAtTime(level * 0.22, audioCtx.currentTime + 0.05);
}
function setEngineFreq(freq){
  if(!audioCtx) return;
  engineOsc.frequency.setTargetAtTime(freq, audioCtx.currentTime, 0.05);
}
function playPointSound(){
  if(!audioCtx) return;
  const t = audioCtx.currentTime;
  const o = audioCtx.createOscillator();
  const g = audioCtx.createGain();
  o.type = 'square';
  o.frequency.setValueAtTime(950, t);
  g.gain.setValueAtTime(0, t);
  g.gain.linearRampToValueAtTime(0.16, t + 0.005);
  g.gain.exponentialRampToValueAtTime(0.001, t + 0.22);
  o.connect(g); g.connect(masterGain);
  o.start(t); o.stop(t + 0.24);
}
function playCollisionSound(){
  if(!audioCtx) return;
  const t = audioCtx.currentTime;
  const o = audioCtx.createOscillator();
  const g = audioCtx.createGain();
  o.type = 'sine';
  o.frequency.setValueAtTime(200, t);
  o.frequency.exponentialRampToValueAtTime(45, t + 0.35);
  g.gain.setValueAtTime(0.0001, t);
  g.gain.linearRampToValueAtTime(0.6, t + 0.01);
  g.gain.exponentialRampToValueAtTime(0.001, t + 0.5);
  o.connect(g); g.connect(masterGain);
  o.start(t); o.stop(t + 0.6);
  const bufferSize = audioCtx.sampleRate * 0.35;
  const buffer = audioCtx.createBuffer(1, bufferSize, audioCtx.sampleRate);
  const data = buffer.getChannelData(0);
  for(let i=0;i<bufferSize;i++) data[i] = (Math.random()*2-1) * Math.exp(-i/bufferSize*3);
  const src = audioCtx.createBufferSource();
  const sg = audioCtx.createGain();
  sg.gain.value = 0.2;
  src.buffer = buffer;
  src.connect(sg); sg.connect(masterGain);
  src.start(t);
}

/* ====== Game loop ====== */
function startGame(){
  ensureAudio();
  if(audioCtx && audioCtx.state === 'suspended') audioCtx.resume();
  score = 0; obstacles.length = 0; spawnTimer = 0;
  resetPlayer();
  running = true;
  lastTime = performance.now();
  document.getElementById('startPanel').style.display = 'none';
  document.getElementById('gameOverPanel').style.display = 'none';
  requestAnimationFrame(loop);
}
function endGame(){
  running = false;
  if(score > best){ best = Math.floor(score); localStorage.setItem('whiteFerrariBest', best); }
  updateHUD();
  document.getElementById('goScore').textContent = `Score: ${Math.floor(score)} — النتيجة: ${Math.floor(score)}`;
  document.getElementById('startPanel').style.display = 'none';
  document.getElementById('gameOverPanel').style.display = 'block';
  if(audioCtx) setEngineLevel(0);
}
function loop(now){
  const dt = Math.min(0.05, (now - lastTime)/1000);
  lastTime = now;
  update(dt);
  render();
  if(running) requestAnimationFrame(loop);
}

/* ====== Update ====== */
let backgroundOffset = 0;
function update(dt){
  score += dt * 8; // slower score growth for easy
  updateHUD();

  // input accel
  let ax=0, ay=0;
  if(keys.left) ax -= 1; if(keys.right) ax += 1; if(keys.up) ay -= 1; if(keys.down) ay += 1;
  if(ax !== 0 && ay !== 0){ const s = Math.SQRT1_2; ax *= s; ay *= s; }
  player.vx += ax * player.accel * dt; player.vy += ay * player.accel * dt;
  const frictionFactor = Math.pow(1 - Math.min(0.99, player.friction * dt), 1);
  player.vx *= frictionFactor; player.vy *= frictionFactor;
  const sp = Math.hypot(player.vx, player.vy);
  if(sp > player.maxSpeed){ const s = player.maxSpeed / sp; player.vx *= s; player.vy *= s; }
  player.x += player.vx * dt; player.y += player.vy * dt;
  clampPlayer();

  // audio engine mapping
  if(audioCtx){
    const speedNorm = Math.min(1, sp / player.maxSpeed);
    const freq = 140 + speedNorm * 600;
    setEngineFreq(freq);
    setEngineLevel(0.16 + speedNorm * 0.84);
  }

  // spawn obstacles (easier: less frequent)
  spawnTimer -= dt;
  if(spawnTimer <= 0){
    spawnObstacle();
    // spawn interval reduces slightly with score but remains generous
    spawnTimer = Math.max(0.7, EASY_SPAWN_BASE - Math.min(0.8, score * 0.008));
  }

  // update obstacles and collisions (using smaller hitbox)
  for(let i = obstacles.length - 1; i >= 0; i--){
    const ob = obstacles[i];
    ob.y += ob.speed * dt;
    if(ob.y > H + 80){
      score += 1 + Math.floor(ob.speed / 180);
      obstacles.splice(i,1);
      if(audioCtx) playPointSound();
      continue;
    }
    // compute forgiving hitboxes
    const playerBox = {
      x: player.x + player.w * (1 - HITBOX_SHRINK) / 2,
      y: player.y + player.h * (1 - HITBOX_SHRINK) / 2,
      w: player.w * HITBOX_SHRINK,
      h: player.h * HITBOX_SHRINK
    };
    const obBox = { x: ob.x, y: ob.y, w: ob.w, h: ob.h };
    if(rectsOverlap(playerBox, obBox)){
      if(audioCtx) playCollisionSound();
      endGame();
      return;
    }
  }

  backgroundOffset += 200 * dt;
  if(backgroundOffset > 1000) backgroundOffset = 0;
}

/* ====== Render (character drawing instead of car image) ====== */
function render(){
  ctx.fillStyle = '#0b0b0d';
  ctx.fillRect(0,0,W,H);

  // road
  const roadWidth = Math.min(W*0.7, 900);
  const cx = W/2;
  const roadLeft = cx - roadWidth/2;
  const roadRight = cx + roadWidth/2;
  const roadGradient = ctx.createLinearGradient(0,0,0,H);
  roadGradient.addColorStop(0, '#131313');
  roadGradient.addColorStop(1, '#0d0d0d');
  ctx.fillStyle = roadGradient;
  ctx.fillRect(roadLeft,0,roadWidth,H);

  // edges
  ctx.strokeStyle = 'rgba(255,255,255,0.03)';
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.moveTo(roadLeft,0); ctx.lineTo(roadLeft,H);
  ctx.moveTo(roadRight,0); ctx.lineTo(roadRight,H);
  ctx.stroke();

  // center dashed
  ctx.strokeStyle = 'rgba(255,255,255,0.18)';
  ctx.lineWidth = Math.max(2, Math.min(6, roadWidth*0.01));
  ctx.setLineDash([roadWidth*0.04, roadWidth*0.06]);
  ctx.lineDashOffset = -backgroundOffset * 0.6;
  ctx.beginPath(); ctx.moveTo(cx, -1000); ctx.lineTo(cx, H + 1000); ctx.stroke(); ctx.setLineDash([]);

  // obstacles
  for(const ob of obstacles){
    ctx.fillStyle = ob.color;
    roundRect(ctx, ob.x, ob.y, ob.w, ob.h, Math.min(12, ob.w*0.08), true, false);
    ctx.fillStyle = 'rgba(255,255,255,0.12)';
    roundRect(ctx, ob.x + ob.w*0.08, ob.y + ob.h*0.08, ob.w*0.3, ob.h*0.2, 4, true, false);
  }

  // draw character sprite at player.x/y
  drawCharacter(ctx, player.x, player.y, player.w, player.h, player.vx);

  // subtle vignette
  const g = ctx.createLinearGradient(0,0,0,H);
  g.addColorStop(0, 'rgba(0,0,0,0.0)');
  g.addColorStop(1, 'rgba(0,0,0,0.18)');
  ctx.fillStyle = g;
  ctx.fillRect(0,0,W,H);
}

/* drawCharacter: stylized emo character with green hair and black clothes */
function drawCharacter(ctx, x, y, w, h, vx){
  ctx.save();
  // tilt slightly based on vx
  const tilt = Math.max(-0.14, Math.min(0.14, vx / player.maxSpeed));
  ctx.translate(x + w/2, y + h/2);
  ctx.rotate(tilt);
  // body proportions
  const headH = h * 0.28;
  const bodyH = h * 0.58;
  const headW = w * 0.7;
  // draw body (hoodie / emo clothes - black)
  ctx.fillStyle = '#0b0b0f';
  roundRect(ctx, -w*0.45, -h*0.08, w*0.9, bodyH, 10, true, false);
  // small chest highlight
  ctx.fillStyle = 'rgba(255,255,255,0.03)';
  roundRect(ctx, -w*0.28, -h*0.01, w*0.56, bodyH*0.2, 6, true, false);
  // draw head (skin: pale)
  const headX = 0;
  const headY = -bodyH/2 - headH/2 + 6;
  ctx.beginPath();
  ctx.fillStyle = '#f5e6db'; // light skin tone
  ctx.ellipse(headX, headY, headW/2, headH/2, 0, 0, Math.PI*2);
  ctx.fill();
  // emo hair: green, swooped, partial covering face
  ctx.fillStyle = '#2ee06a'; // vivid green
  ctx.beginPath();
  // hair base: left spikes
  const spikeCount = 6;
  const hw = headW/2;
  const hh = headH/2;
  for(let i=0;i<=spikeCount;i++){
    const t = i / spikeCount;
    const px = headX - hw + t * (hw*2);
    const py = headY - hh - Math.sin(t * Math.PI) * hh * 0.95;
    if(i===0) ctx.moveTo(px, py);
    else ctx.lineTo(px, py);
  }
  // close shape at back
  ctx.lineTo(headX + hw, headY + hh*0.4);
  ctx.lineTo(headX - hw, headY + hh*0.4);
  ctx.closePath();
  ctx.fill();
  // eye area (simple)
  ctx.fillStyle = '#1b1b1b';
  // left eye
  ctx.beginPath();
  ctx.ellipse(headX - headW*0.18, headY - headH*0.08, headW*0.06, headH*0.04, 0, 0, Math.PI*2);
  ctx.fill();
  // right eye (partly covered by hair)
  ctx.beginPath();
  ctx.ellipse(headX + headW*0.05, headY - headH*0.06, headW*0.06, headH*0.04, 0, 0, Math.PI*2);
  ctx.fill();
  // mouth small line
  ctx.strokeStyle = 'rgba(0,0,0,0.6)';
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.moveTo(headX - headW*0.08, headY + headH*0.15);
  ctx.lineTo(headX + headW*0.08, headY + headH*0.15);
  ctx.stroke();
  // emo accessory: choker
  ctx.fillStyle = '#0a0a0a';
  ctx.fillRect(headX - headW*0.2, headY + headH*0.22, headW*0.4, headH*0.08);

  ctx.restore();
}

/* rounded rectangle */
function roundRect(ctx, x, y, w, h, r, fill, stroke){
  if (typeof r === 'undefined') r = 5;
  ctx.beginPath();
  ctx.moveTo(x + r, y);
  ctx.arcTo(x + w, y, x + w, y + h, r);
  ctx.arcTo(x + w, y + h, x, y + h, r);
  ctx.arcTo(x, y + h, x, y, r);
  ctx.arcTo(x, y, x + w, y, r);
  ctx.closePath();
  if(fill) ctx.fill();
  if(stroke) ctx.stroke();
}

/* UI buttons */
document.getElementById('startBtn').addEventListener('click', startGame);
document.getElementById('restartBtn').addEventListener('click', startGame);
window.addEventListener('keydown', e => {
  if(e.key === ' '){
    e.preventDefault();
    if(!running) startGame();
  }
});

/* init */
resetPlayer();
updateHUD();
render();

</script>
</body>
</html>
