<template>
  <main class="game-shell" aria-label="飞机大战游戏">
    <canvas
      ref="canvasRef"
      class="game-canvas"
      width="620"
      height="920"
      @pointerdown="onPointerDown"
      @pointermove="onPointerMove"
      @pointerup="clearPointer"
      @pointercancel="clearPointer"
    />

    <section class="hud" aria-live="polite">
      <div class="metric">
        <span class="label">分数</span>
        <span class="value">{{ hud.score }}</span>
      </div>
      <div class="metric">
        <span class="label">生命</span>
        <span class="value">{{ hud.lives }}</span>
      </div>
      <div class="metric">
        <span class="label">火力</span>
        <span class="value">{{ hud.power }}</span>
      </div>
      <div class="metric">
        <span class="label">关卡</span>
        <span class="value">{{ hud.level }}</span>
      </div>
    </section>

    <button class="sound-toggle" type="button" :aria-label="muted ? '开启声音' : '关闭声音'" @click="toggleSound">
      {{ muted ? "×" : "♪" }}
    </button>

    <div class="flash" :class="{ hit: hitFlash }" />

    <section class="overlay" :class="{ hidden: state === 'playing' }">
      <div class="menu">
        <h1>{{ menuTitle }}</h1>
        <p class="subtitle">{{ menuMessage }}</p>
        <div class="controls">
          <div class="key">移动：WASD / 方向键</div>
          <div class="key">拖动：鼠标 / 触屏</div>
          <div class="key">暂停：P 或 Esc</div>
          <div class="key">重开：R</div>
        </div>
        <button class="primary-action" type="button" @click="handlePrimaryAction">
          {{ primaryAction }}
        </button>
        <p class="hint">小技巧：炸弹会清屏，护盾可挡一次伤害，紫色超载会短时提高火力。</p>
      </div>
    </section>
  </main>
</template>

<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref } from "vue";

const canvasRef = ref(null);
const muted = ref(false);
const hitFlash = ref(false);
const state = ref("menu");
const hud = reactive({ score: 0, lives: 3, power: 1, level: 1 });

const WIDTH = 620;
const HEIGHT = 920;
const keys = new Set();
const stars = [];
const bullets = [];
const enemyBullets = [];
const enemies = [];
const obstacles = [];
const particles = [];
const pickups = [];

let ctx = null;
let pointer = null;
let lastTime = 0;
let spawnTimer = 0;
let obstacleTimer = 0;
let pickupTimer = 0;
let bossTimer = 0;
let rafId = 0;
let shake = 0;
let musicTimer = 0;
let musicStep = 0;
let audioCtx = null;
let masterGain = null;
let musicGain = null;
let sfxGain = null;

const player = {
  x: WIDTH / 2,
  y: HEIGHT - 115,
  r: 23,
  speed: 440,
  lives: 3,
  invulnerable: 0,
  cooldown: 0,
  power: 1,
  shield: 0,
  overdrive: 0,
  score: 0,
  level: 1,
  fireRate: 0.14
};

const menuTitle = computed(() => {
  if (state.value === "paused") return "暂停中";
  if (state.value === "over") return "任务结束";
  return "星翼突袭";
});

const menuMessage = computed(() => {
  if (state.value === "paused") return "战场先稳住了。点击继续，或按 P / Esc 回到游戏。";
  if (state.value === "over") return `最终分数 ${player.score}，抵达第 ${player.level} 关。再飞一次，把弹幕撕开。`;
  return "拖动、方向键或 WASD 控制战机，自动射击。拾取回血、炸弹、护盾和超载道具，撑过越来越密集的空袭。";
});

const primaryAction = computed(() => (state.value === "paused" ? "继续作战" : state.value === "over" ? "重新开始" : "开始作战"));

function rand(min, max) {
  return Math.random() * (max - min) + min;
}

function clamp(value, min, max) {
  return Math.max(min, Math.min(max, value));
}

function distance(a, b) {
  return Math.hypot(a.x - b.x, a.y - b.y);
}

function syncHud() {
  hud.score = player.score;
  hud.lives = player.lives;
  hud.power = player.power;
  hud.level = player.level;
}

// Web Audio 必须由用户点击后才能播放，所以在开始游戏或点击声音按钮时初始化。
function initAudio() {
  if (audioCtx) return;
  audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  masterGain = audioCtx.createGain();
  musicGain = audioCtx.createGain();
  sfxGain = audioCtx.createGain();
  masterGain.gain.value = muted.value ? 0 : 0.72;
  musicGain.gain.value = 0.18;
  sfxGain.gain.value = 0.7;
  musicGain.connect(masterGain);
  sfxGain.connect(masterGain);
  masterGain.connect(audioCtx.destination);
}

function setMuted(value) {
  muted.value = value;
  if (masterGain && audioCtx) {
    masterGain.gain.setTargetAtTime(value ? 0 : 0.72, audioCtx.currentTime, 0.025);
  }
}

function tone(freq, duration, type = "sine", gain = 0.18, dest = sfxGain, when = 0, slideTo = null) {
  if (!audioCtx || muted.value) return;
  const now = audioCtx.currentTime + when;
  const osc = audioCtx.createOscillator();
  const amp = audioCtx.createGain();
  osc.type = type;
  osc.frequency.setValueAtTime(freq, now);
  if (slideTo) osc.frequency.exponentialRampToValueAtTime(slideTo, now + duration);
  amp.gain.setValueAtTime(0.0001, now);
  amp.gain.exponentialRampToValueAtTime(gain, now + 0.012);
  amp.gain.exponentialRampToValueAtTime(0.0001, now + duration);
  osc.connect(amp);
  amp.connect(dest);
  osc.start(now);
  osc.stop(now + duration + 0.03);
}

