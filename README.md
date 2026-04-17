<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>ゆいきちドキュメント</title>
  <style>
    /* ---------- 基本レイアウト ---------- */
    body { margin: 0; font-family: 'Noto Sans JP', Arial, sans-serif; background:#fff; }
    #toolbar {
      background:#f1f3f4;
      border-bottom:1px solid #dadce0;
      padding:6px 8px;
      display:flex;
      align-items:center;
      gap:4px;
      flex-wrap:wrap;
    }
    #toolbar button { background:none; border:none; padding:4px 8px; cursor:pointer; }
    #toolbar button:hover { background:#e8eaed; }
    #editor {
      padding:12px 20px;
      min-height:80vh;
      outline:none;
      white-space:pre-wrap;
    }
    /* ---------- フォントサイズドロップダウン ---------- */
    select { padding:2px 4px; }
  </style>
</head>
<body>
  <!-- ① ツールバー -->
  <div id="toolbar">
    <button onclick="execCmd('undo')">↺</button>
    <button onclick="execCmd('redo')">↻</button>
    <button onclick="execCmd('bold')"><b>太</b></button>
    <button onclick="execCmd('italic')"><i>斜</i></button>
    <button onclick="execCmd('underline')"><u>下</u></button>
    <button onclick="execCmd('strikethrough')"><s>取り消し線</s></button>
    <select onchange="execCmd('fontName', this.value)">
      <option>Arial</option>
      <option>Courier New</option>
      <option>Georgia</option>
      <option>Helvetica</option>
      <option>Times New Roman</option>
      <option>Verdana</option>
    </select>
    <select onchange="execCmd('fontSize', this.value)">
      <!-- Google Docs スタイル (ポイント) -->
      <option value="1">8</option>
      <option value="2">10</option>
      <option value="3">12</option>
      <option value="4">14</option>
      <option value="5">18</option>
      <option value="6">24</option>
      <option value="7">36</option>
    </select>
    <button onclick="saveDocument()">💾 保存</button>
    <button onclick="loadDocument()">📂 読み込み</button>
  </div>

  <!-- ② エディタ（contenteditable） -->
  <div id="editor" contenteditable="true">
    ここにテキストを書いてみてください...
  </div>

  <!-- ③ スクリプト -->
  <script>
    const editor = document.getElementById('editor');

    /* コマンド実行ユーティリティ */
    function execCmd(cmd, value = null) {
      document.execCommand(cmd, false, value);
      editor.focus();
    }

    /* 簡易保存（localStorage） */
    function saveDocument() {
      const content = editor.innerHTML;
      localStorage.setItem('yulimitDoc', content);
      alert('保存しました！');
    }

    function loadDocument() {
      const content = localStorage.getItem('yulimitDoc');
      if (content) {
        editor.innerHTML = content;
        alert('読み込み成功！');
      } else {
        alert('保存データがありません。');
      }
    }

    /* 初期化時に自動で読み込み（任意） */
    window.addEventListener('load', loadDocument);
  </script>
</body>
</html>
