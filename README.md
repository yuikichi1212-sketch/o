<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>THE ULTIMATE VOXEL PVP - COMPLETE EDITION</title>
    <style>
        body { margin: 0; overflow: hidden; background: #87ceeb; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; }
        #crosshair {
            position: absolute; top: 50%; left: 50%; width: 14px; height: 14px;
            border: 2px solid rgba(255,255,255,0.8); transform: translate(-50%, -50%); pointer-events: none;
        }
        #ui {
            position: absolute; bottom: 30px; left: 50%; transform: translateX(-50%);
            display: flex; gap: 10px; background: rgba(0,0,0,0.7); padding: 15px; border-radius: 12px;
            border: 2px solid #444; backdrop-filter: blur(5px);
        }
        .slot {
            width: 60px; height: 60px; border: 2px solid #555; display: flex;
            flex-direction: column; justify-content: center; align-items: center;
            color: white; font-size: 10px; position: relative;
        }
        .slot.active { border-color: #00ffcc; box-shadow: 0 0 15px #00ffcc; }
        .count { position: absolute; bottom: 2px; right: 4px; font-weight: bold; }
        #status {
            position: absolute; top: 20px; left: 20px; color: white;
            background: rgba(0,0,0,0.6); padding: 15px; border-radius: 8px;
        }
        #hp-bar { width: 200px; height: 12px; background: #222; margin-top: 8px; border-radius: 6px; overflow: hidden; }
        #hp-fill { width: 100%; height: 100%; background: linear-gradient(90deg, #ff416c, #ff4b2b); transition: 0.2s; }
        #overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.8); color: white; display: flex;
            flex-direction: column; justify-content: center; align-items: center; cursor: pointer;
        }
    </style>
