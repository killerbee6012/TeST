<!DOCTYPE html>
<html>
<head>
    <title>Mario-ähnliches Laufspiel (Final)</title>
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
        let gameActive = false;
        let score = 0;
        let scrollSpeed = 3;
        const groundHeight = 40;
        let playerWidth = 30;
        let playerHeight = 50;
        let isJumping = false;
        let isSliding = false;
        let slideTimer = 0;

        const Engine = Matter.Engine,
              Render = Matter.Render,
              Runner = Matter.Runner,
              Bodies = Matter.Bodies,
              Composite = Matter.Composite,
              Events = Matter.Events;

        const engine = Engine.create();
        const world = engine.world;
        engine.gravity.y = 0.8;

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

        let player;
        let ground;
        let obstacles = [];
        let clouds = [];
        let backgroundOffset = 0;

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

        function drawPlayer() {
            ctx.save();
            ctx.translate(player.position.x, player.position.y);
            ctx.rotate(player.angle);
            
            if (isSliding) {
                ctx.fillStyle = '#FF0000';
                ctx.fillRect(-playerWidth/2, -15, playerWidth, 30);
                ctx.fillStyle = '#0000FF';
                ctx.fillRect(-playerWidth/2, -5, playerWidth, 20);
            } else {
                ctx.fillStyle = '#FF0000';
                ctx.fillRect(-playerWidth/2, -playerHeight/2, playerWidth, playerHeight);
                ctx.fillStyle = '#0000FF';
                ctx.fillRect(-playerWidth/2, -playerHeight/6, playerWidth, playerHeight/3);
            }

            ctx.fillStyle = '#FFC0CB';
            ctx.beginPath();
            ctx.arc(0, isSliding ? -10 : -playerHeight/4, playerWidth/3, 0, Math.PI * 2);
            ctx.fill();
            
            ctx.restore();
        }

        function createGround() {
            ground = Bodies.rectangle(400, 400 - groundHeight/2, 800, groundHeight, {
                isStatic: true,
                label: 'ground',
                render: { visible: false }
            });
            Composite.add(world, ground);
        }

        function drawGround() {
            ctx.fillStyle = '#4CAF50';
            ctx.fillRect(0, 400 - groundHeight, 800, groundHeight);
            ctx.fillStyle = '#388E3C';
            for (let x = -backgroundOffset % 40; x < 800; x += 40) {
                ctx.fillRect(x, 400 - groundHeight, 20, 5);
            }
        }

        function createObstacle() {
            const types = ['pipe', 'block'];
            const type = types[Math.floor(Math.random() * types.length)];
            let obstacle;
            const x = 850;

            if (type === 'pipe') {
                const pipeHeight = Math.random() * 50 + 50;
                obstacle = Bodies.rectangle(x, 400 - groundHeight - pipeHeight/2, 
                                            50, pipeHeight, {
                    isStatic: true,
                    label: 'obstacle'
                });
            } else {
                obstacle = Bodies.rectangle(x, 400 - groundHeight - 30, 
                                            40, 40, {
                    isStatic: true,
                    label: 'obstacle'
                });
            }

            obstacles.push({ body: obstacle, type: type });
            Composite.add(world, obstacle);
        }

        function drawObstacles() {
            obstacles.forEach(obstacle => {
                ctx.save();
                ctx.translate(obstacle.body.position.x, obstacle.body.position.y);
                ctx.rotate(obstacle.body.angle);

                if (obstacle.type === 'pipe') {
                    ctx.fillStyle = '#00AA00';
                    ctx.fillRect(-25, -obstacle.body.bounds.max.y + obstacle.body.position.y, 50, obstacle.body.bounds.max.y - obstacle.body.bounds.min.y);
                    ctx.fillStyle = '#007700';
                    ctx.fillRect(-20, -obstacle.body.bounds.max.y + obstacle.body.position.y + 5, 40, 8);
                } else {
                    ctx.fillStyle = '#B87333';
                    ctx.fillRect(-20, -20, 40, 40);
                    ctx.fillStyle = '#A05A2C';
                    ctx.fillRect(-15, -15, 30, 5);
                }

                ctx.restore();
            });
        }

        function startGame() {
            gameActive = true;
            score = 0;
            scrollSpeed = 3;
            isJumping = false;
            isSliding = false;
            document.getElementById('startButton').style.display = 'none';
            document.getElementById('gameOver').style.display = 'none';
            Composite.clear(world);
            obstacles = [];
            clouds = [];
            createPlayer();
            createGround();
            createClouds();

            obstacleInterval = setInterval(() => {
                if (gameActive) createObstacle();
            }, 2000);

            if (!runner) {
                runner = Runner.create();
                Runner.run(runner, engine);
            }

            requestAnimationFrame(gameLoop);
        }

        function gameOver() {
            gameActive = false;
            clearInterval(obstacleInterval);
            document.getElementById('gameOver').style.display = 'block';
            document.getElementById('startButton').style.display = 'block';
        }

        let runner;
        let obstacleInterval;
        function gameLoop() {
            if (!gameActive) return;
            ctx.clearRect(0, 0, 800, 400);
            ctx.fillStyle = '#87CEEB';
            ctx.fillRect(0, 0, 800, 400 - groundHeight);
            drawClouds();
            drawGround();
            drawObstacles();
            drawPlayer();

            obstacles.forEach(obstacle => {
                Matter.Body.setPosition(obstacle.body, {
                    x: obstacle.body.position.x - scrollSpeed,
                    y: obstacle.body.position.y
                });
                if (obstacle.body.position.x < -100) {
                    Composite.remove(world, obstacle.body);
                    obstacles = obstacles.filter(o => o.body.id !== obstacle.body.id);
                }
            });

            backgroundOffset += scrollSpeed;
            if (backgroundOffset >= 800) backgroundOffset = 0;

            if (isSliding) {
                slideTimer++;
                if (slideTimer > 60) {
                    isSliding = false;
                    playerHeight = 50;
                    Matter.Body.setPosition(player, {
                        x: player.position.x,
                        y: 400 - groundHeight - playerHeight / 2
                    });
                }
            }

            const playerBottom = player.position.y + (isSliding ? 15 : playerHeight / 2);
            isJumping = playerBottom < 400 - groundHeight - 5;

            score++;
            document.getElementById('score').textContent = `Score: ${score}`;

            if (score % 500 === 0) {
                scrollSpeed += 0.5;
            }

            requestAnimationFrame(gameLoop);
        }

        document.getElementById('startButton').addEventListener('click', startGame);

        document.addEventListener('keydown', (e) => {
            if (!gameActive) return;

            if (e.code === 'Space' && !isJumping && !isSliding) {
                Matter.Body.setVelocity(player, { x: 0, y: -10 });
                isJumping = true;
            }

            if ((e.code === 'ShiftLeft' || e.code === 'ShiftRight') && !isSliding && !isJumping) {
                isSliding = true;
                slideTimer = 0;
                playerHeight = 30;
                Matter.Body.setPosition(player, {
                    x: player.position.x,
                    y: 400 - groundHeight - playerHeight / 2
                });
            }
        });

        document.addEventListener('keyup', (e) => {
            if ((e.code === 'ShiftLeft' || e.code === 'ShiftRight') && isSliding) {
                isSliding = false;
                playerHeight = 50;
                Matter.Body.setPosition(player, {
                    x: player.position.x,
                    y: 400 - groundHeight - playerHeight / 2
                });
            }
        });

        Events.on(engine, 'collisionStart', (event) => {
            const pairs = event.pairs;
            for (let i = 0; i < pairs.length; i++) {
                const pair = pairs[i];
                if ((pair.bodyA.label === 'player' && pair.bodyB.label === 'obstacle') ||
                    (pair.bodyB.label === 'player' && pair.bodyA.label === 'obstacle')) {
                    const obstacle = pair.bodyA.label === 'obstacle' ? pair.bodyA : pair.bodyB;
                    const obstacleHeight = obstacle.bounds.max.y - obstacle.bounds.min.y;
                    if (!isSliding || obstacleHeight < 60) {
                        gameOver();
                    }
                }

                if ((pair.bodyA.label === 'player' && pair.bodyB.label === 'ground') ||
                    (pair.bodyB.label === 'player' && pair.bodyA.label === 'ground')) {
                    isJumping = false;
                }
            }
        });

        Render.run(render);
        createClouds();
    </script>
</body>
</html>
