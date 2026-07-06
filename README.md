<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Super Arcade Soccer 1v1</title>
    <style>
        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
        }

        body {
            background: #1a1a1a;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            color: #fff;
            overflow: hidden;
        }

        h1 {
            margin-bottom: 10px;
            font-size: 2.5rem;
            text-transform: uppercase;
            text-shadow: 0 4px 10px rgba(0,0,0,0.5);
            color: #4eed58;
        }

        #scoreboard {
            display: flex;
            gap: 30px;
            font-size: 2rem;
            font-weight: bold;
            margin-bottom: 15px;
            background: #2a2a2a;
            padding: 10px 30px;
            border-radius: 50px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.3);
            border: 2px solid #444;
        }

        .player-score { color: #3498db; }
        .cpu-score { color: #e74c3c; }

        #gameContainer {
            position: relative;
            box-shadow: 0 10px 30px rgba(0,0,0,0.7);
            border-radius: 8px;
            overflow: hidden;
        }

        canvas {
            background: #2c3e50;
            display: block;
        }

        #goalMessage {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%) scale(0);
            font-size: 5rem;
            font-weight: 900;
            color: #f1c40f;
            text-shadow: 0 0 20px rgba(241, 196, 15, 0.7), 4px 4px 0 #000;
            transition: transform 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
            pointer-events: none;
            z-index: 10;
            text-transform: uppercase;
        }

        #goalMessage.show {
            transform: translate(-50%, -50%) scale(1);
        }

        #controls-tip {
            margin-top: 15px;
            color: #888;
            font-size: 0.9rem;
            text-align: center;
        }
    </style>
