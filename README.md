<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>KIVDA Wave Game</title>
  <style>
    body {
      margin: 0;
      overflow: hidden;
      background: #111;
      font-family: sans-serif;
    }
    canvas {
      display: block;
      background: #222;
    }
    #ui {
      position: absolute;
      top: 10px;
      left: 10px;
      font-size: 18px;
      color: white;
      z-index: 2;
    }
    #mobileControls button {
      font-size: 24px;
      margin: 2px;
      padding: 10px;
      background: #333;
      color: white;
      border: none;
      border-radius: 8px;
    }
  </style>
</head>
<body>
<canvas id="gameCanvas"></canvas>
<div id="ui">Health: 100 | Score: 0 | Wave: 1</div>

<!-- Mobile Controls -->
<div id="mobileControls" style="position:fixed; bottom:10px; left:10px; z-index:5; display:none;">
  <div style="text-align:center;">
    <button onclick="move('up')">⬆️</button><br>
    <button onclick="move('left')">⬅️</button>
    <button onclick="move('down')">⬇️</button>
    <button onclick="move('right')">➡️</button>
  </div>
</div>

<script>
const canvas = document.getElementById("gameCanvas");
const ctx = canvas.getContext("2d");

function resizeCanvas() {
  canvas.width = window.innerWidth;
  canvas.height = window.innerHeight;
}
resizeCanvas();
window.addEventListener("resize", resizeCanvas);

// Detect mobile
if (/Mobi|Android/i.test(navigator.userAgent)) {
  document.getElementById("mobileControls").style.display = "block";
}

// Game variables
let keys = {};
let bullets = [];
let enemies = [];
let score = 0;
let wave = 1;
let gameOver = false;
let waitingForStat = false;
let currentTarget = null;

let stats = {
  strength: 1,
  speed: 4,
};

const player = {
  x: canvas.width / 2,
  y: canvas.height / 2,
  size: 20,
  speed: stats.speed,
  health: 100,
};

const statOverlay = document.createElement("div");
statOverlay.style = `position:fixed; top:0; left:0; width:100%; height:100%;
  background:#000a; color:white; display:none; flex-direction:column; align-items:center; justify-content:center; z-index:10; font-size:20px;`;
statOverlay.innerHTML = `
  <h2>🎁 Bonus Stat Point!</h2>
  <p>Choose one to continue:</p>
  <button onclick="chooseStat('strength')">+ Strength (Bullet Damage)</button>
  <button onclick="chooseStat('speed')">+ Speed (Movement)</button>
  <button onclick="chooseStat('healing')">+ Heal 30 HP</button>
`;
document.body.appendChild(statOverlay);

function chooseStat(type) {
  if (type === "strength") stats.strength++;
  else if (type === "speed") {
    stats.speed++;
    player.speed = stats.speed;
  } else if (type === "healing") {
    player.health = Math.min(100, player.health + 30);
  }
  statOverlay.style.display = "none";
  waitingForStat = false;
  spawnEnemies(wave * 3);
}

function move(dir) {
  if (dir === "up") player.y -= player.speed;
  if (dir === "down") player.y += player.speed;
  if (dir === "left") player.x -= player.speed;
  if (dir === "right") player.x += player.speed;
}

document.addEventListener("keydown", e => {
  keys[e.key.toLowerCase()] = true;
});
document.addEventListener("keyup", e => {
  keys[e.key.toLowerCase()] = false;
});

function getNearestEnemy() {
  let minDist = Infinity;
  let nearest = null;
  for (const e of enemies) {
    const dist = Math.hypot(player.x - e.x, player.y - e.y);
    if (dist < minDist) {
      minDist = dist;
      nearest = e;
    }
  }
  return nearest;
}

function shoot() {
  currentTarget = getNearestEnemy();
  if (!currentTarget) return;

  let dx = currentTarget.x - player.x;
  let dy = currentTarget.y - player.y;
  const len = Math.hypot(dx, dy);
  dx /= len;
  dy /= len;

  bullets.push({
    x: player.x,
    y: player.y,
    size: 5,
    speed: 6,
    dx,
    dy,
    damage: stats.strength,
  });
}