function noise(duration, gain = 0.2, cutoff = 900, when = 0) {
  if (!audioCtx || muted.value) return;
  const now = audioCtx.currentTime + when;
  const buffer = audioCtx.createBuffer(1, Math.ceil(audioCtx.sampleRate * duration), audioCtx.sampleRate);
  const data = buffer.getChannelData(0);
  for (let i = 0; i < data.length; i += 1) data[i] = (Math.random() * 2 - 1) * (1 - i / data.length);
  const src = audioCtx.createBufferSource();
  const filter = audioCtx.createBiquadFilter();
  const amp = audioCtx.createGain();
  src.buffer = buffer;
  filter.type = "lowpass";
  filter.frequency.value = cutoff;
  amp.gain.setValueAtTime(gain, now);
  amp.gain.exponentialRampToValueAtTime(0.0001, now + duration);
  src.connect(filter);
  filter.connect(amp);
  amp.connect(sfxGain);
  src.start(now);
}

function playSound(name) {
  if (!audioCtx || muted.value) return;
  if (name === "shot") tone(860, 0.045, "square", 0.035, sfxGain, 0, 1180);
  if (name === "enemyShot") tone(210, 0.08, "sawtooth", 0.045, sfxGain, 0, 150);
  if (name === "hit") tone(180, 0.08, "triangle", 0.09, sfxGain, 0, 90);
  if (name === "boom") {
    noise(0.32, 0.24, 760);
    tone(72, 0.34, "sawtooth", 0.12, sfxGain, 0, 42);
  }
  if (name === "pickup") {
    tone(640, 0.09, "sine", 0.09);
    tone(960, 0.12, "sine", 0.08, sfxGain, 0.06);
  }
  if (name === "shield") {
    tone(520, 0.12, "triangle", 0.08);
    tone(780, 0.18, "triangle", 0.07, sfxGain, 0.08);
  }
  if (name === "bomb") {
    noise(0.52, 0.32, 520);
    tone(92, 0.45, "sawtooth", 0.16, sfxGain, 0, 34);
    tone(220, 0.14, "square", 0.07, sfxGain, 0.04, 120);
  }
  if (name === "overdrive") {
    tone(740, 0.08, "square", 0.06);
    tone(1110, 0.12, "square", 0.05, sfxGain, 0.07);
    tone(1480, 0.14, "triangle", 0.04, sfxGain, 0.15);
  }
  if (name === "hurt") {
    noise(0.24, 0.18, 1400);
    tone(120, 0.28, "sawtooth", 0.11, sfxGain, 0, 58);
  }
  if (name === "start") {
    tone(440, 0.1, "triangle", 0.08);
    tone(660, 0.12, "triangle", 0.08, sfxGain, 0.08);
    tone(990, 0.15, "triangle", 0.07, sfxGain, 0.18);
  }
}

function updateMusic(dt) {
  if (!audioCtx || muted.value || state.value !== "playing") return;
  musicTimer -= dt;
  if (musicTimer > 0) return;
  const bass = [55, 55, 65.41, 73.42, 82.41, 73.42, 65.41, 49];
  const lead = [220, 277.18, 329.63, 415.3, 329.63, 277.18, 246.94, 196];
  const beat = musicStep % 8;
  tone(bass[beat], 0.22, "sawtooth", beat % 2 ? 0.035 : 0.06, musicGain);
  if (beat % 2 === 0) tone(lead[beat], 0.12, "triangle", 0.035, musicGain, 0.03);
  if (beat === 0 || beat === 4) noise(0.08, 0.025, 280, 0.01);
  musicStep += 1;
  musicTimer = 0.245;
}

function resetStars() {
  stars.length = 0;
  for (let i = 0; i < 110; i += 1) {
    stars.push({ x: rand(0, WIDTH), y: rand(0, HEIGHT), size: rand(0.7, 2.2), speed: rand(45, 190), alpha: rand(0.35, 0.95) });
  }
}

function resetGame() {
  bullets.length = 0;
  enemyBullets.length = 0;
  enemies.length = 0;
  obstacles.length = 0;
  particles.length = 0;
  pickups.length = 0;
  Object.assign(player, {
    x: WIDTH / 2,
    y: HEIGHT - 115,
    lives: 3,
    power: 1,
    shield: 0,
    overdrive: 0,
    score: 0,
    level: 1,
    invulnerable: 1.6,
    cooldown: 0
  });
  spawnTimer = 0.35;
  obstacleTimer = 2.4;
  pickupTimer = 8;
  bossTimer = 32;
  shake = 0;
  musicTimer = 0;
  musicStep = 0;
  syncHud();
}

function startGame() {
  initAudio();
  if (audioCtx.state === "suspended") audioCtx.resume();
  playSound("start");
  resetGame();
  state.value = "playing";
  lastTime = performance.now();
  cancelAnimationFrame(rafId);
  rafId = requestAnimationFrame(loop);
}

function togglePause() {
  if (state.value === "playing") {
    state.value = "paused";
    return;
  }
  if (state.value === "paused") {
    if (audioCtx && audioCtx.state === "suspended") audioCtx.resume();
    state.value = "playing";
    lastTime = performance.now();
    rafId = requestAnimationFrame(loop);
  }
}

function handlePrimaryAction() {
  if (state.value === "paused") togglePause();
  else startGame();
}

function toggleSound() {
  initAudio();
  if (audioCtx.state === "suspended") audioCtx.resume();
  setMuted(!muted.value);
  if (!muted.value) playSound("pickup");
}

function addParticles(x, y, color, count, speed = 240) {
  for (let i = 0; i < count; i += 1) {
    const angle = rand(0, Math.PI * 2);
    const ttl = rand(0.28, 0.72);
    particles.push({
      x,
      y,
      vx: Math.cos(angle) * rand(30, speed),
      vy: Math.sin(angle) * rand(30, speed),
      life: ttl,
      ttl,
      size: rand(2, 5),
      color
    });
  }
}

