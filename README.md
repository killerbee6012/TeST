<!DOCTYPE html>
<html>
<head>
  <title>Jump & Slide Game</title>
  <style>
    body {
      margin: 0; padding: 0;
      display: flex; justify-content: center; align-items: center;
      height: 100vh; background: #333;
      font-family: Arial, sans-serif;
      overflow: hidden;
    }
    #gameContainer {
      position: relative;
      width: 800px; height: 400px;
      border: 4px solid #555;
      background: linear-gradient(to bottom, #87CEEB 60%, #4CAF50 40%);
      overflow: hidden;
    }
    #score {
      position: absolute;
      top: 10px; left: 10px;
      color: white; font-size: 24px;
      z-index: 10;
      text-shadow: 1px 1px 3px black;
    }
    #gameOver {
      position: absolute;
      top: 50%; left: 50%;
      transform: translate(-50%, -50%);
      font-size: 48px; color: red;
      display: none;
      z-index: 20;
      text-shadow: 2px 2px 5px black;
    }
    #startButton {
      position: absolute;
      top: 50%; left: 50%;
      transform: translate(-50%, -50%);
      padding: 15px 30px;
      font-size: 24px;
      background: #FF5722; color: white;
      border: none; border-radius: 5px;
      cursor: pointer;
      z-index: 20;
    }
    #startButton:hover {
      background: #E64A19;
    }
    canvas {
      display: block;
      background: transparent;
    }
  </style>
