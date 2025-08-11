<!DOCTYPE html>
<html>
<head>
  <title>🐍 Змейка: Фруктовая буря</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body {
      margin: 0;
      padding: 10px;
      text-align: center;
      font-family: 'Arial', sans-serif;
      background: #0a0a0a;
      color: #fff;
      touch-action: manipulation;
    }
    canvas {
      background: #000;
      border: 2px solid #333;
      display: block;
      margin: 20px auto;
      image-rendering: pixelated;
    }
    #score {
      font-size: 24px;
      font-weight: bold;
      margin: 10px;
    }
    #best {
      font-size: 18px;
      color: gold;
    }
    .controls {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      grid-template-rows: repeat(3, 1fr);
      gap: 10px;
      width: 200px;
      margin: 20px auto;
    }
    .btn {
      background: #0095dd;
      color: white;
      border: none;
      padding: 20px;
      font-size: 20px;
      border-radius: 10px;
      cursor: pointer;
    }
    .btn:active { background: #0077bb; }
    .btn:nth-child(1), .btn:nth-child(3), .btn:nth-child(7), .btn:nth-child(9) {
      visibility: hidden;
    }
    button#restart-btn {
      padding: 12px 24px;
      font-size: 18px;
      background: #28a745;
      color: white;
      border: none;
      border-radius: 6px;
      cursor: pointer;
    }
  </style>