function spawnEnemy(forceBoss = false) {
  const boss = forceBoss || (bossTimer <= 0 && enemies.every((enemy) => enemy.type !== "boss"));
  if (boss) {
    bossTimer = Math.max(22, 38 - player.level * 2);
    const hp = 120 + player.level * 40;
    enemies.push({ type: "boss", x: WIDTH / 2, y: -70, r: 62, hp, maxHp: hp, speed: 55 + player.level * 4, vx: 125, shoot: 1.1, score: 900 + player.level * 90 });
    return;
  }

  const elite = Math.random() < Math.min(0.18, 0.05 + player.level * 0.018);
  const hp = elite ? 16 + player.level * 3 : 5 + player.level * 1.4;
  enemies.push({
    type: elite ? "elite" : "scout",
    x: rand(35, WIDTH - 35),
    y: -45,
    r: elite ? 28 : 21,
    hp,
    maxHp: hp,
    speed: rand(110, 180) + player.level * 9,
    vx: Math.random() < 0.32 ? rand(-90, 90) : 0,
    phase: rand(0, Math.PI * 2),
    shoot: elite ? rand(1.2, 2.3) : rand(2.2, 4.4),
    score: elite ? 210 : 90
  });
}

function spawnObstacle() {
  const size = rand(24, 52) + player.level * 1.4;
  const sides = Math.floor(rand(8, 13));
  const points = Array.from({ length: sides }, (_, index) => ({ angle: (Math.PI * 2 * index) / sides, radius: rand(0.62, 1.05) }));
  const craters = Array.from({ length: 4 }, () => ({ x: rand(-0.34, 0.34), y: rand(-0.3, 0.3), rx: rand(0.08, 0.15), ry: rand(0.05, 0.11), angle: rand(0, Math.PI) }));
  const hp = Math.round(size * 0.45 + player.level * 2.5);
  obstacles.push({
    x: rand(size, WIDTH - size),
    y: -size - 20,
    r: size,
    hp,
    maxHp: hp,
    speed: rand(88, 150) + player.level * 6,
    vx: rand(-38, 38),
    spin: rand(-2.4, 2.4),
    angle: rand(0, Math.PI * 2),
    points,
    craters,
    score: Math.round(size * 4)
  });
}

function choosePickupKind() {
  const roll = Math.random();
  if (roll < 0.34) return "power";
  if (roll < 0.56) return "life";
  if (roll < 0.72) return "shield";
  if (roll < 0.88) return "bomb";
  return "overdrive";
}

function spawnPickup(x = rand(40, WIDTH - 40), y = -24, kind = choosePickupKind()) {
  pickups.push({ kind, x, y, r: kind === "bomb" ? 20 : 17, speed: rand(95, 135), spin: rand(-2.2, 2.2), angle: rand(0, Math.PI * 2) });
}

function firePlayer() {
  const effectivePower = clamp(player.power + (player.overdrive > 0 ? 2 : 0), 1, 7);
  const lanes = [-1, 0, 1, -2, 2, -3, 3];
  const spread = lanes.slice(0, Math.min(lanes.length, effectivePower * 2 - 1));
  playSound("shot");
  spread.forEach((lane, index) => {
    bullets.push({
      x: player.x + lane * 10,
      y: player.y - 28,
      vx: lane * 22,
      vy: -690 - effectivePower * 18,
      r: index === 0 ? 5 : 4,
      damage: 5 + effectivePower * 0.6,
      color: player.overdrive > 0 ? "#c77dff" : lane === 0 ? "#76f7b6" : "#51d7ff"
    });
  });
}

function fireEnemy(enemy) {
  playSound("enemyShot");
  const shots = enemy.type === "boss" ? 5 : enemy.type === "elite" ? 3 : 1;
  for (let i = 0; i < shots; i += 1) {
    const spread = shots === 1 ? 0 : (i / (shots - 1) - 0.5) * 0.52;
    const angle = Math.atan2(player.y - enemy.y, player.x - enemy.x) + spread;
    const speed = enemy.type === "boss" ? 250 : 220;
    enemyBullets.push({ x: enemy.x, y: enemy.y + enemy.r * 0.5, vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed, r: enemy.type === "boss" ? 6 : 5, color: enemy.type === "boss" ? "#ffd166" : "#ff5a77" });
  }
}

// 主循环只负责推进一帧：输入、生成、运动、碰撞、声音节拍和绘制都在这里串起来。
function update(dt) {
  const levelTarget = 1 + Math.floor(player.score / 1400);
  if (levelTarget !== player.level) {
    player.level = levelTarget;
    addParticles(player.x, player.y, "#76f7b6", 26, 300);
  }

  player.invulnerable = Math.max(0, player.invulnerable - dt);
  player.shield = Math.max(0, player.shield - dt);
  player.overdrive = Math.max(0, player.overdrive - dt);
  player.cooldown -= dt;
  if (player.cooldown <= 0) {
    firePlayer();
    const rateBonus = player.overdrive > 0 ? 0.045 : 0;
    player.cooldown = Math.max(0.055, player.fireRate - player.power * 0.012 - rateBonus);
  }

  movePlayer(dt);
  updateStars(dt);

  spawnTimer -= dt;
  obstacleTimer -= dt;
  pickupTimer -= dt;
  bossTimer -= dt;
  if (spawnTimer <= 0) {
    spawnEnemy();
    spawnTimer = Math.max(0.22, rand(0.7, 1.18) - player.level * 0.045);
  }
  if (obstacleTimer <= 0) {
    spawnObstacle();
    obstacleTimer = Math.max(0.95, rand(2.2, 3.7) - player.level * 0.09);
  }
  if (pickupTimer <= 0) {
    spawnPickup();
    pickupTimer = rand(9, 14);
  }

  updateBullets(dt);
  updateEnemies(dt);
  updateObstacles(dt);
  updatePickups(dt);
  updateParticles(dt);
  handleCollisions();
  shake = Math.max(0, shake - dt * 18);
  updateMusic(dt);
  syncHud();
}

