<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>3D Hyper PVP Arena</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #000; font-family: sans-serif; }
        #crosshair {
            position: absolute; top: 50%; left: 50%;
            width: 20px; height: 20px;
            border: 2px solid #0f0; border-radius: 50%;
            transform: translate(-50%, -50%); pointer-events: none;
        }
        #ui {
            position: absolute; bottom: 20px; left: 20px;
            color: #0f0; font-size: 24px; text-shadow: 2px 2px #000;
        }
        #msg {
            position: absolute; top: 20%; left: 50%;
            transform: translateX(-50%); color: white; text-align: center;
        }
    </style>
</head>
<body>
    <div id="crosshair"></div>
    <div id="ui">Score: <span id="score">0</span></div>
    <div id="msg">クリックしてスタート<br>(WASDで移動 / Spaceでジャンプ / クリックで射撃)</div>

    <script type="importmap">
        { "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js" } }
    </script>

    <script type="module">
        import * as THREE from 'three';

        // --- シーン設定 ---
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x050505);
        scene.fog = new THREE.FogExp2(0x050505, 0.05);

        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        // --- 照明 ---
        const light = new THREE.HemisphereLight(0xeeeeff, 0x777788, 1);
        scene.add(light);

        // --- プレイヤー変数 ---
        let moveForward = false, moveBackward = false, moveLeft = false, moveRight = false, canJump = false;
        const velocity = new THREE.Vector3();
        const direction = new THREE.Vector3();
        let score = 0;

        // --- オブジェクト管理 ---
        const objects = []; // 当たり判定用
        const bullets = [];
        const enemies = [];

        // --- マップ生成 (面白い構造) ---
        const floorGeo = new THREE.PlaneGeometry(200, 200);
        const floorMat = new THREE.MeshPhongMaterial({ color: 0x222222 });
        const floor = new THREE.Mesh(floorGeo, floorMat);
        floor.rotation.x = -Math.PI / 2;
        scene.add(floor);
        objects.push(floor);

        // ランダムな障害物と浮遊プラットフォーム
        for (let i = 0; i < 50; i++) {
            const h = Math.random() * 10 + 2;
            const boxGeo = new THREE.BoxGeometry(5, h, 5);
            const boxMat = new THREE.MeshPhongMaterial({ color: Math.random() * 0xffffff });
            const box = new THREE.Mesh(boxGeo, boxMat);
            box.position.set(Math.random() * 160 - 80, h/2, Math.random() * 160 - 80);
            scene.add(box);
            objects.push(box);
        }

        // --- 敵の生成 ---
        function spawnEnemy() {
            const enemyGeo = new THREE.BoxGeometry(2, 2, 2);
            const enemyMat = new THREE.MeshBasicMaterial({ color: 0xff0000 });
            const enemy = new THREE.Mesh(enemyGeo, enemyMat);
            enemy.position.set(Math.random() * 100 - 50, 1, Math.random() * 100 - 50);
            scene.add(enemy);
            enemies.push(enemy);
        }
        for(let i=0; i<10; i++) spawnEnemy();

        // --- コントロール ---
        document.addEventListener('click', () => {
            document.body.requestPointerLock();
            document.getElementById('msg').style.display = 'none';
        });

        const onKeyDown = (e) => {
            switch (e.code) {
                case 'KeyW': moveForward = true; break;
                case 'KeyS': moveBackward = true; break;
                case 'KeyA': moveLeft = true; break;
                case 'KeyD': moveRight = true; break;
                case 'Space': if (canJump) velocity.y += 15; canJump = false; break;
            }
        };
        const onKeyUp = (e) => {
            switch (e.code) {
                case 'KeyW': moveForward = false; break;
                case 'KeyS': moveBackward = false; break;
                case 'KeyA': moveLeft = false; break;
                case 'KeyD': moveRight = false; break;
            }
        };
        document.addEventListener('keydown', onKeyDown);
        document.addEventListener('keyup', onKeyUp);

        // マウス視点移動
        document.addEventListener('mousemove', (e) => {
            if (document.pointerLockElement) {
                camera.rotation.y -= e.movementX * 0.002;
                camera.rotation.x -= e.movementY * 0.002;
                camera.rotation.x = Math.max(-Math.PI/2, Math.min(Math.PI/2, camera.rotation.x));
            }
        });

        // 射撃
        document.addEventListener('mousedown', (e) => {
            if (document.pointerLockElement) {
                const bGeo = new THREE.SphereGeometry(0.2);
                const bMat = new THREE.MeshBasicMaterial({ color: 0xffff00 });
                const bullet = new THREE.Mesh(bGeo, bMat);
                bullet.position.copy(camera.position);
                
                const dir = new THREE.Vector3(0, 0, -1).applyQuaternion(camera.quaternion);
                bullet.userData.velocity = dir.multiplyScalar(2);
                scene.add(bullet);
                bullets.push(bullet);
            }
        });

        camera.position.y = 1.6;

        // --- メインループ ---
        let prevTime = performance.now();
        function animate() {
            requestAnimationFrame(animate);
            const time = performance.now();
            const delta = (time - prevTime) / 1000;

            // 移動計算
            velocity.x -= velocity.x * 10.0 * delta;
            velocity.z -= velocity.z * 10.0 * delta;
            velocity.y -= 9.8 * 4.0 * delta; // 重力

            direction.z = Number(moveForward) - Number(moveBackward);
            direction.x = Number(moveRight) - Number(moveLeft);
            direction.normalize();

            if (moveForward || moveBackward) velocity.z -= direction.z * 400.0 * delta;
            if (moveLeft || moveRight) velocity.x -= direction.x * 400.0 * delta;

            camera.translateX(-velocity.x * delta);
            camera.translateZ(velocity.z * delta);
            camera.position.y += velocity.y * delta;

            if (camera.position.y < 1.6) {
                velocity.y = 0;
                camera.position.y = 1.6;
                canJump = true;
            }

            // 弾の移動と判定
            bullets.forEach((b, index) => {
                b.position.add(b.userData.velocity);
                if(b.position.length() > 200) {
                    scene.remove(b);
                    bullets.splice(index, 1);
                }
                // 敵との当たり判定
                enemies.forEach((en, ei) => {
                    if(b.position.distanceTo(en.position) < 2) {
                        scene.remove(en);
                        enemies.splice(ei, 1);
                        scene.remove(b);
                        bullets.splice(index, 1);
                        score += 100;
                        document.getElementById('score').innerText = score;
                        spawnEnemy();
                    }
                });
            });

            renderer.render(scene, camera);
            prevTime = time;
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
