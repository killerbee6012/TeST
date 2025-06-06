<!DOCTYPE html>
<html>
<head>
    <title>Mario-ähnliches Laufspiel (Slide + Jump)</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            background: #333;
            font-family: Arial, sans-serif;
            overflow: hidden;
        }
        #gameContainer {
            position: relative;
            width: 800px;
            height: 400px;
            border: 4px solid #555;
            box-shadow: 0 0 20px rgba(0, 0, 0, 0.7);
            overflow: hidden;
        }
        #gameCanvas {
            position: absolute;
            background: linear-gradient(to bottom, #87CEEB 60%, #4CAF50 60%);
        }
        #score {
            position: absolute;
            top: 10px;
            left: 10px;
            font-size: 24px;
            color: white;
            text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.8);
            z-index: 10;
        }
        #gameOver {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            font-size: 48px;
            color: red;
            text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.8);
            display: none;
            z-index: 10;
        }
        #startButton {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            padding: 15px 30px;
            font-size: 24px;
            background: #FF5722;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            z-index: 10;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
        }
        #startButton:hover {
            background: #E64A19;
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
        // Game Variables
        let gameActive = false;
        let score = 0;
        let scrollSpeed = 3;
        const groundHeight = 40;
        let playerWidth = 30;
        let playerHeight = 50;
        let isJumping = false;
        let isSliding = false;
        let slideTimer = 0;

        // Matter.js Setup
        const Engine = Matter.Engine,
              Render = Matter.Render,
              Runner = Matter.Runner,
              Bodies = Matter.Bodies,
              Composite = Matter.Composite,
              Events = Matter.Events;

        // Engine erstellen
        const engine = Engine.create();
        const world = engine.world;
        engine.gravity.y = 0.8;

        // Canvas & Renderer
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const render = Render.create({
            canvas: canvas,
            engine: engine,
            options: {
                width: 800,
                height: 400,
                wireframes: false,
                background: 'transparent'
            }
        });

        // Game Elements
        let player;
        let ground;
        let obstacles = [];
        let clouds = [];
        let backgroundOffset = 0;

        // Create Clouds
        function createClouds() {
            for (let i = 0; i < 5; i++) {
                clouds.push({
                    x: Math.random() * 800,
                    y: Math.random() * 100 + 20,
                    width: Math.random() * 60 + 40,
                    speed: Math.random() * 0.5 + 0.2
                });
            }
        }

        // Draw Clouds
        function drawClouds() {
            ctx.save();
            clouds.forEach(cloud => {
                ctx.beginPath();
                ctx.arc(cloud.x, cloud.y, cloud.width/2, 0, Math.PI * 2);
                ctx.arc(cloud.x - cloud.width/3, cloud.y - cloud.width/6, cloud.width/3, 0, Math.PI * 2);
                ctx.arc(cloud.x + cloud.width/3, cloud.y - cloud.width/6, cloud.width/3, 0, Math.PI * 2);
                ctx.arc(cloud.x - cloud.width/4, cloud.y + cloud.width/6, cloud.width/3, 0, Math.PI * 2);
                ctx.arc(cloud.x + cloud.width/4, cloud.y + cloud.width/6, cloud.width/3, 0, Math.PI * 2);
                ctx.fillStyle = 'rgba(255, 255, 255, 0.8)';
                ctx.fill();
                
                cloud.x -= cloud.speed;
                if (cloud.x < -cloud.width) {
                    cloud.x = 800 + cloud.width;
                    cloud.y = Math.random() * 100 + 20;
                }
            });
            ctx.restore();
        }

        // Create Player
        function createPlayer() {
            player = Bodies.rectangle(100, 400 - groundHeight - playerHeight/2, 
                                    playerWidth, playerHeight, {
                restitution: 0.2,
                friction: 0.3,
                density: 0.05,
                label: 'player',
                chamfer: { radius: 5 }
            });
            Composite.add(world, player);
        }

        // Draw Player
        function drawPlayer() {
            ctx.save();
            ctx.translate(player.position.x, player.position.y);
            ctx.rotate(player.angle);
            
            if (isSliding) {
                // Slide position (flach, auf Boden)
                ctx.fillStyle = '#FF0000';
                ctx.fillRect(-playerWidth/2, -15, playerWidth, 30);
                
                ctx.fillStyle = '#0000FF';
                ctx.fillRect(-playerWidth/2, -5, playerWidth, 20);
            } else {
                // Normal position
                ctx.fillStyle = '#FF0000';
                ctx.fillRect(-playerWidth/2, -playerHeight/2, playerWidth, playerHeight);
                
                ctx.fillStyle = '#0000FF';
                ctx.fillRect(-playerWidth/2, -playerHeight/6, playerWidth, playerHeight/3);
            }
            
            // Kopf in beiden Positionen
            ctx.fillStyle = '#FFC0CB';
            ctx.beginPath();
            ctx.arc(0, isSliding ? -10 : -playerHeight/4, playerWidth/3, 0, Math.PI * 2);
            ctx.fill();
            
            ctx.restore();
        }

        // Create Ground
        function createGround() {
            ground = Bodies.rectangle(400, 400 - groundHeight/2, 800, groundHeight, {
                isStatic: true,
                label: 'ground',
                render: { visible: false }
            });
            Composite.add(world, ground);
        }

        // Draw Ground
        function drawGround() {
            ctx.fillStyle = '#4CAF50';
            ctx.fillRect(0, 400 - groundHeight, 800, groundHeight);
            
            // Ground details
            ctx.fillStyle = '#388E3C';
            for (let x = -backgroundOffset % 40; x < 800; x += 40) {
                ctx.fillRect(x, 400 - groundHeight, 20, 5);
            }
        }

        // Create Obstacles
        function createObstacle() {
            const types = ['pipe', 'block', 'low'];
            const type = types[Math.floor(Math.random() * types.length)];
            
            let obstacle;
            const x = 850;
            
            if (type === 'pipe') {
                // Röhren: hoch, zum Überspringen
                const pipeHeight = Math.random() * 50 + 50;
                obstacle = Bodies.rectangle(x, 400 - groundHeight - pipeHeight/2, 
                                          50, pipeHeight, {
                    isStatic: true,
                    label: 'obstacle',
                    obstacleType: 'pipe'
                });
            } else if (type === 'block') {
                // Kleine Blöcke: tödlich bei Kontakt
                obstacle = Bodies.rectangle(x, 400 - groundHeight - 30, 
                                          40, 40, {
                    isStatic: true,
                    label: 'obstacle',
                    obstacleType: 'block'
                });
            } else {
                // Hängendes niedriges Hindernis (drunter sliden)
                const lowHeight = 20;
                obstacle = Bodies.rectangle(x, 400 - groundHeight - playerHeight - lowHeight/2 + 5, 
                                          70, lowHeight, {
                    isStatic: true,
                    label: 'obstacle',
                    obstacleType: 'low'
                });
            }
            
            obstacles.push({
                body: obstacle,
                type: type
            });
            Composite.add(world, obstacle);
        }

        // Draw Obstacles
        function drawObstacles() {
            obstacles.forEach(obstacle => {
                ctx.save();
                ctx.translate(obstacle.body.position.x, obstacle.body.position.y);
                ctx.rotate(obstacle.body.angle);
                
                if (obstacle.type === 'pipe') {
                    ctx.fillStyle = '#00AA00';
                    const height = obstacle.body.bounds.max.y - obstacle.body.bounds.min.y;
                    ctx.fillRect(-25, -height/2, 50, height);
                    // Pipe details
                    ctx.fillStyle = '#007700';
                    ctx.fillRect(-20, -height/2 + 5, 40, 8);
                } else if (obstacle.type === 'block') {
                    ctx.fillStyle = '#B87333';
                    ctx.fillRect(-20, -20, 40, 40);
                    // Block details
                    ctx.fillStyle = '#A05A2C';
                    ctx.fillRect(-15, -15, 30, 10);
                } else if (obstacle.type === 'low') {
                    ctx.fillStyle = '#663300';
                    ctx.fillRect(-35, -10, 70, 20);
                }
                
                ctx.restore();
            });
        }

        // Update Obstacles
        function updateObstacles() {
            obstacles.forEach((obstacle, index) => {
                Matter.Body.translate(obstacle.body, {x: -scrollSpeed, y: 0});
                
                if (obstacle.body.position.x < -50) {
                    Composite.remove(world, obstacle.body);
                    obstacles.splice(index, 1);
                    score++;
                    updateScore();
                }
            });
        }

        // Update Score Display
        function updateScore() {
            document.getElementById('score').textContent = 'Score: ' + score;
        }

        // Reset Game
        function resetGame() {
            // Entferne alle Hindernisse
            obstacles.forEach(obstacle => Composite.remove(world, obstacle.body));
            obstacles = [];

            // Spieler zurücksetzen
            Matter.Body.setPosition(player, {x: 100, y: 400 - groundHeight - playerHeight/2});
            Matter.Body.setVelocity(player, {x: 0, y: 0});
            isJumping = false;
            isSliding = false;
            slideTimer = 0;

            score = 0;
            updateScore();
            gameActive = true;
            document.getElementById('gameOver').style.display = 'none';
            document.getElementById('startButton').style.display = 'none';
        }

        // Collision Check for Game Over
        Events.on(engine, 'collisionStart', event => {
            if (!gameActive) return;

            const pairs = event.pairs;
            pairs.forEach(pair => {
                let labels = [pair.bodyA.label, pair.bodyB.label];
                if (labels.includes('player') && labels.includes('obstacle')) {
                    // Kollisions-Objekt finden
                    let obstacleBody = pair.bodyA.label === 'obstacle' ? pair.bodyA : pair.bodyB;
                    let obstacleInfo = obstacles.find(o => o.body === obstacleBody);
                    if (!obstacleInfo) return;

                    // Bei Low-Hindernissen muss man sliding sein, sonst Game Over
                    if (obstacleInfo.type === 'low' && !isSliding) {
                        endGame();
                    } else if (obstacleInfo.type !== 'low') {
                        // Pipe oder Block: Kontakt -> Game Over
                        endGame();
                    }
                }
            });
        });

        // End Game Funktion
        function endGame() {
            gameActive = false;
            document.getElementById('gameOver').style.display = 'block';
            document.getElementById('startButton').style.display = 'block';
        }

        // Handle Keyboard Input
        document.addEventListener('keydown', e => {
            if (!gameActive) return;

            if (e.code === 'Space') {
                if (!isJumping && !isSliding) {
                    Matter.Body.applyForce(player, player.position, {x: 0, y: -0.15});
                    isJumping = true;
                }
            } else if (e.code === 'ShiftLeft' || e.code === 'ShiftRight') {
                if (!isSliding && !isJumping) {
                    isSliding = true;
                    slideTimer = 30; // Slide für 30 Frames (~0.5s bei 60fps)
                    // Verändere Form (niedriger)
                    Matter.Body.scale(player, 1, 0.5);
                }
            }
        });

        // Kein Pfeiltasten-Handling, alles deaktiviert

        // Handle Key Up for Slide End
        document.addEventListener('keyup', e => {
            if (!gameActive) return;

            if ((e.code === 'ShiftLeft' || e.code === 'ShiftRight') && isSliding) {
                isSliding = false;
                // Ursprüngliche Größe zurücksetzen
                Matter.Body.scale(player, 1, 2);
            }
        });

        // Überwache Bodenkontakt, um Springen zurückzusetzen
        Events.on(engine, 'afterUpdate', () => {
            if (!gameActive) return;

            // Prüfe, ob Spieler am Boden ist (nahe ground.y - groundHeight/2)
            if (player.position.y >= 400 - groundHeight - (isSliding ? playerHeight/4 : playerHeight/2) - 1) {
                isJumping = false;
            }

            // Slide Timer (optional automatisch stoppen)
            if (isSliding) {
                slideTimer--;
                if (slideTimer <= 0) {
                    isSliding = false;
                    Matter.Body.scale(player, 1, 2);
                }
            }
        });

        // Game Loop
        function gameLoop() {
            if (!gameActive) return;

            backgroundOffset += scrollSpeed;

            // Hintergrund zeichnen
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            drawClouds();
            drawGround();
            drawObstacles();
            drawPlayer();

            // Hindernisse bewegen & generieren
            updateObstacles();
            if (Math.random() < 0.02) {
                createObstacle();
            }

            Engine.update(engine, 1000 / 60);
            requestAnimationFrame(gameLoop);
        }

        // Init
        function init() {
            createGround();
            createPlayer();
            createClouds();
            updateScore();
            Render.run(render);
            Runner.run(Runner.create(), engine);
        }

        // Start Button
        document.getElementById('startButton').addEventListener('click', () => {
            if (!gameActive) {
                resetGame();
                gameLoop();
            }
        });

        init();
    </script>
</body>
</html>