</head>
<body>
  <h1>🐍 Фруктовая змейка</h1>
  <div id="score">Очки: 0</div>
  <div id="best">Рекорд: 0</div>
  <canvas id="game" width="300" height="300"></canvas>
  <div class="controls">
    <button class="btn"></button>
    <button class="btn" onclick="setDir(0,-1)">↑</button>
    <button class="btn"></button>
    <button class="btn" onclick="setDir(-1,0)">←</button>
    <button class="btn">●</button>
    <button class="btn" onclick="setDir(1,0)">→</button>
    <button class="btn"></button>
    <button class="btn" onclick="setDir(0,1)">↓</button>
    <button class="btn"></button>
  </div>
  <button id="restart-btn" onclick="restart()">Начать заново</button>

  <script>
    // Telegram WebApp
    const tg = window.Telegram?.WebApp;
    if (tg) {
      tg.ready();
      tg.expand();
    }

    // Элементы
    const canvas = document.getElementById("game");
    const ctx = canvas.getContext("2d");
    const scoreDisplay = document.getElementById("score");
    const bestDisplay = document.getElementById("best");

    // Настройки
    const gridSize = 15;
    const tileCount = 20;

    // Фрукты
    const FRUITS = [
      { emoji: "🍎", points: 10, weight: 6 },
      { emoji: "🍌", points: 10, weight: 5 },
      { emoji: "🥝", points: 10, weight: 5 },
      { emoji: "🍒", points: 10, weight: 4 },
      { emoji: "🍈", points: 20, weight: 2, big: true },
      { emoji: "🍉", points: 20, weight: 2, big: true }
    ];

    // Состояние
    let snake = [{ x: 10, y: 10 }];
    let dx = 0, dy = 0;
    let score = 0;
    let bestScore = parseInt(localStorage.getItem("snakeBest")) || 0;
    let food = null;
    let gameRunning = false;
    let gameStarted = false;
    let gameInterval;

    // Звуки (короткие base64)
    const eatSound = new Audio("data:audio/wav;base64,UklGRiQAAABXQVZFZm10IBAAAAABAAEARKwAAIhYAQACABAAZGF0YQAAAAA=");
    const dieSound = new Audio("data:audio/wav;base64,UklGRiQAAABXQVZFZm10IBAAAAABAAEARKwAAIhYAQACABAAZGF0YQAAAAI=");
    const startSound = new Audio("data:audio/wav;base64,UklGRiQAAABXQVZFZm10IBAAAAABAAEARKwAAIhYAQACABAAZGF0YQAAABA=");

    eatSound.volume = 0.3;
    dieSound.volume = 0.4;
    startSound.volume = 0.5;

    // Рекорд
    bestDisplay.textContent = `Рекорд: ${bestScore}`;

    // Выбор фрукта с учётом веса
    function getRandomFruit() {
      const totalWeight = FRUITS.reduce((sum, f) => sum + f.weight, 0);
      let rand = Math.random() * totalWeight;
      for (let fruit of FRUITS) {
        rand -= fruit.weight;
        if (rand <= 0) return { ...fruit };
      }
      return FRUITS[0];
    }

    // Анимация старта
    function showStartAnimation() {
      ctx.fillStyle = "black";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      
      let step = 0;
      const anim = setInterval(() => {
        ctx.fillStyle = "rgba(0,0,0,0.8)";
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        
        ctx.fillStyle = "lime";
        ctx.font = "20px Arial";
        ctx.fillText("🐍 ЗМЕЙКА", 90, 120);
        
        ctx.fillStyle = "white";
        ctx.font = "16px Arial";
        const texts = ["просыпается...", "тянется...", "голодна!"];
        ctx.fillText(texts[step % 3], 95, 150);
        
        step++;
        if (step > 15) {
          clearInterval(anim);
          draw();
          showHint();
        }
      }, 200);
    }

    function showHint() {
      ctx.fillStyle = "rgba(0, 0, 0, 0.5)";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "white";
      ctx.font = "18px Arial";
      ctx.fillText("Нажми стрелку", 90, 140);
      ctx.fillText("чтобы начать", 90, 170);
    }

    function placeFood() {
      food = getRandomFruit();
      food.x = Math.floor(Math.random() * tileCount);
      food.y = Math.floor(Math.random() * tileCount);
      
      for (let part of snake) {
        if (part.x === food.x && part.y === food.y) {
          placeFood();
        }
      }
    }

    function draw() {
      ctx.fillStyle = "black";
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      // Змейка
      ctx.fillStyle = "lime";
      for (let part of snake) {
        ctx.fillRect(part.x * gridSize, part.y * gridSize, gridSize - 2, gridSize - 2);
      }

      // Фрукт
      if (food) {
        ctx.font = food.big ? "24px Arial" : "18px Arial";
        ctx.fillText(food.emoji, food.x * gridSize + 2, food.y * gridSize + 14);
      }
    }

    function setDir(x, y) {
      if (!gameStarted) {
        gameStarted = true;
        gameRunning = true;
        startSound.play();
        clearInterval(gameInterval);
        gameInterval = setInterval(gameLoop, 150);
      }

      if (x !== 0 && dx === 0) { dx = x; dy = 0; }
      if (y !== 0 && dy === 0) { dy = y; dx = 0; }
    }

    function gameLoop() {
      if (!gameRunning) return;

      const head = { x: snake[0].x + dx, y: snake[0].y + dy };

      if (head.x < 0 || head.x >= tileCount || head.y < 0 || head.y >= tileCount) {
        gameOver();
        return;
      }

      for (let part of snake) {
        if (part.x === head.x && part.y === head.y) {
          gameOver();
          return;
        }
      }

      snake.unshift(head);

      if (head.x === food.x && head.y === food.y) {
        score += food.points;
        scoreDisplay.textContent = `Очки: ${score}`;
        eatSound.play();
        placeFood();
      } else {
        snake.pop();
      }

      draw();
    }

    function gameOver() {
      gameRunning = false;
      clearInterval(gameInterval);
      dieSound.play();

      // Обновляем рекорд
      if (score > bestScore) {
        bestScore = score;
        localStorage.setItem("snakeBest", bestScore);
        bestDisplay.textContent = `Рекорд: ${bestScore} 🌟`;
      }

      ctx.fillStyle = "rgba(0,0,0,0.8)";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "red";
      ctx.font = "30px Arial";
      ctx.fillText("ПОТРАЧЕНО!", 60, 140);
      ctx.fillStyle = "white";
      ctx.font = "20px Arial";
      ctx.fillText(`Очки: ${score}`, 100, 180);
    }

    function restart() {
      clearInterval(gameInterval);
      snake = [{ x: 10, y: 10 }];
      dx = 0; dy = 0;
      score = 0;
      scoreDisplay.textContent = "Очки: 0";
      gameRunning = false;
      gameStarted = false;
      placeFood();
      draw();
      showStartAnimation();
    }

    // Старт
    window.onload = () => {
      placeFood();
      draw();
      showStartAnimation();
    };
  </script>
</body>
</html>
