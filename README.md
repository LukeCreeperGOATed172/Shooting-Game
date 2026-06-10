<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Pseudo-3D Space Shooter</title>
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
            width: 100%;
            height: 100%;
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

// Resize canvas to fit window
function resize() {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
}
window.addEventListener('resize', resize);
resize();

// --- Game Variables ---
let score = 0;
let highScore = 0; // Local variable bypasses Tracking Prevention storage blocks
let gameOver = false;
let spawnTimer = 0;

const mouse = { x: canvas.width / 2, y: canvas.height / 2 };
const stars = [];
const lasers = [];
const asteroids = [];

// Initialize 3D depth stars
for (let i = 0; i < 200; i++) {
    stars.push({
        x: (Math.random() - 0.5) * canvas.width * 2,
        y: (Math.random() - 0.5) * canvas.height * 2,
        z: Math.random() * canvas.width
    });
}

// --- Inputs ---
window.addEventListener('mousemove', (e) => {
    mouse.x = e.clientX;
    mouse.y = e.clientY;
});

window.addEventListener('click', () => {
    if (gameOver) {
        resetGame();
    } else {
        // Fire laser from ship's current screen position
        lasers.push({
            x: mouse.x,
            y: mouse.y,
            z: 10 // Starts close, flies deep into distance
        });
    }
});

function spawnAsteroid() {
    // Spawns deep in the distance (z = canvas.width), flying toward screen
    asteroids.push({
        x: (Math.random() - 0.5) * canvas.width * 2,
        y: (Math.random() - 0.5) * canvas.height * 2,
        z: canvas.width,
        size: Math.random() * 30 + 20,
        speed: Math.random() * 4 + 4 + (score * 0.1)
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

// --- Core Game Loop ---
function animate() {
    requestAnimationFrame(animate);

    // Clear Screen
    ctx.fillStyle = '#020208';
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    const cx = canvas.width / 2;
    const cy = canvas.height / 2;

    // 1. Draw Starfield (Pseudo-3D Projection)
    ctx.fillStyle = '#ffffff';
    for (let i = 0; i < stars.length; i++) {
        let s = stars[i];
        s.z -= 2; // Move stars closer
        if (s.z <= 0) s.z = canvas.width; // Reset deep

        // Project 3D coordinates to 2D Screen
        let sx = (s.x / s.z) * cx + cx;
        let sy = (s.y / s.z) * cy + cy;
        let size = (1 - s.z / canvas.width) * 3;

        if (sx >= 0 && sx <= canvas.width && sy >= 0 && sy <= canvas.height) {
            ctx.fillRect(sx, sy, size, size);
        }
    }

    // 2. Process Lasers (Move into background depth)
    for (let i = lasers.length - 1; i >= 0; i--) {
        let l = lasers[i];
        l.z += 15; // Move away from player

        // Project to screen
        let lRadius = (1 - l.z / canvas.width) * 8;
        if (lRadius < 1) lRadius = 1;

        // Draw laser dot traveling outward
        ctx.fillStyle = '#ff0055';
        ctx.beginPath();
        ctx.arc(l.x, l.y, lRadius, 0, Math.PI * 2);
        ctx.fill();

        if (l.z > canvas.width) {
            lasers.splice(i, 1);
        }
    }

    // 3. Process Asteroids (Move forward toward player)
    if (!gameOver) {
        spawnTimer++;
        let spawnRate = Math.max(15, 45 - score * 0.5);
        if (spawnTimer > spawnRate) {
            spawnAsteroid();
            spawnTimer = 0;
        }
    }

    for (let i = asteroids.length - 1; i >= 0; i--) {
        let a = asteroids[i];
        a.z -= a.speed; // Move closer

        // 3D Perspective Projection Formulas
        let ax = (a.x / a.z) * cx + cx;
        let ay = (a.y / a.z) * cy + cy;
        let aSize = (1 - a.z / canvas.width) * a.size * 2;

        if (aSize < 0) aSize = 0;

        // Draw Asteroid (Rock outline filled with grey)
        if (a.z > 0) {
            ctx.fillStyle = '#55555c';
            ctx.strokeStyle = '#888893';
            ctx.lineWidth = 2;
            ctx.beginPath();
            ctx.arc(ax, ay, aSize, 0, Math.PI * 2);
            ctx.fill();
            ctx.stroke();

            // Collision check: Lasers hitting Asteroid
            for (let j = lasers.length - 1; j >= 0; j--) {
                let l = lasers[j];
                // Calculate distance on the flat 2D plane
                let dist = Math.hypot(l.x - ax, l.y - ay);
                if (dist < aSize && a.z > 100) { // Hit registered
                    asteroids.splice(i, 1);
                    lasers.splice(j, 1);
                    score += 10;
                    document.getElementById('current-score').innerText = score;
                    break;
                }
            }

            // Collision check: Asteroid hits player