</head>
<body>
  <div id="gameContainer">
    <div id="score">Score: 0</div>
    <div id="gameOver">Game Over!</div>
    <button id="startButton">Spiel Starten</button>
    <canvas id="gameCanvas" width="800" height="400"></canvas>
  </div>

  <script src="https://cdnjs.cloudflare.com/ajax/libs/matter-js/0.18.0/matter.min.js"></script>
  <script>
    // Setup Matter.js
    const { Engine, Render, Runner, Bodies, Composite, Body, Events, Vector } = Matter;

    // Engine & world
    const engine = Engine.create();
    const world = engine.world;
    engine.gravity.y = 1.2;

    // Canvas and context
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');

    // Game state
    let gameActive = false;
    let score = 0;
    let scrollSpeed = 4;

    // Player properties
    const playerWidth = 30;
    const playerHeight = 50;
    const slideHeight = 25;

    // Bodies
    let player;
    let ground;
    let obstacles = [];

    // Flags
    let isJumping = false;
    let isSliding = false;
    let slideFrames = 0;
    const maxSlideFrames = 30;

    // HTML Elements
    const scoreEl = document.getElementById('score');
    const gameOverEl = document.getElementById('gameOver');
    const startButton = document.getElementById('startButton');

    // Create ground
    function createGround() {
      ground = Bodies.rectangle(400, 400 - 20, 800, 40, { isStatic: true, label: 'ground' });
      Composite.add(world, ground);
    }

    // Create player
    function createPlayer() {
      player = Bodies.rectangle(100, 400 - 20 - playerHeight/2, playerWidth, playerHeight, {
        label: 'player',
        friction: 0.001,
        restitution: 0,
      });
      Composite.add(world, player);
    }

    // Create obstacles
    function createObstacle() {
      // Zufällig zwischen "hoch" (pipe) und "niedrig" (block)
      let obstacle;
      const x = 850;
      if (Math.random() < 0.5) {
        // Pipe - hoch, zwingt zum Sliden
        const height = 100; // hoch
        obstacle = Bodies.rectangle(x, 400 - 20 - height/2, 50, height, { isStatic: true, label: 'obstacle' });
        obstacle.obstacleType = 'high';
      } else {
        // Block - niedrig, kann übersprungen werden
        const height = 40;
        obstacle = Bodies.rectangle(x, 400 - 20 - height/2, 40, height, { isStatic: true, label: 'obstacle' });
        obstacle.obstacleType = 'low';
      }
      obstacles.push(obstacle);
      Composite.add(world, obstacle);
    }

    // Reset game
    function resetGame() {
      Composite.clear(world, false);
      obstacles = [];
      isJumping = false;
      isSliding = false;
      slideFrames = 0;
      score = 0;
      scrollSpeed = 4;
      gameOverEl.style.display = 'none';

      createGround();
      createPlayer();
    }

    // Draw functions
    function drawPlayer() {
      ctx.save();
      ctx.translate(player.position.x, player.position.y);
      ctx.fillStyle = 'red';

      if (isSliding) {
        ctx.fillRect(-playerWidth/2, -slideHeight/2, playerWidth, slideHeight);
      } else {
        ctx.fillRect(-playerWidth/2, -playerHeight/2, playerWidth, playerHeight);
      }

      // Kopf
      ctx.fillStyle = 'pink';
      ctx.beginPath();
      ctx.arc(0, isSliding ? -slideHeight/2 : -playerHeight/2 - 5, playerWidth/3, 0, Math.PI * 2);
      ctx.fill();

      ctx.restore();
    }

    function drawGround() {
      ctx.fillStyle = '#4CAF50';
      ctx.fillRect(0, 400 - 40, 800, 40);
    }

    function drawObstacles() {
      ctx.fillStyle = '#654321';
      obstacles.forEach(obstacle => {
        ctx.save();
        ctx.translate(obstacle.position.x, obstacle.position.y);
        ctx.fillRect(-obstacle.bounds.max.x + obstacle.position.x, -obstacle.bounds.max.y + obstacle.position.y,
                     obstacle.bounds.max.x - obstacle.bounds.min.x,
                     obstacle.bounds.max.y - obstacle.bounds.min.y);
        ctx.restore();
      });
    }

    // Game Loop
    function gameLoop() {
      if (!gameActive) return;

      ctx.clearRect(0, 0, canvas.width, canvas.height);

      drawGround();
      drawPlayer();
      drawObstacles();

      // Move obstacles
      obstacles.forEach(obstacle => {
        Body.translate(obstacle, { x: -scrollSpeed, y: 0 });
      });

      // Entferne Hindernisse, die links raus sind
      obstacles = obstacles.filter(obstacle => {
        if (obstacle.position.x < -50) {
          Composite.remove(world, obstacle);
          return false;
        }
        return true;
      });

      // Slide-Frames runterzählen
      if (isSliding) {
        slideFrames--;
        if (slideFrames <= 0) {
          endSlide();
        }
        // Spieler auf Boden fixieren
        Body.setPosition(player, {x: player.position.x, y: 400 - 20 - slideHeight/2});
        Body.setVelocity(player, {x: 0, y: 0});
      }

      // Prüfe, ob Spieler auf Boden (kleiner Abstand)
      if (player.position.y > 400 - 20 - playerHeight/2 - 1) {
        isJumping = false;
      }

      // Kollisionsprüfung mit Hindernissen
      for (const obstacle of obstacles) {
        if (Matter.SAT.collides(player, obstacle).collided) {
          endGame();
          return;
        }
      }

      // Score erhöhen
      score += 0.1;
      scoreEl.textContent = 'Score: ' + Math.floor(score);

      requestAnimationFrame(gameLoop);
    }

    // Slide starten & beenden
    function startSlide() {
      if (isJumping || isSliding) return;
      isSliding = true;
      slideFrames = maxSlideFrames;

      // Player Größe halbieren (nur Höhe)
      Body.scale(player, 1, slideHeight / playerHeight);
      // Spieler Position auf Boden fixieren
      Body.setPosition(player, {x: player.position.x, y: 400 - 20 - slideHeight/2});
      Body.setVelocity(player, {x: 0, y: 0});
    }

    function endSlide() {
      if (!isSliding) return;
      isSliding = false;

      // Player Größe zurücksetzen
      Body.scale(player, 1, playerHeight / slideHeight);
    }

    // Game Over
    function endGame() {
      gameActive = false;
      gameOverEl.style.display = 'block';
      startButton.style.display = 'block';
    }

    // Input
    document.addEventListener('keydown', e => {
      if (!gameActive) return;

      if (e.code === 'Space') {
        if (!isJumping && !isSliding) {
          // Springen
          Body.applyForce(player, player.position, {x: 0, y: -0.15});
          isJumping = true;
        }
      }

      if (e.code === 'ShiftLeft' || e.code === 'ShiftRight') {
        startSlide();
      }
    });

    document.addEventListener('keyup', e => {
      if (!gameActive) return;

      if (e.code === 'ShiftLeft' || e.code === 'ShiftRight') {
        endSlide();
      }
    });

    // Start Button
    startButton.addEventListener('click', () => {
      resetGame();
      gameActive = true;
      startButton.style.display = 'none';
      gameOverEl.style.display = 'none';

      Runner.run(engine);
      gameLoop();

      // Hindernisse regelmäßig erzeugen
      obstacleInterval = setInterval(() => {
        if (gameActive) createObstacle();
      }, 1800);
    });

  </script>
</body>
</html>
