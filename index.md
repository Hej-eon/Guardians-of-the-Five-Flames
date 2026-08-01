<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Guardians of the Five Flames</title>
  <style>
    body {
      background: #1b0f0a;
      color: #f5e3c8;
      font-family: monospace;
      display: flex;
      flex-direction: column;
      align-items: center;
    }
    #ui {
      margin: 10px;
    }
    canvas {
      border: 4px solid #f5e3c8;
      image-rendering: pixelated;
      background: #2b1810;
    }
  </style>
</head>
<body>
  <div id="ui">
    <span id="worldLabel"></span> |
    <span id="levelLabel"></span> |
    <span id="livesLabel"></span>
  </div>
  <canvas id="gameCanvas" width="480" height="320"></canvas>

  <script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');

    const TILE_SIZE = 32;
    const COLS = 15;
    const ROWS = 10;

    // Tile types
    const TILE_EMPTY = 0;
    const TILE_SOLID = 1;
    const TILE_SOFT = 2;
    const TILE_EXIT = 3;

    // Simple Kurdish-themed world names
    const WORLDS = [
      { name: "World 1 - Agir (Fire)" },
      { name: "World 2 - Zagros (Mountains)" },
      { name: "World 3 - Roj (Sun)" },
      { name: "World 4 - Çem (River)" },
      { name: "World 5 - Sterk (Stars)" }
    ];

    let currentWorld = 0;
    let currentLevel = 0;
    let lives = 3;

    document.getElementById('livesLabel').textContent = `Lives: ${lives}`;

    // Simple level maps (you can expand later)
    // 0 = empty, 1 = solid, 2 = soft, 3 = exit
    function createBaseMap() {
      const map = [];
      for (let r = 0; r < ROWS; r++) {
        const row = [];
        for (let c = 0; c < COLS; c++) {
          if (r === 0 || r === ROWS - 1 || c === 0 || c === COLS - 1) {
            row.push(TILE_SOLID);
          } else if (r % 2 === 0 && c % 2 === 0) {
            row.push(TILE_SOLID);
          } else if (Math.random() < 0.3) {
            row.push(TILE_SOFT);
          } else {
            row.push(TILE_EMPTY);
          }
        }
        map.push(row);
      }
      // Exit tile somewhere
      map[ROWS - 2][COLS - 2] = TILE_EXIT;
      // Clear spawn area
      map[1][1] = TILE_EMPTY;
      map[1][2] = TILE_EMPTY;
      map[2][1] = TILE_EMPTY;
      return map;
    }

    const levels = [];
    for (let w = 0; w < 5; w++) {
      levels[w] = [];
      for (let l = 0; l < 3; l++) {
        levels[w][l] = createBaseMap();
      }
    }

    let map = levels[currentWorld][currentLevel];

    const player = {
      x: 1,
      y: 1,
      speed: 1,
    };

    const bombs = [];
    const explosions = [];

    const keys = {};

    window.addEventListener('keydown', (e) => {
      keys[e.key] = true;
      if (e.key === ' ') {
        placeBomb();
      }
    });

    window.addEventListener('keyup', (e) => {
      keys[e.key] = false;
    });

    function placeBomb() {
      const existing = bombs.find(b => b.x === player.x && b.y === player.y);
      if (existing) return;
      bombs.push({
        x: player.x,
        y: player.y,
        timer: 60, // frames
        range: 3
      });
    }

    function updatePlayer() {
      let nx = player.x;
      let ny = player.y;

      if (keys['ArrowLeft']) nx--;
      if (keys['ArrowRight']) nx++;
      if (keys['ArrowUp']) ny--;
      if (keys['ArrowDown']) ny++;

      if (isWalkable(nx, ny)) {
        player.x = nx;
        player.y = ny;
      }
    }

    function isWalkable(x, y) {
      if (x < 0 || y < 0 || x >= COLS || y >= ROWS) return false;
      const t = map[y][x];
      return t === TILE_EMPTY || t === TILE_SOFT || t === TILE_EXIT;
    }

    function updateBombs() {
      for (let i = bombs.length - 1; i >= 0; i--) {
        const b = bombs[i];
        b.timer--;
        if (b.timer <= 0) {
          explodeBomb(b);
          bombs.splice(i, 1);
        }
      }
    }

    function explodeBomb(bomb) {
      const dirs = [
        { dx: 0, dy: 0 }, // center
        { dx: 1, dy: 0 },
        { dx: -1, dy: 0 },
        { dx: 0, dy: 1 },
        { dx: 0, dy: -1 }
      ];

      dirs.forEach(dir => {
        for (let r = 0; r <= bomb.range; r++) {
          const x = bomb.x + dir.dx * r;
          const y = bomb.y + dir.dy * r;
          if (x < 0 || y < 0 || x >= COLS || y >= ROWS) break;
          const t = map[y][x];
          explosions.push({ x, y, timer: 20 });

          if (t === TILE_SOLID) break;
          if (t === TILE_SOFT) {
            map[y][x] = TILE_EMPTY;
            break;
          }
        }
      });

      // Damage player
      if (isInExplosion(bomb)) {
        loseLife();
      }
    }

    function isInExplosion(bomb) {
      // Simple check: player on bomb tile or in range line
      if (player.x === bomb.x && player.y === bomb.y) return true;
      if (player.y === bomb.y && Math.abs(player.x - bomb.x) <= bomb.range) return true;
      if (player.x === bomb.x && Math.abs(player.y - bomb.y) <= bomb.range) return true;
      return false;
    }

    function loseLife() {
      lives--;
      document.getElementById('livesLabel').textContent = `Lives: ${lives}`;
      player.x = 1;
      player.y = 1;
      if (lives <= 0) {
        alert("Game Over");
        lives = 3;
        currentWorld = 0;
        currentLevel = 0;
        map = levels[currentWorld][currentLevel];
        updateLabels();
      }
    }

    function updateExplosions() {
      for (let i = explosions.length - 1; i >= 0; i--) {
        const e = explosions[i];
        e.timer--;
        if (e.timer <= 0) {
          explosions.splice(i, 1);
        }
      }
    }

    function checkExit() {
      if (map[player.y][player.x] === TILE_EXIT) {
        nextLevel();
      }
    }

    function nextLevel() {
      currentLevel++;
      if (currentLevel >= 3) {
        currentLevel = 0;
        currentWorld++;
        if (currentWorld >= 5) {
          alert("You protected all five flames! Congratulations!");
          currentWorld = 0;
        }
      }
      map = levels[currentWorld][currentLevel];
      player.x = 1;
      player.y = 1;
      updateLabels();
    }

    function updateLabels() {
      document.getElementById('worldLabel').textContent = WORLDS[currentWorld].name;
      document.getElementById('levelLabel').textContent = `Level ${currentLevel + 1}/3`;
    }

    function draw() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);

      // Draw map
      for (let y = 0; y < ROWS; y++) {
        for (let x = 0; x < COLS; x++) {
          const t = map[y][x];
          const px = x * TILE_SIZE;
          const py = y * TILE_SIZE;

          if (t === TILE_SOLID) {
            ctx.fillStyle = "#5b2b1f"; // solid stone
          } else if (t === TILE_SOFT) {
            ctx.fillStyle = "#a65c3a"; // breakable block
          } else if (t === TILE_EXIT) {
            ctx.fillStyle = "#f5c542"; // flame portal
          } else {
            ctx.fillStyle = "#2b1810"; // floor
          }
          ctx.fillRect(px, py, TILE_SIZE, TILE_SIZE);

          // Kurdish rug pattern hint
          if (t === TILE_EMPTY && (x + y) % 4 === 0) {
            ctx.fillStyle = "#c93b3b";
            ctx.fillRect(px + 8, py + 8, 4, 4);
          }
        }
      }

      // Draw bombs
      bombs.forEach(b => {
        ctx.fillStyle = "#000000";
        ctx.beginPath();
        ctx.arc(b.x * TILE_SIZE + TILE_SIZE / 2, b.y * TILE_SIZE + TILE_SIZE / 2, 10, 0, Math.PI * 2);
        ctx.fill();
      });

      // Draw explosions
      explosions.forEach(e => {
        ctx.fillStyle = "#ffdd55";
        ctx.fillRect(e.x * TILE_SIZE + 4, e.y * TILE_SIZE + 4, TILE_SIZE - 8, TILE_SIZE - 8);
      });

      // Draw player (guardian)
      ctx.fillStyle = "#3af5a0";
      ctx.fillRect(player.x * TILE_SIZE + 6, player.y * TILE_SIZE + 4, TILE_SIZE - 12, TILE_SIZE - 8);
      ctx.fillStyle = "#ffffff";
      ctx.fillRect(player.x * TILE_SIZE + 10, player.y * TILE_SIZE + 8, 4, 4); // eye

      requestAnimationFrame(loop);
    }

    function loop() {
      updatePlayer();
      updateBombs();
      updateExplosions();
      checkExit();
      draw();
    }

    updateLabels();
    loop();
  </script>
</body>
</html>
