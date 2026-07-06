<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cyber Stadium - Arcade Soccer</title>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@700;900&family=Rajdhani:wght@500;700&display=swap" rel="stylesheet">
    
    <style>
        :root {
            --bg-color: #050811;
            --neon-green: #39ff14;
            --neon-blue: #00f0ff;
            --neon-yellow: #fffb00;
            --panel-bg: rgba(14, 23, 41, 0.75);
            --font-display: 'Orbitron', sans-serif;
            --font-ui: 'Rajdhani', sans-serif;
        }

        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            background-color: var(--bg-color);
            color: #fff;
            font-family: var(--font-ui);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            overflow: hidden;
            background: radial-gradient(circle at center, #0f1932 0%, #050811 100%);
        }

        /* BARRA SUPERIOR / PLACAR */
        .game-header {
            text-align: center;
            margin-bottom: 15px;
            width: 100%;
            max-width: 800px;
        }

        .title {
            font-family: var(--font-display);
            font-size: 2rem;
            text-transform: uppercase;
            letter-spacing: 2px;
            background: linear-gradient(90deg, var(--neon-yellow), var(--neon-green));
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
            margin-bottom: 10px;
        }

        .scoreboard {
            display: flex;
            justify-content: center;
            align-items: center;
            gap: 40px;
            background: var(--panel-bg);
            padding: 10px 40px;
            border-radius: 50px;
            border: 1px solid rgba(255, 255, 255, 0.1);
            box-shadow: 0 4px 20px rgba(0, 0, 0, 0.5);
            backdrop-filter: blur(10px);
        }

        .team-score {
            font-family: var(--font-display);
            font-size: 2.5rem;
            font-weight: 900;
        }

        #score-player { color: var(--neon-yellow); text-shadow: 0 0 10px rgba(255, 251, 0, 0.5); }
        #score-cpu { color: var(--neon-blue); text-shadow: 0 0 10px rgba(0, 240, 255, 0.5); }

        .vs {
            font-size: 1.2rem;
            color: #556789;
            font-weight: bold;
            text-transform: uppercase;
        }

        /* ÁREA DO JOGO (CANVAS) */
        .canvas-container {
            position: relative;
            border: 4px solid #1e293b;
            border-radius: 12px;
            box-shadow: 0 0 40px rgba(57, 255, 20, 0.15);
            background-color: #0a140d;
        }

        canvas {
            display: block;
            cursor: none;
        }

        /* PAINEL DE INSTRUÇÕES */
        .controls-panel {
            margin-top: 15px;
            background: var(--panel-bg);
            padding: 12px 30px;
            border-radius: 8px;
            border: 1px solid rgba(255, 255, 255, 0.05);
            font-size: 1.1rem;
            color: #94a3b8;
            display: flex;
            gap: 30px;
        }

        .controls-panel strong {
            color: #fff;
        }

        /* MENSAGEM DE GOL OVERLAY */
        .goal-overlay {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%) scale(0);
            font-family: var(--font-display);
            font-size: 5rem;
            font-weight: 900;
            color: var(--neon-green);
            text-shadow: 0 0 30px var(--neon-green);
            transition: transform 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
            pointer-events: none;
            z-index: 10;
        }

        .goal-overlay.active {
            transform: translate(-50%, -50%) scale(1);
        }
    </style>
