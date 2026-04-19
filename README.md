<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>Realistic Voxel Arena</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Segoe UI', sans-serif; }
        #crosshair {
            position: absolute; top: 50%; left: 50%;
            width: 8px; height: 8px; background: rgba(255,255,255,0.5);
            border-radius: 50%; transform: translate(-50%, -50%); pointer-events: none;
        }
        #ui-container {
            position: absolute; bottom: 20px; left: 50%; transform: translateX(-50%);
            display: flex; gap: 10px; background: rgba(0,0,0,0.6); padding: 10px; border-radius: 10px;
        }
        .slot {
            width: 50px; height: 50px; border: 2px solid #555;
            display: flex; justify-content: center; align-items: center; color: white; font-size: 12px;
        }
        .active { border-color: #fff; box-shadow: 0 0 10px #fff; }
        #info { position: absolute; top: 20px; left: 20px; color: white; }
    </style>
</head>
<body>
    <div id="crosshair"></div>
    <div id="info">Room: 001 | Player: Guest</div>
    <div id="ui-container">
        <div class="slot active" id="slot1">剣</div>
        <div class="slot" id="slot2">弓</div>
        <div class="slot" id="slot3">石壁</div>
        <div class="slot" id="slot4">回復</div>
    </div>

    <script type="importmap">
        { "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js" } }
    </script>

    <script type="module">
        import * as THREE from 'three';

        // --- 基本設定 ---
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x87ceeb); // リアルな空色
        scene.fog = new THREE.Fog(0x87ceeb, 1, 50);

        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.shadowMap.enabled = true; // 影を有効に
        document.body.appendChild(renderer.domElement);

        // --- ライティング ---
        const sun = new THREE.DirectionalLight(0xffffff, 1.5);
        sun.position.set(10, 20, 10);
        sun.castShadow = true;
        scene.add(sun);
        scene.add(new THREE.AmbientLight(0x404040, 1));

        // --- プレイヤー状態 ---
        const player = {
            height: 1.7,
            speed: 0.12, // 歩行速度をリアルに修正
            velocity: new THREE.Vector3(),
            onGround: false,
            weapon: null,
            inventory: ['sword', 'bow', 'block', 'heal'],
            activeSlot: 0
        };

        // --- マップ生成（リアルなボクセル） ---
        const blockSize = 1;
        const geometry = new THREE.BoxGeometry(blockSize, blockSize, blockSize);
        const material = new THREE.MeshStandardMaterial({ color: 0x777777, roughness: 0.8 });
        
        const worldGroup = new THREE.Group();
        for (let x = -15; x < 15; x++) {
            for (let z = -15; z < 15; z++) {
                const h = Math.floor(Math.random() * 2); // 基本の地面
                for(let y = 0; y <= h; y++) {
                    const block = new THREE.Mesh(geometry, material);
                    block.position.set(x, y - 1, z);
                    block.receiveShadow = true;
                    block.castShadow = true;
                    worldGroup.add(block);
                }
            }
        }
        scene.add(worldGroup);

        // --- 武器（剣）の作成 ---
        const swordGeo = new THREE.BoxGeometry(0.1, 0.8, 0.2);
        const swordMat = new THREE.MeshStandardMaterial({ color: 0xcccccc, metalness: 0.8 });
        const sword = new THREE.Mesh(swordGeo, swordMat);
        camera.add(sword); // カメラに追従
        sword.position.set(0.5, -0.4, -0.6);
        sword.rotation.x = Math.PI / 4;
        scene.add(camera);

        // --- 操作系 ---
        const keys = {};
        document.addEventListener('keydown', (e) => keys[e.code] = true);
        document.addEventListener('keyup', (e) => keys[e.code] = false);
        document.addEventListener('mousedown', (e) => {
            if (document.pointerLockElement) {
                if(e.button === 0) attack(); // 左クリ：攻撃
            } else {
                document.body.requestPointerLock();
            }
        });

        function attack() {
            // 剣を振るアニメーション
            sword.rotation.z += 1.5;
            setTimeout(() => sword.rotation.z -= 1.5, 150);
        }

        // --- 移動・カメラロジック ---
        let pitch = 0;
        let yaw = 0;
        document.addEventListener('mousemove', (e) => {
            if (document.pointerLockElement) {
                yaw -= e.movementX * 0.0015;
                pitch -= e.movementY * 0.0015;
                pitch = Math.max(-Math.PI/2.2, Math.min(Math.PI/2.2, pitch));
                
                camera.rotation.set(pitch, yaw, 0, 'YXZ');
            }
        });

        function updateMovement() {
            const dir = new THREE.Vector3();
            const forward = new THREE.Vector3(0, 0, -1).applyQuaternion(new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0, 1, 0), yaw));
            const right = new THREE.Vector3(1, 0, 0).applyQuaternion(new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0, 1, 0), yaw));

            if (keys['KeyW']) dir.add(forward);
            if (keys['KeyS']) dir.sub(forward);
            if (keys['KeyA']) dir.sub(right);
            if (keys['KeyD']) dir.add(right);

            if (dir.length() > 0) {
                dir.normalize().multiplyScalar(player.speed);
                camera.position.add(dir);
                // 歩行時の揺れ（ヘッドボブ）
                camera.position.y = player.height + Math.sin(Date.now() * 0.01) * 0.03;
            }
        }

        // --- メインループ ---
        camera.position.y = player.height;

        function animate() {
            requestAnimationFrame(animate);
            updateMovement();
            renderer.render(scene, camera);
        }
        animate();

        // リサイズ対応
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>
