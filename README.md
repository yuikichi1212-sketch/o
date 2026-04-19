<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>Ultimate Voxel Survivor & PVP</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Courier New', Courier, monospace; }
        #crosshair {
            position: absolute; top: 50%; left: 50%;
            width: 12px; height: 12px; border: 1px solid white;
            transform: translate(-50%, -50%); pointer-events: none;
        }
        #ui {
            position: absolute; bottom: 20px; left: 50%; transform: translateX(-50%);
            display: flex; gap: 8px; background: rgba(0,0,0,0.8); padding: 12px; border-radius: 5px; border: 1px solid #444;
        }
        .slot {
            width: 60px; height: 60px; border: 2px solid #333;
            display: flex; flex-direction: column; justify-content: center; align-items: center;
            color: #aaa; font-size: 10px; transition: 0.2s;
        }
        .slot.active { border-color: #0f0; color: #fff; box-shadow: 0 0 10px rgba(0,255,0,0.3); }
        #mode-indicator {
            position: absolute; top: 20px; right: 20px; color: #f00; font-weight: bold; font-size: 20px;
            text-transform: uppercase; letter-spacing: 2px;
        }
        #instructions {
            position: absolute; top: 20px; left: 20px; color: #ccc; font-size: 12px;
            background: rgba(0,0,0,0.5); padding: 10px;
        }
    </style>
