<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D Space Shooter</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background-color: #000;
            font-family: 'Courier New', Courier, monospace;
            user-select: none;
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
            background: rgba(0, 0, 0, 0.85);
            padding: 30px;
            border: 2px solid #ff0055;
            border-radius: 10px;
            box-shadow: 0 0 20px rgba(255, 0, 85, 0.4);
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
// --- Setup Scene, Camera, and Renderer ---
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// --- Lighting ---
const ambientLight = new THREE.AmbientLight(0x333333);
scene.add(ambientLight);

const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
directionalLight.position.set(5, 10, 7);
scene.add(directionalLight);

// --- Game Variables ---
let score = 0;
let highScore = localStorage.getItem("spaceracer_highscore") ? parseInt(localStorage.getItem("spaceracer_highscore")) : 0;
let gameOver = false;
let isLoopRunning = false; // Safety lock flag to prevent WebGL blackouts
let asteroidTimer = null;

const lasers = [];
const asteroids = [];
const mouse = { x: 0, y: 0 };

// Initialize High Score HUD
document.getElementById('high-score-hud').innerText = highScore;

// --- Create Starfield Background ---
const starGeometry = new THREE.BufferGeometry();
const starCount = 500;
const starPositions = new Float32Array(starCount * 3);
for(let i = 0; i < starCount * 3; i += 3) {
    starPositions[i] = (Math.random() - 0.5) * 100;       
    starPositions[i+1] = (Math.random() - 0.5) * 100;     
    starPositions[i+2] = -Math.random() * 200;            
}
starGeometry.setAttribute('position', new THREE.BufferAttribute(starPositions, 3));
const starMaterial = new THREE.PointsMaterial({ color: 0xffffff, size: 0.5 });
const starField = new THREE.Points(starGeometry, starMaterial);
scene.add(starField);

// --- Create Player Ship ---
const playerGroup = new THREE.Group();

const hullGeo = new THREE.ConeGeometry(0.6, 2.5, 4);
hullGeo.rotateX(Math.PI / 2); 
const hullMat = new THREE.MeshStandardMaterial({ color: 0x00ffcc, roughness: 0.2 });
const hull = new THREE.Mesh(hullGeo, hullMat);
playerGroup.add(hull);

const wingGeo = new THREE.BoxGeometry(2.5, 0.1, 0.8);
const wingMat = new THREE.MeshStandardMaterial({ color: 0x0088cc });
const wings = new THREE.Mesh(wingGeo, wingMat);
wings.position.set(0, -0.2, -0.2);
playerGroup.add(wings);

playerGroup.position.set(0, 0, -5); 
scene.add(playerGroup);

// Camera layout adjusted to align better over the plane coordinates
camera.position.set(0, 2.5, 2.5);
camera.lookAt(0, 0, -10);

// --- Input Handling ---
window.addEventListener('mousemove', (e) => {
    // This fetches the exact pixel bounds of the canvas, ignoring body padding or margins
    const rect = renderer.domElement.getBoundingClientRect();
    
    // Calculates the mouse position strictly relative to the game viewport boundaries
    const canvasX = e.clientX - rect.left;
    const canvasY = e.clientY - rect.top;
    
    // Maps the precise coordinates to Three.js normalized 3D space (-1 to 1)
    mouse.x = (canvasX / rect.width) * 2 - 1;
    mouse.y = -(canvasY / rect.height) * 2 + 1;
});

// --- Game Actions ---
function spawnLaser() {
    if (gameOver) return;
    const laserGeo = new THREE.CylinderGeometry(0.05, 0.05, 0.8, 4);
    laserGeo.rotateX(Math.PI / 2);
    const laserMat = new THREE.MeshBasicMaterial({ color: 0xff0055 });
    const laser = new THREE.Mesh(laserGeo, laserMat);
    
    laser.position.copy(playerGroup.position);
    laser.position.z -= 1.5; 
    
    scene.add(laser);
    lasers.push(laser);
}

