<!DOCTYPE html>
<html>
<head>
  <title>🚀 Космическая змейка</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body {
      margin: 0;
      padding: 10px;
      text-align: center;
      font-family: 'Arial', sans-serif;
      background: #000;
      color: #0f0;
      touch-action: manipulation;
    }
    canvas {
      background: #000;
      border: 1px solid #333;
      image-rendering: pixelated;
      display: block;
      margin: 20px auto;
    }
    #score {
      font-size: 24px;
      font-weight: bold;
      color: #0f0;
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
      background: #0066cc;
      color: white;
      border: none;
      padding: 20px;
      font-size: 20px;
      border-radius: 10px;
      cursor: pointer;
    }
    button#restart-btn {
      padding: 12px 24px;
      font-size: 18px;
      background: #00cc44;
      color: white;
      border: none;
      border-radius: 6px;
      cursor: pointer;
    }
  </style>
</head>
<body>
  <h1>🚀 КОСМИЧЕСКАЯ ЗМЕЙКА</h1>
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

    // Звёзды в фоне
    const stars = [];
    for (let i = 0; i < 50; i++) {
      stars.push({
        x: Math.random() * canvas.width,
        y: Math.random() * canvas.height,
        r: Math.random() * 1.5,
        a: Math.random()
      });
    }

    // Фрукты-планеты
    const FRUITS = [
      { emoji: "🪐", points: 10, weight: 5 },
      { emoji: "🌍", points: 10, weight: 5 },
      { emoji: "🪷", points: 10, weight: 4 },
      { emoji: "☄️", points: 15, weight: 3 },
      { emoji: "🛸", points: 25, weight: 2 },
      { emoji: "🪸", points: 10, weight: 4 },
      { emoji: "🍒", points: 10, weight: 6 },
      { emoji: "🍉", points: 20, weight: 3 },
      { emoji: "🍇", points: 10, weight: 5 },
      { emoji: "🪻", points: 10, weight: 4 }
    ];

    // Состояние
    let snake = [{ x: 10, y: 10 }];
    let dx = 0, dy = 0;
    let score = 0;
    let bestScore = parseInt(localStorage.getItem("spaceSnakeBest")) || 0;
    let food = null;
    let gameRunning = false;
    let gameStarted = false;
    let gameInterval;
    let popups = [];
    let celebration = null; // Анимация уровня
    let eatEffects = [];

    // Рекорд
    bestDisplay.textContent = `Рекорд: ${bestScore}`;

    // Звук (тихий бип)
    function playSound() {
      const ctx = new (window.AudioContext || window.webkitAudioContext)();
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      osc.connect(gain);
      gain.connect(ctx.destination);
      osc.frequency.setValueAtTime(300, ctx.currentTime);
      gain.gain.setValueAtTime(0.1, ctx.currentTime);
      gain.gain.exponentialRampToValueAtTime(0.01, ctx.currentTime + 0.3);
      osc.start();
      osc.stop(ctx.currentTime + 0.3);
    }

    // Всплывающие очки
    function addPopup(x, y, text, color = "#00ff00") {
      popups.push({
        x: x * gridSize + gridSize / 2,
        y: y * gridSize,
        text: text,
        color: color,
        alpha: 1,
        dy: -1.5
      });
    }

    // Эффект поедания
    function addEatEffect(x, y) {
      for (let i = 0; i < 8; i++) {
        eatEffects.push({
          x: x * gridSize + gridSize / 2,
          y: y * gridSize + gridSize / 2,
          dx: (Math.random() - 0.5) * 6,
          dy: (Math.random() - 0.5) * 6,
          alpha: 1,
          r: Math.random() * 2 + 1
        });
      }
    }

    // Анимация праздника (каждые 100 очков)
    function startCelebration() {
      celebration = { text: "LEVEL UP 🚀", alpha: 1 };
      setTimeout(() => {
        if (celebration) celebration.alpha = 0;
      }, 1500);
    }

    // Выбор фрукта
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
      let step = 0;
      const texts = ["включаем двигатели...", "выход в эфир...", "змейка-корабль, старт!"];
      const anim = setInterval(() => {
        drawSpace();
        ctx.fillStyle = "rgba(0,0,0,0.7)";
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        
        ctx.fillStyle = "#00ff00";
        ctx.font = "20px Arial";
        ctx.fillText("🚀 КОСМОС", 80, 120);
        
        ctx.fillStyle = "white";
        ctx.font = "16px Arial";
        ctx.fillText(texts[step % 3], 60, 150);
        
        step++;
        if (step > 12) {
          clearInterval(anim);
          draw();
          showHint();
        }
      }, 250);
    }

    function showHint() {
      ctx.fillStyle = "rgba(0,0,0,0.5)";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "white";
      ctx.font = "18px Arial";
      ctx.fillText("Нажми стрелку", 90, 140);
      ctx.fillText("для старта", 90, 170);
    }

    function placeFood() {
      food = getRandomFruit();
      food.x = Math.floor(Math.random() * tileCount);
      food.y = Math.floor(Math.random() * tileCount);
      
      // Не на змейке
      for (let part of snake) {
        if (part.x === food.x && part.y === food.y) {
          placeFood();
        }
      }
    }

    function drawSpace() {
      ctx.fillStyle = "black";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      
      // Звёзды
      stars.forEach(s => {
        ctx.fillStyle = `rgba(255, 255, 255, ${s.a})`;
        ctx.beginPath();
        ctx.arc(s.x, s.y, s.r, 0, Math.PI * 2);
        ctx.fill();
      });
    }

    function draw() {
      drawSpace();

      // Змейка (цельная)
      ctx.fillStyle = "#00ff00";
      for (let i = 0; i < snake.length; i++) {
        const part = snake[i];
        ctx.fillRect(part.x * gridSize, part.y * gridSize, gridSize, gridSize);
        if (i > 0) {
          ctx.fillStyle = "#00cc00";
        }
      }

      // Голова — светится
      const head = snake[0];
      ctx.fillStyle = "#00ff00";
      ctx.fillRect(head.x * gridSize, head.y * gridSize, gridSize, gridSize);

      // Фрукт
      if (food) {
        ctx.font = food.points >= 20 ? "24px Arial" : "18px Arial";
        ctx.fillText(food.emoji, food.x * gridSize + 2, food.y * gridSize + 14);
      }

      // Всплывающие очки
      popups.forEach((p, i) => {
        ctx.fillStyle = `rgba(${p.color === "yellow" ? "255,255,0" : "0,255,0"}, ${p.alpha})`;
        ctx.font = "bold 16px Arial";
        ctx.fillText(p.text, p.x, p.y);
        p.y += p.dy;
        p.alpha -= 0.02;
        if (p.alpha <= 0) popups.splice(i, 1);
      });

      // Эффект поедания
      eatEffects.forEach((e, i) => {
        ctx.fillStyle = `rgba(255, 255, 0, ${e.alpha})`;
        ctx.beginPath();
        ctx.arc(e.x, e.y, e.r, 0, Math.PI * 2);
        ctx.fill();
        e.x += e.dx;
        e.y += e.dy;
        e.alpha -= 0.03;
        if (e.alpha <= 0) eatEffects.splice(i, 1);
      });

      // Анимация уровня
      if (celebration) {
        ctx.fillStyle = `rgba(255, 215, 0, ${celebration.alpha})`;
        ctx.font = "bold 24px Arial";
        const w = ctx.measureText(celebration.text).width;
        ctx.fillText(celebration.text, (canvas.width - w) / 2, 150);
        if (celebration.alpha > 0) celebration.alpha -= 0.01;
      }
    }

    function setDir(x, y) {
      if (!gameStarted) {
        gameStarted = true;
        gameRunning = true;
        clearInterval(gameInterval);
        gameInterval = setInterval(gameLoop, 150);
      }

      if (x !== 0 && dx === 0) { dx = x; dy = 0; }
      if (y !== 0 && dy === 0) { dy = y; dx = 0; }
    }

    function gameLoop() {
      if (!gameRunning) return;

      let head = { x: snake[0].x + dx, y: snake[0].y + dy };

      // Проход сквозь стену
      if (head.x < 0) head.x = tileCount - 1;
      if (head.x >= tileCount) head.x = 0;
      if (head.y < 0) head.y = tileCount - 1;
      if (head.y >= tileCount) head.y = 0;

      // Самопересечение
      for (let part of snake) {
        if (part.x === head.x && part.y === head.y) {
          gameOver();
          return;
        }
      }

      snake.unshift(head);

      if (head.x === food.x && head.y === food.y) {
        const points = food.points;
        score += points;
        scoreDisplay.textContent = `Очки: ${score}`;
        playSound();
        addPopup(head.x, head.y, `+${points}`, points >= 20 ? "yellow" : "green");
        addEatEffect(head.x, head.y);
        placeFood();

        // Анимация каждые 100 очков
        if (score % 100 === 0 && score <= 1000) {
          startCelebration();
        }
      } else {
        snake.pop();
      }

      draw();
    }

    function gameOver() {
      gameRunning = false;
      clearInterval(gameInterval);

      if (score > bestScore) {
        bestScore = score;
        localStorage.setItem("spaceSnakeBest", bestScore);
        bestDisplay.textContent = `Рекорд: ${bestScore} 🌟`;
      }

      drawSpace();
      ctx.fillStyle = "rgba(0,0,0,0.8)";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "red";
      ctx.font = "30px Arial";
      ctx.fillText("МИССИЯ", 80, 130);
      ctx.fillText("ЗАВЕРШЕНА", 60, 170);
      ctx.fillStyle = "white";
      ctx.font = "20px Arial";
      ctx.fillText(`Очки: ${score}`, 100, 210);
    }

    function restart() {
      clearInterval(gameInterval);
      snake = [{ x: 10, y: 10 }];
      dx = 0; dy = 0;
      score = 0;
      scoreDisplay.textContent = "Очки: 0";
      gameRunning = false;
      gameStarted = false;
      popups = [];
      eatEffects = [];
      celebration = null;
      placeFood();
      draw();
      showStartAnimation();
    }

    // Старт
    window.addEventListener('load', () => {
      placeFood();
      draw();
      showStartAnimation();
    });
  </script>
</body>
</html>