</head>
<body>

    <div class="game-header">
        <h1 class="title">Cyber Stadium Soccer</h1>
        <div class="scoreboard">
            <div>
                <span style="font-size: 0.9rem; color: #aaa; display:block;">BRASIL</span>
                <span id="score-player" class="team-score">0</span>
            </div>
            <div class="vs">VS</div>
            <div>
                <span style="font-size: 0.9rem; color: #aaa; display:block;">CPU</span>
                <span id="score-cpu" class="team-score">0</span>
            </div>
        </div>
    </div>

    <div class="canvas-container">
        <div id="goal-text" class="goal-overlay">GOOOOOL!</div>
        <canvas id="gameCanvas" width="800" height="450"></canvas>
    </div>

    <div class="controls-panel">
        <div>Mover: <strong>W, A, S, D</strong> ou <strong>Setas</strong></div>
        <div>Objetivo: <strong>Empurre a bola no gol direito!</strong></div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const goalText = document.getElementById('goal-text');

        // VARIÁVEIS DO PLACAR
        let playerPoints = 0;
        let cpuPoints = 0;

        // TECLAS PRESSIONADAS
        const keys = {};
        window.addEventListener('keydown', e => keys[e.key.toLowerCase()] = true);
        window.addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);

        // CONFIGURAÇÕES DO CAMPO E ENTIEDADES
        const goalHeight = 120;
        const goalTop = (canvas.height - goalHeight) / 2;

        class Entity {
            constructor(x, y, radius, color, speed) {
                this.x = x;
                this.y = y;
                this.radius = radius;
                this.color = color;
                this.speed = speed;
                this.vx = 0;
                this.vy = 0;
            }

            draw() {
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
                ctx.fillStyle = this.color;
                ctx.fill();
                ctx.closePath();
            }
        }

        const player = new Entity(150, canvas.height / 2, 22, '#fffb00', 4.5);
        const cpu = new Entity(650, canvas.height / 2, 22, '#00f0ff', 3.8);
        const ball = new Entity(canvas.width / 2, canvas.height / 2, 12, '#ffffff', 0);
        ball.friction = 0.98; // Atrito da bola com o gramado

        function initPositions() {
            player.x = 150; player.y = canvas.height / 2; player.vx = 0; player.vy = 0;
            cpu.x = 650; cpu.y = canvas.height / 2; cpu.vx = 0; cpu.vy = 0;
            ball.x = canvas.width / 2; ball.y = canvas.height / 2; ball.vx = 0; ball.vy = 0;
        }

        // CONTROLAR COLISÃO FÍSICA ENTRE CÍRCULOS (Jogador/Bola)
        function handleCollision(ent, ballObj) {
            let dx = ballObj.x - ent.x;
            let dy = ballObj.y - ent.y;
            let dist = Math.sqrt(dx * dx + dy * dy);
            let minDist = ent.radius + ballObj.radius;

            if (dist < minDist) {
                // Afasta a bola imediatamente para não grudar
                let angle = Math.atan2(dy, dx);
                let overlap = minDist - dist;
                ballObj.x += Math.cos(angle) * overlap;
                ballObj.y += Math.sin(angle) * overlap;

                // Transfere velocidade do impacto e adiciona força extra de chute
                ballObj.vx = Math.cos(angle) * (Math.abs(ent.vx) + 5);
                ballObj.vy = Math.sin(angle) * (Math.abs(ent.vy) + 5);
            }
        }

        // DESENHAR LINHAS DO CAMPO ESTILO VAPORWAVE/NEON
        function drawPitch() {
            // Gramado de fundo
            ctx.fillStyle = '#06130b';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // Linhas brancas neon do campo
            ctx.strokeStyle = 'rgba(57, 255, 20, 0.3)';
            ctx.lineWidth = 3;

            // Linha do meio campo
            ctx.beginPath();
            ctx.moveTo(canvas.width / 2, 0);
            ctx.lineTo(canvas.width / 2, canvas.height);
            ctx.stroke();

            // Círculo central
            ctx.beginPath();
            ctx.arc(canvas.width / 2, canvas.height / 2, 70, 0, Math.PI * 2);
            ctx.stroke();

            // Áreas do Gol (Esquerda e Direita)
            ctx.strokeRect(0, goalTop - 40, 80, goalHeight + 80);
            ctx.strokeRect(canvas.width - 80, goalTop - 40, 80, goalHeight + 80);

            // Traves Físicas (Desenho Visual)
            ctx.fillStyle = '#334155';
            ctx.fillRect(0, goalTop, 8, goalHeight);
            ctx.fillRect(canvas.width - 8, goalTop, 8, goalHeight);
        }

        function triggerGoal(team) {
            if(team === 'player') { playerPoints++; document.getElementById('score-player').innerText = playerPoints; }
            if(team === 'cpu') { cpuPoints++; document.getElementById('score-cpu').innerText = cpuPoints; }
            
            goalText.classList.add('active');
            setTimeout(() => {
                goalText.classList.remove('active');
                initPositions();
            }, 1500);
        }

        // LOOP PRINCIPAL DO JOGO
        function update() {
            // 1. MOVIMENTAÇÃO JOGADOR (WASD / Setas)
            player.vx = 0; player.vy = 0;
            if (keys['w'] || keys['arrowup']) { player.vy = -player.speed; }
            if (keys['s'] || keys['arrowdown']) { player.vy = player.speed; }
            if (keys['a'] || keys['arrowleft']) { player.vx = -player.speed; }
            if (keys['d'] || keys['arrowright']) { player.vx = player.speed; }
            
            player.x += player.vx;
            player.y += player.vy;

            // 2. INTELIGÊNCIA ARTIFICIAL SIMPLES DO BOT (CPU)
            // O bot segue a bola se ela estiver do lado dele do campo
            let targetY = ball.y;
            let targetX = ball.x;

            cpu.vx = 0; cpu.vy = 0;
            if (cpu.x < targetX) { cpu.vx = cpu.speed * 0.7; }
            if (cpu.x > targetX) { cpu.vx = -cpu.speed; }
            if (cpu.y < targetY - 5) { cpu.vy = cpu.speed; }
            if (cpu.y > targetY + 5) { cpu.vy = -cpu.speed; }

            cpu.x += cpu.vx;
            cpu.y += cpu.vy;

            // 3. FÍSICA E MOVIMENTO DA BOLA
            ball.vx *= ball.friction;
            ball.vy *= ball.friction;
            ball.x += ball.vx;
            ball.y += ball.vy;

            // Limitar jogadores dentro das bordas do campo
            [player, cpu].forEach(ent => {
                if (ent.x - ent.radius < 0) ent.x = ent.radius;
                if (ent.x + ent.radius > canvas.width) ent.x = canvas.width - ent.radius;
                if (ent.y - ent.radius < 0) ent.y = ent.radius;
                if (ent.y + ent.radius > canvas.height) ent.y = canvas.height - ent.radius;
            });

            // 4. COLISÕES COM AS BORDAS / DETECÇÃO DE GOL
            // Paredes Topo e Fundo da Bola
            if (ball.y - ball.radius < 0 || ball.y + ball.radius > canvas.height) {
                ball.vy = -ball.vy;
            }

            // Paredes Laterais da Bola (Checa se entrou no gol ou bateu na linha de fundo)
            if (ball.x - ball.radius < 0) {
                if (ball.y > goalTop && ball.y < goalTop + goalHeight) {
                    triggerGoal('cpu'); // Gol da CPU
                } else {
                    ball.vx = -ball.vx; // Bateu na parede de fundo
                    ball.x = ball.radius;
                }
            }
            if (ball.x + ball.radius > canvas.width) {
                if (ball.y > goalTop && ball.y < goalTop + goalHeight) {
                    triggerGoal('player'); // Gol do Jogador
                } else {
                    ball.vx = -ball.vx; // Bateu na parede de fundo
                    ball.x = canvas.width - ball.radius;
                }
            }

            // 5. PROCESSAR COLISÕES ENTRE CORPOS
            handleCollision(player, ball);
            handleCollision(cpu, ball);

            // 6. RENDERIZAÇÃO GRÁFICA
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            drawPitch();
            
            // Detalhes estéticos (Sombras brilhantes)
            ctx.shadowBlur = 15;
            ctx.shadowColor = player.color;
            player.draw();
            
            ctx.shadowColor = cpu.color;
            cpu.draw();
            
            ctx.shadowColor = '#ffffff';
            ball.draw();
            ctx.shadowBlur = 0; // Desativa sombra para otimizar frames

            requestAnimationFrame(update);
        }

        // Inicia o jogo
        initPositions();
        update();
    </script>
</body>
</html>