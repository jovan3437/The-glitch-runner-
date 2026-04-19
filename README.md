<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Glitch-Runner: Virus Edition</title>
    <style>
        body, html {
            margin: 0; padding: 0; width: 100%; height: 100%;
            background-color: #050505; color: #0f0;
            font-family: 'Courier New', Courier, monospace;
            overflow: hidden; touch-action: none; 
        }
        
        /* ADJUSTED: Pushed down 60px to leave room for AdMob Banner at the top */
        #game-container { 
            position: relative; 
            width: 100%; 
            height: calc(100% - 60px); 
            margin-top: 60px; 
        }
        
        canvas { display: block; width: 100%; height: 100%; background: #000; }
        
        .ui-layer {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            display: flex; flex-direction: column; justify-content: center;
            align-items: center; background: rgba(0, 0, 0, 0.9);
            text-align: center; z-index: 10;
        }

        button {
            background: #000; color: #0f0; border: 2px solid #0f0;
            padding: 15px 30px; margin: 10px; cursor: pointer;
            font-family: inherit; font-size: 1.2rem; border-radius: 5px;
            box-shadow: 0 0 10px #0f0;
        }

        #hud {
            position: absolute; top: 15px; left: 15px;
            pointer-events: none; font-size: 1.1rem; z-index: 5;
        }
        .text-red { color: #ff3333; }
        #upgradeScreen, #gameOverScreen { display: none; }
    </style>
</head>
<body>

<div id="game-container">
    <div id="hud">
        <div>CORE_STABILITY: <span id="scoreDisplay">0</span></div>
        <div style="color: #0ff;">PAGES_STOLEN: <span id="dataDisplay">0</span></div>
        <div class="text-red">CORRUPTION: <span id="corruptionDisplay">0</span>%</div>
    </div>
    
    
    <canvas id="gameCanvas"></canvas>

    <div id="startScreen" class="ui-layer">
        <h1 style="font-size: 3rem;">👾 VIRUS.EXE</h1>
        <p>Double Tap to Double Jump.<br>Steal the 📄, avoid the Corrupt Blocks.</p>
        <button onclick="startGame()">UPLOAD VIRUS</button>
    </div>

    <div id="upgradeScreen" class="ui-layer">
        <h2>EVOLVE CODE</h2>
        <button onclick="applyPatch('doubleJump')">Unlock: Double Jump (+20% Corrupt)</button>
        <button onclick="applyPatch('dataMultiplier')">Unlock: Data Efficiency (+30% Corrupt)</button>
    </div>

    <div id="gameOverScreen" class="ui-layer">
        <h1 class="text-red">SYSTEM REBOOTED</h1>
        <p>Your virus was quarantined.</p>
        <button onclick="resetGame()">RE-INFECT</button>
    </div>
</div>

<script>
    // ==========================================
    // ADMOB CONFIGURATION & BRIDGE
    // ==========================================
    
    // PASTE YOUR ADMOB UNIT ID HERE:
    const ADMOB_BANNER_ID = "ca-app-pub-3893159799848007/6322819668"; 

    function requestAdMobBanner() {
        console.log("Requesting AdMob Banner with ID: " + ADMOB_BANNER_ID);
        
        // This triggers the ad if you are using a standard Android Interface
        if (window.AndroidInterface && typeof window.AndroidInterface.showBanner === 'function') {
            window.AndroidInterface.showBanner(ADMOB_BANNER_ID);
        } 
        // This triggers the ad if you are using iOS / WebKit messaging
        else if (window.webkit && window.webkit.messageHandlers && window.webkit.messageHandlers.adHandler) {
            window.webkit.messageHandlers.adHandler.postMessage({action: "showBanner", id: ADMOB_BANNER_ID});
        }
    }
    // ==========================================

    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');

    function resize() {
        const container = document.getElementById('game-container');
        canvas.width = container.clientWidth;
        canvas.height = container.clientHeight;
    }
    window.addEventListener('resize', resize);
    resize();

    // Game Variables
    let hasStarted = false, isPaused = false, isGameOver = false;
    let frames = 0, score = 0, data = 0, corruption = 0;
    let gameSpeed = 6, nextPatchScore = 1000;
    
    // Upgrades
    let hasDoubleJump = false;
    let dataMultiplier = 1;

    const player = {
        x: 60, y: 0, size: 40, dy: 0,
        gravity: 0.7, jumpForce: -12,
        jumpCount: 0,
        
        draw() {
            ctx.save();
            // Glitchy vibration effect based on corruption
            let gx = (Math.random() - 0.5) * (corruption / 10);
            let gy = (Math.random() - 0.5) * (corruption / 10);
            
            ctx.font = `${this.size}px serif`;
            ctx.textAlign = "center";
            ctx.textBaseline = "middle";
            
            // Draw Virus Emoji
            ctx.fillText("👾", this.x + (this.size/2) + gx, this.y + (this.size/2) + gy);
            
            // Add a neon glow
            ctx.shadowBlur = 15;
            ctx.shadowColor = '#0f0';
            ctx.restore();
        },
        update() {
            const ground = canvas.height - 60;
            this.dy += this.gravity;
            this.y += this.dy;

            if (this.y + this.size >= ground) {
                this.y = ground - this.size;
                this.dy = 0;
                this.jumpCount = 0;
            }
        },
        jump() {
            // Logic for single or double jump
            const maxJumps = hasDoubleJump ? 2 : 1;
            if (this.jumpCount < maxJumps) {
                this.dy = this.jumpForce;
                this.jumpCount++;
            }
        }
    };

    let obstacles = [];
    let dataTokens = [];

    // --- Input Logic (Tap/Click) ---
    canvas.addEventListener('touchstart', (e) => {
        e.preventDefault();
        if (hasStarted && !isPaused && !isGameOver) player.jump();
    });
    canvas.addEventListener('mousedown', (e) => {
        if (hasStarted && !isPaused && !isGameOver) player.jump();
    });

    window.startGame = () => {
        document.getElementById('startScreen').style.display = 'none';
        hasStarted = true;
        player.y = canvas.height / 2;
        
        // CALL ADMOB WHEN THE GAME STARTS
        requestAdMobBanner();
        
        loop();
    };

    window.applyPatch = (type) => {
        if (type === 'doubleJump') { hasDoubleJump = true; corruption += 20; }
        if (type === 'dataMultiplier') { dataMultiplier += 1; corruption += 30; }
        nextPatchScore += 1000;
        upgradeScreen.style.display = 'none';
        isPaused = false;
        loop();
    };

    window.resetGame = () => {
        location.reload(); // Simplest way to reset all variables
    };

    function loop() {
        if (!hasStarted || isPaused || isGameOver) return;

        ctx.fillStyle = "#000";
        ctx.fillRect(0, 0, canvas.width, canvas.height);

        // Draw Floor
        ctx.strokeStyle = "#0f0";
        ctx.lineWidth = 4;
        ctx.beginPath();
        ctx.moveTo(0, canvas.height - 60);
        ctx.lineTo(canvas.width, canvas.height - 60);
        ctx.stroke();

        // Glitch Effects
        if (corruption > 0 && Math.random() < (corruption/200)) {
            ctx.translate((Math.random()-0.5)*10, 0);
        }

        player.update();
        player.draw();

        // Spawn Obstacles
        if (frames % 80 === 0) {
            obstacles.push({ x: canvas.width, y: canvas.height - 110, w: 30, h: 50 });
        }

        // Spawn Data (Pages)
        if (frames % 60 === 0) {
            dataTokens.push({ x: canvas.width, y: canvas.height - 150 - Math.random() * 150 });
        }

        // Process Obstacles
        obstacles.forEach((obs, i) => {
            obs.x -= gameSpeed;
            ctx.fillStyle = "#f00";
            ctx.fillRect(obs.x, obs.y, obs.w, obs.h);
            
            // Hitbox
            if (player.x < obs.x + obs.w && player.x + player.size > obs.x &&
                player.y < obs.y + obs.h && player.y + player.size > obs.y) {
                isGameOver = true;
                document.getElementById('gameOverScreen').style.display = 'flex';
            }
        });

        // Process Data (Pages 📄)
        dataTokens.forEach((token, i) => {
            token.x -= gameSpeed;
            ctx.font = "30px serif";
            ctx.fillText("📄", token.x, token.y);

            if (Math.abs(player.x - token.x) < 30 && Math.abs(player.y - token.y) < 30) {
                data += (1 * dataMultiplier);
                dataTokens.splice(i, 1);
            }
        });

        // Cleanup
        obstacles = obstacles.filter(o => o.x > -50);
        dataTokens = dataTokens.filter(d => d.x > -50);

        frames++;
        score++;
        if (score >= nextPatchScore) {
            isPaused = true;
            document.getElementById('upgradeScreen').style.display = 'flex';
        }

        document.getElementById('scoreDisplay').innerText = score;
        document.getElementById('dataDisplay').innerText = data;
        document.getElementById('corruptionDisplay').innerText = corruption;

        requestAnimationFrame(loop);
    }
</script>
</body>
</html>
# The-glitch-runner-
#game #freegame #runnergame #webgame #free #poki    
on desktop site for Play 
tap only on game area to jump 

