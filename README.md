const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const scoreEl = document.getElementById('score');
const bestScoreEl = document.getElementById('best-score');
const startButton = document.getElementById('startButton');

const road = { left: 72, width: 276, top: 0, bottom: canvas.height };
const laneCount = 3;
const roadGap = road.width / laneCount;

const keys = {
  ArrowLeft: false,
  ArrowRight: false,
  a: false,
  d: false,
};

let gameStarted = false;
let gameOver = false;
let score = 0;
let bestScore = Number(localStorage.getItem('neon-highway-best')) || 0;
let lastTime = 0;
let spawnTimer = 0;
let lastSpawnAt = 0;
let roadScroll = 0;

const player = {
  width: 42,
  height: 82,
  x: road.left + roadGap * 1 + roadGap / 2 - 21,
  y: canvas.height - 120,
  color: '#60a5fa',
  speed: 0,
};

const traffic = [];

bestScoreEl.textContent = bestScore;
scoreEl.textContent = score;

function resetGame() {
  gameStarted = true;
  gameOver = false;
  score = 0;
  roadScroll = 0;
  spawnTimer = 600;
  lastSpawnAt = 0;
  traffic.length = 0;
  player.x = road.left + roadGap * 1 + roadGap / 2 - player.width / 2;
  player.y = canvas.height - 120;
  scoreEl.textContent = score;
  startButton.textContent = 'Restart Race';
}

function randomBetween(min, max) {
  return Math.random() * (max - min) + min;
}

function laneCenter(lane) {
  return road.left + lane * roadGap + roadGap / 2;
}

function spawnTraffic() {
  const lane = Math.floor(Math.random() * laneCount);
  const vehicle = {
    lane,
    x: laneCenter(lane) - 23,
    y: -90,
    width: 46,
    height: 82,
    speed: randomBetween(240, 410),
    color: ['#f87171', '#fbbf24', '#a78bfa', '#34d399', '#f472b6'][Math.floor(Math.random() * 5)],
    passed: false,
  };

  traffic.push(vehicle);
}

function update(delta) {
  if (!gameStarted || gameOver) {
    return;
  }

  roadScroll += delta * 0.28;
  score += delta * 0.02;
  scoreEl.textContent = Math.floor(score);

  const moveInput = (keys.ArrowLeft || keys.a ? -1 : 0) + (keys.ArrowRight || keys.d ? 1 : 0);
  player.x += moveInput * delta * 0.46;
  player.x = Math.max(road.left + 8, Math.min(player.x, road.left + road.width - player.width - 8));

  spawnTimer -= delta;
  if (spawnTimer <= 0) {
    spawnTraffic();
    spawnTimer = randomBetween(900, 1600);
  }

  for (const vehicle of traffic) {
    vehicle.y += (vehicle.speed * delta) / 1000;

    if (!vehicle.passed && vehicle.y > player.y + player.height + 10) {
      vehicle.passed = true;
    }

    if (vehicle.y > canvas.height + 120) {
      vehicle.removed = true;
    }

    if (intersects(player, vehicle)) {
      gameOver = true;
      updateBestScore();
      startButton.textContent = 'Try Again';
      return;
    }
  }

  for (let i = traffic.length - 1; i >= 0; i -= 1) {
    if (traffic[i].removed) {
      traffic.splice(i, 1);
    }
  }
}

function updateBestScore() {
  bestScore = Math.max(bestScore, Math.floor(score));
  localStorage.setItem('neon-highway-best', String(bestScore));
  bestScoreEl.textContent = bestScore;
}

function intersects(a, b) {
  return (
    a.x < b.x + b.width &&
    a.x + a.width > b.x &&
    a.y < b.y + b.height &&
    a.y + a.height > b.y
  );
}

