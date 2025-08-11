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

  // Всплывающие очки
  let popups = [];

  // Звуки (в base64 — короткие WAV)
  const eatSound = new Audio("data:audio/wav;base64,UklGRiQAAABXQVZFZm10IBAAAAABAAEARKwAAIhYAQACABAAZGF0YQAAAAA=");
  const dieSound = new Audio("data:audio/wav;base64,UklGRiQAAABXQVZFZm10IBAAAAABAAEARKwAAIhYAQACABAAZGF0YQAAAAI=");
  const startSound = new Audio("data:audio/wav;base64,UklGRiQAAABXQVZFZm10IBAAAAABAAEARKwAAIhYAQACABAAZGF0YQAAABA=");

  // Уменьшим громкость
  eatSound.volume = 0.4;
  dieSound.volume = 0.5;
  startSound.volume = 0.6;

  // Рекорд
  bestDisplay.textContent = `Рекорд: ${bestScore}`;

  // Всплывающий текст
  function addPopup(x, y, text, color = "white") {
    popups.push({
      x: x * gridSize + gridSize / 2,
      y: y * gridSize,
      text: text,
      color: color,
      alpha: 1,
      dy: -1.5  // движение вверх
    });
  }

  // Выбор фрукта с весом
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

    // Всплывающие очки
    popups.forEach((p, i) => {
      ctx.fillStyle = `rgba(${p.color === "yellow" ? "255,255,0" : "255,255,255"}, ${p.alpha})`;
      ctx.font = "bold 16px Arial";
      ctx.fillText(p.text, p.x, p.y);
      p.y += p.dy;
      p.alpha -= 0.03;
      if (p.alpha <= 0) popups.splice(i, 1);
    });
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
      const points = food.points;
      score += points;
      scoreDisplay.textContent = `Очки: ${score}`;
      eatSound.play();
      addPopup(head.x, head.y, `+${points}`, points === 20 ? "yellow" : "white");
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
    popups = [];
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
