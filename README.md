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
            pointer-events: none;
        }
        #gameover-screen h1 { font-size: 50px; margin: 0; }
        #gameover-screen p { color: #fff; font-size: 20px; }
    </style>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
</head>
<body>

    <div id="hud">SCORE: 0</div>
    <div id="gameover-screen">
        <h1>GAME OVER</h1>
        <p id="final-score">Final Score: 0</p>
        <p style="font-size: 16px; color: #888;">Click anywhere to restart</p>
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
let gameOver = false;
const lasers = [];
const asteroids = [];
const mouse = { x: 0, y: 0 };

// --- Create Starfield Background ---
const starGeometry = new THREE.BufferGeometry();
const starCount = 500;
const starPositions = new Float32Array(starCount * 3);
for(let i = 0; i < starCount * 3; i += 3) {
    starPositions[i] = (Math.random() - 0.5) * 100;       // X
    starPositions[i+1] = (Math.random() - 0.5) * 100;     // Y
    starPositions[i+2] = -Math.random() * 200;            // Z (In front of camera)
}
starGeometry.setAttribute('position', new THREE.BufferAttribute(starPositions, 3));
const starMaterial = new THREE.PointsMaterial({ color: 0xffffff, size: 0.5 });
const starField = new THREE.Points(starGeometry, starMaterial);
scene.add(starField);

// --- Create Player Ship (Group of 3D Shapes) ---
const playerGroup = new THREE.Group();

// Main hull (Cone)
const hullGeo = new THREE.ConeGeometry(0.6, 2.5, 4);
hullGeo.rotateX(Math.PI / 2); // Point it forward along Z-axis
const hullMat = new THREE.MeshStandardMaterial({ color: 0x00ffcc, roughness: 0.2 });
const hull = new THREE.Mesh(hullGeo, hullMat);
playerGroup.add(hull);

// Wings (Box)
const wingGeo = new THREE.BoxGeometry(2.5, 0.1, 0.8);
const wingMat = new THREE.MeshStandardMaterial({ color: 0x0088cc });
const wings = new THREE.Mesh(wingGeo, wingMat);
wings.position.set(0, -0.2, -0.2);
playerGroup.add(wings);

playerGroup.position.set(0, 0, -5); // Position in front of camera
scene.add(playerGroup);

// Camera positioning behind the player
camera.position.set(0, 2, 2);
camera.lookAt(0, 0, -10);

// --- Input Handling ---
window.addEventListener('mousemove', (e) => {
    // Normalize mouse coordinates mapping to -1 to +1
    mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
});

window.addEventListener('click', () => {
    if (gameOver) {
        resetGame();
        return;
    }
    spawnLaser();
});

// --- Game Actions ---
function spawnLaser() {
    const laserGeo = new THREE.CylinderGeometry(0.05, 0.05, 0.8, 4);
    laserGeo.rotateX(Math.PI / 2);
    const laserMat = new THREE.MeshBasicMaterial({ color: 0xff0055 });
    const laser = new THREE.Mesh(laserGeo, laserMat);
    
    // Match player's current position
    laser.position.copy(playerGroup.position);
    // Push it slightly ahead of the ship
    laser.position.z -= 1.5; 
    
    scene.add(laser);
    lasers.push(laser);
}

function spawnAsteroid() {
    if (gameOver) return;

    // Distorted sphere geometry for an organic asteroid look
    const radius = Math.random() * 0.6 + 0.4;
    const asteroidGeo = new THREE.DodecahedronGeometry(radius, 1);
    const asteroidMat = new THREE.MeshStandardMaterial({ color: 0x666666, roughness: 0.9 });
    const asteroid = new THREE.Mesh(asteroidGeo, asteroidMat);

    // Spawn randomly ahead in the distance
    asteroid.position.set(
        (Math.random() - 0.5) * 12,
        (Math.random() - 0.5) * 8,
        -50 
    );
    
    // Add random rotation speeds
    asteroid.userData = {
        rotX: Math.random() * 0.02,
        rotY: Math.random() * 0.02,
        speed: Math.random() * 0.15 + 0.1 + (score * 0.002) // Speed scales up with score
    };

    scene.add(asteroid);
    asteroids.push(asteroid);

    // Dynamic spawn timers based on score
    setTimeout(spawnAsteroid, Math.max(200, 1000 - score * 5));
}

// Bounding Box Collision Check
function checkCollision(mesh1, mesh2, distanceThreshold) {
    return mesh1.position.distanceTo(mesh2.position) < distanceThreshold;
}

function resetGame() {
    score = 0;
    gameOver = false;
    document.getElementById('hud').innerText = `SCORE: ${score}`;
    document.getElementById('gameover-screen').style.display = 'none';
    
    // Clean arrays
    lasers.forEach(l => scene.remove(l));
    asteroids.forEach(a => scene.remove(a));
    lasers.length = 0;
    asteroids.length = 0;
    
    playerGroup.position.set(0, 0, -5);
    animate();
}

// --- Main Animation Loop ---
function animate() {
    if (gameOver) {
        document.getElementById('final-score').innerText = `Final Score: ${score}`;
        document.getElementById('gameover-screen').style.display = 'block';
        return; 
    }

    requestAnimationFrame(animate);

    // 1. Move Player smoothly towards target mouse coordinates
    const targetX = mouse.x * 6;
    const targetY = mouse.y * 4;
    playerGroup.position.x += (targetX - playerGroup.position.x) * 0.1;
    playerGroup.position.y += (targetY - playerGroup.position.y) * 0.1;

    // Subtle banked turning effect based on movement
    playerGroup.rotation.z = -(targetX - playerGroup.position.x) * 0.2;
    playerGroup.rotation.y = (targetX - playerGroup.position.x) * 0.1;

    // 2. Animate Starfield background (moving past player)
    const positions = starField.geometry.attributes.position.array;
    for(let i = 2; i < positions.length; i += 3) {
        positions[i] += 0.5; // Move star closer to camera
        if (positions[i] > 0) {
            positions[i] = -200; // Reset star back to distance
        }
    }
    starField.geometry.attributes.position.needsUpdate = true;

    // 3. Update Lasers
    for (let i = lasers.length - 1; i >= 0; i--) {
        lasers[i].position.z -= 0.6; // Fly forward

        // Remove distant lasers
        if (lasers[i].position.z < -60) {
            scene.remove(lasers[i]);
            lasers.splice(i, 1);
        }
    }

    // 4. Update Asteroids
    for (let i = asteroids.length - 1; i >= 0; i--) {
        const ast = asteroids[i];
        ast.position.z += ast.userData.speed; // Fly towards player
        ast.rotation.x += ast.userData.rotX;
        ast.rotation.y += ast.userData.rotY;

        // Player Collision check (Threshold roughly matching physical boundaries)
        if (checkCollision(playerGroup, ast, 1.0)) {
            gameOver = true;
        }

        // Laser Collisions check
        for (let j = lasers.length - 1; j >= 0; j--) {
            if (checkCollision(lasers[j], ast, 0.8)) {
                scene.remove(ast);
                scene.remove(lasers[j]);
                asteroids.splice(i, 1);
                lasers.splice(j, 1);
                
                score += 10;
                document.getElementById('hud').innerText = `SCORE: ${score}`;
                break;
            }
        }

        // Clean up passed asteroids
        if (ast && ast.position.z > 2) {
            scene.remove(ast);
            asteroids.splice(i, 1);
        }
    }

    renderer.render(scene, camera);
}

// Window resizing adjustment
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
