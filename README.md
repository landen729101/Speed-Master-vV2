<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no" />
<title>Speed Master - Addictive Traffic Dodge</title>
<style>
  body, html {
    margin:0; padding:0; background:#111; color:#eee; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    overflow: hidden;
    user-select:none;
    -webkit-tap-highlight-color: transparent;
  }
  #gameCanvas {
    display:block;
    margin: 10px auto;
    background: linear-gradient(to top, #222, #444);
    border-radius: 15px;
    box-shadow: 0 0 20px #0f0;
    touch-action:none;
  }
  #startScreen, #gameOverScreen {
    position: fixed;
    inset: 0;
    background: rgba(0,0,0,0.95);
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    z-index: 100;
    color: #0f0;
    text-align: center;
    padding: 20px;
  }
  input, button {
    padding: 14px 22px;
    font-size: 1.3em;
    border-radius: 12px;
    border: none;
    margin-top: 18px;
    background: #0a0;
    color: #eee;
    font-weight: 700;
    cursor: pointer;
    user-select:none;
    transition: background 0.3s ease;
    min-width: 180px;
  }
  input:focus, button:hover {
    background: #0f0;
    color: #000;
    outline: none;
  }
  #leaderboardList, #gameOverLeaderboard {
    margin-top: 20px;
    width: 280px;
    max-height: 160px;
    overflow-y: auto;
    background: #111;
    border: 2px solid #0f0;
    border-radius: 12px;
    padding: 10px;
    text-align: left;
  }
  #leaderboardList li, #gameOverLeaderboard li {
    margin: 6px 0;
    font-weight: 700;
    font-size: 1.1em;
  }
  #scoreDisplay {
    position: fixed;
    top: 10px;
    left: 50%;
    transform: translateX(-50%);
    font-size: 2em;
    font-weight: 900;
    color: #0f0;
    user-select:none;
    z-index: 50;
    text-shadow: 0 0 8px #0f0;
  }
  #instructions {
    position: fixed;
    bottom: 12px;
    left: 50%;
    transform: translateX(-50%);
    font-size: 1.1em;
    color: #080;
    user-select:none;
    text-shadow: 0 0 3px #050;
  }
</style>
</head>
<body>

<div id="startScreen">
  <h1>Speed Master</h1>
  <label for="playerName">Enter your name:</label>
  <input id="playerName" type="text" maxlength="12" placeholder="Your name" autocomplete="off" />
  <button id="startBtn">Start Race</button>

  <h2>Leaderboard</h2>
  <ol id="leaderboardList"></ol>
  <p style="margin-top:12px; font-size:0.9em; color:#0a0;">Tap left or right side to move your car</p>
</div>

<canvas id="gameCanvas" width="360" height="540" tabindex="0"></canvas>
<div id="scoreDisplay" style="display:none;">Score: 0</div>
<div id="instructions">Tap left or right side to move</div>

<div id="gameOverScreen" style="display:none;">
  <h2>Game Over!</h2>
  <p id="finalScore"></p>
  <button id="restartBtn">Play Again</button>
  <h3>Leaderboard</h3>
  <ol id="gameOverLeaderboard"></ol>
</div>

<audio id="bgMusic" loop src="https://cdn.pixabay.com/download/audio/2022/03/29/audio_ebff882d41.mp3?filename=retro-game-loop-10870.mp3" crossorigin="anonymous"></audio>
<audio id="crashSound" src="https://cdn.pixabay.com/download/audio/2022/03/23/audio_75bb4940e7.mp3?filename=explosion-2-6590.mp3" crossorigin="anonymous"></audio>