function movePlayer(dt) {
  let dx = 0;
  let dy = 0;
  if (keys.has("ArrowLeft") || keys.has("a")) dx -= 1;
  if (keys.has("ArrowRight") || keys.has("d")) dx += 1;
  if (keys.has("ArrowUp") || keys.has("w")) dy -= 1;
  if (keys.has("ArrowDown") || keys.has("s")) dy += 1;

  if (pointer) {
    player.x += (pointer.x - player.x) * Math.min(1, dt * 12);
    player.y += (pointer.y - player.y) * Math.min(1, dt * 12);
  } else if (dx || dy) {
    const mag = Math.hypot(dx, dy) || 1;
    player.x += (dx / mag) * player.speed * dt;
    player.y += (dy / mag) * player.speed * dt;
  }
  player.x = clamp(player.x, 28, WIDTH - 28);
  player.y = clamp(player.y, 100, HEIGHT - 42);
}

function updateStars(dt) {
  stars.forEach((star) => {
    star.y += star.speed * dt * (1 + player.level * 0.02);
    if (star.y > HEIGHT + 4) {
      star.y = -4;
      star.x = rand(0, WIDTH);
    }
  });
}

function updateBullets(dt) {
  for (let i = bullets.length - 1; i >= 0; i -= 1) {
    const b = bullets[i];
    b.x += b.vx * dt;
    b.y += b.vy * dt;
    if (b.y < -20 || b.x < -20 || b.x > WIDTH + 20) bullets.splice(i, 1);
  }
  for (let i = enemyBullets.length - 1; i >= 0; i -= 1) {
    const b = enemyBullets[i];
    b.x += b.vx * dt;
    b.y += b.vy * dt;
    if (b.y > HEIGHT + 30 || b.y < -40 || b.x < -40 || b.x > WIDTH + 40) enemyBullets.splice(i, 1);
  }
}

function updateEnemies(dt) {
  for (let i = enemies.length - 1; i >= 0; i -= 1) {
    const enemy = enemies[i];
    enemy.y += enemy.speed * dt;
    enemy.phase += dt * 2.2;
    enemy.x += (enemy.vx + Math.sin(enemy.phase) * (enemy.type === "boss" ? 38 : 28)) * dt;
    if (enemy.type === "boss") {
      if (enemy.y > 105) enemy.speed = 10;
      if (enemy.x < 80 || enemy.x > WIDTH - 80) enemy.vx *= -1;
    }
    enemy.shoot -= dt;
    if (enemy.y > 20 && enemy.shoot <= 0) {
      fireEnemy(enemy);
      enemy.shoot = enemy.type === "boss" ? 0.85 : rand(1.45, 3.2);
    }
    if (enemy.y > HEIGHT + 80) enemies.splice(i, 1);
  }
}

function updateObstacles(dt) {
  for (let i = obstacles.length - 1; i >= 0; i -= 1) {
    const obstacle = obstacles[i];
    obstacle.x += obstacle.vx * dt;
    obstacle.y += obstacle.speed * dt;
    obstacle.angle += obstacle.spin * dt;
    if (obstacle.x < obstacle.r || obstacle.x > WIDTH - obstacle.r) obstacle.vx *= -1;
    if (obstacle.y > HEIGHT + obstacle.r + 30) obstacles.splice(i, 1);
  }
}

function updatePickups(dt) {
  for (let i = pickups.length - 1; i >= 0; i -= 1) {
    const item = pickups[i];
    item.y += item.speed * dt;
    item.angle += item.spin * dt;
    if (item.y > HEIGHT + 30) pickups.splice(i, 1);
  }
}

function updateParticles(dt) {
  for (let i = particles.length - 1; i >= 0; i -= 1) {
    const p = particles[i];
    p.x += p.vx * dt;
    p.y += p.vy * dt;
    p.vx *= 0.98;
    p.vy *= 0.98;
    p.life -= dt;
    if (p.life <= 0) particles.splice(i, 1);
  }
}

// 碰撞分成三类：玩家子弹打目标、敌方伤害玩家、玩家拾取道具。
function handleCollisions() {
  hitObstaclesWithBullets();
  hitEnemiesWithBullets();

  if (player.invulnerable <= 0) {
    for (let i = enemyBullets.length - 1; i >= 0; i -= 1) {
      if (distance(player, enemyBullets[i]) < player.r + enemyBullets[i].r) {
        enemyBullets.splice(i, 1);
        damagePlayer();
        break;
      }
    }
    if (player.invulnerable <= 0) {
      for (let i = enemies.length - 1; i >= 0; i -= 1) {
        if (distance(player, enemies[i]) < player.r + enemies[i].r * 0.75) {
          enemies.splice(i, 1);
          damagePlayer();
          break;
        }
      }
    }
    if (player.invulnerable <= 0) {
      for (let i = obstacles.length - 1; i >= 0; i -= 1) {
        if (distance(player, obstacles[i]) < player.r + obstacles[i].r * 0.68) {
          addParticles(obstacles[i].x, obstacles[i].y, "#b7c0c8", 24, 300);
          obstacles.splice(i, 1);
          damagePlayer();
          break;
        }
      }
    }
  }

  for (let i = pickups.length - 1; i >= 0; i -= 1) {
    const item = pickups[i];
    if (distance(player, item) < player.r + item.r) {
      applyPickup(item);
      player.score += 80;
      pickups.splice(i, 1);
    }
  }
}

