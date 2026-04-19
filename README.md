<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>ULTIMATE REAL-PVP ARENA</title>
    <style>
        body { margin: 0; overflow: hidden; background: #050505; font-family: 'Arial', sans-serif; }
        #ui {
            position: absolute; bottom: 30px; left: 30px; color: #00fff2;
            background: rgba(0,0,0,0.7); padding: 20px; border-radius: 10px; border: 1px solid #00fff2;
        }
        #hp-bar { width: 200px; height: 10px; background: #222; margin-top: 10px; border-radius: 5px; }
        #hp-fill { width: 100%; height: 100%; background: #00fff2; transition: 0.2s; }
        #crosshair {
            position: absolute; top: 50%; left: 50%; width: 10px; height: 10px;
            border: 1px solid #fff; border-radius: 50%; transform: translate(-50%, -50%); pointer-events: none;
        }
        #overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.8); color: white; display: flex;
            flex-direction: column; justify-content: center; align-items: center; cursor: pointer;
        }
    </style>
</head>
<body>
    <div id="crosshair"></div>
    <div id="ui">
        <div style="font-size: 12px;">PLAYER NEURAL LINK</div>
        <div id="hp-bar"><div id="hp-fill"></div></div>
        <div style="margin-top:10px;">KILLS: <span id="score">0</span> | AMMO: ∞</div>
        <div style="font-size: 10px; color: #888; margin-top:5px;">[1] Sword [2] Gun | WASD: Move</div>
    </div>
    <div id="overlay">
        <h1 style="letter-spacing: 10px;">REAL-PVP ARENA</h1>
        <p>クリックしてシステムを起動</p>
    </div>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/cannon.js/0.6.2/cannon.min.js"></script>

    <script>
        // --- 設定 ---
        let hp = 100, score = 0, isDead = false, activeWeapon = 1;
        const enemies = [], bullets = [], keys = {};

        // --- 物理ワールド ---
        const world = new CANNON.World();
        world.gravity.set(0, -18, 0);

        // --- Three.js ---
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0a0a10);
        scene.fog = new THREE.FogExp2(0x0a0a10, 0.02);

        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.shadowMap.enabled = true;
        document.body.appendChild(renderer.domElement);

        // --- 地形生成 (リアルな凸凹地面) ---
        const groundMat = new THREE.MeshStandardMaterial({ color: 0x1a1a1a, roughness: 0.8 });
        const groundGeo = new THREE.PlaneGeometry(200, 200, 64, 64);
        groundGeo.rotateX(-Math.PI / 2);
        
        const vertices = groundGeo.attributes.position.array;
        for (let i = 0; i < vertices.length; i += 3) {
            const x = vertices[i], z = vertices[i+2];
            vertices[i+1] = Math.sin(x * 0.1) * 2 + Math.cos(z * 0.1) * 2; // スムーズな起伏
        }
        const groundMesh = new THREE.Mesh(groundGeo, groundMat);
        groundMesh.receiveShadow = true;
        scene.add(groundMesh);

        const physGround = new CANNON.Body({ mass: 0 });
        physGround.addShape(new CANNON.Plane());
        physGround.quaternion.setFromAxisAngle(new CANNON.Vector3(1,0,0), -Math.PI/2);
        world.addBody(physGround);

        // --- プレイヤー物理 ---
        const playerBody = new CANNON.Body({ mass: 60, fixedRotation: true });
        playerBody.addShape(new CANNON.Sphere(1));
        playerBody.position.set(0, 10, 0); // 空中からスポーンして落下
        world.addBody(playerBody);

        // --- 手・武器のビジュアル ---
        const armGroup = new THREE.Group();
        const hand = new THREE.Mesh(new THREE.BoxGeometry(0.2, 0.2, 0.8), new THREE.MeshStandardMaterial({color: 0xffdbac}));
        hand.position.set(0.5, -0.4, -0.6);
        
        const sword = new THREE.Mesh(new THREE.BoxGeometry(0.1, 1.5, 0.2), new THREE.MeshStandardMaterial({color: 0xcccccc, metalness: 0.8}));
        sword.position.set(0, 0.6, -0.4);
        sword.rotation.x = -Math.PI/4;

        const gun = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.25, 0.6), new THREE.MeshStandardMaterial({color: 0x333333}));
        gun.position.set(0, 0, -0.4);
        gun.visible = false;

        hand.add(sword, gun);
        armGroup.add(hand);
        camera.add(armGroup);
        scene.add(camera);
        scene.add(new THREE.AmbientLight(0xffffff, 0.5));
        const sun = new THREE.DirectionalLight(0xffffff, 1);
        sun.position.set(10, 50, 10);
        sun.castShadow = true;
        scene.add(sun);

        // --- 操作 ---
        document.getElementById('overlay').onclick = () => {
            document.body.requestPointerLock();
            document.getElementById('overlay').style.display = 'none';
        };

        window.addEventListener('keydown', (e) => {
            keys[e.code] = true;
            if(e.code === 'Digit1') { activeWeapon = 1; sword.visible = true; gun.visible = false; }
            if(e.code === 'Digit2') { activeWeapon = 2; sword.visible = false; gun.visible = true; }
        });
        window.addEventListener('keyup', (e) => keys[e.code] = false);

        let yaw = 0, pitch = 0;
        document.addEventListener('mousemove', (e) => {
            if (document.pointerLockElement) {
                yaw -= e.movementX * 0.002;
                pitch -= e.movementY * 0.002;
                pitch = Math.max(-Math.PI/2.2, Math.min(Math.PI/2.2, pitch));
                camera.rotation.set(pitch, yaw, 0, 'YXZ');
            }
        });

        // --- 攻撃機能 ---
        window.onmousedown = () => {
            if (!document.pointerLockElement) return;
            // 攻撃アニメ
            hand.rotation.x += 0.5;
            setTimeout(() => hand.rotation.x -= 0.5, 100);

            if (activeWeapon === 2) {
                const bBody = new CANNON.Body({ mass: 0.1 });
                bBody.addShape(new CANNON.Sphere(0.1));
                const dir = new THREE.Vector3(0,0,-1).applyQuaternion(camera.quaternion);
                bBody.position.set(playerBody.position.x + dir.x, playerBody.position.y + dir.y, playerBody.position.z + dir.z);
                bBody.velocity.set(dir.x*60, dir.y*60, dir.z*60);
                world.addBody(bBody);
                const bMesh = new THREE.Mesh(new THREE.SphereGeometry(0.1), new THREE.MeshBasicMaterial({color: 0x00fff2}));
                scene.add(bMesh);
                bullets.push({body: bBody, mesh: bMesh, life: 100});
            } else {
                // 近接判定
                enemies.forEach(en => {
                    if (playerBody.position.distanceTo(en.body.position) < 3) en.hp -= 2;
                });
            }
        };

        // --- CPU AI ---
        function spawnEnemy() {
            const eBody = new CANNON.Body({ mass: 50, fixedRotation: true });
            eBody.addShape(new CANNON.Cylinder(0.5, 0.5, 2, 8));
            eBody.position.set(Math.random()*40-20, 5, Math.random()*40-20);
            world.addBody(eBody);
            const eMesh = new THREE.Mesh(new THREE.CapsuleGeometry(0.5, 1), new THREE.MeshStandardMaterial({color: 0xff4444}));
            scene.add(eMesh);
            enemies.push({body: eBody, mesh: eMesh, hp: 5});
        }
        for(let i=0; i<4; i++) spawnEnemy();

        // --- ループ ---
        function animate() {
            requestAnimationFrame(animate);
            world.step(1/60);

            if (!isDead) {
                // 移動制御
                const moveSpeed = 8;
                const front = new THREE.Vector3(0,0,-1).applyQuaternion(new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0,1,0), yaw));
                const right = new THREE.Vector3(1,0,0).applyQuaternion(new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0,1,0), yaw));
                let v = new THREE.Vector3();
                if(keys['KeyW']) v.add(front);
                if(keys['KeyS']) v.sub(front);
                if(keys['KeyA']) v.sub(right);
                if(keys['KeyD']) v.add(right);
                
                if(v.length() > 0) {
                    v.normalize().multiplyScalar(moveSpeed);
                    playerBody.velocity.x = v.x;
                    playerBody.velocity.z = v.z;
                }
                if(keys['Space'] && Math.abs(playerBody.velocity.y) < 0.1) playerBody.velocity.y = 10;

                camera.position.copy(playerBody.position);
                camera.position.y += 0.8; // 目線の高さ
                // ヘッドボブ
                if(v.length() > 0) camera.position.y += Math.sin(Date.now()*0.01)*0.05;
            }

            // 弾丸更新
            bullets.forEach((b, i) => {
                b.mesh.position.copy(b.body.position);
                enemies.forEach(en => {
                    if(b.body.position.distanceTo(en.body.position) < 1.2) { en.hp -= 1; b.life = 0; }
                });
                b.life--;
                if(b.life <= 0) { world.remove(b.body); scene.remove(b.mesh); bullets.splice(i, 1); }
            });

            // 敵AI更新
            enemies.forEach((en, i) => {
                en.mesh.position.copy(en.body.position);
                const toP = new THREE.Vector3().subVectors(playerBody.position, en.body.position);
                if(toP.length() < 20) {
                    toP.y = 0;
                    en.body.velocity.x = toP.normalize().x * 3;
                    en.body.velocity.z = toP.z * 3;
                    if(toP.length() < 1.5) hp -= 0.1; // 攻撃を受ける
                }
                if(en.hp <= 0) {
                    world.remove(en.body); scene.remove(en.mesh); enemies.splice(i, 1);
                    score++; document.getElementById('score').innerText = score;
                    setTimeout(spawnEnemy, 3000);
                }
            });

            document.getElementById('hp-fill').style.width = hp + "%";
            if(hp <= 0 && !isDead) { isDead = true; alert("CONNECTION LOST. SCORE: " + score); location.reload(); }

            renderer.render(scene, camera);
        }
        animate();
    </script>
</body>
</html>