</head>
<body>
    <div id="crosshair"></div>
    <div id="mode-indicator">Survival Mode</div>
    <div id="instructions">
        [1-4] 武器・ツール切替<br>
        [左クリ] 攻撃/採掘<br>
        [右クリ] ブロック設置<br>
        [M] モード切替(Survival/PVP)
    </div>
    <div id="ui">
        <div class="slot active" id="slot0"><span>剣</span><div id="inv-0">1</div></div>
        <div class="slot" id="slot1"><span>ツルハシ</span><div id="inv-1">1</div></div>
        <div class="slot" id="slot2"><span>石材</span><div id="inv-2">0</div></div>
        <div class="slot" id="slot3"><span>金</span><div id="inv-3">0</div></div>
    </div>

    <script type="importmap">
        { "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js" } }
    </script>

    <script type="module">
        import * as THREE from 'three';

        // --- 初期設定 ---
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x11141a);
        scene.fog = new THREE.FogExp2(0x11141a, 0.04);

        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.shadowMap.enabled = true;
        document.body.appendChild(renderer.domElement);

        // --- ライト (リアルな夜明け風) ---
        const ambientLight = new THREE.AmbientLight(0x404040, 1.2);
        scene.add(ambientLight);
        const sun = new THREE.PointLight(0xffccaa, 2, 100);
        sun.position.set(20, 50, 20);
        sun.castShadow = true;
        scene.add(sun);

        // --- ゲームデータ ---
        let isSurvival = true;
        let activeSlot = 0;
        const inventory = [1, 1, 0, 0]; // 剣, ツルハシ, 石材, 金
        const player = { height: 1.7, speed: 0.1, isAttacking: false };

        // --- マップ生成 (高低差のあるリアルなボクセル) ---
        const blocks = [];
        const boxGeo = new THREE.BoxGeometry(1, 1, 1);
        
        function generateMap() {
            for (let x = -20; x < 20; x++) {
                for (let z = -20; z < 20; z++) {
                    // ノイズの代わりにサイン波を組み合わせて複雑な地形を作成
                    const h = Math.floor(Math.sin(x * 0.2) * 3 + Math.cos(z * 0.2) * 3 + 4);
                    for (let y = 0; y < h; y++) {
                        let color = 0x555555;
                        if (y === h - 1) color = 0x334433; // 草っぽさ
                        if (y < 2) color = 0x222222; // 深層

                        const mat = new THREE.MeshStandardMaterial({ color: color, roughness: 0.9 });
                        const block = new THREE.Mesh(boxGeo, mat);
                        block.position.set(x, y, z);
                        block.receiveShadow = true;
                        block.castShadow = true;
                        scene.add(block);
                        blocks.push(block);
                    }
                }
            }
        }
        generateMap();

        // --- 武器モデルとアニメーション ---
        const weaponPivot = new THREE.Group();
        const swordGeo = new THREE.BoxGeometry(0.1, 1.2, 0.2);
        const swordMat = new THREE.MeshStandardMaterial({ color: 0x88aaff, metalness: 0.9 });
        const sword = new THREE.Mesh(swordGeo, swordMat);
        sword.position.set(0.6, -0.5, -0.8);
        sword.rotation.x = -Math.PI / 3;
        weaponPivot.add(sword);
        camera.add(weaponPivot);
        scene.add(camera);

        // --- 操作系 ---
        const keys = {};
        document.addEventListener('keydown', (e) => {
            keys[e.code] = true;
            if(e.code === 'KeyM') toggleMode();
            if(['Digit1','Digit2','Digit3','Digit4'].includes(e.code)) {
                activeSlot = parseInt(e.key) - 1;
                updateUI();
            }
        });
        document.addEventListener('keyup', (e) => keys[e.code] = false);

        function toggleMode() {
            isSurvival = !isSurvival;
            document.getElementById('mode-indicator').innerText = isSurvival ? "Survival Mode" : "PVP Mode";
            document.getElementById('mode-indicator').style.color = isSurvival ? "#0f0" : "#f00";
        }

        function updateUI() {
            document.querySelectorAll('.slot').forEach((s, i) => {
                s.classList.toggle('active', i === activeSlot);
                document.getElementById(`inv-${i}`).innerText = inventory[i];
            });
        }

        // --- 攻撃/採掘アニメーション ---
        function performAction() {
            if (player.isAttacking) return;
            player.isAttacking = true;

            // アニメーション (Tween風)
            const startRotation = weaponPivot.rotation.x;
            let progress = 0;
            const anim = setInterval(() => {
                progress += 0.15;
                weaponPivot.rotation.x += 0.3;
                weaponPivot.rotation.y -= 0.1;
                if (progress >= 1) {
                    clearInterval(anim);
                    weaponPivot.rotation.set(0,0,0);
                    player.isAttacking = false;
                }
            }, 20);

            // 採掘・攻撃判定
            if (isSurvival && activeSlot === 1) { // ツルハシ
                // 前方のブロックを削除してアイテムを増やす処理（簡易版）
                inventory[2] += 1; 
                updateUI();
            }
        }

        document.addEventListener('mousedown', (e) => {
            if (document.pointerLockElement) {
                if(e.button === 0) performAction();
            } else {
                document.body.requestPointerLock();
            }
        });

        // --- カメラ移動ロジック ---
        let yaw = 0, pitch = 0;
        document.addEventListener('mousemove', (e) => {
            if (document.pointerLockElement) {
                yaw -= e.movementX * 0.002;
                pitch -= e.movementY * 0.002;
                pitch = Math.max(-Math.PI/2.1, Math.min(Math.PI/2.1, pitch));
                camera.rotation.set(pitch, yaw, 0, 'YXZ');
            }
        });

        function update() {
            const move = new THREE.Vector3();
            if (keys['KeyW']) move.z -= 1;
            if (keys['KeyS']) move.z += 1;
            if (keys['KeyA']) move.x -= 1;
            if (keys['KeyD']) move.x += 1;
            
            if (move.length() > 0) {
                move.normalize().multiplyScalar(player.speed).applyQuaternion(new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0, 1, 0), yaw));
                camera.position.add(move);
                // ヘッドボブ
                camera.position.y = 2.5 + Math.sin(Date.now() * 0.008) * 0.05;
            }
        }

        camera.position.set(0, 10, 5); // 高い位置からスタート

        function animate() {
            requestAnimationFrame(animate);
            update();
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