</head>
<body>

    <h1>Arcade Soccer</h1>

    <div id="scoreboard">
        <span class="player-score" id="pScore">0</span>
        <span>x</span>
        <span class="cpu-score" id="cScore">0</span>
    </div>

    <div id="gameContainer">
        <div id="goalMessage">GOL!!</div>
        <canvas id="gameCanvas" width="800" height="500"></canvas>
    </div>

    <div id="controls-tip">
        Controles: Use as <b>Setas do Teclado</b> ou <b>WASD</b> para mover seu jogador (Azul).
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const goalMsg = document.getElementById('goalMessage');

        // Placar
        let playerScore = 0;
        let cpuScore = 0;

        // Teclas pressionadas
        const keys = {};

        // Configuração das Traves (Gols)
        const goalWidth = 20;
        const goalHeight = 140;
        const goalY = (canvas.height - goalHeight) / 2;

        // Entidades do Jogo
        const player = {
            x: 150,
            y: canvas.height / 2,
            radius: 22,
            speed: 5,
            color: '#3498db',
            targetColor: '#2980b9'
        };

        const cpu = {
            x: canvas.width - 150,
            y: canvas.height / 2,
            radius: 22,
            speed: 4.2, // Ligeiramente mais lento que o player para ser justo
            color: '#e74c3c',
            targetColor: '#c0392b'
        };

        const ball = {
            x: canvas.width / 2,
            y: canvas.height / 2,
            radius: 12,
            vx: 0,
            vy: 0,
            friction: 0.98,
            color: '#ffffff'
        };

        // Ouvintes de teclado
        window.addEventListener('keydown', e => keys[e.key.toLowerCase()] = true);
        window.addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);

        function resetPositions() {
            player.x = 150;
            player.y = canvas.height / 2;
            
            cpu.x = canvas.width - 150;
            cpu.y = canvas.height / 2;

            ball.x = canvas.width / 2;
            ball.y = canvas.height / 2;
            ball.vx = 0;
            ball.vy = 0;
        }

        function triggerGoal(winner) {
            if(winner === 'player') {
                playerScore++;
                document.getElementById('pScore').innerText = playerScore;
                goalMsg.innerText = "SEU GOL!!";
                goalMsg.style.color = "#3498db";
            } else {
                cpuScore++;
                document.getElementById('cScore').innerText = cpuScore;
                goalMsg.innerText = "GOL DA CPU!";
                goalMsg.style.color = "#e74c3c";
            }

            goalMsg.classList.add('show');
            
            setTimeout(() => {
                goalMsg.classList.remove('show');
                resetPositions();
            }, 1500);
        }

        // Desenhar o Campo de Futebol
        function drawField() {
            // Gramado
            ctx.fillStyle = '#2e7d32';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // Linhas de marcação (Brancas)
            ctx.strokeStyle = 'rgba(255, 255, 255, 0.4)';
            ctx.lineWidth = 4;

            // Linha do Meio
            ctx.beginPath();
            ctx.moveTo(canvas.width / 2, 0);
            ctx.lineTo(canvas.width / 2, canvas.height);
            ctx.stroke();

            // Círculo Central
            ctx.beginPath();
            ctx.arc(canvas.width / 2, canvas.height / 2, 80, 0, Math.PI * 2);
            ctx.stroke();

            // Ponto Central
            ctx.beginPath();
            ctx.arc(canvas.width / 2, canvas.height / 2, 5, 0, Math.PI * 2);
            ctx.fillStyle = 'rgba(255, 255, 255, 0.6)';
            ctx.fill();

            // Áreas de Gol (Esquerda e Direita)
            ctx.strokeRect(0, goalY - 40, 80, goalHeight + 80);
            ctx.strokeRect(canvas.width - 80, goalY - 40, 80, goalHeight + 80);

            // Desenhar as Redes/Traves Físicas
            ctx.fillStyle = '#fff';
            // Gol Esquerdo
            ctx.fillRect(0, goalY, goalWidth, 6); // Trave Cima
            ctx.fillRect(0, goalY + goalHeight - 6, goalWidth, 6); // Trave Baixo
            ctx.fillRect(0, goalY, 4, goalHeight); // Fundo da rede
            
            // Gol Direito
            ctx.fillRect(canvas.width - goalWidth, goalY, goalWidth, 6); 
            ctx.fillRect(canvas.width - goalWidth, goalY + goalHeight - 6, goalWidth, 6);
            ctx.fillRect(canvas.width - 4, goalY, 4, goalHeight);
        }

        function drawCircle(x, y, radius, color, strokeColor) {
            ctx.beginPath();
            ctx.arc(x, y, radius, 0, Math.PI * 2);
            ctx.fillStyle = color;
            ctx.fill();
            if(strokeColor) {
                ctx.strokeStyle = strokeColor;
                ctx.lineWidth = 3;
                ctx.stroke();
            }
        }

        // Lógica de colisão elástica simples entre jogador/bola
        function handleCollision(entity) {
            let dx = ball.x - entity.x;
            let dy = ball.y - entity.y;
            let distance = Math.sqrt(dx * dx + dy * dy);
            let minDist = entity.radius + ball.radius;

            if (distance < minDist) {
                // Ângulo do impacto
                let angle = Math.atan2(dy, dx);
                
                // Força do impacto baseada na velocidade do empurrão + bônus
                let force = 6; 
                
                // Afasta a bola para não grudar no jogador
                ball.x = entity.x + Math.cos(angle) * minDist;
                ball.y = entity.y + Math.sin(angle) * minDist;

                // Aplica a nova velocidade à bola
                ball.vx = Math.cos(angle) * force + (ball.vx * 0.5);
                ball.vy = Math.sin(angle) * force + (ball.vy * 0.5);
            }
        }

        function update() {
            // Se o texto de Gol estiver ativo, pausa o jogo momentaneamente
            if (goalMsg.classList.contains('show')) return;

            // --- MOVIMENTO DO JOGADOR ---
            if (keys['arrowup'] || keys['w']) player.y -= player.speed;
            if (keys['arrowdown'] || keys['s']) player.y += player.speed;
            if (keys['arrowleft'] || keys['a']) player.x -= player.speed;
            if (keys['arrowright'] || keys['d']) player.x += player.speed;

            // Limites do Player (Não sair do campo)
            player.x = Math.max(player.radius, Math.min(canvas.width - player.radius, player.x));
            player.y = Math.max(player.radius, Math.min(canvas.height - player.radius, player.y));

            // --- INTELIGÊNCIA ARTIFICIAL (CPU) ---
            // A CPU persegue a bola se ela estiver na metade direita do campo, senão volta para a defesa
            let targetX = canvas.width - 150;
            let targetY = canvas.height / 2;

            if (ball.x > canvas.width / 3) {
                targetX = ball.x - 20; // Tenta se posicionar um pouco atrás da bola para empurrar pro gol
                targetY = ball.y;
            }

            // Movimento suave da CPU em direção ao alvo
            if (cpu.x < targetX) cpu.x += cpu.speed;
            if (cpu.x > targetX) cpu.x -= cpu.speed;
            if (cpu.y < targetY) cpu.y += cpu.speed;
            if (cpu.y > targetY) cpu.y -= cpu.speed;

            // Limites da CPU
            cpu.x = Math.max(canvas.width / 2, Math.min(canvas.width - cpu.radius, cpu.x)); // Fica na sua metade
            cpu.y = Math.max(cpu.radius, Math.min(canvas.height - cpu.radius, cpu.y));


            // --- FÍSICA DA BOLA ---
            ball.x += ball.vx;
            ball.y += ball.vy;
            ball.vx *= ball.friction; // Desaceleração natural (atrito da grama)
            ball.vy *= ball.friction;

            // Colisões da bola com as paredes laterais (com exceção da área de gol)
            // Paredes Superiores e Inferiores
            if (ball.y - ball.radius < 0) {
                ball.y = ball.radius;
                ball.vy = -ball.vy;
            }
            if (ball.y + ball.radius > canvas.height) {
                ball.y = canvas.height - ball.radius;
                ball.vy = -ball.vy;
            }

            // Paredes da Esquerda (Fora do Gol)
            if (ball.x - ball.radius < 0) {
                if (ball.y > goalY && ball.y < goalY + goalHeight) {
                    // É GOL DA CPU!
                    triggerGoal('cpu');
                } else {
                    ball.x = ball.radius;
                    ball.vx = -ball.vx;
                }
            }

            // Paredes da Direita (Fora do Gol)
            if (ball.x + ball.radius > canvas.width) {
                if (ball.y > goalY && ball.y < goalY + goalHeight) {
                    // É GOL DO JOGADOR!
                    triggerGoal('player');
                } else {
                    ball.x = canvas.width - ball.radius;
                    ball.vx = -ball.vx;
                }
            }

            // Processar colisões físicas de toque
            handleCollision(player);
            handleCollision(cpu);
        }

        function draw() {
            drawField();

            // Desenhar os Jogadores (com visual estilo botão de futebol de mesa moderno)
            drawCircle(player.x, player.y, player.radius, player.color, player.targetColor);
            drawCircle(player.x, player.y, player.radius * 0.5, 'rgba(255,255,255,0.2)'); // Detalhe interno

            drawCircle(cpu.x, cpu.y, cpu.radius, cpu.color, cpu.targetColor);
            drawCircle(cpu.x, cpu.y, cpu.radius * 0.5, 'rgba(255,255,255,0.2)');

            // Desenhar a Bola (Estilo clássico gomos)
            drawCircle(ball.x, ball.y, ball.radius, ball.color, '#333');
            // Linhas da bola
            ctx.strokeStyle = '#333';
            ctx.lineWidth = 1;
            ctx.beginPath();
            ctx.moveTo(ball.x - ball.radius, ball.y);
            ctx.lineTo(ball.x + ball.radius, ball.y);
            ctx.moveTo(ball.x, ball.y - ball.radius);
            ctx.lineTo(ball.x, ball.y + ball.radius);
            ctx.stroke();
        }

        // Loop principal do jogo
        function gameLoop() {
            update();
            draw();
            requestAnimationFrame(gameLoop);
        }

        // Iniciar
        gameLoop();
    </script>
</body>
</html>