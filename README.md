# parallel-memory
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>平行・垂直 神経衰弱</title>
  <style>
    body { font-family: sans-serif; text-align: center; transition: background-color 0.3s; }

    #start-menu {
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 20px;
      margin-top: 100px;
    }

    .menu-button {
      padding: 10px 30px;
      font-size: 20px;
      cursor: pointer;
    }

    #info {
      margin-top: 20px;
    }

    #turn-info {
      font-size: 36px;
      font-weight: bold;
      margin-top: 10px;
    }

    #board-wrapper {
      display: flex;
      justify-content: center;
      align-items: flex-start;
      gap: 20px;
      margin-top: 20px;
    }

    #score-left, #score-right {
      writing-mode: vertical-rl;
      text-orientation: upright;
      font-size: 24px;
      font-weight: bold;
      text-align: center;
      background: white;
      padding: 10px;
      border: 2px solid #ccc;
      border-radius: 10px;
      height: 420px;
      display: none; /* 初期は非表示 */
      align-items: center;
      justify-content: center;
    }

    #game-board {
      display: grid;
      grid-template-columns: repeat(6, 100px);
      grid-gap: 10px;
      justify-content: center;
    }

    canvas.card {
      width: 100px;
      height: 140px;
      background: #004080;
      border-radius: 10px;
      cursor: pointer;
      transition: transform 0.2s;
    }

    canvas.card.flipping {
      transform: rotateY(180deg);
    }

    canvas.preview-card {
      width: 200px !important;
      height: 280px !important;
      border: 2px solid black;
      background-color: white;
      pointer-events: none;
    }

    .preview-container {
      position: relative;
      display: inline-block;
    }

    #result-label {
      font-size: 14px;
      font-weight: normal;
      white-space: nowrap;
      background: white;
      padding: 2px 8px;
      border-radius: 4px;
      position: absolute;
      bottom: 8px;
      left: 50%;
      transform: translateX(-50%);
    }

    #preview {
      position: fixed;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      display: none;
      justify-content: center;
      gap: 20px;
      z-index: 10;
      pointer-events: none;
      padding: 10px;
      border-radius: 12px;
      flex-direction: column;
      align-items: center;
    }
  </style>
</head>
<body>
<div id="start-menu">
  <h1>平行モード 神経衰弱</h1>
  <button class="menu-button" onclick="startGame('solo')">1人モード</button>
  <button class="menu-button" onclick="startGame('versus')">対戦モード</button>
</div>

<div id="info" style="display:none;">
  <div style="font-size: 20px; font-weight: bold;">めくった回数：<span id="count">0</span></div>
  <div id="turn-info"></div>
</div>

<div id="preview"><div id="preview-canvas"></div></div>

<div id="board-wrapper">
  <div id="score-left">プレイヤー１：0</div>
  <div id="game-board"></div>
  <div id="score-right">プレイヤー２：0</div>
</div>

<div id="menu-return" style="margin-top: 20px; display: none;">
  <button class="menu-button" onclick="returnToMenu()">メニューにもどる</button>
</div>

<script defer>
let currentPlayer = 0;
let scores = [0, 0];
let gameMode = "solo";
let board = document.getElementById("game-board");
let preview = document.getElementById("preview");
let cards = [], flipped = [], matched = [], flipCount = 0;

function startGame(mode) {
  gameMode = mode;
  currentPlayer = 0;
  scores = [0, 0];

  document.getElementById("start-menu").style.display = "none";
  document.getElementById("info").style.display = "block";
  document.getElementById("menu-return").style.display = "block";

  // ゲームボード表示＋得点表示切替
  document.getElementById("score-left").style.display = mode === "versus" ? "flex" : "none";
  document.getElementById("score-right").style.display = mode === "versus" ? "flex" : "none";

  document.getElementById("score-left").textContent = "プレイヤー１：0";
  document.getElementById("score-right").textContent = "プレイヤー２：0";

  document.getElementById("turn-info").textContent = "";
  if (mode === "versus") updateTurnInfo();

  updateBackgroundForPlayer();
  initGame();
}

function updateBackgroundForPlayer() {
  document.body.style.backgroundColor = gameMode === "versus"
    ? (currentPlayer === 0 ? "#ffe6ea" : "#e6f0ff")
    : "white";
}

function updateTurnInfo() {
  const turnInfo = document.getElementById("turn-info");
  turnInfo.textContent = `プレイヤー${currentPlayer + 1}の番です`;
  turnInfo.style.color = currentPlayer === 0 ? "#e60033" : "#0066cc";
}

