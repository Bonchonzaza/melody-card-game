<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Melody Card Game</title>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/tailwindcss/2.2.19/tailwind.min.css">
  <style>
    .card-hover {
      transition: transform 0.2s;
    }
    .card-hover:hover {
      transform: scale(1.02);
    }
    .gradient-bg {
      background: linear-gradient(to bottom, #e9d5ff, #bfdbfe);
    }
  </style>
</head>
<body class="gradient-bg min-h-screen">
  <div id="app" class="container mx-auto p-8 max-w-4xl"></div>

  <script>
    const app = document.getElementById('app');
    
    // Game state เกมรอบแรก
    let gameState = 'start';
    let players = 1;
    let roles = Array(6).fill(null);
    let selectedSong = '';
    let timeLeft = 10;
    let scores = Array(6).fill(0);
    let micStates = Array(6).fill(false);
    let currentRound = 1;
    let winner = null;

    const songs = [
      { id: 1, title: 'สองใจ', artist: 'ดา Endorphine', difficulty: 'ง่าย' },
      { id: 2, title: 'วันเกิดฉันปีนี้', artist: 'Three Man Down', difficulty: 'ปานกลาง' },
      { id: 3, title: 'แน่ใจไหม', artist: 'นนท์ ธนนท์', difficulty: 'ยาก' }
    ];

    function renderStart() {
      app.innerHTML = `
        <div class="bg-white rounded-lg shadow-lg p-8 text-center">
          <h1 class="text-4xl font-bold text-purple-700 mb-6">ยินดีต้อนรับสู่ Melody Card Game</h1>
          <button 
            class="px-6 py-3 bg-gradient-to-r from-purple-600 to-blue-600 text-white rounded-lg hover:from-purple-700 hover:to-blue-700 transition"
            onclick="startGame()"
          >
            เริ่มเกม
          </button>
        </div>
      `;
    }

    function renderWaiting() {
      app.innerHTML = `
        <div class="bg-white rounded-lg shadow-lg p-8 text-center">
          <h2 class="text-3xl font-bold text-purple-700 mb-6">รอผู้เล่น... (${players}/6)</h2>
          <div class="flex justify-center gap-4">
            ${Array(6).fill(null).map((_, i) => `
              <div class="w-12 h-12 rounded-full ${i < players ? 'bg-green-500' : 'bg-gray-300'}"></div>
            `).join('')}
          </div>
        </div>
      `;
    }

    function renderRoleSelection() {
      app.innerHTML = `
        <div class="bg-white rounded-lg shadow-lg p-8 text-center">
          <h2 class="text-3xl font-bold text-purple-700 mb-6">เลือกการ์ดของคุณ (รอบที่ ${currentRound}/3)</h2>
          <div class="grid grid-cols-2 md:grid-cols-3 gap-6">
            ${roles.map((role, i) => `
              <div 
                class="bg-white rounded-lg shadow p-6 cursor-pointer card-hover ${role ? 'bg-purple-100' : ''}"
                onclick="assignRole(${i})"
              >
                <div class="text-2xl font-bold mb-2">ผู้เล่น ${i + 1}</div>
                <div>${role || '?'}</div>
              </div>
            `).join('')}
          </div>
        </div>
      `;
    }

    function renderSongSelection() {
      app.innerHTML = `
        <div class="bg-white rounded-lg shadow-lg p-8 text-center">
          <h2 class="text-3xl font-bold text-purple-700 mb-6">เลือกเพลง (รอบที่ ${currentRound}/3)</h2>
          <div class="space-y-4">
            ${songs.map(song => `
              <div 
                class="bg-white rounded-lg shadow p-4 cursor-pointer card-hover ${selectedSong === song.title ? 'bg-purple-100' : ''}"
                onclick="selectSong('${song.title}')"
              >
                <h3 class="font-bold">${song.title}</h3>
                <p class="text-gray-600">${song.artist}</p>
                <p class="text-sm text-purple-600">ระดับความยาก: ${song.difficulty}</p>
              </div>
            `).join('')}
          </div>
          <button 
            class="mt-6 px-6 py-3 bg-gradient-to-r from-purple-600 to-blue-600 text-white rounded-lg ${!selectedSong ? 'opacity-50 cursor-not-allowed' : 'hover:from-purple-700 hover:to-blue-700'}"
            onclick="startGamePlay()"
            ${!selectedSong ? 'disabled' : ''}
          >
            เริ่มเล่น
          </button>
        </div>
      `;
    }

    function renderPlaying() {
      app.innerHTML = `
        <div class="bg-white rounded-lg shadow-lg p-8 text-center">
          <h2 class="text-3xl font-bold text-purple-700 mb-2">รอบที่ ${currentRound}/3</h2>
          <h3 class="text-xl text-purple-600 mb-4">เพลง: ${selectedSong}</h3>
          <div class="text-2xl font-bold mb-6">เวลา: ${timeLeft} วินาที</div>
          <div class="grid grid-cols-2 md:grid-cols-3 gap-6">
            ${roles.map((role, i) => `
              <div class="bg-white rounded-lg shadow p-4">
                <div class="font-bold mb-2">ผู้เล่น ${i + 1}</div>
                <div class="text-sm mb-2">${role}</div>
                <button
                  class="w-full px-4 py-2 rounded-lg ${micStates[i] ? 'bg-red-500' : 'bg-green-500'} text-white"
                  onclick="toggleMic(${i})"
                >
                  ${micStates[i] ? '🔊 ปิดไมค์' : '🔇 เปิดไมค์'}
                </button>
              </div>
            `).join('')}
          </div>
        </div>
      `;
    }

    function renderScores() {
      app.innerHTML = `
        <div class="bg-white rounded-lg shadow-lg p-8 text-center">
          <h2 class="text-3xl font-bold text-purple-700 mb-6">ผลการแข่งขัน</h2>
          ${winner !== null ? `
            <h3 class="text-2xl text-green-600 mb-4">
              ผู้ชนะ: ผู้เล่น ${winner + 1} (คะแนน: ${scores[winner]})
            </h3>
          ` : ''}
          <div class="grid gap-4 mb-6">
            ${scores.map((score, i) => `
              <div class="bg-white rounded-lg shadow p-4">
                <div class="font-bold">ผู้เล่น ${i + 1}</div>
                <div class="text-xl">${score} คะแนน</div>
                <div class="text-sm text-gray-600">${roles[i]}</div>
              </div>
            `).join('')}
          </div>
          <button 
            class="px-6 py-3 bg-gradient-to-r from-purple-600 to-blue-600 text-white rounded-lg hover:from-purple-700 hover:to-blue-700"
            onclick="resetGame()"
          >
            เล่นใหม่
          </button>
        </div>
      `;
    }

    // Game functions
    function startGame() {
      gameState = 'waiting';
      renderWaiting();
      simulatePlayerJoin();
    }

    function simulatePlayerJoin() {
      const interval = setInterval(() => {
        if (players < 6) {
          players++;
          renderWaiting();
        } else {
          clearInterval(interval);
          gameState = 'roleSelection';
          renderRoleSelection();
        }
      }, 1000);
    }

    function assignRole(index) {
      roles[index] = index === 5 ? 'singer' : 'listener';
      renderRoleSelection();
      if (roles.filter(r => r).length === 6) {
        gameState = 'songSelection';
        renderSongSelection();
      }
    }

    function selectSong(title) {
      selectedSong = title;
      renderSongSelection();
    }

    function startGamePlay() {
      if (!selectedSong) return;
      gameState = 'playing';
      startTimer();
      renderPlaying();
    }

    function toggleMic(playerIndex) {
      micStates[playerIndex] = !micStates[playerIndex];
      renderPlaying();
    }

    function startTimer() {
      const timer = setInterval(() => {
        timeLeft--;
        if (timeLeft >= 0) {
          renderPlaying();
        } else {
          clearInterval(timer);
          calculateScores();
          if (currentRound < 3) {
            startNextRound();
          } else {
            determineWinner();
          }
        }
      }, 1000);
    }

    function calculateScores() {
      micStates.forEach((isMicOn, index) => {
        if (roles[index] === 'singer' && isMicOn) {
          scores[index] += 10;
        } else if (roles[index] === 'listener' && !isMicOn) {
          scores[index] += 5;
        }
      });
    }

    // เริ่มเล่นรอบต่อไป
    function startNextRound() {
      currentRound++;
      timeLeft = 10;
      micStates = Array(6).fill(false);
      selectedSong = '';
      gameState = 'songSelection';
      renderSongSelection();
    }

    function determineWinner() {
      const maxScore = Math.max(...scores);
      winner = scores.indexOf(maxScore);
      gameState = 'scores';
      renderScores();
    }
    // รีเซ็ตเกม
    function resetGame() {
      gameState = 'start';
      players = 1;
      roles = Array(6).fill(null);
      selectedSong = '';
      timeLeft = 10;
      scores = Array(6).fill(0);
      micStates = Array(6).fill(false);
      currentRound = 1;
      winner = null;
      renderStart();
    }

    // Initialize game
    renderStart();
  </script>
</body>
</html>
