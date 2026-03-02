<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <style>
        body { background: #555; display: flex; justify-content: center; padding: 50px; }
        .test-paper { 
            background: white; width: 600px; padding: 40px; 
            box-shadow: 10px 10px 0px rgba(0,0,0,0.2); font-family: "MS Mincho", serif;
            position: relative; border: 1px solid #ccc;
        }
        .header { display: flex; justify-content: border-between; border-bottom: 2px solid #000; margin-bottom: 30px; }
        .title { font-size: 28px; flex-grow: 1; }
        .name-box { border-left: 2px solid #000; padding-left: 20px; font-size: 20px; width: 250px; }
        .score-circle { 
            position: absolute; top: 40px; right: 40px; 
            width: 120px; height: 120px; border: 4px solid #ff0000; 
            border-radius: 50%; color: #ff0000; display: flex; 
            flex-direction: column; align-items: center; justify-content: center;
            transform: rotate(-10deg);
        }
        .score-value { font-size: 50px; font-weight: bold; line-height: 1; }
        .question { font-size: 24px; margin: 30px 0; display: flex; align-items: center; }
        .answer-line { border-bottom: 1px solid #000; width: 100px; display: inline-block; margin-left: 20px; height: 30px; }
        .wrong-mark { color: #ff0000; font-size: 40px; margin-left: -80px; font-family: sans-serif; }
    </style>
</head>
<body>

<div class="test-paper">
    <div class="score-circle">
        <div style="font-size: 16px;">点数</div>
        <div class="score-value">0</div>
    </div>

    <div class="header">
        <div class="title">算数 小テスト（たしざん）</div>
        <div class="name-box">氏名： 与魔 場下</div>
    </div>

    <div class="question">① 1 + 1 = <span class="answer-line"></span> <span class="wrong-mark">／</span></div>
    <div class="question">② 2 + 3 = <span class="answer-line"></span> <span class="wrong-mark">／</span></div>
    <div class="question">③ 5 + 0 = <span class="answer-line"></span> <span class="wrong-mark">／</span></div>
    <div class="question">④ 10 + 10 = <span class="answer-line"></span> <span class="wrong-mark">／</span></div>
    <div class="question">⑤ 100 + 1 = <span class="answer-line"></span> <span class="wrong-mark">／</span></div>
    
    <p style="text-align: right; color: #ff0000; font-weight: bold; margin-top: 50px;">もっとがんばりましょう！</p>
</div>

</body>
</html>
