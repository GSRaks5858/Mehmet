<!DOCTYPE html>
<html>
<head>
  <title>🐍 Yılandan Kaçış</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { font-family: Arial, sans-serif; background: #222; color: #fff; text-align: center; }
    canvas { background: #333; display: block; margin: 20px auto; border: 3px solid #fff; }
    #hud { margin-top: 10px; font-size: 18px; }

    #gameOver, #gameWin {
      font-size: 28px; color: yellow; display: none;
      margin-top: 10px;
    }
    #restartBtn, #continueBtn {
      padding: 10px 20px;
      font-size: 18px;
      margin-top: 15px;
      cursor: pointer;
      display: none;
      border: none;
      border-radius: 6px;
      background-color: #4CAF50;
      color: white;
      transition: background-color 0.3s ease;
    }
    #restartBtn:hover, #continueBtn:hover {
      background-color: #45a049;
    }

    #menu {
      margin-top: 40px;
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 15px;
    }
    #menu button {
      width: 200px;
      padding: 12px 0;
      font-size: 20px;
      cursor: pointer;
      border: none;
      border-radius: 6px;
      background-color: #4CAF50;
      color: white;
      transition: background-color 0.3s ease;
    }
    #menu button:hover {
      background-color: #45a049;
    }

    #gameCanvas, #hud, #gameOver, #gameWin, #restartBtn, #continueBtn {
      display: none;
    }
  </style>