</head>
<body>
    <div id="crosshair"></div>
    <div id="status">
        <div>PLAYER STATUS</div>
        <div id="hp-bar"><div id="hp-fill"></div></div>
        <div style="margin-top:10px;">SCORE: <span id="score">0</span></div>
    </div>
    <div id="ui">
        <div class="slot active" id="s0"><span>SWORD</span><div class="count">∞</div></div>
        <div class="slot" id="s1"><span>PICKAXE</span><div class="count">∞</div></div>
        <div class="slot" id="s2"><span>BLOCK</span><div class="count" id="block-count">0</div></div>
    </div>
    <div id="overlay">
        <h1>ULTIMATE VOXEL PVP</h1>
        <p>クリックしてゲーム開始</p>
        <p style="font-size: 14px; color: #aaa;">WASD: 移動 | 左クリ: 攻撃・採掘 | 右クリ: 設置 | 1-3: 切替</p>
    </div>

    <script type="importmap">
        { "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js" } }
    </script>

    <script type="module">
        import * as THREE from 'three';

        // --- Core Setup ---
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x87ceeb);
        scene.fog = new THREE.Fog(0x87ceeb, 10, 80);

        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.shadowMap.enabled = true;
        document.body.appendChild(renderer.domElement);

        // --- Lights ---
        const ambient = new THREE.AmbientLight(0xffffff, 0.6);
        scene.add(ambient);
        const sun = new THREE.DirectionalLight(0xffffff, 1.2);
        sun.position.set(50, 80, 50);
        sun.castShadow = true;
        scene.add(sun);

        // --- Game State ---
        let hp = 100, score = 0, blockInv = 10;
        let activeSlot = 0; // 0:Sword, 1:Pickaxe, 2:Block
        let isAttacking = false;
        const keys = {};
        const enemies = [];
        const worldBlocks = [];

        // --- Realistic Terrain Generation ---
        const boxGeo = new THREE.BoxGeometry(1, 1, 1);
        const terrainGroup = new THREE.Group();
        for (let x = -30; x < 30; x++) {
            for (let z = -30; z < 30; z++) {
                const noise = Math.sin(x * 0.15) * 2.5 + Math.cos(z * 0.15) * 2.5;
                const h = Math.floor(noise + 5);
                for (let y = 0; y < h; y++) {
                    const color = y === h - 1 ? 0x55aa55 : (y > h - 3 ? 0x8b4513 : 0x777777);
                    const block = new THREE.Mesh(boxGeo, new THREE.MeshStandardMaterial({ color }));
                    block.position.set(x, y, z);
                    block.receiveShadow = true;
                    if (y === h - 1) block.castShadow = true;
                    terrainGroup.add(block);
                    worldBlocks.push(block);
                }
            }
        }
        scene.add(terrainGroup);

        // --- Player Model (Hand & Weapon) ---
        const handRoot = new THREE.Group();
        const arm = new THREE.Mesh(new THREE.BoxGeometry(0.3, 0.3, 1), new THREE.MeshStandardMaterial({ color: 0xffdbac }));
        arm.position.set(0.5, -0.4, -0.6);
        
        const sword = new THREE.Mesh(new THREE.BoxGeometry(0.1, 1.5, 0.3), new THREE.MeshStandardMaterial({ color: 0xdddddd, metalness: 0.8, roughness: 0.2 }));
        sword.position.set(0, 0.6, -0.4);
        sword.rotation.x = -Math.PI/4;
        
        const heldBlock = new THREE.Mesh(boxGeo, new THREE.MeshStandardMaterial({ color: 0x8b4513 }));
        heldBlock.scale.set(0.4, 0.4, 0.4);
        heldBlock.position.set(0, 0.2, -0.5);
        heldBlock.visible = false;

        arm.add(sword);
        arm.add(heldBlock);
        handRoot.add(arm);
        camera.add(handRoot);
        scene.add(camera);

        // --- CPU AI (Enemies) ---
        function spawnEnemy() {
            const bot = new THREE.Group();
            const body = new THREE.Mesh(new THREE.BoxGeometry(0.8, 1.8, 0.8), new THREE.MeshStandardMaterial({ color: 0xff3333 }));
            body.position.y = 0.9;
            bot.add(body);
            bot.position.set(Math.random()*40-20, 10, Math.random()*40-20);
            scene.add(bot);
            enemies.push({ mesh: bot, hp: 5, lastHit: 0 });
        }
        for(let i=0; i<4; i++) spawnEnemy();

        // --- Controls ---
        const overlay = document.getElementById('overlay');
        overlay.onclick = () => {
            document.body.requestPointerLock();
            overlay.style.display = 'none';
        };

        document.addEventListener('keydown', (e) => {
            keys[e.code] = true;
            if(e.code === 'Digit1') switchSlot(0);
            if(e.code === 'Digit2') switchSlot(1);
            if(e.code === 'Digit3') switchSlot(2);
        });
        document.addEventListener('keyup', (e) => keys[e.code] = false);

        function switchSlot(n) {
            activeSlot = n;
            document.querySelectorAll('.slot').forEach((s, i) => s.classList.toggle('active', i === n));
            sword.visible = (n === 0 || n === 1);
            heldBlock.visible = (n === 2);
        }

        // --- Combat & Interaction ---
        let yaw = 0, pitch = 0;
        document.addEventListener('mousemove', (e) => {
            if (document.pointerLockElement) {
                yaw -= e.movementX * 0.002;
                pitch -= e.movementY * 0.002;
                pitch = Math.max(-Math.PI/2.2, Math.min(Math.PI/2.2, pitch));
                camera.rotation.set(pitch, yaw, 0, 'YXZ');
            }
        });

        window.onmousedown = (e) => {
            if (!document.pointerLockElement) return;
            if (e.button === 0) performAttack(); // Left Click
            if (e.button === 2) placeBlock();    // Right Click
        };

        function performAttack() {
            if (isAttacking) return;
            isAttacking = true;
            
            // Animation
            const tl = setInterval(() => {
                arm.rotation.x += 0.2;
                if(arm.rotation.x > 1) {
                    clearInterval(tl);
                    setTimeout(() => { arm.rotation.x = 0; isAttacking = false; }, 100);
                }
            }, 20);

            // Logic
            if (activeSlot === 0) { // Sword
                enemies.forEach((en, i) => {
                    if (camera.position.distanceTo(en.mesh.position) < 4) {
                        en.hp--;
                        if(en.hp <= 0) {
                            scene.remove(en.mesh);
                            enemies.splice(i, 1);
                            score += 1000;
                            document.getElementById('score').innerText = score;
                            setTimeout(spawnEnemy, 5000);
                        }
                    }
                });
            } else if (activeSlot === 1) { // Pickaxe
                blockInv++;
                document.getElementById('block-count').innerText = blockInv;
            }
        }

        function placeBlock() {
            if (activeSlot === 2 && blockInv > 0) {
                const b = new THREE.Mesh(boxGeo, new THREE.MeshStandardMaterial({ color: 0x8b4513 }));
                const dir = new THREE.Vector3(0, 0, -1).applyQuaternion(camera.quaternion);
                b.position.copy(camera.position).add(dir.multiplyScalar(3));
                b.position.round();
                scene.add(b);
                blockInv--;
                document.getElementById('block-count').innerText = blockInv;
            }
        }

        // --- Game Loop ---
        camera.position.set(0, 15, 0);

        function animate() {
            requestAnimationFrame(animate);
            
            // Movement
            const move = new THREE.Vector3();
            if (keys['KeyW']) move.z -= 1;
            if (keys['KeyS']) move.z += 1;
            if (keys['KeyA']) move.x -= 1;
            if (keys['KeyD']) move.x += 1;
            
            if (move.length() > 0) {
                move.normalize().multiplyScalar(0.12).applyQuaternion(new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0, 1, 0), yaw));
                camera.position.add(move);
                // Head Bobbing
                camera.position.y = 6 + Math.sin(Date.now() * 0.01) * 0.04;
            }

            // Enemy AI
            enemies.forEach(en => {
                const diff = camera.position.clone().sub(en.mesh.position);
                diff.y = 0;
                if(diff.length() > 1.2) {
                    en.mesh.position.add(diff.normalize().multiplyScalar(0.04));
                }
                en.mesh.lookAt(camera.position.x, en.mesh.position.y, camera.position.z);

                // Attack Player
                if (diff.length() < 1.5 && Date.now() - en.lastHit > 1000) {
                    hp -= 10;
                    en.lastHit = Date.now();
                    document.getElementById('hp-fill').style.width = hp + "%";
                    if(hp <= 0) alert("GAME OVER! スコア: " + score);
                }
            });

            renderer.render(scene, camera);
        }
        animate();

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>
