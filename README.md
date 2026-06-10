<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D Space Shooter</title>
    <style>
        html, body {
            margin: 0;
            padding: 0;
            width: 100%;
            height: 100%;
            overflow: hidden;
            background-color: #020208;
            font-family: 'Courier New', Courier, monospace;
            user-select: none;
        }
        canvas {
            display: block;
            position: absolute;
            top: 0;
            left: 0;
            width: 100vw;
            height: 100vh;
            z-index: 1;
        }
        #hud {
            position: absolute;
            top: 20px;
            left: 20px;
            color: #00ffcc;
            font-size: 24px;
            font-weight: bold;
            pointer-events: none;
            text-shadow: 0 0 8px rgba(0,255,220,0.6);
            z-index: 10;
        }
        #gameover-screen {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            color: #ff0055;
            text-align: center;
            display: none;
            background: rgba(5, 5, 15, 0.95);
            padding: 35px;
            border: 2px solid #ff0055;
            border-radius: 10px;
            box-shadow: 0 0 25px rgba(255, 0, 85, 0.5);
            z-index: 20;
        }
        #gameover-screen h1 { font-size: 50px; margin: 0 0 10px 0; }
        #gameover-screen p { color: #fff; font-size: 20px; margin: 5px 0; }
    </style>
</head>
<body>

    <div id="hud">
        SCORE: <span id="current-score">0</span><br>
        HIGH SCORE: <span id="high-score-hud">0</span>
    </div>
    
    <div id="gameover-screen">
        <h1>GAME OVER</h1>
        <p id="final-score">Final Score: 0</p>
        <p id="high-score-notice" style="color: #00ffcc; font-weight: bold; display: none;">NEW HIGH SCORE!</p>
        <p style="font-size: 16px; color: #888; margin-top: 20px; font-weight: bold;">CLICK ANYWHERE TO RESTART</p>
    </div>

    <canvas id="gameCanvas"></canvas>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

// Forces absolute pixel dimensions based on the window size
function resize() {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
}
window.addEventListener('resize', resize);
resize();

// --- Game Variables ---
let score = 0;
let highScore = 0; 
let gameOver = false;
let spawnTimer = 0;

const mouse = { x: window.innerWidth / 2, y: window.innerHeight / 2 };
const stars = [];
const lasers = [];
const asteroids = [];

// Initialize Starfield Depth Matrix
for (let i = 0; i < 200; i++) {
    stars.push({
        x: (Math.random() - 0.5) * 2000,
        y: (Math.random() - 0.5) * 2000,
        z: Math.random() * 1000
    });
}

// Track inputs relative to actual screen
window.addEventListener('mousemove', (e) => {
    mouse.x = e.clientX;
    mouse.y = e.clientY;
});

window.addEventListener('click', () => {
    if (gameOver) {
        resetGame();
    } else {
        lasers.push({ x: mouse.x, y: mouse.y, z: 10 });
    }
});

function spawnAsteroid() {
    asteroids.push({
        x: (Math.random() - 0.5) * canvas.width * 2,
        y: (Math.random() - 0.5) * canvas.height * 2,
        z: 1000,
        size: Math.random() * 30 + 20,
        speed: Math.random() * 4 + 5 + (score * 0.1)
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

// --- Game Loop ---
function animate() {
    requestAnimationFrame(animate);

    // Dynamic fallback safeguard: If canvas loses size, force recalculation
    if (canvas.width === 0 || canvas.height === 0) {
        resize();
    }

    // Paint Background Space
    ctx.fillStyle = '#020208';
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    const cx = canvas.width / 2;
    const cy = canvas.height / 2;

    // 1. Render Starfield 
    ctx.fillStyle = '#ffffff';
    for (let i = 0; i < stars.length; i++) {
        let s = stars[i];
        s.z -= 4; 
        if (s.z <= 0) s.z = 1000; 

        let sx = (s.x / s.z) * cx + cx;
        let sy = (s.y / s.z) * cy + cy;
        let size = (1 - s.z / 1000) * 3;

        if (sx >= 0 && sx <= canvas.width && sy >= 0 && sy <= canvas.height) {
            ctx.fillRect(sx, sy, size, size);
        }
    }

    // 2. Render Lasers
    ctx.fillStyle = '#ff0055';
    for (let i = lasers.length - 1; i >= 0; i--) {
        let l = lasers[i];
        l.z += 20; 

        let lRadius = (1 - l.z / 1000) * 8;
        if (lRadius < 1) lRadius = 1;

        ctx.beginPath();
        ctx.arc(l.x, l.y, lRadius, 0, Math.PI * 2);
        ctx.fill();

        if (l.z > 1000) {
            lasers.splice(i, 1);
        }
    }

    // 3. Render Asteroids
    if (!gameOver) {
        spawnTimer++;
        let spawnRate = Math.max(12, 40 - score * 0.5);
        if (spawnTimer > spawnRate) {
            spawnAsteroid();
            spawnTimer = 0;
        }
    }

    for (let i = asteroids.length - 1; i >= 0; i--) {
        let a = asteroids[i];
        a.z -= a.speed; 

        let ax = (a.x / a.z) * cx + cx;
        let ay = (a.y / a.z) * cy + cy;
        let aSize = (1 - a.z / 1000) * a.size * 2;

        if (aSize < 0) aSize = 0;

        if (a.z > 0) {
            ctx.fillStyle = '#44444a';
            ctx.strokeStyle = '#666670';
            ctx.lineWidth = 2;
            ctx.beginPath();
            ctx.arc(ax, ay, aSize, 0, Math.PI * 2);
            ctx.fill();
            ctx.stroke();

            // Laser hit checks
            for (let j = lasers.length - 1; j >= 0; j--) {
                let l = lasers[j];
                let dist = Math.hypot(l.x - ax, l.y - ay);
                if (dist < aSize && a.z > 100) { 
                    asteroids.splice(i, 1);
                    lasers.splice(j, 1);
                    score += 10;
                    document.getElementById('current-score').innerText = score;
                    break;
                }
            }

            // Player crash check
            if (a.z < 30) {
                let playerDist = Math.hypot(mouse.x - ax, mouse.y - ay);
                if (playerDist < aSize + 20) {
                    gameOver = true;
                    document.getElementById('final-score').innerText = `Final Score: ${score}`;
                    if (score > highScore) {
                        highScore = score;
                        document.getElementById('high-score-hud').innerText = highScore;
                        document.getElementById('high-score-notice').style.display = 'block';
                    } else {
                        document.getElementById('high-score-notice').style.display = 'none';
                    }
                    document.getElementById('gameover-screen').style.display = 'block';
                }
            }
        }

        if (a.z <= 0) {
            asteroids.splice(i, 1);
        }
    }

    // 4. Render Player Craft
    if (!gameOver) {
        ctx.fillStyle = '#00ffcc';
        ctx.strokeStyle = '#0088cc';
        ctx.lineWidth = 3;

        ctx.beginPath();
        ctx.moveTo(mouse.x, mouse.y - 15);
        ctx.lineTo(mouse.x - 25, mouse.y + 15);
        ctx.lineTo(mouse.x, mouse.y + 5);
        ctx.lineTo(mouse.x + 25, mouse.y + 15);
        ctx.closePath();
        ctx.fill();
        ctx.stroke();
        
        ctx.strokeStyle = 'rgba(0, 255, 204, 0.15)';
        ctx.lineWidth = 1;
        ctx.beginPath();
        ctx.arc(mouse.x, mouse.y, 35, 0, Math.PI*2);
        ctx.stroke();
    }
}

// Spark loop engine
animate();
</script>
</body>
</html>