function spawnAsteroid() {
    if (gameOver) return;

    const radius = Math.random() * 0.6 + 0.4;
    const asteroidGeo = new THREE.DodecahedronGeometry(radius, 1);
    const asteroidMat = new THREE.MeshStandardMaterial({ color: 0x666666, roughness: 0.9 });
    const asteroid = new THREE.Mesh(asteroidGeo, asteroidMat);

    asteroid.position.set(
        (Math.random() - 0.5) * 16,
        (Math.random() - 0.5) * 11,
        -50 
    );
    
    asteroid.userData = {
        rotX: Math.random() * 0.02,
        rotY: Math.random() * 0.02,
        speed: Math.random() * 0.15 + 0.1 + (score * 0.002) 
    };

    scene.add(asteroid);
    asteroids.push(asteroid);

    asteroidTimer = setTimeout(spawnAsteroid, Math.max(200, 1000 - score * 5));
}

function checkCollision(mesh1, mesh2, distanceThreshold) {
    return mesh1.position.distanceTo(mesh2.position) < distanceThreshold;
}

function handleGameOver() {
    gameOver = true;
    clearTimeout(asteroidTimer); // Clear async operations instantly on death
    document.getElementById('final-score').innerText = `Final Score: ${score}`;
    
    if (score > highScore) {
        highScore = score;
        localStorage.setItem("spaceracer_highscore", highScore);
        document.getElementById('high-score-hud').innerText = highScore;
        document.getElementById('high-score-notice').style.display = 'block';
    } else {
        document.getElementById('high-score-notice').style.display = 'none';
    }
    
    document.getElementById('gameover-screen').style.display = 'block';
}

function resetGame() {
    clearTimeout(asteroidTimer);

    // Clean old elements out of the 3D pipeline memory
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
    // Notice: We don't execute animate() here. The existing loop picks up the state change!
}

// --- Main Animation Loop ---
function animate() {
    requestAnimationFrame(animate);
    isLoopRunning = true;

    // PERSPECTIVE CORRECTION: Multipliers updated + vertical adjustment added (+1.2)
    // This shifts the ship up so it tracks directly under your physical mouse position
    const targetX = mouse.x * 7.5;
    const targetY = (mouse.y * 5.2) + 1.2; 
    
    // Aesthetic bank rolling on turn
    playerGroup.rotation.z = -(targetX - playerGroup.position.x) * 0.15;
    playerGroup.rotation.y = (targetX - playerGroup.position.x) * 0.08;

    playerGroup.position.x = targetX;
    playerGroup.position.y = targetY;

    // Starfield movement
    const positions = starField.geometry.attributes.position.array;
    for(let i = 2; i < positions.length; i += 3) {
        positions[i] += 0.5; 
        if (positions[i] > 0) {
            positions[i] = -200; 
        }
    }
    starField.geometry.attributes.position.needsUpdate = true;

    // Update Lasers
    for (let i = lasers.length - 1; i >= 0; i--) {
        lasers[i].position.z -= 0.7; 

        if (lasers[i].position.z < -60) {
            scene.remove(lasers[i]);
            lasers.splice(i, 1);
        }
    }

    // Update Asteroids
    for (let i = asteroids.length - 1; i >= 0; i--) {
        const ast = asteroids[i];
        ast.position.z += ast.userData.speed; 
        ast.rotation.x += ast.userData.rotX;
        ast.rotation.y += ast.userData.rotY;

        // Collision Check
        if (!gameOver && checkCollision(playerGroup, ast, 1.1)) {
            handleGameOver();
        }

        // Laser Hits
        for (let j = lasers.length - 1; j >= 0; j--) {
            if (checkCollision(lasers[j], ast, 0.9)) {
                scene.remove(ast);
                scene.remove(lasers[j]);
                asteroids.splice(i, 1);
                lasers.splice(j, 1);
                
                score += 10;
                document.getElementById('current-score').innerText = score;
                break;
            }
        }

        if (ast && ast.position.z > 2) {
            scene.remove(ast);
            asteroids.splice(i, 1);
        }
    }

    renderer.render(scene, camera);
}

window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// Run
spawnAsteroid();
animate();
</script>
</body>
</html>
