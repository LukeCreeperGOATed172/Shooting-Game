<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D Space Shooter - Absolute Alignment Edition</title>
    <style>
        html, body {
            margin: 0;
            padding: 0;
            width: 100%;
            height: 100%;
            overflow: hidden;
            background-color: #000;
            font-family: 'Courier New', Courier, monospace;
            user-select: none;
        }
        /* Forces the game window to ignore GitHub's default layout padding rules */
        canvas {
            position: absolute;
            top: 0;
            left: 0;
            width: 100vw !important;
            height: 100vh !important;
            display: block;
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
            line-height: 1.5;
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
            pointer-events: auto;
            background: rgba(0, 0, 0, 0.9);
            padding: 30px;
            border: 2px solid #ff0055;
            border-radius: 10px;
            box-shadow: 0 0 20px rgba(255, 0, 85, 0.5);
            z-index: 20;
            cursor: pointer;
        }
        #gameover-screen h1 { font-size: 50px; margin: 0 0 10px 0; }
        #gameover-screen p { color: #fff; font-size: 20px; margin: 5px 0; }
    </style>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
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
        <p style="font-size: 16px; color: #888; margin-top: 20px;">Click anywhere to restart</p>
    </div>

<script>
// --- Core Setup ---
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// --- Lighting Engine ---
const ambientLight = new THREE.AmbientLight(0x444444);
scene.add(ambientLight);

const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
directionalLight.position.set(5, 10, 7);
scene.add(directionalLight);

// --- State Trackers ---
let score = 0;
let highScore = 0; 
let gameOver = false;
let asteroidTimer = null;

const lasers = [];
const asteroids = [];
const mouse = new THREE.Vector2();
const raycaster = new THREE.Raycaster();

// Anchor an invisible plane at Z = -5. The ship will slide along this flat target grid.
const movementPlane = new THREE.Plane(new THREE.Vector3(0, 0, 1), 5);
const targetWorldPosition = new THREE.Vector3();

// --- Background Assembly ---
const starGeometry = new THREE.BufferGeometry();
const starCount = 400;
const starPositions = new Float32Array(starCount * 3);
for(let i = 0; i < starCount * 3; i += 3) {
    starPositions[i] = (Math.random() - 0.5) * 120;       
    starPositions[i+1] = (Math.random() - 0.5) * 120;     
    starPositions[i+2] = -Math.random() * 200;            
}
starGeometry.setAttribute('position', new THREE.BufferAttribute(starPositions, 3));
const starMaterial = new THREE.PointsMaterial({ color: 0xffffff, size: 0.5 });
const starField = new THREE.Points(starGeometry, starMaterial);
scene.add(starField);

// --- Construct Player Ship Mesh ---
const playerGroup = new THREE.Group();

const hullGeo = new THREE.ConeGeometry(0.5, 2.2, 4);
hullGeo.rotateX(Math.PI / 2); 
const hullMat = new THREE.MeshStandardMaterial({ color: 0x00ffcc, roughness: 0.2 });
const hull = new THREE.Mesh(hullGeo, hullMat);
playerGroup.add(hull);

const wingGeo = new THREE.BoxGeometry(2.4, 0.08, 0.7);
const wingMat = new THREE.MeshStandardMaterial({ color: 0x0088cc });
const wings = new THREE.Mesh(wingGeo, wingMat);
wings.position.set(0, -0.15, -0.2);
playerGroup.add(wings);

playerGroup.position.set(0, 0, -5); 
scene.add(playerGroup);

// Anchor Camera Properties
camera.position.set(0, 3.0, 3.5);
camera.lookAt(0, 0, -8);

// --- Controls Matrix ---
window.addEventListener('mousemove', (e) => {
    // Calculates position directly bounded to renderer dimensions rather than standard window windowing
    const rect = renderer.domElement.getBoundingClientRect();
    mouse.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
});

window.addEventListener('click', () => {
    if (gameOver) {
        resetGame();
        return;
    }
    spawnLaser();
});

// --- Functional Routines ---
function spawnLaser() {
    if (gameOver) return;
    const laserGeo = new THREE.CylinderGeometry(0.04, 0.04, 0.8, 4);
    laserGeo.rotateX(Math.PI / 2);
    const laserMat = new THREE.MeshBasicMaterial({ color: 0xff0055 });
    const laser = new THREE.Mesh(laserGeo, laserMat);
    
    laser.position.copy(playerGroup.position);
    laser.position.z -= 1.2; 
    
    scene.add(laser);
    lasers.push(laser);
}

function spawnAsteroid() {
    if (gameOver) return;

    const radius = Math.random() * 0.5 + 0.4;
    const asteroidGeo = new THREE.DodecahedronGeometry(radius, 1);
    const asteroidMat = new THREE.MeshStandardMaterial({ color: 0x666666, roughness: 0.9 });
    const asteroid = new THREE.Mesh(asteroidGeo, asteroidMat);

    asteroid.position.set(
        (Math.random() - 0.5) * 18,
        (Math.random() - 0.5) * 12,
        -60 
    );
    
    asteroid.userData = {
        rotX: Math.random() * 0.02,
        rotY: Math.random() * 0.02,
        speed: Math.random() * 0.12 + 0.12 + (score * 0.003) 
    };

    scene.add(asteroid);
    asteroids.push(asteroid);

    asteroidTimer = setTimeout(spawnAsteroid, Math.max(200, 1000 - score * 5));
}

function handleGameOver() {
    gameOver = true;
    clearTimeout(asteroidTimer); 
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

function resetGame() {
    clearTimeout(asteroidTimer);

    lasers.forEach(l => scene.remove(l));
    asteroids.forEach(a => scene.remove(a));
    lasers.length = 0;
    asteroids.length = 0;
    
    score = 0;
    gameOver = false;
    document.getElementById('current-score').innerText = score;
    document.getElementById('gameover-screen').style.display = 'none';
    
    playerGroup.position.set(0, 0, -5);
    
    spawnAsteroid();
}

// --- Frame Engine Loop ---
function animate() {
    requestAnimationFrame(animate);

    // FOOLPROOF ALIGNMENT: Shoot ray out from camera position to calculate real intersections
    raycaster.setFromCamera(mouse, camera);
    raycaster.ray.intersectPlane(movementPlane, targetWorldPosition);