function spawnEnemies(count) {
  if (wave % 5 === 0) {
    enemies.push({
      x: Math.random() * canvas.width,
      y: Math.random() * canvas.height,
      size: 40,
      speed: 1.2 + wave * 0.1,
      health: 30 + wave * 10,
      isBoss: true,
    });
  } else {
    for (let i = 0; i < count; i++) {
      let side = Math.floor(Math.random() * 4);
      let x = side === 0 ? 0 : side === 1 ? canvas.width : Math.random() * canvas.width;
      let y = side === 2 ? 0 : side === 3 ? canvas.height : Math.random() * canvas.height;

      enemies.push({
        x,
        y,
        size: 20,
        speed: 1 + wave * 0.2,
        health: 3 + wave,
        isBoss: false,
      });
    }
  }
}

let lastShot = 0;

function update() {
  if (gameOver || waitingForStat) return;

  // Movement
  if (keys["w"] || keys["arrowup"]) player.y -= player.speed;
  if (keys["s"] || keys["arrowdown"]) player.y += player.speed;
  if (keys["a"] || keys["arrowleft"]) player.x -= player.speed;
  if (keys["d"] || keys["arrowright"]) player.x += player.speed;

  if (Date.now() - lastShot > 300) {
    shoot();
    lastShot = Date.now();
  }

  bullets = bullets.filter(b => b.x >= 0 && b.x <= canvas.width && b.y >= 0 && b.y <= canvas.height);
  bullets.forEach(b => {
    b.x += b.dx * b.speed;
    b.y += b.dy * b.speed;
  });

  const newEnemies = [];
  let bossDefeated = false;
  enemies.forEach(e => {
    const dx = player.x - e.x;
    const dy = player.y - e.y;
    const dist = Math.hypot(dx, dy);
    e.x += (dx / dist) * e.speed;
    e.y += (dy / dist) * e.speed;

    if (dist < player.size + e.size) {
      player.health -= e.isBoss ? 1.5 : 0.5;
    }

    let hit = false;
    bullets = bullets.filter(b => {
      const bDist = Math.hypot(b.x - e.x, b.y - e.y);
      if (!hit && bDist < e.size) {
        e.health -= b.damage;
        hit = true;
        return false;
      }
      return true;
    });

    if (e.health > 0) newEnemies.push(e);
    else {
      score += e.isBoss ? 100 : 10;
      if (e.isBoss) bossDefeated = true;
    }
  });
  enemies = newEnemies;

  if (enemies.length === 0 && !waitingForStat) {
    wave++;

    if (bossDefeated) {
      waitingForStat = true;
      statOverlay.style.display = "flex";
    } else {
      spawnEnemies(wave * 3);
    }
  }

  if (player.health <= 0 && !gameOver) {
    gameOver = true;
    setTimeout(() => {
      alert("Game Over! Your score: " + score);
      location.reload();
    }, 100);
  }

  document.getElementById("ui").innerText = `Health: ${Math.max(0, Math.floor(player.health))} | Score: ${score} | Wave: ${wave}`;
}

function draw() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  ctx.save();
  ctx.globalAlpha = 0.04;
  ctx.font = "bold 100px sans-serif";
  ctx.fillStyle = "white";
  ctx.textAlign = "center";
  ctx.fillText("iALLiP KIVDA", canvas.width / 2, canvas.height / 2);
  ctx.restore();

  ctx.fillStyle = "lime";
  ctx.beginPath();
  ctx.arc(player.x, player.y, player.size, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = "yellow";
  bullets.forEach(b => {
    ctx.beginPath();
    ctx.arc(b.x, b.y, b.size, 0, Math.PI * 2);
    ctx.fill();
  });

  enemies.forEach(e => {
    ctx.fillStyle = e.isBoss ? "purple" : "red";
    ctx.beginPath();
    ctx.arc(e.x, e.y, e.size, 0, Math.PI * 2);
    ctx.fill();
  });
}

function gameLoop() {
  update();
  draw();
  requestAnimationFrame(gameLoop);
}

spawnEnemies(3);
gameLoop();
</script>
</body>
</html>
