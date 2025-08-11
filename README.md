<!DOCTYPE html>
<html>
<head>
  <title>Змейка в Telegram</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
  body {
    margin: 0;
    padding: 10px;
    text-align: center;
    font-family: sans-serif;
    background: #111;
    color: white;
  }
  canvas {
    background: #000;
    border: 2px solid #333;
    display: block;
    margin: 20px auto;
  }
  #score {
    font-size: 24px;
    font-weight: bold;
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
  <h1>🐍 Змейка в Telegram</h1>
  <div id="score">Очки: 0</div>
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
    // Подключаем Telegram WebApp
    const tg = window.Telegram?.WebApp;
    if (tg) {
      tg.ready(); // Готовим интерфейс
      tg.expand(); // Раскрываем на весь экран
    }

    const canvas = document.getElementById("game");
    const ctx = canvas.getContext("2d");
    const scoreDisplay = document.getElementById("score");
    
    const gridSize = 15;
    const tileCount = 20;
    let score = 0;
    let snake = [{ x: 10, y: 10 }];
    let dx = 0;
    let dy = 0;
    let food = { x: 5, y: 5 };
    let gameRunning = true;
    let gameInterval;

    function setDir(x, y) {
      if (x !== 0 && dx === 0) { dx = x; dy = 0; }
      if (y !== 0 && dy === 0) { dy = y; dx = 0; }
    }

    function startGame() {
      clearInterval(gameInterval);
      snake = [{ x: 10, y: 10 }];
      dx = 0; dy = 0;
      score = 0;
      scoreDisplay.textContent = "Очки: 0";
      gameRunning = true;
      placeFood();
      draw();
      gameInterval = setInterval(gameLoop, 150);
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
        score += 10;
        scoreDisplay.textContent = `Очки: ${score}`;
        placeFood();
      } else {
        snake.pop();
      }

      draw();
    }

    function draw() {
      ctx.fillStyle = "black";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "lime";
      for (let part of snake) {
        ctx.fillRect(part.x * gridSize, part.y * gridSize, gridSize - 2, gridSize - 2);
      }
      ctx.fillStyle = "red";
      ctx.fillRect(food.x * gridSize, food.y * gridSize, gridSize - 2, gridSize - 2);
    }

    function placeFood() {
      food = {
        x: Math.floor(Math.random() * tileCount),
        y: Math.floor(Math.random() * tileCount)
      };
      for (let part of snake) {
        if (part.x === food.x && part.y === food.y) placeFood();
      }
    }

    function gameOver() {
      gameRunning = false;
      clearInterval(gameInterval);
      ctx.fillStyle = "rgba(0,0,0,0.8)";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "white";
      ctx.font = "30px Arial";
      ctx.fillText("ПОТРАЧЕНО!", 80, 150);
      ctx.font = "20px Arial";
      ctx.fillText("Нажми 'Начать'", 80, 190);

      // Можно отправить очки в бота
      if (tg) {
        tg.sendData(JSON.stringify({ type: "game_over", score: score }));
      }
    }

    function restart() {
      gameRunning = true;
      startGame();
    }

    window.onload = startGame;
  </script>
</body>
</html>