function applyPickup(item) {
  const color = pickupStyle(item.kind).color;
  if (item.kind === "power") {
    player.power = clamp(player.power + 1, 1, 5);
    playSound("pickup");
  } else if (item.kind === "life") {
    player.lives = clamp(player.lives + 1, 0, 6);
    playSound("pickup");
  } else if (item.kind === "shield") {
    player.shield = Math.max(player.shield, 7);
    player.invulnerable = Math.max(player.invulnerable, 0.6);
    playSound("shield");
  } else if (item.kind === "bomb") {
    triggerBomb(item.x, item.y);
    playSound("bomb");
  } else if (item.kind === "overdrive") {
    player.overdrive = 8;
    playSound("overdrive");
  }
  addParticles(item.x, item.y, color, item.kind === "bomb" ? 46 : 24, item.kind === "bomb" ? 520 : 280);
}

// 炸弹是稀有道具：拾取后立刻清除弹幕并重创屏幕内目标。
function triggerBomb(x, y) {
  enemyBullets.length = 0;
  player.score += enemies.length * 120 + obstacles.length * 70;
  enemies.forEach((enemy) => addParticles(enemy.x, enemy.y, enemy.type === "boss" ? "#ffd166" : "#ff5a77", enemy.type === "boss" ? 60 : 28, 460));
  obstacles.forEach((obstacle) => addParticles(obstacle.x, obstacle.y, "#b7c0c8", 24, 360));
  enemies.length = 0;
  obstacles.length = 0;
  addParticles(x, y, "#ffd166", 90, 680);
  shake = Math.max(shake, 18);
}

function hitObstaclesWithBullets() {
  for (let i = obstacles.length - 1; i >= 0; i -= 1) {
    const obstacle = obstacles[i];
    for (let j = bullets.length - 1; j >= 0; j -= 1) {
      const bullet = bullets[j];
      if (distance(obstacle, bullet) < obstacle.r * 0.72 + bullet.r) {
        obstacle.hp -= bullet.damage;
        bullets.splice(j, 1);
        addParticles(bullet.x, bullet.y, "#b7c0c8", 4, 150);
        playSound("hit");
        if (obstacle.hp <= 0) {
          player.score += obstacle.score;
          addParticles(obstacle.x, obstacle.y, "#aeb7c0", 30, 340);
          playSound("boom");
          obstacles.splice(i, 1);
          shake = Math.max(shake, 5);
        }
        break;
      }
    }
  }
}

function hitEnemiesWithBullets() {
  for (let i = enemies.length - 1; i >= 0; i -= 1) {
    const enemy = enemies[i];
    for (let j = bullets.length - 1; j >= 0; j -= 1) {
      const bullet = bullets[j];
      if (distance(enemy, bullet) < enemy.r + bullet.r) {
        enemy.hp -= bullet.damage;
        bullets.splice(j, 1);
        addParticles(bullet.x, bullet.y, bullet.color, 3, 120);
        playSound("hit");
        if (enemy.hp <= 0) {
          player.score += enemy.score;
          addParticles(enemy.x, enemy.y, enemy.type === "boss" ? "#ffd166" : "#51d7ff", enemy.type === "boss" ? 70 : 22, 420);
          playSound("boom");
          if (Math.random() < (enemy.type === "boss" ? 0.85 : 0.12)) spawnPickup(enemy.x, enemy.y);
          enemies.splice(i, 1);
          shake = Math.max(shake, enemy.type === "boss" ? 10 : 4);
        }
        break;
      }
    }
  }
}

function damagePlayer() {
  if (player.shield > 0) {
    player.shield = 0;
    player.invulnerable = 1.1;
    shake = 7;
    addParticles(player.x, player.y, "#58f5ff", 34, 380);
    playSound("shield");
    return;
  }
  player.lives -= 1;
  player.power = Math.max(1, player.power - 1);
  player.invulnerable = 1.45;
  shake = 12;
  hitFlash.value = true;
  window.setTimeout(() => {
    hitFlash.value = false;
  }, 120);
  addParticles(player.x, player.y, "#ff5a77", 34, 360);
  playSound("hurt");
  if (player.lives <= 0) {
    addParticles(player.x, player.y, "#ffd166", 80, 520);
    playSound("boom");
    state.value = "over";
  }
}

function draw() {
  if (!ctx) return;
  ctx.save();
  if (shake > 0) ctx.translate(rand(-shake, shake), rand(-shake, shake));
  drawBackground();
  bullets.forEach(drawBullet);
  enemyBullets.forEach(drawEnemyBullet);
  obstacles.forEach(drawObstacle);
  pickups.forEach(drawPickup);
  enemies.forEach(drawEnemy);
  drawPlayer();
  particles.forEach(drawParticle);
  ctx.restore();
}

function drawBackground() {
  const gradient = ctx.createLinearGradient(0, 0, 0, HEIGHT);
  gradient.addColorStop(0, "#0b2433");
  gradient.addColorStop(0.55, "#071018");
  gradient.addColorStop(1, "#02070b");
  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, WIDTH, HEIGHT);

  ctx.save();
  ctx.globalCompositeOperation = "screen";
  stars.forEach((star) => {
    ctx.globalAlpha = star.alpha;
    ctx.fillStyle = "#d8fbff";
    ctx.fillRect(star.x, star.y, star.size, star.size * 2.6);
  });
  ctx.restore();

  ctx.strokeStyle = "rgba(81, 215, 255, 0.08)";
  ctx.lineWidth = 1;
  for (let y = -80; y < HEIGHT; y += 80) {
    const drift = (performance.now() * 0.045) % 80;
    ctx.beginPath();
    ctx.moveTo(0, y + drift);
    ctx.lineTo(WIDTH, y + 26 + drift);
    ctx.stroke();
  }
}