<script>
(() => {
  const canvas = document.getElementById('gameCanvas');
  const ctx = canvas.getContext('2d');
  const startScreen = document.getElementById('startScreen');
  const gameOverScreen = document.getElementById('gameOverScreen');
  const leaderboardList = document.getElementById('leaderboardList');
  const gameOverLeaderboard = document.getElementById('gameOverLeaderboard');
  const playerNameInput = document.getElementById('playerName');
  const startBtn = document.getElementById('startBtn');
  const restartBtn = document.getElementById('restartBtn');
  const scoreDisplay = document.getElementById('scoreDisplay');
  const finalScoreText = document.getElementById('finalScore');
  const bgMusic = document.getElementById('bgMusic');
  const crashSound = document.getElementById('crashSound');

  const WIDTH = canvas.width;
  const HEIGHT = canvas.height;
  const laneCount = 3;
  const laneWidth = WIDTH / laneCount;

  const carWidth = 50;
  const carHeight = 90;
  const playerY = HEIGHT - carHeight - 20;

  // Score unlock milestones and their car colors (for variety, more realistic looking)
  const unlocks = [
    {score: 0, color: '#2ecc71'},       // Green basic car
    {score: 10, color: '#3498db'},      // Blue
    {score: 25, color: '#e74c3c'},      // Red
    {score: 50, color: '#f1c40f'},      // Yellow
    {score: 75, color: '#9b59b6'},      // Purple
    {score: 150, color: '#e67e22'},     // Orange
    {score: 300, color: '#1abc9c'},     // Turquoise
    {score: 500, color: '#34495e'},     // Dark gray
    {score: 1000, color: '#ffffff'},    // White super car
  ];

  // Helper to get current unlocked car color based on score
  function getCurrentCarColor(score) {
    let color = unlocks[0].color;
    for (let i = 0; i < unlocks.length; i++) {
      if (score >= unlocks[i].score) color = unlocks[i].color;
      else break;
    }
    return color;
  }

  let playerLane = 1;
  let targetLane = 1;
  let playerX = laneToX(playerLane);

  let traffic = [];
  let speed = 3;
  let score = 0;
  let gameOver = false;
  let explosion = null;

  // Particle explosion for crash effect
  class Particle {
    constructor(x, y) {
      this.x = x;
      this.y = y;
      this.radius = Math.random() * 5 + 3;
      this.color = `hsl(${Math.random() * 60 + 30}, 100%, 60%)`;
      this.speedX = (Math.random() - 0.5) * 8;
      this.speedY = (Math.random() - 0.5) * 8;
      this.alpha = 1;
      this.decay = Math.random() * 0.03 + 0.02;
    }
    update() {
      this.x += this.speedX;
      this.y += this.speedY;
      this.alpha -= this.decay;
      this.radius *= 0.95;
    }
    draw() {
      ctx.save();
      ctx.globalAlpha = this.alpha;
      ctx.fillStyle = this.color;
      ctx.beginPath();
      ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
      ctx.fill();
      ctx.restore();
    }
  }

  class Explosion {
    constructor(x, y) {
      this.particles = [];
      for (let i = 0; i < 30; i++) {
        this.particles.push(new Particle(x, y));
      }
      this.done = false;
    }
    update() {
      this.particles.forEach(p => p.update());
      this.particles = this.particles.filter(p => p.alpha > 0.05 && p.radius > 0.5);
      if (this.particles.length === 0) this.done = true;
    }
    draw() {
      this.particles.forEach(p => p.draw());
    }
  }

  function laneToX(lane) {
    return lane * laneWidth + laneWidth / 2 - carWidth / 2;
  }

  function drawCar(x, y, color) {
    ctx.fillStyle = color;
    ctx.strokeStyle = '#222';
    ctx.lineWidth = 3;
    // Car body
    ctx.beginPath();
    ctx.moveTo(x + 10, y);
    ctx.lineTo(x + carWidth - 10, y);
    ctx.quadraticCurveTo(x + carWidth, y + 10, x + carWidth, y + 30);
    ctx.lineTo(x + carWidth, y + carHeight - 20);
    ctx.quadraticCurveTo(x + carWidth - 10, y + carHeight, x + 10, y + carHeight);
    ctx.lineTo(x, y + carHeight - 20);
    ctx.lineTo(x, y + 30);
    ctx.quadraticCurveTo(x, y + 10, x + 10, y);
    ctx.closePath();
    ctx.fill();
    ctx.stroke();

    // Windows - lighter
    ctx.fillStyle = 'rgba(255,255,255,0.5)';
    ctx.beginPath();
    ctx.moveTo(x + 12, y + 10);
    ctx.lineTo(x + carWidth - 12, y + 10);
    ctx.lineTo(x + carWidth - 20, y + 30);
    ctx.lineTo(x + 20, y + 30);
    ctx.closePath();
    ctx.fill();

    // Wheels - dark circles
    ctx.fillStyle = '#222';
    ctx.beginPath();
    ctx.ellipse(x + 15, y + carHeight - 10, 8, 12, 0, 0, Math.PI * 2);
    ctx.ellipse(x + carWidth - 15, y + carHeight - 10, 8, 12, 0, 0, Math.PI * 2);
    ctx.fill();
  }

  // Traffic car class
  class TrafficCar {
    constructor(lane, y, color) {
      this.lane = lane;
      this.x = laneToX(lane);
      this.y = y;
      this.color = color;
      this.passed = false;
    }
    update() {
      this.y += speed;
    }
    draw() {
      drawCar(this.x, this.y, this.color);
    }
  }

  // Spawn traffic ensuring at least 1 gap lane free
  function spawnTraffic() {
    const lanes = [0,1,2];
    // Check which lanes are safe to spawn cars (no cars close to top)
    const safeLanes = lanes.filter(l => !traffic.some(c => c.lane === l && c.y < 150));

    if(safeLanes.length === 0) return;

    // Decide how many cars to spawn (1 or 2 max, leave at least one gap)
    let carsToSpawn = safeLanes.length === 1 ? 1 : (Math.random() < 0.35 ? 2 : 1);

    // Shuffle safe lanes
    for(let i = safeLanes.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [safeLanes[i], safeLanes[j]] = [safeLanes[j], safeLanes[i]];
    }

    for(let i = 0; i < carsToSpawn; i++) {
      const lane = safeLanes[i];
      const color = unlocks[Math.floor(Math.random() * Math.min(unlocks.length, getCurrentUnlockIndex() + 2))].color;
      traffic.push(new TrafficCar(lane, -carHeight - Math.random() * 200, color));
    }
  }

  // Returns index of current unlock based on score
  function getCurrentUnlockIndex() {
    let idx = 0;
    for(let i = 0; i < unlocks.length; i++) {
      if(score >= unlocks[i].score) idx = i;
    }
    return idx;
  }

  // Collision detection between player and traffic cars
  function checkCollision(a, b) {
    return !(a.x > b.x + carWidth - 10 || a.x + carWidth - 10 < b.x || a.y > b.y + carHeight - 10 || a.y + carHeight - 10 < b.y);
  }

  // Draw lanes lines
  function drawLanes() {
    ctx.strokeStyle = '#0f0';
    ctx.lineWidth = 2;
    ctx.setLineDash([20, 15]);
    for(let i = 1; i < laneCount; i++) {
      const x = i * laneWidth;
      ctx.beginPath();
      ctx.moveTo(x, 0);
      ctx.lineTo(x, HEIGHT);
      ctx.stroke();
    }
    ctx.setLineDash([]);
  }

  // Update game state
  function update() {
    if(gameOver) {
      if(explosion) {
        explosion.update();
        if(explosion.done) {
          showGameOverScreen();
        }
      }
      return;
    }

    // Smooth lane movement
    let desiredX = laneToX(targetLane);
    if(Math.abs(playerX - desiredX) < 5) {
      playerX = desiredX;
      playerLane = targetLane;
    } else {
      playerX += (desiredX - playerX) * 0.3;
    }

    // Spawn traffic occasionally, speed up spawn rate with score
    if(Math.random() < 0.02 + score * 0.0003) {
      spawnTraffic();
    }

    // Update traffic cars
    traffic.forEach(car => car.update());

    // Remove cars off screen
    traffic = traffic.filter(car => car.y < HEIGHT + carHeight);

    // Check collisions
    for(let car of traffic) {
      if(checkCollision({x:playerX, y:playerY, width:carWidth, height:carHeight}, car)) {
        // Crash!
        gameOver = true;
        explosion = new Explosion(playerX + carWidth/2, playerY + carHeight/2);
        crashSound.play();
        bgMusic.pause();
      }
      // Mark passed cars for scoring
      if(!car.passed && car.y > playerY + carHeight) {
        score++;
        car.passed = true;

        // Speed up gradually every 10 points
        if(score % 10 === 0) speed += 0.3;
      }
    }

    scoreDisplay.textContent = `Score: ${score}`;
  }

  // Draw everything
  function draw() {
    // Clear canvas
    ctx.clearRect(0, 0, WIDTH, HEIGHT);

    // Draw lanes
    drawLanes();

    // Draw player car with unlocked color
    const playerColor = getCurrentCarColor(score);
    drawCar(playerX, playerY, playerColor);

    // Draw traffic cars
    traffic.forEach(car => car.draw());

    // Explosion
    if(explosion) explosion.draw();
  }

  // Game loop
  function loop() {
    update();
    draw();
    if(!gameOver || (explosion && !explosion.done)) {
      requestAnimationFrame(loop);
    }
  }

  // Start game
  function startGame() {
    const name = playerNameInput.value.trim();
    if(!name) {
      alert('Please enter your name to start!');
      playerNameInput.focus();
      return;
    }
    playerName = name;
    startScreen.style.display = 'none';
    gameOverScreen.style.display = 'none';
    scoreDisplay.style.display = 'block';

    // Reset state
    traffic = [];
    speed = 3;
    score = 0;
    gameOver = false;
    explosion = null;
    playerLane = 1;
    targetLane = 1;
    playerX = laneToX(playerLane);

    bgMusic.currentTime = 0;
    bgMusic.play();

    loop();
  }

  // Show game over screen and save leaderboard
  function showGameOverScreen() {
    scoreDisplay.style.display = 'none';
    gameOverScreen.style.display = 'flex';
    finalScoreText.textContent = `Your score: ${score}`;
    saveScore(playerName, score);
    loadLeaderboard(gameOverLeaderboard);
  }

  // Save score to localStorage leaderboard
  function saveScore(name, score) {
    let leaderboard = JSON.parse(localStorage.getItem('speedMasterLeaderboard') || '[]');
    leaderboard.push({name, score});
    leaderboard.sort((a,b) => b.score - a.score);
    if(leaderboard.length > 10) leaderboard = leaderboard.slice(0,10);
    localStorage.setItem('speedMasterLeaderboard', JSON.stringify(leaderboard));
  }

  // Load leaderboard list element
  function loadLeaderboard(olElement) {
    const leaderboard = JSON.parse(localStorage.getItem('speedMasterLeaderboard') || '[]');
    olElement.innerHTML = '';
    leaderboard.forEach(({name, score}) => {
      const li = document.createElement('li');
      li.textContent = `${name} - ${score}`;
      olElement.appendChild(li);
    });
  }
