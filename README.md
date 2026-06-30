// html
<html>
  <head>
    <title>カニが柿を食べるゲーム</title>
    <style>
      #gameover {
        display: none;
      }
    </style>
  </head>
  <body>
    <h1>カニが柿を食べるゲーム</h1>
    <button id="start">スタート</button>
    <div id="kani"></div>
    <div id="kaki"></div>
    <div id="gomi"></div>
    <p id="result"></p>
    <p id="score">スコア: 0</p>
    <div id="gameover">
      <p>ゲームオーバー</p>
      <button id="retry">再チャレンジ</button>
    </div>

    <script>
      let score = 0;
      let kakiInterval;
      let gomiInterval;
      let kani;

      document.getElementById('start').addEventListener('click', () => {
        startGame();
      });

      document.getElementById('retry').addEventListener('click', () => {
        retryGame();
      });

      function startGame() {
        createKani();
        kakiInterval = setInterval(() => {
          createKaki();
        }, 1000);
        gomiInterval = setInterval(() => {
          createGomi();
        }, 2000);
      }

      function createKani() {
        kani = document.createElement('div');
        kani.textContent = 'カニ';
        kani.style.position = 'absolute';
        kani.style.top = '50%';
        kani.style.left = '50%';
        document.getElementById('kani').appendChild(kani);

        document.addEventListener('touchstart', (e) => {
          moveKaniTo(e.touches[0].clientX, e.touches[0].clientY);
        });
      }

      function createKaki() {
        const kaki = document.createElement('div');
        kaki.textContent = '柿';
        kaki.style.position = 'absolute';
        kaki.style.top = '0%';
        kaki.style.left = `${Math.floor(Math.random() * 100)}%`;
        document.getElementById('kaki').appendChild(kaki);

        kakiInterval = setInterval(() => {
          moveKaki(kaki);
        }, 10);
      }

      function createGomi() {
        const gomi = document.createElement('div');
        gomi.textContent = 'ゴミ';
        gomi.style.position = 'absolute';
        gomi.style.top = '0%';
        gomi.style.left = `${Math.floor(Math.random() * 100)}%`;
        document.getElementById('gomi').appendChild(gomi);

        gomiInterval = setInterval(() => {
          moveGomi(gomi);
        }, 10);
      }

      function moveKaki(kaki) {
        const top = kaki.style.top;
        if (top === '100%') {
          kaki.remove();
        } else {
          kaki.style.top = `${parseInt(top.substring(0, top.length - 1)) + 1}%`;
          checkCollisionKaki(kaki);
        }
      }

      function moveGomi(gomi) {
        const top = gomi.style.top;
        if (top === '100%') {
          gomi.remove();
        } else {
          gomi.style.top = `${parseInt(top.substring(0, top.length - 1)) + 1}%`;
          checkCollisionGomi(gomi);
        }
      }

      function eatKaki(kaki) {
        score++;
        document.getElementById('score').textContent = `スコア: ${score}`;
        kaki.remove();
      }

      function gameover() {
        document.getElementById('gameover').style.display = 'block';
        document.getElementById('kani').style.display = 'none';
        document.getElementById('kaki').style.display = 'none';
        document.getElementById('gomi').style.display = 'none';
        clearInterval(kakiInterval);
        clearInterval(gomiInterval);
      }

      function retryGame() {
        document.getElementById('gameover').style.display = 'none';
        document.getElementById('kani').style.display = 'block';
        document.getElementById('kaki').style.display = 'block';
        document.getElementById('gomi').style.display = 'block';
        score = 0;
        document.getElementById('score').textContent = `スコア: ${score}`;
        startGame();
      }

      function moveKaniTo(x, y) {
        kani.style.left = `${x}px`;
        kani.style.top = `${y}px`;
      }

      function checkCollisionKaki(kaki) {
        const kaniRect = kani.getBoundingClientRect();
        const kakiRect = kaki.getBoundingClientRect();

        if (kaniRect.left < kakiRect.right &&
            kaniRect.right > kakiRect.left &&
            kaniRect.top < kakiRect.bottom &&
            kaniRect.bottom > kakiRect.top) {
          eatKaki(kaki);
        }
      }

      function checkCollisionGomi(gomi) {
        const kaniRect = kani.getBoundingClientRect();
        const gomiRect = gomi.getBoundingClientRect();

        if (kaniRect.left < gomiRect.right &&
            kaniRect.right > gomiRect.left &&
            kaniRect.top < gomiRect.bottom &&
            kaniRect.bottom > gomiRect.top) {
          gameover();
        }
      }
    </script>
  </body>
</html>
