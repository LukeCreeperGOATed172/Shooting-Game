<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D Cockpit Space Shooter</title>
    <style>
        html, body {
            margin: 0;
            padding: 0;
            width: 100%;
            height: 100%;
            overflow: hidden;
            background-color: #03030c;
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
            background: rgba(4, 4, 12, 0.95);
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
        <h1>SHIP DESTROYED</h1>
        <p id="final-score">Final Score: 0</p>
        <p id="high-score-notice" style="color: #00ffcc; font-weight: bold; display: none;">NEW HIGH SCORE!</p>
        <p style="font-size: 16px; color: #888; margin-top: 20px; font-weight: bold;">CLICK ANYWHERE TO RESTART</p>
    </div>

    <canvas id="gameCanvas"></canvas>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

function resize() {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
}
window.addEventListener('resize', resize);
resize();

// --- Game Settings & State ---
let score = 0;
let highScore = 0; 
let gameOver = false;
let spawnTimer = 0;

// FOV and perspective warp settings
const FOV = 400; 
const PLAYER_Z = 20; // The depth plane where your player ship physically exists

const mouse = { x: 0, y: 0 };
const stars = [];
const lasers = [];
const asteroids = [];

// Generate stars with localized 3D coordinate vectors
for (let i = 0; i < 300; i++) {
    stars.push({
        x: (Math.random() - 0.5) * 3000,
        y: (Math.random() - 0.5) * 2000,
        z: Math.random() * 1000
    });
}

window.addEventListener('mousemove', (e) => {
    // Translate mouse screen pixels into actual 3D virtual coordinates
    mouse.x = (e.clientX - canvas.width / 2) * (PLAYER_Z / FOV);
    mouse.y = (e.clientY - canvas.height / 2) * (PLAYER_Z / FOV);
});

window.addEventListener('click', () => {
    if (gameOver) {
        resetGame();
    } else {
        // Fire laser directly out of the ship's 3D positioning
        lasers.push({
            x: mouse.x,
            y: mouse.y,
            z: PLAYER_Z,
            speed: 25
        });
    }
});

function spawnAsteroid() {
    // Spawns targets wildly distributed across real far-off 3D space dimensions
    asteroids.push({
        x: (Math.random() - 0.5) * 1500,
        y: (Math.random() - 0.5) * 1000,
        z: 1000, 
        size: Math.random() * 25 + 20, 
        speed: Math.random() * 4 + 6 + (score * 0.08)
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

// --- Main Engine Loop ---
function animate() {
    requestAnimationFrame(animate);

    // Render dark space vacuum background
    ctx.fillStyle = '#03030c';
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    // Calculate center origin viewpoint matrix
    const cx = canvas.width / 2;
    const cy = canvas.height / 2;

    // 1. Render 3D Background Stars
    ctx.fillStyle = '#ffffff';
    for (let i = 0; i < stars.length; i++) {
        let s = stars[i];
        s.z -= 5; 
        if (s.z <= 0) s.z = 1000; 

        // Matrix transformations project 3D coordinate properties into flat screen values
        let sx = (s.x / s.z) * FOV + cx;
        let sy = (s.y / s.z) * FOV + cy;
        let scale = (1 - s.z / 1000) * 2.5;

        if (sx >= 0 && sx <= canvas.width && sy >= 0 && sy <= canvas.height) {
            ctx.fillRect(sx, sy, scale, scale);
        }
    }

    // 2. Update and Track Lasers
    for (let i = lasers.length - 1; i >= 0; i--) {
        let l = lasers[i];
        l.z += l.speed; // Fly deep into the horizon

        let lx = (l.x / l.z) * FOV + cx;
        let ly = (l.y / l.z) * FOV + cy;
        let lRadius = (1 - l.z / 1000) * 12;
        if (lRadius < 1) lRadius = 1;

        ctx.fillStyle = '#ff0055';
        ctx.shadowBlur = 10;
        ctx.shadowColor = '#ff0055';
        ctx.beginPath();
        ctx.arc(lx, ly, lRadius, 0, Math.PI * 2);
