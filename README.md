[memory_game.html](https://github.com/user-attachments/files/24414664/memory_game.html)
<!doctype html>
<html lang="ja" class="h-full">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>神経衰弱ゲーム（単体版）</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <style>
    body { box-sizing: border-box; }
    .card-matched { opacity: 0.5; pointer-events: none; }
  </style>
</head>

<body class="h-full">
  <div id="app" class="w-full h-full"></div>

  <script>
    const defaultConfig = {
      game_title: "神経衰弱ゲーム",
      subtitle: "",
      play_button: "ゲームスタート",
      new_game_button: "もう一度プレイ",
      background_color: "#f0f4f8",
      card_back_color: "#4f46e5",
      card_front_color: "#ffffff",
      primary_action_color: "#6366f1",
      text_color: "#1e293b",
      font_family: "system-ui",
      font_size: 16
    };

    // ★ 堅牢シャッフル
    function shuffle(array) {
      for (let i = array.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [array[i], array[j]] = [array[j], array[i]];
      }
      return array;
    }

    // ★ key（文字列）で一致判定する版
    const allNumberPairs = [
      { percent: "10%", decimal: "0.1",  key: "0.1" },
      { percent: "25%", decimal: "0.25", key: "0.25" },
      { percent: "50%", decimal: "0.5",  key: "0.5" },
      { percent: "75%", decimal: "0.75", key: "0.75" },
      { percent: "20%", decimal: "0.2",  key: "0.2" },
      { percent: "40%", decimal: "0.4",  key: "0.4" },
      { percent: "60%", decimal: "0.6",  key: "0.6" },
      { percent: "80%", decimal: "0.8",  key: "0.8" },
      { percent: "30%", decimal: "0.3",  key: "0.3" },
      { percent: "90%", decimal: "0.9",  key: "0.9" },
      { percent: "15%", decimal: "0.15", key: "0.15" }
    ];

    let gameState = {
      cards: [],
      flippedCards: [],
      matchedPairs: 0,
      moves: 0,
      score: 0,
      isProcessing: false,
      gameStarted: false,
      difficulty: 'easy', // 'easy' | 'medium' | 'hard' | 'vs-computer' | 'vs-computer-hard'
      totalPairs: 4,
      startTime: null,
      elapsedTime: 0,
      timerInterval: null,
      isVsComputer: false,
      currentPlayer: 'player',
      playerScore: 0,
      computerScore: 0,
      computerMemory: []
    };

    function initializeGame(difficulty) {
      if (difficulty) gameState.difficulty = difficulty;

      let pairsCount;
      gameState.isVsComputer = false;

      if (gameState.difficulty === 'easy') {
        pairsCount = 4;  // 8枚
      } else if (gameState.difficulty === 'medium') {
        pairsCount = 8;  // 16枚
      } else if (gameState.difficulty === 'hard') {
        pairsCount = 10; // 20枚
      } else if (gameState.difficulty === 'vs-computer') {
        pairsCount = 6;  // 12枚
        gameState.isVsComputer = true;
      } else if (gameState.difficulty === 'vs-computer-hard') {
        pairsCount = 8;  // 16枚
        gameState.isVsComputer = true;
      }

      gameState.totalPairs = pairsCount;

      // ★ 毎回ランダムにペア候補を選ぶ
      const candidates = shuffle([...allNumberPairs]).slice(0, pairsCount);

      const allCards = [];
      candidates.forEach(pair => {
        allCards.push({ display: pair.percent, key: pair.key, type: 'percent' });
        allCards.push({ display: pair.decimal, key: pair.key, type: 'decimal' });
      });

      shuffle(allCards);

      gameState.cards = allCards.map((card, index) => ({
        id: index,
        display: card.display,
        key: card.key,
        type: card.type,
        isFlipped: false,
        isMatched: false
      }));

      gameState.flippedCards = [];
      gameState.matchedPairs = 0;
      gameState.moves = 0;
      gameState.score = 0;
      gameState.isProcessing = false;

      gameState.startTime = Date.now();
      gameState.elapsedTime = 0;
      gameState.currentPlayer = 'player';
      gameState.playerScore = 0;
      gameState.computerScore = 0;
      gameState.computerMemory = [];

      if (gameState.timerInterval) clearInterval(gameState.timerInterval);
      gameState.timerInterval = setInterval(() => {
        if (gameState.matchedPairs < gameState.totalPairs) {
          gameState.elapsedTime = Math.floor((Date.now() - gameState.startTime) / 1000);
          updateTimerDisplay();
        }
      }, 1000);
    }

    function updateTimerDisplay() {
      const timerElement = document.getElementById('timer-display');
      if (timerElement) timerElement.textContent = `⏱️ ${gameState.elapsedTime}秒`;
    }

    function createCard(card, config) {
      const baseSize = config.font_size || defaultConfig.font_size;
      const fontFamily = config.font_family || defaultConfig.font_family;
      const cardBackColor = config.card_back_color || defaultConfig.card_back_color;
      const cardFrontColor = config.card_front_color || defaultConfig.card_front_color;

      const cardDiv = document.createElement('div');
      cardDiv.className = 'relative w-20 h-28 cursor-pointer transition-all duration-300 hover:scale-105';
      cardDiv.dataset.cardId = card.id;

      const cardInner = document.createElement('div');
      cardInner.className = 'w-full h-full rounded-xl shadow-lg flex items-center justify-center';
      cardInner.style.fontFamily = `${fontFamily}, Arial, sans-serif`;

      if (card.isMatched) cardInner.classList.add('card-matched');

      if (card.isFlipped || card.isMatched) {
        cardInner.style.backgroundColor = cardFrontColor;
        cardInner.style.fontSize = `${baseSize * 1.8}px`;
        cardInner.style.color = card.type === 'percent' ? '#2563eb' : (config.text_color || defaultConfig.text_color);
        cardInner.style.fontWeight = 'bold';
        cardInner.textContent = card.display;
      } else {
        cardInner.style.backgroundColor = cardBackColor;
        cardInner.innerHTML = `<div style="font-size: ${baseSize * 1.5}px; color: white;">?</div>`;
      }

      cardDiv.appendChild(cardInner);
      cardDiv.addEventListener('click', () => handleCardClick(card.id));
      return cardDiv;
    }

    function handleCardClick(cardId) {
      if (gameState.isProcessing) return;
      if (gameState.isVsComputer && gameState.currentPlayer === 'computer') return;

      const card = gameState.cards.find(c => c.id === cardId);
      if (!card || card.isFlipped || card.isMatched) return;

      card.isFlipped = true;
      gameState.flippedCards.push(card);

      if (gameState.isVsComputer) {
        if (gameState.computerMemory.findIndex(m => m.id === card.id) === -1) {
          gameState.computerMemory.push({ id: card.id, key: card.key, display: card.display });
        }
      }

      renderGame();

      if (gameState.flippedCards.length === 2) {
        gameState.isProcessing = true;
        gameState.moves++;

        const [card1, card2] = gameState.flippedCards;

        if (card1.key === card2.key && card1.id !== card2.id) {
          setTimeout(() => {
            card1.isMatched = true;
            card2.isMatched = true;
            gameState.matchedPairs++;

            if (gameState.isVsComputer) {
              if (gameState.currentPlayer === 'player') gameState.playerScore++;
              else gameState.computerScore++;
            } else {
              gameState.score += 10;
            }

            gameState.flippedCards = [];
            gameState.isProcessing = false;

            if (gameState.matchedPairs === gameState.totalPairs) {
              if (gameState.timerInterval) {
                clearInterval(gameState.timerInterval);
                gameState.timerInterval = null;
              }
              gameState.elapsedTime = Math.floor((Date.now() - gameState.startTime) / 1000);
              setTimeout(() => showCompletionMessage(), 200);
            } else if (gameState.isVsComputer && gameState.currentPlayer === 'computer') {
              setTimeout(() => computerTurn(), 700);
            }

            renderGame();
          }, 500);
        } else {
          setTimeout(() => {
            card1.isFlipped = false;
            card2.isFlipped = false;
            gameState.flippedCards = [];
            gameState.isProcessing = false;

            if (gameState.isVsComputer) {
              gameState.currentPlayer = gameState.currentPlayer === 'player' ? 'computer' : 'player';
              renderGame();
              if (gameState.currentPlayer === 'computer') setTimeout(() => computerTurn(), 700);
            } else {
              renderGame();
            }
          }, 800);
        }
      }
    }

    function computerTurn() {
      if (gameState.matchedPairs >= gameState.totalPairs) return;
      if (gameState.isProcessing) return;

      const availableCards = gameState.cards.filter(c => !c.isMatched && !c.isFlipped);
      if (availableCards.length < 2) return;

      let card1ToFlip = null;
      let card2ToFlip = null;

      const useMemory = Math.random() < 0.6;

      if (useMemory) {
        for (let i = 0; i < gameState.computerMemory.length; i++) {
          for (let j = i + 1; j < gameState.computerMemory.length; j++) {
            const mem1 = gameState.computerMemory[i];
            const mem2 = gameState.computerMemory[j];
            if (mem1.key === mem2.key) {
              const c1 = gameState.cards.find(c => c.id === mem1.id && !c.isMatched && !c.isFlipped);
              const c2 = gameState.cards.find(c => c.id === mem2.id && !c.isMatched && !c.isFlipped);
              if (c1 && c2) { card1ToFlip = c1; card2ToFlip = c2; break; }
            }
          }
          if (card1ToFlip) break;
        }
      }

      if (!card1ToFlip) {
        card1ToFlip = availableCards[Math.floor(Math.random() * availableCards.length)];
        const rest = availableCards.filter(c => c.id !== card1ToFlip.id);
        card2ToFlip = rest[Math.floor(Math.random() * rest.length)];
      }

      setTimeout(() => {
        if (card1ToFlip && !card1ToFlip.isMatched && !card1ToFlip.isFlipped) {
          card1ToFlip.isFlipped = true;
          gameState.flippedCards.push(card1ToFlip);
          if (gameState.computerMemory.findIndex(m => m.id === card1ToFlip.id) === -1) {
            gameState.computerMemory.push({ id: card1ToFlip.id, key: card1ToFlip.key, display: card1ToFlip.display });
          }
          renderGame();
        }
      }, 300);

      setTimeout(() => {
        if (card2ToFlip && !card2ToFlip.isMatched && !card2ToFlip.isFlipped && gameState.flippedCards.length === 1) {
          card2ToFlip.isFlipped = true;
          gameState.flippedCards.push(card2ToFlip);
          if (gameState.computerMemory.findIndex(m => m.id === card2ToFlip.id) === -1) {
            gameState.computerMemory.push({ id: card2ToFlip.id, key: card2ToFlip.key, display: card2ToFlip.display });
          }
          renderGame();

          gameState.isProcessing = true;
          gameState.moves++;

          const [firstCard, secondCard] = gameState.flippedCards;

          if (firstCard.key === secondCard.key && firstCard.id !== secondCard.id) {
            setTimeout(() => {
              firstCard.isMatched = true;
              secondCard.isMatched = true;
              gameState.matchedPairs++;
              gameState.computerScore++;
              gameState.flippedCards = [];
              gameState.isProcessing = false;

              if (gameState.matchedPairs === gameState.totalPairs) {
                if (gameState.timerInterval) {
                  clearInterval(gameState.timerInterval);
                  gameState.timerInterval = null;
                }
                gameState.elapsedTime = Math.floor((Date.now() - gameState.startTime) / 1000);
                setTimeout(() => showCompletionMessage(), 200);
              } else {
                renderGame();
                setTimeout(() => computerTurn(), 700);
              }
            }, 500);
          } else {
            setTimeout(() => {
              firstCard.isFlipped = false;
              secondCard.isFlipped = false;
              gameState.flippedCards = [];
              gameState.isProcessing = false;
              gameState.currentPlayer = 'player';
              renderGame();
            }, 800);
          }
        }
      }, 1000);
    }

    function showCompletionMessage() {
      const overlay = document.createElement('div');
      overlay.className = 'fixed inset-0 flex items-center justify-center p-4';
      overlay.style.backgroundColor = 'rgba(0, 0, 0, 0.7)';
      overlay.style.zIndex = '1000';

      const config = defaultConfig;
      const baseSize = config.font_size;
      const fontFamily = config.font_family;
      const primaryColor = config.primary_action_color;
      const cardFrontColor = config.card_front_color;
      const textColor = config.text_color;

      const messageBox = document.createElement('div');
      messageBox.className = 'rounded-2xl shadow-2xl p-8 max-w-md text-center';
      messageBox.style.backgroundColor = cardFrontColor;
      messageBox.style.fontFamily = `${fontFamily}, Arial, sans-serif`;

      const icon = document.createElement('div');
      icon.style.fontSize = `${baseSize * 3}px`;
      icon.className = 'mb-4';

      const title = document.createElement('h2');
      title.style.fontSize = `${baseSize * 1.8}px`;
      title.style.color = textColor;
      title.className = 'font-bold mb-4';

      if (gameState.isVsComputer) {
        if (gameState.playerScore > gameState.computerScore) { icon.textContent = '🎉'; title.textContent = 'あなたの勝ち！'; }
        else if (gameState.playerScore < gameState.computerScore) { icon.textContent = '😢'; title.textContent = 'コンピューターの勝ち'; }
        else { icon.textContent = '🤝'; title.textContent = '引き分け！'; }

        const finalScore = document.createElement('p');
        finalScore.style.fontSize = `${baseSize * 1.4}px`;
        finalScore.style.color = primaryColor;
        finalScore.className = 'font-bold mb-6';
        finalScore.textContent = `あなた: ${gameState.playerScore} - コンピューター: ${gameState.computerScore}`;

        messageBox.appendChild(icon);
        messageBox.appendChild(title);
        messageBox.appendChild(finalScore);
      } else {
        icon.textContent = '🎉';
        title.textContent = 'おめでとうございます！';

        const timeStats = document.createElement('p');
        timeStats.style.fontSize = `${baseSize * 1.2}px`;
        timeStats.style.color = textColor;
        timeStats.className = 'mb-3 font-semibold';
        timeStats.textContent = `クリアタイム: ${gameState.elapsedTime}秒`;

        const scoreDisplay = document.createElement('p');
        scoreDisplay.style.fontSize = `${baseSize * 1.4}px`;
        scoreDisplay.style.color = primaryColor;
        scoreDisplay.className = 'font-bold mb-6';
        scoreDisplay.textContent = `スコア: ${gameState.score}点`;

        messageBox.appendChild(icon);
        messageBox.appendChild(title);
        messageBox.appendChild(timeStats);
        messageBox.appendChild(scoreDisplay);
      }

      const closeButton = document.createElement('button');
      closeButton.style.backgroundColor = primaryColor;
      closeButton.style.fontSize = `${baseSize}px`;
      closeButton.textContent = '閉じる';
      closeButton.className = 'px-6 py-3 rounded-lg text-white font-semibold hover:opacity-90 transition-opacity';
      closeButton.addEventListener('click', () => document.body.removeChild(overlay));

      messageBox.appendChild(closeButton);
      overlay.appendChild(messageBox);
      document.body.appendChild(overlay);
    }

    function renderGame() {
      const config = defaultConfig;
      const app = document.getElementById('app');

      if (!gameState.gameStarted) {
        renderStartScreen(app, config);
        return;
      }

      const baseSize = config.font_size;
      const fontFamily = config.font_family;
      const backgroundColor = config.background_color;
      const primaryColor = config.primary_action_color;
      const textColor = config.text_color;
      const gameTitle = config.game_title;
      const newGameButton = config.new_game_button;

      app.innerHTML = '';
      app.style.backgroundColor = backgroundColor;
      app.style.fontFamily = `${fontFamily}, Arial, sans-serif`;
      app.className = 'w-full h-full overflow-auto flex flex-col items-center justify-center p-8';

      const container = document.createElement('div');
      container.className = 'w-full max-w-2xl';

      const header = document.createElement('div');
      header.className = 'text-center mb-8';

      const title = document.createElement('h1');
      title.style.fontSize = `${baseSize * 2}px`;
      title.style.color = textColor;
      title.textContent = gameTitle;
      title.className = 'font-bold mb-2';

      header.appendChild(title);

      if (gameState.isVsComputer) {
        const turn = document.createElement('div');
        turn.style.fontSize = `${baseSize * 1.3}px`;
        turn.style.color = gameState.currentPlayer === 'player' ? primaryColor : '#10b981';
        turn.textContent = gameState.currentPlayer === 'player' ? '👤 あなたのターン' : '🤖 コンピューターのターン';
        turn.className = 'mb-3 font-bold';

        const score = document.createElement('div');
        score.style.fontSize = `${baseSize * 1.1}px`;
        score.style.color = textColor;
        score.textContent = `あなた: ${gameState.playerScore} | コンピューター: ${gameState.computerScore} | ペア: ${gameState.matchedPairs}/${gameState.totalPairs}`;
        score.className = 'mb-4';

        header.appendChild(turn);
        header.appendChild(score);
      } else {
        const timer = document.createElement('div');
        timer.id = 'timer-display';
        timer.style.fontSize = `${baseSize * 1.5}px`;
        timer.style.color = primaryColor;
        timer.textContent = `⏱️ ${gameState.elapsedTime}秒`;
        timer.className = 'mb-3 font-bold';

        const stats = document.createElement('div');
        stats.style.fontSize = `${baseSize * 1.1}px`;
        stats.style.color = textColor;
        stats.textContent = `手数: ${gameState.moves} | ペア: ${gameState.matchedPairs}/${gameState.totalPairs} | スコア: ${gameState.score}点`;
        stats.className = 'mb-4';

        header.appendChild(timer);
        header.appendChild(stats);
      }

      const buttonRow = document.createElement('div');
      buttonRow.className = 'flex gap-3 justify-center items-center';

      const reset = document.createElement('button');
      reset.style.backgroundColor = primaryColor;
      reset.textContent = newGameButton;
      reset.className = 'px-6 py-2 rounded-lg text-white font-semibold hover:opacity-90 transition-opacity';
      reset.addEventListener('click', () => { initializeGame(); renderGame(); });

      const back = document.createElement('button');
      back.style.backgroundColor = config.card_front_color;
      back.style.color = textColor;
      back.style.border = `2px solid ${primaryColor}`;
      back.textContent = '最初のページへ戻る';
      back.className = 'px-6 py-2 rounded-lg font-semibold hover:opacity-90 transition-opacity';
      back.addEventListener('click', () => {
        if (gameState.timerInterval) { clearInterval(gameState.timerInterval); gameState.timerInterval = null; }
        gameState.gameStarted = false;
        gameState.isVsComputer = false;
        renderGame();
      });

      buttonRow.appendChild(reset);
      buttonRow.appendChild(back);
      header.appendChild(buttonRow);

      const grid = document.createElement('div');
      let gridCols;
      if (gameState.difficulty === 'easy') gridCols = 'grid-cols-4';
      else if (gameState.difficulty === 'medium' || gameState.difficulty === 'vs-computer-hard') gridCols = 'grid-cols-4';
      else if (gameState.difficulty === 'vs-computer') gridCols = 'grid-cols-4';
      else gridCols = 'grid-cols-5';
      grid.className = `grid ${gridCols} gap-3 justify-items-center`;

      gameState.cards.forEach(card => grid.appendChild(createCard(card, config)));

      container.appendChild(header);
      container.appendChild(grid);
      app.appendChild(container);
    }

    function renderStartScreen(app, config) {
      const baseSize = config.font_size;
      const backgroundColor = config.background_color;
      const primaryColor = config.primary_action_color;
      const textColor = config.text_color;
      const gameTitle = config.game_title;

      app.innerHTML = '';
      app.style.backgroundColor = backgroundColor;
      app.className = 'w-full h-full flex items-center justify-center p-8';

      const box = document.createElement('div');
      box.className = 'text-center max-w-md';

      const icon = document.createElement('div');
      icon.style.fontSize = `${baseSize * 4}px`;
      icon.textContent = '🃏';
      icon.className = 'mb-6';

      const title = document.createElement('h1');
      title.style.fontSize = `${baseSize * 2.5}px`;
      title.style.color = textColor;
      title.textContent = gameTitle;
      title.className = 'font-bold mb-4';

      const instructions = document.createElement('div');
      instructions.style.fontSize = `${baseSize}px`;
      instructions.style.color = textColor;
      instructions.className = 'mb-8 text-left space-y-2';
      instructions.innerHTML = `
        <p>• カードをクリックしてめくる</p>
        <p>• パーセントと小数のペアを見つける</p>
        <p>• 例：25%と0.25はペア！</p>
      `;

      const label = document.createElement('p');
      label.style.fontSize = `${baseSize * 1.2}px`;
      label.style.color = textColor;
      label.textContent = '難易度を選択：';
      label.className = 'font-semibold mb-4';

      const buttons = document.createElement('div');
      buttons.className = 'flex flex-col gap-3 justify-center items-center';

      function makeButton(text, color, difficulty) {
        const b = document.createElement('button');
        b.style.backgroundColor = color;
        b.style.fontSize = `${baseSize * 1.1}px`;
        b.textContent = text;
        b.className = 'w-64 px-6 py-3 rounded-lg text-white font-bold hover:opacity-90 transition-opacity';
        b.addEventListener('click', () => {
          initializeGame(difficulty);
          gameState.gameStarted = true;
          renderGame();
        });
        return b;
      }

      buttons.appendChild(makeButton('① かんたん (8枚)',  primaryColor, 'easy'));
      buttons.appendChild(makeButton('② ふつう (16枚)',     primaryColor, 'medium'));
      buttons.appendChild(makeButton('③ むずかしい (20枚)', primaryColor, 'hard'));
      buttons.appendChild(makeButton('④ 対戦 (12枚)', '#10b981', 'vs-computer'));
      buttons.appendChild(makeButton('⑤ 対戦 (16枚)', '#10b981', 'vs-computer-hard'));

      box.appendChild(icon);
      box.appendChild(title);
      box.appendChild(instructions);
      box.appendChild(label);
      box.appendChild(buttons);
      app.appendChild(box);
    }

    // 起動
    initializeGame('easy');
    renderStartScreen(document.getElementById('app'), defaultConfig);
  </script>
</body>
</html>