function initGame() {
  const baseAngles = [15, 30, 45, 60, 75, 90, 105, 120, 135, 150, 165, 180];
  const selected = shuffle(baseAngles).slice(0, 9);
  cards = [];
  selected.forEach(angle => { cards.push(angle); cards.push(angle); });
  shuffle(cards);
  board.innerHTML = "";
  flipped = [];
  matched = [];
  flipCount = 0;
  document.getElementById("count").textContent = 0;

  cards.forEach((angle, index) => {
    const canvas = document.createElement("canvas");
    canvas.width = 100;
    canvas.height = 140;
    canvas.className = "card";
    board.appendChild(canvas);

    canvas.dataset.angle = angle;
    canvas.dataset.index = index;
    drawBack(canvas);

    canvas.addEventListener("click", () => {
      if (flipped.length >= 2 || flipped.includes(index) || matched.includes(index)) return;
      flipCount++;
      document.getElementById("count").textContent = flipCount;
      canvas.classList.add("flipping");
      setTimeout(() => {
        drawAngle(canvas, angle);
        canvas.classList.remove("flipping");
      }, 100);
      flipped.push(index);

      if (flipped.length === 2) {
        setTimeout(() => {
          showPreview();
          const [i1, i2] = flipped;
          const a1 = cards[i1], a2 = cards[i2];
          if (a1 === a2) {
            matched.push(i1, i2);
            if (gameMode === "versus") {
              scores[currentPlayer]++;
              document.getElementById("score-left").textContent = `プレイヤー１：${scores[0]}`;
              document.getElementById("score-right").textContent = `プレイヤー２：${scores[1]}`;
            }
            setTimeout(() => {
              flipped = [];
              hidePreview();
              if (matched.length === cards.length) {
                if (gameMode === "versus") {
                  const winner = scores[0] > scores[1] ? "プレイヤー１の勝ち！" : scores[1] > scores[0] ? "プレイヤー２の勝ち！" : "引き分け！";
                  alert(`${winner}（プレイヤー１：${scores[0]} - プレイヤー２：${scores[1]}）`);
                } else {
                  alert(`クリア！${flipCount}回でそろいました！`);
                }
              }
            }, 2000);
          } else {
            setTimeout(() => {
              drawBack(board.children[i1]);
              drawBack(board.children[i2]);
              flipped = [];
              hidePreview();
              if (gameMode === "versus") {
                currentPlayer = 1 - currentPlayer;
                updateTurnInfo();
                updateBackgroundForPlayer();
              }
            }, 2000);
          }
        }, 500);
      }
    });
  });
}

function drawAngle(canvas, angle) {
  const ctx = canvas.getContext("2d");
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.fillStyle = "white";
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  ctx.strokeStyle = "black";
  ctx.lineWidth = 2;
  ctx.strokeRect(0, 0, canvas.width, canvas.height);
  ctx.save();
  ctx.translate(canvas.width / 2, canvas.height / 2);
  ctx.rotate((angle * Math.PI) / 180);
  ctx.beginPath();
  ctx.moveTo(-canvas.width * 0.4, 0);
  ctx.lineTo(canvas.width * 0.4, 0);
  ctx.lineWidth = 4;
  ctx.stroke();
  ctx.restore();
}

function drawBack(canvas) {
  const ctx = canvas.getContext("2d");
  ctx.fillStyle = "#004080";
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  ctx.fillStyle = "white";
  ctx.font = "20px sans-serif";
  ctx.textAlign = "center";
  ctx.fillText("？", canvas.width / 2, canvas.height / 2 + 8);
}

function showPreview() {
  preview.innerHTML = "";
  preview.style.display = "flex";

  const angle1 = cards[flipped[0]];
  const angle2 = cards[flipped[1]];
  const canvas = document.createElement("canvas");
  canvas.width = 200;
  canvas.height = 280;
  canvas.className = "preview-card";

  const container = document.createElement("div");
  container.className = "preview-container";
  container.appendChild(canvas);
  preview.appendChild(container);

  const ctx = canvas.getContext("2d");
  ctx.fillStyle = "white";
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  ctx.strokeStyle = "black";
  ctx.lineWidth = 2;
  ctx.strokeRect(0, 0, canvas.width, canvas.height);

  ctx.save();
  ctx.translate(70, 140);
  ctx.rotate((angle1 * Math.PI) / 180);
  ctx.beginPath();
  ctx.moveTo(-50, 0);
  ctx.lineTo(50, 0);
  ctx.lineWidth = 4;
  ctx.stroke();
  ctx.restore();

  ctx.save();
  ctx.translate(130, 140);
  ctx.rotate((angle2 * Math.PI) / 180);
  ctx.beginPath();
  ctx.moveTo(-50, 0);
  ctx.lineTo(50, 0);
  ctx.lineWidth = 4;
  ctx.stroke();
  ctx.restore();

  const label = document.createElement("div");
  label.id = "result-label";
  label.textContent = (angle1 === angle2) ? "平行です" : "平行ではありません";
  container.appendChild(label);
}

function hidePreview() {
  preview.innerHTML = "";
  preview.style.display = "none";
}

function shuffle(arr) {
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
  return arr;
}

function returnToMenu() {
  document.getElementById("start-menu").style.display = "flex";
  document.getElementById("info").style.display = "none";
  document.getElementById("menu-return").style.display = "none";
  document.getElementById("score-left").style.display = "none";
  document.getElementById("score-right").style.display = "none";
  board.innerHTML = "";
  hidePreview();
  document.body.style.backgroundColor = "white";
}
</script>
</body>
</html>
