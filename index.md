---
layout: default
---

<style>
  :root {
    color-scheme: light dark;
    font-family: "Segoe UI", system-ui, -apple-system, sans-serif;
  }

  body {
    margin: 0;
    padding: 24px;
    display: flex;
    justify-content: center;
    background: #0f1115;
    color: #eef1f6;
  }

  .snake-app {
    max-width: 720px;
    width: 100%;
    background: #171a22;
    border-radius: 18px;
    padding: 24px;
    box-shadow: 0 20px 40px rgba(0, 0, 0, 0.35);
  }

  h1 {
    font-size: 2rem;
    margin: 0 0 8px;
  }

  p {
    margin: 0 0 16px;
    color: #c8cfdd;
  }

  .hud {
    display: flex;
    flex-wrap: wrap;
    gap: 16px;
    align-items: center;
    margin: 16px 0;
  }

  .stat {
    background: #222736;
    padding: 10px 16px;
    border-radius: 999px;
    font-weight: 600;
  }

  .controls {
    display: flex;
    gap: 12px;
    flex-wrap: wrap;
  }

  button {
    background: #4c7cf5;
    border: none;
    color: white;
    padding: 10px 18px;
    border-radius: 10px;
    font-weight: 600;
    cursor: pointer;
    transition: transform 0.1s ease, background 0.2s ease;
  }

  button:hover {
    background: #3b6be2;
  }

  button:active {
    transform: scale(0.98);
  }

  canvas {
    width: 100%;
    max-width: 600px;
    aspect-ratio: 1;
    display: block;
    margin: 0 auto;
    background: linear-gradient(135deg, #111620, #1c2230);
    border-radius: 16px;
    border: 2px solid #2d3345;
  }

  .tips {
    margin-top: 16px;
    font-size: 0.95rem;
    color: #9aa3b5;
  }
</style>

<div class="snake-app">
  <h1>Snake im Browser</h1>
  <p>Steuere die Schlange mit den Pfeiltasten oder WASD. Sammle Snacks, vermeide Wände und deinen eigenen Körper.</p>

  <div class="hud">
    <div class="stat">Punkte: <span id="score">0</span></div>
    <div class="stat">Bestwert: <span id="best-score">0</span></div>
    <div class="stat">Tempo: <span id="speed">1.0x</span></div>
  </div>

  <canvas id="game" width="600" height="600" aria-label="Snake Spiel" role="img"></canvas>

  <div class="controls">
    <button id="start">Start</button>
    <button id="pause">Pause</button>
    <button id="reset">Neu starten</button>
  </div>

  <div class="tips">
    Tipp: Mit der Leertaste pausierst du das Spiel. Jeder Snack erhöht das Tempo leicht.
  </div>
</div>

<script>
  const canvas = document.getElementById("game");
  const ctx = canvas.getContext("2d");
  const scoreEl = document.getElementById("score");
  const bestScoreEl = document.getElementById("best-score");
  const speedEl = document.getElementById("speed");
  const startButton = document.getElementById("start");
  const pauseButton = document.getElementById("pause");
  const resetButton = document.getElementById("reset");

  const gridSize = 24;
  const cellSize = canvas.width / gridSize;
  const baseSpeed = 140;

  let snake;
  let direction;
  let nextDirection;
  let food;
  let score;
  let bestScore = 0;
  let speedMultiplier;
  let timer;
  let running = false;
  let paused = false;

  const palette = {
    snake: "#7ee787",
    head: "#1f6feb",
    food: "#ff7b72",
    grid: "rgba(255, 255, 255, 0.04)",
  };

  const drawGrid = () => {
    ctx.strokeStyle = palette.grid;
    ctx.lineWidth = 1;
    for (let i = 0; i <= gridSize; i += 1) {
      const pos = i * cellSize;
      ctx.beginPath();
      ctx.moveTo(pos, 0);
      ctx.lineTo(pos, canvas.height);
      ctx.stroke();
      ctx.beginPath();
      ctx.moveTo(0, pos);
      ctx.lineTo(canvas.width, pos);
      ctx.stroke();
    }
  };

  const resetGame = () => {
    snake = [
      { x: 10, y: 12 },
      { x: 9, y: 12 },
      { x: 8, y: 12 },
    ];
    direction = { x: 1, y: 0 };
    nextDirection = direction;
    score = 0;
    speedMultiplier = 1;
    food = spawnFood();
    updateHUD();
  };

  const updateHUD = () => {
    scoreEl.textContent = score;
    bestScoreEl.textContent = bestScore;
    speedEl.textContent = `${speedMultiplier.toFixed(1)}x`;
  };

  const spawnFood = () => {
    let position;
    do {
      position = {
        x: Math.floor(Math.random() * gridSize),
        y: Math.floor(Math.random() * gridSize),
      };
    } while (snake.some((segment) => segment.x === position.x && segment.y === position.y));
    return position;
  };

  const drawCell = (x, y, color) => {
    ctx.fillStyle = color;
    ctx.fillRect(x * cellSize, y * cellSize, cellSize, cellSize);
  };

  const render = () => {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    drawGrid();
    snake.forEach((segment, index) => {
      drawCell(segment.x, segment.y, index === 0 ? palette.head : palette.snake);
    });
    drawCell(food.x, food.y, palette.food);
  };

  const tick = () => {
    if (!running || paused) {
      return;
    }

    direction = nextDirection;
    const head = { x: snake[0].x + direction.x, y: snake[0].y + direction.y };

    const hitWall = head.x < 0 || head.x >= gridSize || head.y < 0 || head.y >= gridSize;
    const hitSelf = snake.some((segment) => segment.x === head.x && segment.y === head.y);

    if (hitWall || hitSelf) {
      running = false;
      clearInterval(timer);
      if (score > bestScore) {
        bestScore = score;
      }
      updateHUD();
      return;
    }

    snake.unshift(head);

    if (head.x === food.x && head.y === food.y) {
      score += 10;
      speedMultiplier = Math.min(2.5, speedMultiplier + 0.1);
      food = spawnFood();
    } else {
      snake.pop();
    }

    updateHUD();
    render();
    scheduleNextTick();
  };

  const scheduleNextTick = () => {
    clearInterval(timer);
    timer = setInterval(tick, baseSpeed / speedMultiplier);
  };

  const startGame = () => {
    if (running) {
      return;
    }
    running = true;
    paused = false;
    scheduleNextTick();
  };

  const togglePause = () => {
    if (!running) {
      return;
    }
    paused = !paused;
    pauseButton.textContent = paused ? "Weiter" : "Pause";
  };

  const setDirection = (x, y) => {
    if (direction.x === -x && direction.y === -y) {
      return;
    }
    nextDirection = { x, y };
  };

  const handleKey = (event) => {
    switch (event.key) {
      case "ArrowUp":
      case "w":
      case "W":
        setDirection(0, -1);
        break;
      case "ArrowDown":
      case "s":
      case "S":
        setDirection(0, 1);
        break;
      case "ArrowLeft":
      case "a":
      case "A":
        setDirection(-1, 0);
        break;
      case "ArrowRight":
      case "d":
      case "D":
        setDirection(1, 0);
        break;
      case " ":
        event.preventDefault();
        togglePause();
        break;
      default:
        break;
    }
  };

  startButton.addEventListener("click", startGame);
  pauseButton.addEventListener("click", togglePause);
  resetButton.addEventListener("click", () => {
    running = false;
    clearInterval(timer);
    resetGame();
    render();
    pauseButton.textContent = "Pause";
  });
  document.addEventListener("keydown", handleKey);

  resetGame();
  render();
</script>
