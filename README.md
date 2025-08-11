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
  const gridSize = 15;        // Размер одной клетки
  const tileCount = 20;
  const moveSpeed = 5;        // Пикселей за шаг (для плавности)
  const smoothStep = 0.1;     // Скорость интерполяции

  // Звёзды
  const stars = [];
  for (let i = 0; i < 60; i++) {
    stars.push({
      x: Math.random() * canvas.width,
      y: Math.random() * canvas.height,
      r: Math.random() * 1.8,
      a: Math.random()
    });
  }

  // Фрукты — с разными очками и весами
  const FRUITS = [
    { emoji: "🍒", points: 5,  weight: 10 },
    { emoji: "🍎", points: 10, weight: 8 },
    { emoji: "🍊", points: 12, weight: 7 },
    { emoji: "🍋", points: 8,  weight: 6 },
    { emoji: "🍌", points: 15, weight: 5 },
    { emoji: "🥝", points: 10, weight: 6 },
    { emoji: "🍇", points: 14, weight: 5 },
    { emoji: "🍓", points: 11, weight: 6 },
    { emoji: "🫐", points: 9,  weight: 5 },
    { emoji: "🍍", points: 18, weight: 3 },
    { emoji: "🥭", points: 20, weight: 2 },
    { emoji: "🍉", points: 25, weight: 2 },
    { emoji: "🍈", points: 22, weight: 2 },
    { emoji: "🍑", points: 13, weight: 4 },
    { emoji: "🫒", points: 30, weight: 1 }, // Оливка — редкая и ценная
    { emoji: "🥑", points: 35, weight: 1 },
    { emoji: "🌽", points: 40, weight: 1 }, // Кукуруза — шутка, но редкая
    { emoji: "🥕", points: 50, weight: 1 }  // Морковка — суперредкая
  ];

  // Состояние
  let snake = [{ x: 10, y: 10 }];
  let snakePos = { x: 10 * gridSize, y: 10 * gridSize }; // Плавная позиция
  let dx = 0, dy = 0;
  let nextDx = 0, nextDy = 0; // Буфер направления
  let score = 0;
  let bestScore = parseInt(localStorage.getItem("spaceSnakeBest")) || 0;
  let food = null;
  let gameRunning = false;
  let gameStarted = false;
  let gameInterval;
  let popups = [];
  let eatEffects = [];
  let celebration = null;

  // Рекорд
  bestDisplay.textContent = `Рекорд: ${bestScore}`;

  // Звук
  function playSound(freq = 300) {
    try {
      const ctx = new (window.AudioContext || window.webkitAudioContext)();
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      osc.connect(gain);
      gain.connect(ctx.destination);
      osc.frequency.setValueAtTime(freq, ctx.currentTime);
      gain.gain.setValueAtTime(0.1, ctx.currentTime);
      gain.gain.exponentialRampToValueAtTime(0.01, ctx.currentTime + 0.2);
      osc.start();
      osc.stop(ctx.currentTime + 0.2);
    } catch (e) {}
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
    for (let i = 0; i < 10; i++) {
      eatEffects.push({
        x: x * gridSize + gridSize / 2,
        y: y * gridSize + gridSize / 2,
        dx: (Math.random() - 0.5) * 8,
        dy: (Math.random() - 0.5) * 8,
        alpha: 1,
        r: Math.random() * 2 + 1
      });
    }
  }

  // Анимация уровня
  function startCelebration() {
    celebration = { text: "🚀 LEVEL UP!", alpha: 1 };
    setTimeout(() => { if (celebration) celebration.alpha = 0; }, 1500);
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
    const texts = ["включаем двигатели...", "выход в эфир...", "змейка-корабль, сосал?!"];
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
    food.gridX = Math.floor(Math.random() * tileCount);
    food.gridY = Math.floor(Math.random() * tileCount);
    
    // Не на змейке
    for (let part of snake) {
      if (part.x === food.gridX && part.y === food.gridY) {
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

    // Плавное отображение змейки
    for (let i = 0; i < snake.length; i++) {
      const part = snake[i];
      const x = part.x * gridSize;
      const y = part.y * gridSize;

      ctx.fillStyle = i === 0 ? "#00ff44" : "#00cc22";
      ctx.beginPath();
      ctx.arc(x + gridSize/2, y + gridSize/2, gridSize/2 - 1, 0, Math.PI * 2);
      ctx.fill();

      // Глаза у головы
      if (i === 0) {
        ctx.fillStyle = "white";
        const eyeSize = 2;
        const eyeX = dx === 1 ? x + gridSize - 5 : dx === -1 ? x + 5 : x + 7;
        const eyeY = dy === 1 ? y + gridSize - 5 : dy === -1 ? y + 5 : y + 7;
        ctx.beginPath();
        ctx.arc(eyeX, eyeY, eyeSize, 0, Math.PI * 2);
        ctx.fill();
        ctx.fillStyle = "black";
        ctx.beginPath();
        ctx.arc(eyeX + (dx || 0), eyeY + (dy || 0), 0.8, 0, Math.PI * 2);
        ctx.fill();
      }
    }

    // Фрукт — ровно в клетке
    if (food) {
      ctx.font = `${gridSize}px Arial`;
      ctx.textAlign = "center";
      ctx.textBaseline = "middle";
      ctx.fillText(
        food.emoji,
        food.gridX * gridSize + gridSize / 2,
        food.gridY * gridSize + gridSize / 2
      );
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

    // Эффекты
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
      gameInterval = setInterval(gameLoop, 80); // Плавная частота
    }
    // Буфер направления
    if (x !== 0 && dx === 0) { nextDx = x; nextDy = 0; }
    if (y !== 0 && dy === 0) { nextDy = y; nextDx = 0; }
  }

  function gameLoop() {
    if (!gameRunning) return;

    // Применяем новое направление
    if (nextDx !== 0 || nextDy !== 0) {
      dx = nextDx;
      dy = nextDy;
      nextDx = 0;
      nextDy = 0;
    }

    // Плавное движение
    snakePos.x += dx * moveSpeed;
    snakePos.y += dy * moveSpeed;

    // Проверка, когда сдвиг на целую клетку
    if (Math.abs(snakePos.x - snake[0].x * gridSize) >= gridSize) {
      const head = { x: snake[0].x + dx, y: snake[0].y + dy };

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
      snakePos.x = head.x * gridSize;
      snakePos.y = head.y * gridSize;

      if (head.x === food.gridX && head.y === food.gridY) {
        const points = food.points;
        score += points;
        scoreDisplay.textContent = `Очки: ${score}`;
        playSound(200 + points * 2);
        addPopup(head.x, head.y, `+${points}`, points > 20 ? "yellow" : "green");
        addEatEffect(head.x, head.y);
        placeFood();

        if (score % 100 === 0 && score <= 1000) {
          startCelebration();
        }
      } else {
        snake.pop();
      }
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
    snakePos = { x: 10 * gridSize, y: 10 * gridSize };
    dx = 0; dy = 0; nextDx = 0; nextDy = 0;
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

  window.addEventListener('load', () => {
    placeFood();
    draw();
    showStartAnimation();
  });
</scrстартт
