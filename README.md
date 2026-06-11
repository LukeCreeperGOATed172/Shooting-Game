<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Arcade 3D Shooter</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #050510;
            color: #ffffff;
            font-family: 'Courier New', Courier, monospace;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            overflow: hidden;
            user-select: none;
        }
        #game-container {
            position: relative;
            width: 800px;
            height: 600px;
            border: 3px solid #00ffcc;
            box-shadow: 0 0 30px rgba(0, 255, 204, 0.3);
        }
        canvas {
            display: block;
            background-color: #010105;
            width: 800px;
            height: 600px;
        }
        #hud {
            position: absolute;
            top: 20px;
            left: 20px;
            color: #00ffcc;
            font-size: 22px;
            font-weight: bold;
            pointer-events: none;
            text-shadow: 0 0 5px #00ffcc;
            z-index: 5;
        }
        #gameover-screen {
            position: absolute;
            top: 0;
            left: 0;
            width: 800px;
            height: 600px;
            background: rgba(0, 0, 0, 0.9);
            display: none;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            z-index: 10;
        }
        #gameover-screen h1 { color: #ff0055; font-size: 45px; margin: 0 0 10px 0; text-shadow: 0 0 10px #ff0055; }
        #gameover-screen p { color: #fff; font-size: 20px; margin: 5px 0; }
    </style>
</head>
<body>

    <div id="game-container">
        <div id="hud">
            SCORE: <span id="current-score">0</span><br>
            HIGH: <span id="high-score-hud">0</span>
        </div>
        
        <div id="gameover-screen">
            <h1>GAME OVER</h1>
            <p id="final-score">Final Score: 0</p>
            <p id="high-score-notice" style="color: #00ffcc; font-weight: bold; display: none;">NEW HIGH SCORE!</p>
            <p style="font-size: 16px; color: #888; margin-top: 30px;">CLICK ANYWHERE TO RESTART</p>
        </div>

        <canvas id="gameCanvas" width="800" height="600"></canvas>
    </div>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

// --- Game Variables ---
let score = 0;
let highScore = 0;
let gameOver = false;
let spawnTimer = 0;

const mouse = { x: 400, y: 300 };
const stars = [];
const lasers = [];
const asteroids = [];

// Populate fixed star grid
for (let i = 0; i < 150; i++) {
    stars.push({
        x: Math.random() * 1600 - 800,
        y: Math.random() * 1200 - 600,
        z: Math.random() * 800
    });
}

// Track mouse positioning inside our bounded container box
canvas.addEventListener('mousemove', (e) => {
    const rect = canvas.getBoundingClientRect();
    mouse.x = e.clientX - rect.left;
    mouse.y = e.clientY - rect.top;
});

canvas.addEventListener('click', () => {
    if (gameOver) {
        resetGame();
    } else {
        // Fire from current position
        lasers.push({
            x: mouse.x - 400,
            y: mouse.y - 300,
            z: 10
        });
    }
});

function spawnAsteroid() {
    asteroids.push({
        x: Math.random() * 1000 - 500,
        y: Math.random() * 800 - 400,
        z: 800,
        size: Math.random() * 20 + 20,
        speed: Math.random() * 3 + 5 + (score * 0.05)
    });
}

function resetGame() {
    score = 0;
    gameOver = false;
    lasers.length = 0;
    asteroids.length = 0;
    document.getElementById('current-score').innerText = score;
    document.getElementById('gameover-screen').style.display = 'none';
}