</head>
<body>
  <h1>🐍 Yılandan Kaçış</h1>

  <!-- Menü -->
  <div id="menu">
    <button id="startBtn">▶️ Oyuna Başla</button>
    <button id="exitBtn">❌ Oyundan Çık</button>
  </div>

  <canvas id="gameCanvas" width="600" height="400"></canvas>
  <div id="hud">
    🧃 Skor: <span id="score">0</span> |
    ❤️ Can: <span id="lives">3</span> |
    ⏱️ Süre: <span id="timer">60</span> |
    🔢 Seviye: <span id="level">1</span> |
    🩸 Can Objeleri: <span id="heartsCount">0</span>
  </div>
  <div id="gameOver">💀 OYUN BİTTİ! Yılanlara yem oldun!</div>
  <div id="gameWin">🏆 OYUNU KAZANDIN! Helal sana gardaş!</div>
  <button id="restartBtn">🔁 Tekrar Oyna</button>
  <button id="continueBtn">▶️ Sonraki Seviyeye Geç</button>

  <audio id="eatSound" src="https://www.soundjay.com/button/beep-07.wav"></audio>
  <audio id="hitSound" src="https://www.soundjay.com/button/beep-10.wav"></audio>
  <audio id="bombSound" src="https://www.soundjay.com/explosion/sounds/explosion-01.mp3"></audio>

  <script>
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");

    // Dino (player)
    const dino = { x: 50, y: 180, size: 30, speed: 3 };

    // Game variables
    let apples = [{ x: 300, y: 200 }];       // Elmalar (skor için)
    let hearts = [];                          // Can objeleri (can artırır)
    let bombs = [];                           // Bossun bombaları
    let snakes = [{ x: 600, y: 100, speed: dino.speed * 0.8 }];  // Düşman yılanlar
    let boss = null;                         // Boss objesi

    let score = 0, lives = 3, timeLeft = 60;
    let currentLevel = 1, maxLevel = 5;
    let gameOver = false, gameWin = false;
    let keys = {};
    let lastTime = 0;
    let timerInterval;

    const eatSound = document.getElementById("eatSound");
    const hitSound = document.getElementById("hitSound");
    const bombSound = document.getElementById("bombSound");

    // HTML element refs
    const scoreSpan = document.getElementById("score");
    const livesSpan = document.getElementById("lives");
    const timerSpan = document.getElementById("timer");
    const levelSpan = document.getElementById("level");
    const heartsCountSpan = document.getElementById("heartsCount");

    const menuDiv = document.getElementById("menu");
    const startBtn = document.getElementById("startBtn");
    const exitBtn = document.getElementById("exitBtn");
    const hud = document.getElementById("hud");
    const gameOverDiv = document.getElementById("gameOver");
    const gameWinDiv = document.getElementById("gameWin");
    const restartBtn = document.getElementById("restartBtn");
    const continueBtn = document.getElementById("continueBtn");

    document.addEventListener("keydown", (e) => keys[e.key.toLowerCase()] = true);
    document.addEventListener("keyup", (e) => keys[e.key.toLowerCase()] = false);

    startBtn.onclick = () => {
      menuDiv.style.display = "none";
      canvas.style.display = "block";
      hud.style.display = "block";
      restartBtn.style.display = "none";
      continueBtn.style.display = "none";
      gameOverDiv.style.display = "none";
      gameWinDiv.style.display = "none";

      resetGame(currentLevel);
      lastTime = performance.now();
      requestAnimationFrame(gameLoop);

      startTimer();
    };

    exitBtn.onclick = () => {
      if (confirm("Oyundan çıkmak istediğine emin misin?")) {
        window.close();
      }
    };

    restartBtn.onclick = () => {
      menuDiv.style.display = "block";
      canvas.style.display = "none";
      hud.style.display = "none";
      restartBtn.style.display = "none";
      continueBtn.style.display = "none";
      gameOverDiv.style.display = "none";
      gameWinDiv.style.display = "none";
      clearInterval(timerInterval);
      currentLevel = 1;
    };

    continueBtn.onclick = () => {
      continueBtn.style.display = "none";
      gameWinDiv.style.display = "none";
      if (currentLevel < maxLevel) {
        currentLevel++;
        resetGame(currentLevel);
        lastTime = performance.now();
        requestAnimationFrame(gameLoop);
        startTimer();
      } else {
        // Son seviyeden sonra oyun kazanıldı sayılır
        gameWin = true;
        endGame(true);
      }
    };

    function startTimer() {
      clearInterval(timerInterval);
      timeLeft = 60;
      timerSpan.textContent = timeLeft;
      // Zamanlayıcı + 20. ve 40. saniyelerde can objesi çıkacak
      timerInterval = setInterval(() => {
        if (gameOver || gameWin) {
          clearInterval(timerInterval);
          return;
        }
        timeLeft--;
        timerSpan.textContent = timeLeft;

        if (timeLeft === 40 || timeLeft === 20) {
          spawnHeart();
        }

        if (timeLeft <= 0) {
          clearInterval(timerInterval);
          if (currentLevel === maxLevel) {
            gameWin = true;
            endGame(true);
          } else {
            // Seviye tamamlandı, devam butonu göster
            continueBtn.style.display = "inline-block";
            gameWinDiv.textContent = `🎉 Seviye ${currentLevel} tamamlandı! Sonraki seviyeye geçmek için devam et.`;
            gameWinDiv.style.display = "block";
          }
        }
      }, 1000);
    }

    function resetGame(level) {
      dino.x = 50;
      dino.y = 180;
      score = 0;
      lives = 3;
      timeLeft = 60;
      apples = [randomItem()];
      hearts = [];
      bombs = [];
      snakes = [{ x: 600, y: 100, speed: dino.speed * 0.8 }];

      boss = null;
      if (level === maxLevel) {
        // 5. seviyede boss oluştur
        boss = {
          x: 500,
          y: 150,
          width: 45,
          height: 60, // biraz uzun
          health: 10,
          speed: 1.2,
          bombCooldown: 0
        };
        snakes = []; // normal yılanlar yok boss var
      }

      keys = {};
      gameOver = false;
      gameWin = false;

      scoreSpan.textContent = score;
      livesSpan.textContent = lives;
      timerSpan.textContent = timeLeft;
      levelSpan.textContent = level;
      heartsCountSpan.textContent = hearts.length;
    }

    function spawnHeart() {
      hearts.push(randomItem());
      heartsCountSpan.textContent = hearts.length;
    }

    function spawnBomb() {
      if (!boss) return;
      // Boss rastgele pozisyonundan bomba fırlatır
      bombs.push({
        x: boss.x + boss.width / 2,
        y: boss.y + boss.height,
        size: 20,
        speed: 3
      });
    }

    function update(deltaTime) {
      if (gameOver || gameWin) return;

      // Dino hareket
      if (keys['w']) dino.y -= dino.speed * deltaTime;
      if (keys['s']) dino.y += dino.speed * deltaTime;
      if (keys['a']) dino.x -= dino.speed * deltaTime;
      if (keys['d']) dino.x += dino.speed * deltaTime;

      dino.x = Math.max(0, Math.min(canvas.width - dino.size, dino.x));
      dino.y = Math.max(0, Math.min(canvas.height - dino.size, dino.y));

      // Elmalar için skor alma
      for (let i = 0; i < apples.length; i++) {
        if (isCollidingRect(dino, { x: apples[i].x, y: apples[i].y, size: 30 })) {
          score++;
          eatSound.play();
          apples.splice(i, 1);
          apples.push(randomItem());
          scoreSpan.textContent = score;

          if (score % 10 === 0 && currentLevel < maxLevel) {
            // Yeni yılan ekle
            snakes.push({ x: Math.random() * 500 + 50, y: Math.random() * 300 + 50, speed: dino.speed * 0.8 });
          }
          break;
        }
      }

      // Can objelerini toplama (❤️ artar)
      for (let i = 0; i < hearts.length; i++) {
        if (isCollidingRect(dino, { x: hearts[i].x, y: hearts[i].y, size: 30 })) {
          lives++;
          eatSound.play();
          hearts.splice(i, 1);
          heartsCountSpan.textContent = hearts.length;
          livesSpan.textContent = lives;
          break;
        }
      }

      // Yılanlar hareket ve çarpma kontrolü
      snakes.forEach(snake => {
        const dx = dino.x - snake.x;
        const dy = dino.y - snake.y;
        const dist = Math.sqrt(dx * dx + dy * dy);
        if (dist > 0) {
          snake.x += (dx / dist) * snake.speed * deltaTime;
          snake.y += (dy / dist) * snake.speed * deltaTime;
        }

        if (isCollidingRect(dino, { x: snake.x, y: snake.y, size: 30 })) {
          hitSound.play();
          lives--;
          livesSpan.textContent = lives;
          dino.x = 50;
          dino.y = 180;
          if (lives <= 0) {
            gameOver = true;
            endGame(false);
          }
        }
      });

      // Boss hareket ve bomba saldırısı
      if (boss) {
        // Basit yapay zeka: Dinoya yaklaşır
        const dx = dino.x - boss.x;
        const dy = dino.y - boss.y;
        const dist = Math.sqrt(dx * dx + dy * dy);
        if (dist > 0) {
          boss.x += (dx / dist) * boss.speed * deltaTime;
          boss.y += (dy / dist) * boss.speed * deltaTime;
          // Canvas sınırları içinde kal
          boss.x = Math.max(0, Math.min(canvas.width - boss.width, boss.x));
          boss.y = Math.max(0, Math.min(canvas.height - boss.height, boss.y));
        }

        // Bombaları rastgele belli aralıklarla fırlatır
        boss.bombCooldown -= deltaTime;
        if (boss.bombCooldown <= 0) {
          spawnBomb();
          boss.bombCooldown = 120; // cooldown 120 frame civarı (~2 saniye)
        }
      }

      // Bombalar hareket eder, yılanla çarpışma kontrolü
      for (let i = bombs.length -1; i >= 0; i--) {
        bombs[i].y += bombs[i].speed * deltaTime;
        // Bombalar yılan ile değil, dinoya çarparsa patlamaz, bossun bomba yemesi isteniyor
        // Bu yüzden bombalar sadece dino ve canvas dışına çıkınca silinir

        // Dino ile çarpışma kontrolü (yok, bomba zarar vermiyor dinoya)

        // Canvas dışına çıktıysa sil
        if (bombs[i].y > canvas.height) {
          bombs.splice(i,1);
        }
      }

      // Dino ile boss bomba çarpışması kontrolü
      // Bu mantıkla bomba boss'a çarptığında canı azalır
      for (let i = bombs.length -1; i >=0; i--) {
        if (boss && isCollidingRect(
          { x: bombs[i].x, y: bombs[i].y, size: bombs[i].size },
          { x: boss.x, y: boss.y, width: boss.width, height: boss.height }
          )) {
          boss.health--;
          bombSound.play();
          bombs.splice(i,1);
          if (boss.health <= 0) {
            gameWin = true;
            endGame(true);
          }
        }
      }
    }

    function draw() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);

      // Dino çiz
      ctx.fillStyle = "lightgreen";
      ctx.font = "28px serif";
      ctx.fillText("🦖", dino.x, dino.y + 25);

      // Elmalar (skor)
      apples.forEach(a => ctx.fillText("🍎", a.x, a.y + 25));

      // Can objeleri
      hearts.forEach(h => ctx.fillText("🩸", h.x, h.y + 25));

      // Yılanlar
      ctx.fillStyle = "red";
      ctx.font = "28px serif";
      snakes.forEach(snake => {
        ctx.fillText("🐍", snake.x, snake.y + 25);
      });

      // Boss
      if (boss) {
        ctx.fillStyle = "purple";
        ctx.font = "42px serif";
        ctx.fillText("🐉", boss.x, boss.y + 40); // boss daha büyük ama abartı değil
        // Boss can barı
        ctx.fillStyle = "red";
        ctx.fillRect(boss.x, boss.y - 10, (boss.health / 10) * boss.width, 5);
      }

      // Bombalar
      ctx.fillStyle = "black";
      ctx.font = "30px serif";
      bombs.forEach(b => ctx.fillText("💣", b.x, b.y + 25));
    }

    function gameLoop(timestamp) {
      if (!lastTime) lastTime = timestamp;
      const deltaTime = (timestamp - lastTime) / 16; // normalize for ~60fps
      lastTime = timestamp;

      update(deltaTime);
      draw();

      if (!gameOver && !gameWin) {
        requestAnimationFrame(gameLoop);
      }
    }

    function endGame(win) {
      clearInterval(timerInterval);
      if (win) {
        canvas.style.display = "none";
        hud.style.display = "none";
        continueBtn.style.display = "none";
        gameWinDiv.style.display = "block";
        restartBtn.style.display = "inline-block";
      } else {
        canvas.style.display = "none";
        hud.style.display = "none";
        gameOverDiv.style.display = "block";
        restartBtn.style.display = "inline-block";
        continueBtn.style.display = "none";
      }
    }

    // Basit dikdörtgen çarpışma kontrolü
    function isCollidingRect(a, b) {
      const aWidth = a.size || a.width || 30;
      const aHeight = a.size || a.height || 30;
      const bWidth = b.size || b.width || 30;
      const bHeight = b.size || b.height || 30;

      return (
        a.x < b.x + bWidth &&
        a.x + aWidth > b.x &&
        a.y < b.y + bHeight &&
        a.y + aHeight > b.y
      );
    }

    // Rastgele pozisyon döndürür (canvas sınırları içinde)
    function randomItem() {
      const padding = 40;
      return {
        x: Math.random() * (canvas.width - padding * 2) + padding,
        y: Math.random() * (canvas.height - padding * 2) + padding,
        size: 30
      };
    }
  </script>
</body>
</html>