function drawPlayer() {
  ctx.save();
  ctx.translate(player.x, player.y);
  const blink = player.invulnerable > 0 && Math.floor(performance.now() / 90) % 2 === 0;

  if (player.shield > 0) {
    const pulse = 1 + Math.sin(performance.now() / 120) * 0.07;
    ctx.save();
    ctx.globalAlpha = 0.34 + Math.sin(performance.now() / 160) * 0.12;
    ctx.strokeStyle = "#58f5ff";
    ctx.fillStyle = "rgba(88, 245, 255, 0.08)";
    ctx.lineWidth = 4;
    ctx.shadowColor = "#58f5ff";
    ctx.shadowBlur = 18;
    ctx.beginPath();
    ctx.ellipse(0, 0, 52 * pulse, 62 * pulse, 0, 0, Math.PI * 2);
    ctx.fill();
    ctx.stroke();
    ctx.restore();
  }

  if (player.overdrive > 0) {
    ctx.save();
    ctx.globalAlpha = 0.4;
    ctx.strokeStyle = "#c77dff";
    ctx.lineWidth = 3;
    ctx.shadowColor = "#c77dff";
    ctx.shadowBlur = 18;
    ctx.beginPath();
    ctx.moveTo(-34, 40);
    ctx.lineTo(-18, 70 + Math.sin(performance.now() / 45) * 8);
    ctx.moveTo(34, 40);
    ctx.lineTo(18, 70 + Math.cos(performance.now() / 45) * 8);
    ctx.stroke();
    ctx.restore();
  }

  ctx.globalAlpha = blink ? 0.45 : 1;

  const hull = ctx.createLinearGradient(0, -38, 0, 34);
  hull.addColorStop(0, "#f5fbff");
  hull.addColorStop(0.38, "#51d7ff");
  hull.addColorStop(1, "#0e5f8d");
  ctx.shadowColor = "#51d7ff";
  ctx.shadowBlur = 15;
  ctx.fillStyle = "rgba(81, 215, 255, 0.16)";
  ctx.beginPath();
  ctx.ellipse(0, 6, 38, 48, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.shadowBlur = 0;
  ctx.fillStyle = "#0b3550";
  ctx.beginPath();
  ctx.moveTo(-7, -4);
  ctx.lineTo(-48, 20);
  ctx.lineTo(-31, 31);
  ctx.lineTo(-5, 20);
  ctx.closePath();
  ctx.fill();

  ctx.beginPath();
  ctx.moveTo(7, -4);
  ctx.lineTo(48, 20);
  ctx.lineTo(31, 31);
  ctx.lineTo(5, 20);
  ctx.closePath();
  ctx.fill();

  ctx.fillStyle = hull;
  ctx.beginPath();
  ctx.moveTo(0, -42);
  ctx.bezierCurveTo(18, -18, 20, 13, 7, 34);
  ctx.lineTo(0, 28);
  ctx.lineTo(-7, 34);
  ctx.bezierCurveTo(-20, 13, -18, -18, 0, -42);
  ctx.closePath();
  ctx.fill();

  ctx.fillStyle = "rgba(245, 251, 255, 0.92)";
  ctx.beginPath();
  ctx.ellipse(0, -14, 8, 18, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.strokeStyle = "rgba(245, 251, 255, 0.62)";
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.moveTo(0, -35);
  ctx.lineTo(0, 24);
  ctx.stroke();

  ctx.fillStyle = "#ffd166";
  ctx.fillRect(-33, 16, 8, 11);
  ctx.fillRect(25, 16, 8, 11);

  const flame = 31 + Math.sin(performance.now() / 55) * 9;
  const flameGradient = ctx.createLinearGradient(0, 22, 0, 48);
  flameGradient.addColorStop(0, "#f5fbff");
  flameGradient.addColorStop(0.4, "#ffd166");
  flameGradient.addColorStop(1, "#ff5a77");
  ctx.fillStyle = flameGradient;
  ctx.beginPath();
  ctx.moveTo(-10, 24);
  ctx.lineTo(0, flame + 17);
  ctx.lineTo(10, 24);
  ctx.closePath();
  ctx.fill();
  ctx.restore();
}

function drawEnemy(enemy) {
  ctx.save();
  ctx.translate(enemy.x, enemy.y);
  const boss = enemy.type === "boss";
  const body = ctx.createLinearGradient(0, -enemy.r, 0, enemy.r);
  body.addColorStop(0, boss ? "#d8b8ff" : enemy.type === "elite" ? "#ffd1dc" : "#ffd8b5");
  body.addColorStop(0.42, boss ? "#7e5cff" : enemy.type === "elite" ? "#ff5a77" : "#ff8a5c");
  body.addColorStop(1, boss ? "#30155e" : enemy.type === "elite" ? "#6b1630" : "#743319");
  ctx.shadowColor = boss ? "#7e5cff" : "#ff5a77";
  ctx.shadowBlur = boss ? 18 : 10;
  ctx.fillStyle = body;
  ctx.beginPath();
  ctx.moveTo(0, boss ? 66 : 30);
  ctx.lineTo(boss ? 62 : 28, boss ? -18 : -12);
  ctx.lineTo(boss ? 25 : 11, boss ? -30 : -20);
  ctx.lineTo(boss ? 13 : 8, boss ? -57 : -30);
  ctx.lineTo(0, boss ? -38 : -16);
  ctx.lineTo(boss ? -13 : -8, boss ? -57 : -30);
  ctx.lineTo(boss ? -25 : -11, boss ? -30 : -20);
  ctx.lineTo(boss ? -62 : -28, boss ? -18 : -12);
  ctx.closePath();
  ctx.fill();

  ctx.shadowBlur = 0;
  ctx.fillStyle = "rgba(255, 255, 255, 0.72)";
  ctx.beginPath();
  ctx.ellipse(0, boss ? -9 : -4, boss ? 16 : 8, boss ? 25 : 14, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = boss ? "#ffd166" : "#51d7ff";
  ctx.fillRect(boss ? -45 : -19, boss ? 6 : 5, boss ? 16 : 8, boss ? 12 : 7);
  ctx.fillRect(boss ? 29 : 11, boss ? 6 : 5, boss ? 16 : 8, boss ? 12 : 7);

  ctx.fillStyle = "rgba(255, 255, 255, 0.82)";
  ctx.fillRect(-enemy.r * 0.62, enemy.r + 9, enemy.r * 1.24, 5);
  ctx.fillStyle = "#76f7b6";
  ctx.fillRect(-enemy.r * 0.62, enemy.r + 9, enemy.r * 1.24 * clamp(enemy.hp / enemy.maxHp, 0, 1), 5);
  ctx.restore();
}

function drawObstacle(obstacle) {
  ctx.save();
  ctx.translate(obstacle.x, obstacle.y);
  ctx.rotate(obstacle.angle);
  const rock = ctx.createRadialGradient(-obstacle.r * 0.3, -obstacle.r * 0.35, obstacle.r * 0.1, 0, 0, obstacle.r);
  rock.addColorStop(0, "#d7d0c2");
  rock.addColorStop(0.45, "#7f8790");
  rock.addColorStop(1, "#323941");
  ctx.fillStyle = rock;
  ctx.strokeStyle = "#c2ccd4";
  ctx.lineWidth = 2;
  ctx.beginPath();
  obstacle.points.forEach((point, index) => {
    const radius = obstacle.r * point.radius;
    const x = Math.cos(point.angle) * radius;
    const y = Math.sin(point.angle) * radius;
    if (index === 0) ctx.moveTo(x, y);
    else ctx.lineTo(x, y);
  });
  ctx.closePath();
  ctx.fill();
  ctx.stroke();

  ctx.fillStyle = "rgba(8, 15, 20, 0.32)";
  obstacle.craters.forEach((crater) => {
    ctx.beginPath();
    ctx.ellipse(crater.x * obstacle.r, crater.y * obstacle.r, obstacle.r * crater.rx, obstacle.r * crater.ry, crater.angle, 0, Math.PI * 2);
    ctx.fill();
  });

  ctx.rotate(-obstacle.angle);
  ctx.fillStyle = "rgba(255, 255, 255, 0.75)";
  ctx.fillRect(-obstacle.r * 0.5, obstacle.r + 8, obstacle.r, 4);
  ctx.fillStyle = "#ffd166";
  ctx.fillRect(-obstacle.r * 0.5, obstacle.r + 8, obstacle.r * clamp(obstacle.hp / obstacle.maxHp, 0, 1), 4);
  ctx.restore();
}

function drawBullet(bullet) {
  ctx.save();
  ctx.fillStyle = bullet.color;
  ctx.shadowColor = bullet.color;
  ctx.shadowBlur = 12;
  ctx.beginPath();
  ctx.ellipse(bullet.x, bullet.y, bullet.r, bullet.r * 2.4, 0, 0, Math.PI * 2);
  ctx.fill();
  ctx.restore();
}

function drawEnemyBullet(bullet) {
  ctx.save();
  ctx.fillStyle = bullet.color;
  ctx.shadowColor = bullet.color;
  ctx.shadowBlur = 10;
  ctx.beginPath();
  ctx.arc(bullet.x, bullet.y, bullet.r, 0, Math.PI * 2);
  ctx.fill();
  ctx.restore();
}

function pickupStyle(kind) {
  const styles = {
    power: { color: "#51d7ff", text: "P" },
    life: { color: "#76f7b6", text: "+" },
    shield: { color: "#58f5ff", text: "S" },
    bomb: { color: "#ffd166", text: "B" },
    overdrive: { color: "#c77dff", text: "O" }
  };
  return styles[kind] || styles.power;
}

function drawPickup(item) {
  ctx.save();
  ctx.translate(item.x, item.y);
  ctx.rotate(item.angle);
  const { color, text } = pickupStyle(item.kind);
  ctx.strokeStyle = color;
  ctx.fillStyle = "rgba(5, 15, 23, 0.76)";
  ctx.lineWidth = 3;
  ctx.shadowColor = color;
  ctx.shadowBlur = 16;

  if (item.kind === "bomb") {
    ctx.beginPath();
    ctx.moveTo(0, -item.r - 2);
    ctx.lineTo(item.r + 2, 0);
    ctx.lineTo(0, item.r + 2);
    ctx.lineTo(-item.r - 2, 0);
    ctx.closePath();
    ctx.fill();
    ctx.stroke();
  } else {
    ctx.beginPath();
    ctx.arc(0, 0, item.r, 0, Math.PI * 2);
    ctx.fill();
    ctx.stroke();
  }

  ctx.rotate(-item.angle);
  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.arc(0, 0, item.r * 0.58, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = "#061018";
  ctx.font = "900 17px system-ui";
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.fillText(text, 0, 0);
  ctx.restore();
}

function drawParticle(particle) {
  ctx.save();
  ctx.globalAlpha = clamp(particle.life / particle.ttl, 0, 1);
  ctx.fillStyle = particle.color;
  ctx.beginPath();
  ctx.arc(particle.x, particle.y, particle.size, 0, Math.PI * 2);
  ctx.fill();
  ctx.restore();
}

function loop(now) {
  if (state.value !== "playing") return;
  const dt = Math.min(0.033, (now - lastTime) / 1000 || 0.016);
  lastTime = now;
  update(dt);
  draw();
  rafId = requestAnimationFrame(loop);
}

function pointerPos(event) {
  const rect = canvasRef.value.getBoundingClientRect();
  return { x: ((event.clientX - rect.left) / rect.width) * WIDTH, y: ((event.clientY - rect.top) / rect.height) * HEIGHT };
}

function onPointerDown(event) {
  if (state.value !== "playing") return;
  canvasRef.value.setPointerCapture(event.pointerId);
  pointer = pointerPos(event);
}

function onPointerMove(event) {
  if (!pointer || state.value !== "playing") return;
  pointer = pointerPos(event);
}

function clearPointer() {
  pointer = null;
}

function onKeyDown(event) {
  const key = event.key.length === 1 ? event.key.toLowerCase() : event.key;
  if (["ArrowLeft", "ArrowRight", "ArrowUp", "ArrowDown", " "].includes(event.key)) event.preventDefault();
  if (key === "p" || key === "Escape") {
    togglePause();
    return;
  }
  if (key === "r") {
    startGame();
    return;
  }
  keys.add(key);
}

function onKeyUp(event) {
  const key = event.key.length === 1 ? event.key.toLowerCase() : event.key;
  keys.delete(key);
}

onMounted(() => {
  ctx = canvasRef.value.getContext("2d");
  resetStars();
  syncHud();
  draw();
  window.addEventListener("keydown", onKeyDown);
  window.addEventListener("keyup", onKeyUp);
});

onBeforeUnmount(() => {
  cancelAnimationFrame(rafId);
  window.removeEventListener("keydown", onKeyDown);
  window.removeEventListener("keyup", onKeyUp);
  if (audioCtx) audioCtx.close();
});
</script>

<style scoped>
.game-shell {
  position: relative;
  display: grid;
  place-items: center;
  width: min(100vw, 620px);
  height: min(100vh, 920px);
  min-height: 620px;
  margin: 0 auto;
  overflow: hidden;
  background: #06111a;
  box-shadow: 0 24px 80px rgba(0, 0, 0, 0.42);
}

.game-canvas {
  display: block;
  width: 100%;
  height: 100%;
  touch-action: none;
  cursor: crosshair;
}

.hud {
  position: absolute;
  inset: 0 0 auto 0;
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 8px;
  padding: 12px;
  pointer-events: none;
}

.metric {
  min-width: 0;
  border: 1px solid rgba(255, 255, 255, 0.16);
  border-radius: 8px;
  padding: 8px 10px;
  background: rgba(7, 16, 24, 0.78);
  backdrop-filter: blur(12px);
}

.label {
  display: block;
  color: #a9b9c4;
  font-size: 11px;
  line-height: 1.2;
  white-space: nowrap;
}

.value {
  display: block;
  margin-top: 2px;
  font-size: clamp(16px, 3.4vw, 22px);
  font-weight: 800;
  line-height: 1.1;
  font-variant-numeric: tabular-nums;
}

.sound-toggle {
  position: absolute;
  right: 12px;
  bottom: 12px;
  z-index: 2;
  width: 44px;
  min-height: 44px;
  border: 1px solid rgba(255, 255, 255, 0.16);
  border-radius: 8px;
  color: #f5fbff;
  background: rgba(7, 16, 24, 0.78);
  box-shadow: 0 12px 28px rgba(0, 0, 0, 0.24);
  backdrop-filter: blur(12px);
  cursor: pointer;
  font-size: 18px;
  line-height: 1;
}

.overlay {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  padding: 24px;
  background: linear-gradient(180deg, rgba(4, 12, 18, 0.72), rgba(4, 12, 18, 0.38));
  transition: opacity 160ms ease, visibility 160ms ease;
}

.overlay.hidden {
  opacity: 0;
  visibility: hidden;
  pointer-events: none;
}

.menu {
  width: min(440px, 100%);
  border: 1px solid rgba(255, 255, 255, 0.16);
  border-radius: 8px;
  padding: 22px;
  background: rgba(5, 15, 23, 0.9);
  box-shadow: 0 22px 60px rgba(0, 0, 0, 0.34);
  backdrop-filter: blur(16px);
}

h1 {
  margin: 0;
  font-size: clamp(36px, 10vw, 68px);
  line-height: 0.95;
  letter-spacing: 0;
}

.subtitle {
  margin: 12px 0 18px;
  color: #a9b9c4;
  font-size: 15px;
  line-height: 1.65;
}

.controls {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 8px;
  margin: 18px 0;
  color: #d8e8f0;
  font-size: 13px;
}

.key {
  border: 1px solid rgba(255, 255, 255, 0.16);
  border-radius: 8px;
  padding: 9px 10px;
  background: rgba(255, 255, 255, 0.05);
}

.primary-action {
  width: 100%;
  min-height: 46px;
  border: 0;
  border-radius: 8px;
  color: #061018;
  background: linear-gradient(135deg, #51d7ff, #76f7b6);
  cursor: pointer;
  font-weight: 900;
  box-shadow: 0 12px 28px rgba(81, 215, 255, 0.2);
}

.primary-action:active,
.sound-toggle:active {
  transform: translateY(1px);
}

.hint {
  margin: 12px 0 0;
  color: #a9b9c4;
  font-size: 12px;
  text-align: center;
}

.flash {
  position: absolute;
  inset: 0;
  pointer-events: none;
  background: rgba(255, 255, 255, 0);
  transition: background 120ms ease;
}

.flash.hit {
  background: rgba(255, 90, 119, 0.22);
}

@media (max-width: 520px) {
  .game-shell {
    width: 100vw;
    height: 100vh;
    min-height: 0;
  }

  .hud {
    grid-template-columns: repeat(2, 1fr);
  }

  .menu {
    padding: 18px;
  }
}
</style>