// --- Main Loop Engine ---
function animate() {
    requestAnimationFrame(animate);

    // Hard render backdrop to stop flickering or black dropouts
    ctx.fillStyle = '#050510';
    ctx.fillRect(0, 0, 800, 600);

    // Center point anchor matrix offsets
    const cx = 400;
    const cy = 300;

    // 1. Draw Starfield
    ctx.fillStyle = '#ffffff';
    for (let i = 0; i < stars.length; i++) {
        let s = stars[i];
        s.z -= 3;
        if (s.z <= 0) s.z = 800;

        let sx = (s.x / s.z) * 400 + cx;
        let sy = (s.y / s.z) * 400 + cy;
        let size = (1 - s.z / 800) * 3;

        if (sx >= 0 && sx <= 800 && sy >= 0 && sy <= 600) {
            ctx.fillRect(sx, sy, size, size);
        }
    }

    // 2. Process Lasers
    ctx.fillStyle = '#ff0055';
    for (let i = lasers.length - 1; i >= 0; i--) {
        let l = lasers[i];
        l.z += 15;

        let lx = (l.x / l.z) * 400 + cx;
        let ly = (l.y / l.z) * 400 + cy;
        let lRadius = (1 - l.z / 800) * 10;
        if (lRadius < 1) lRadius = 1;

        ctx.beginPath();
        ctx.arc(lx, ly, lRadius, 0, Math.PI * 2);
        ctx.fill();

        if (l.z > 800) {
            lasers.splice(i, 1);
        }
    }

    // 3. Spawning Timer Engine
    if (!gameOver) {
        spawnTimer++;
        if (spawnTimer > 25) {
            spawnAsteroid();
            spawnTimer = 0;
        }
    }

    // 4. Process Asteroids & Collision Matrix
    for (let i = asteroids.length - 1; i >= 0; i--) {
        let a = asteroids[i];
        a.z -= a.speed;

        let ax = (a.x / a.z) * 400 + cx;
        let ay = (a.y / a.z) * 400 + cy;
        let aRadius = (400 / a.z) * a.size;

        if (a.z > 0) {
            // Draw Asteroid
            ctx.fillStyle = '#3a3a45';
            ctx.strokeStyle = '#656575';
            ctx.lineWidth = 2;
            ctx.beginPath();
            ctx.arc(ax, ay, Math.max(1, aRadius), 0, Math.PI * 2);
            ctx.fill();
            ctx.stroke();

            // Laser Collision
            for (let j = lasers.length - 1; j >= 0; j--) {
                let l = lasers[j];
                let dist2D = Math.hypot((l.x / l.z) * 400 + cx - ax, (l.y / l.z) * 400 + cy - ay);
                if (dist2D < aRadius + 5 && a.z > 50) {
                    asteroids.splice(i, 1);
                    lasers.splice(j, 1);
                    score += 10;
                    document.getElementById('current-score').innerText = score;
                    break;
                }
            }

            // Player Collision (Locks hitboxes directly onto mouse positions when objects reach Z plane)
            if (!gameOver && a.z < 25) {
                let playerDist = Math.hypot(mouse.x - ax, mouse.y - ay);
                if (playerDist < aRadius + 20) {
                    gameOver = true;
                    document.getElementById('final-score').innerText = `Final Score: ${score}`;
                    if (score > highScore) {
                        highScore = score;
                        document.getElementById('high-score-hud').innerText = highScore;
                        document.getElementById('high-score-notice').style.display = 'block';
                    } else {
                        document.getElementById('high-score-notice').style.display = 'none';
                    }
                    document.getElementById('gameover-screen').style.display = 'flex';
                }
            }
        }

        if (a.z <= 0) {
            asteroids.splice(i, 1);
        }
    }

    // 5. Draw Player Crosshair & Spaceship
    if (!gameOver) {
        ctx.fillStyle = '#00ffcc';
        ctx.strokeStyle = '#0088cc';
        ctx.lineWidth = 3;

        ctx.beginPath();
        ctx.moveTo(mouse.x, mouse.y - 15);
        ctx.lineTo(mouse.x - 20, mouse.y + 15);
        ctx.lineTo(mouse.x, mouse.y + 5);
        ctx.lineTo(mouse.x + 20, mouse.y + 15);
        ctx.closePath();
        ctx.fill();
        ctx.stroke();

        // Crosshairs
        ctx.strokeStyle = 'rgba(0, 255, 204, 0.25)';
        ctx.lineWidth = 1;
        ctx.beginPath();
        ctx.arc(mouse.x, mouse.y, 25, 0, Math.PI * 2);
        ctx.moveTo(mouse.x - 35, mouse.y); ctx.lineTo(mouse.x + 35, mouse.y);
        ctx.moveTo(mouse.x, mouse.y - 35); ctx.lineTo(mouse.x, mouse.y + 35);
        ctx.stroke();
    }
}

// Spark up loop
animate();
</script>
</body>
</html>