function drawBackground() {
  ctx.fillStyle = '#0b1220';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  for (let i = 0; i < 6; i += 1) {
    const hillY = 120 + i * 95;
    ctx.fillStyle = i % 2 === 0 ? '#1f3a5f' : '#1e293b';
    ctx.beginPath();
    ctx.moveTo(0, canvas.height);
    ctx.quadraticCurveTo(canvas.width * 0.25, hillY, canvas.width * 0.5, canvas.height);
    ctx.quadraticCurveTo(canvas.width * 0.75, hillY + 30, canvas.width, canvas.height);
    ctx.closePath();
    ctx.fill();
  }

  ctx.fillStyle = '#1f7a3d';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = '#2d3748';
  ctx.fillRect(road.left, 0, road.width, canvas.height);

  ctx.fillStyle = '#94a3b8';
  ctx.fillRect(road.left - 12, 0, 8, canvas.height);
  ctx.fillRect(road.left + road.width + 4, 0, 8, canvas.height);

  const laneMarkY = (roadScroll % 80) - 80;
  ctx.strokeStyle = '#f8fafc';
  ctx.lineWidth = 5;
  ctx.setLineDash([28, 22]);
  for (let i = 0; i < 10; i += 1) {
    const y = laneMarkY + i * 80;
    ctx.beginPath();
    ctx.moveTo(road.left + roadGap, y);
    ctx.lineTo(road.left + roadGap, y + 45);
    ctx.moveTo(road.left + roadGap * 2, y);
    ctx.lineTo(road.left + roadGap * 2, y + 45);
    ctx.stroke();
  }
  ctx.setLineDash([]);
}

function drawCar(car, isPlayer = false) {
  const x = car.x;
  const y = car.y;
  const w = car.width;
  const h = car.height;

  ctx.save();
  ctx.translate(x, y);

  ctx.fillStyle = isPlayer ? '#60a5fa' : car.color;
  ctx.fillRect(8, 0, w - 16, h);

  ctx.fillStyle = isPlayer ? '#dbeafe' : '#e2e8f0';
  ctx.fillRect(10, 12, w - 20, h * 0.38);

  ctx.fillStyle = '#0f172a';
  ctx.fillRect(6, 14, 7, 18);
  ctx.fillRect(w - 13, 14, 7, 18);
  ctx.fillRect(6, h - 30, 7, 18);
  ctx.fillRect(w - 13, h - 30, 7, 18);

  ctx.fillStyle = '#f8fafc';
  ctx.fillRect(13, 4, w - 26, 8);

  if (isPlayer) {
    ctx.fillStyle = 'rgba(255,255,255,0.7)';
    ctx.fillRect(13, 18, w - 26, 15);
  }

  ctx.restore();
}

function drawGameOverScreen() {
  ctx.fillStyle = 'rgba(15, 23, 42, 0.68)';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = '#f8fafc';
  ctx.textAlign = 'center';
  ctx.font = '700 30px Segoe UI';
  ctx.fillText('Crashed!', canvas.width / 2, canvas.height / 2 - 10);

  ctx.font = '600 18px Segoe UI';
  ctx.fillText(`Score: ${Math.floor(score)}`, canvas.width / 2, canvas.height / 2 + 28);
}

function render() {
  drawBackground();

  for (const vehicle of traffic) {
    drawCar(vehicle, false);
  }

  drawCar(player, true);

  if (gameOver) {
    drawGameOverScreen();
  }
}

function gameLoop(timestamp) {
  const delta = timestamp - lastTime || 16;
  lastTime = timestamp;

  update(delta);
  render();

  requestAnimationFrame(gameLoop);
}

window.addEventListener('keydown', (event) => {
  const key = event.key.toLowerCase();

  if (key === 'arrowleft' || key === 'a') {
    keys.ArrowLeft = true;
    keys.a = true;
  }

  if (key === 'arrowright' || key === 'd') {
    keys.ArrowRight = true;
    keys.d = true;
  }

  if (!gameStarted && (event.key === ' ' || event.key === 'Enter')) {
    resetGame();
  }
});

window.addEventListener('keyup', (event) => {
  const key = event.key.toLowerCase();

  if (key === 'arrowleft' || key === 'a') {
    keys.ArrowLeft = false;
    keys.a = false;
  }

  if (key === 'arrowright' || key === 'd') {
    keys.ArrowRight = false;
    keys.d = false;
  }
});

startButton.addEventListener('click', () => {
  resetGame();
});

render();
requestAnimationFrame(gameLoop);
