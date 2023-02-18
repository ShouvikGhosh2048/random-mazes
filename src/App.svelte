<script lang="ts">
  import { onMount } from "svelte";
  import { tweened } from 'svelte/motion';
  import { linear } from 'svelte/easing';

  const GAME_DURATION = 100;

  function shuffle(list: any[]) {
    for (let i = list.length - 1; i > 0; i--) {
      let j = Math.floor(Math.random() * (i + 1));
      let temp = list[i];
      list[i] = list[j];
      list[j] = temp;
    }
  }

  type GridEntry = 'empty' | 'block' | 'coin' | 'goal';

  // Create the maze using randomized depth-first search - https://en.wikipedia.org/wiki/Maze_generation_algorithm#Randomized_depth-first_search
  function createMaze(grid: GridEntry[][], i: number, j: number) {
    grid[i][j] = 'empty';

    let options = [
      [i,j-2],
      [i,j+2],
      [i-2,j],
      [i+2,j]
    ].filter(([i,j]) =>
      0 <= i && i < grid.length
      && 0 <= j && j < grid[0].length
    );

    shuffle(options);
    options.forEach(([nextI, nextJ]) => {
      if (grid[nextI][nextJ] === 'empty') {
        return;
      }
      grid[(i + nextI)/2][(j + nextJ)/2] = 'empty';
      createMaze(grid, nextI, nextJ);
    });
  }

  function generateRandomGrid() {
    let grid = [] as GridEntry[][];
    for (let i = 0; i < 21; i++) {
      grid.push([]);
      for (let j = 0; j < 21; j++) {
        grid[i].push('block');
      }
    }
    createMaze(grid, 0, 0);

    let possibleGoalPositions = [];
    for (let i = 0; i < 21; i += 2) {
      for (let j = 0; j < 21; j += 2) {
        if (i === 0 && j === 0) {
          continue;
        }

        let numberOfNeighbours = [
          [i,j-1],
          [i,j+1],
          [i-1,j],
          [i+1,j]
        ].filter(([i,j]) =>
          0 <= i && i < grid.length
          && 0 <= j && j < grid[0].length
          && grid[i][j] === 'empty'
        ).length;

        if (numberOfNeighbours === 1) {
          possibleGoalPositions.push([i,j]);
        }
      }
    }

    let [goalI, goalJ] = possibleGoalPositions[Math.floor(Math.random() * possibleGoalPositions.length)];
    grid[goalI][goalJ] = 'goal';

    let totalNumberOfCoins = 0;
    for (let i = 0; i < 21; i++) {
      for (let j = 0; j < 21; j++) {
        if (i === 0 && j === 0 || grid[i][j] !== 'empty') {
          continue;
        }
        
        if (Math.random() < 0.02) {
          grid[i][j] = 'coin';
          totalNumberOfCoins += 1;
        }
      }
    }

    return [grid, totalNumberOfCoins] as [GridEntry[][], number];
  }

  let [grid, totalNumberOfCoins] = generateRandomGrid();

  let playerX = 0;
  let playerY = 0;

  // Location of the center of the canvas while drawing.
  let drawCenter = tweened({
    x: 0,
    y: 0,
  }, {
    duration: 190,
    easing: linear
  });

  let isPlayerMoving = false;
  let currentKey = null; // The last key (w,a,s,d) which was pressed down, if it is still down.
  let numberOfCoins = 0;
  let gameState = 'playing' as 'playing' | 'won' | 'lost';

  let timeLeft = GAME_DURATION;
  let interval = null; // The interval which sets the time left.

  let canvas: undefined | HTMLCanvasElement;
  let ctx: undefined | CanvasRenderingContext2D;

  onMount(() => {
    ctx = canvas.getContext('2d');

    let startTime = Date.now();
    interval = setInterval(() => {
      timeLeft = Math.max(0, GAME_DURATION - Math.floor((Date.now() - startTime) / 1000));
      if (timeLeft === 0 && interval !== null) {
        clearInterval(interval);
        interval = null;

        if (gameState === 'playing') {
          gameState = 'lost';
        }
      }
    }, 990);

    return () => { 
      if (interval !== null) {
        clearInterval(interval);
        interval = null;
      } 
    };
  });

  // Movement
  $: {
    if (currentKey && gameState === 'playing' && !isPlayerMoving) {
      if (currentKey === 'w') {
        if (playerY > 0 && grid[playerY - 1][playerX] !== 'block') {
          playerY--;
        }
      }
      else if (currentKey === 'a') {
        if (playerX > 0 && grid[playerY][playerX - 1] !== 'block') {
          playerX--;
        }
      }
      else if (currentKey === 's') {
        if (playerY + 1 < grid.length && grid[playerY + 1][playerX] !== 'block') {
          playerY++;
        }
      }
      else if (currentKey === 'd') {
        if (playerX + 1 < grid[0].length && grid[playerY][playerX + 1] !== 'block') {
          playerX++;
        }
      }

      isPlayerMoving = true;
      drawCenter.set({
        x: playerX,
        y: playerY,
      }).then(() => {
        isPlayerMoving = false;

        if (grid[playerY][playerX] === 'goal') {
          gameState = 'won';
          if (interval !== null) {
            clearInterval(interval);
            interval = null;
          }
          return;
        }

        // Collect coins
        if (grid[playerY][playerX] === 'coin') {
          grid[playerY][playerX] = 'empty';
          numberOfCoins++;
        }
      });
    }
  }

  $: {
    if (ctx) {
      ctx.clearRect(0, 0, 300, 300);

      for (let i = -3; i <= 3; i++) {
        for (let j = -3; j <= 3; j++) {
          if (playerX + i < 0 || playerX + i >= grid[0].length
          || playerY + j < 0 || playerY + j >= grid.length
          || grid[playerY + j][playerX + i] === 'block') {
            ctx.fillStyle = 'rgb(160, 83, 28)';
            ctx.fillRect(Math.floor((i + 2 - $drawCenter.x + playerX) * 60), Math.floor((j + 2 - $drawCenter.y + playerY) * 60), 60, 60);
          }
          else if (grid[playerY + j][playerX + i] === 'goal') {
            ctx.fillStyle = 'rgb(11, 128, 0)';
            ctx.fillRect(Math.floor((i + 2 - $drawCenter.x + playerX) * 60), Math.floor((j + 2 - $drawCenter.y + playerY) * 60), 60, 60);
          }
          else if (grid[playerY + j][playerX + i] === 'coin') {
            ctx.fillStyle = 'rgb(255, 187, 0)';
            let x = Math.floor((i + 2 - $drawCenter.x + playerX) * 60);
            let y = Math.floor((j + 2 - $drawCenter.y + playerY) * 60);
            ctx.beginPath();
            ctx.moveTo(x + 20, y + 15);
            ctx.lineTo(x + 40, y + 15);
            ctx.arc(x + 40, y + 20, 5, -Math.PI/2, 0);
            ctx.lineTo(x + 45, y + 40);
            ctx.arc(x + 40, y + 40, 5, 0, Math.PI / 2);
            ctx.lineTo(x + 20, y + 45);
            ctx.arc(x + 20, y + 40, 5, Math.PI/2, Math.PI);
            ctx.lineTo(x + 15, y + 20);
            ctx.arc(x + 20, y + 20, 5, Math.PI, Math.PI * 3 / 2);
            ctx.fill();
          }
        }
      }

      // Draw player
      ctx.fillStyle = 'blue';
      ctx.beginPath();
      ctx.moveTo(130, 125);
      ctx.lineTo(170, 125);
      ctx.arc(170, 130, 5, -Math.PI/2, 0);
      ctx.lineTo(175, 170);
      ctx.arc(170, 170, 5, 0, Math.PI / 2);
      ctx.lineTo(130, 175);
      ctx.arc(130, 170, 5, Math.PI/2, Math.PI);
      ctx.lineTo(125, 130);
      ctx.arc(130, 130, 5, Math.PI, Math.PI * 3 / 2);
      ctx.fill();
    }
  }

  function newGame() {
    [grid, totalNumberOfCoins] = generateRandomGrid();

    playerX = 0;
    playerY = 0;
    drawCenter.set({
      x: 0,
      y: 0,
    }, {
      duration: 0, // Immediately set the x and y.
    });
    isPlayerMoving = false;
    currentKey = null;
    numberOfCoins = 0;
    gameState = 'playing';

    timeLeft = GAME_DURATION;
    if (interval) {
      clearInterval(interval);
    }
    let startTime = Date.now();
    interval = setInterval(() => {
      timeLeft = Math.max(0, GAME_DURATION - Math.floor((Date.now() - startTime) / 1000));
      if (timeLeft === 0 && interval !== null) {
        clearInterval(interval);
        interval = null;

        if (gameState === 'playing') {
          gameState = 'lost';
        }
      }
    }, 990);
  };
</script>

<svelte:window  
  on:keydown={(e) => { 
      if (e.key === 'w' || e.key === 'a' || e.key === 's' || e.key === 'd') { 
        currentKey = e.key; 
      }
  }}
  on:keyup={(e) => {
    if (e.key === currentKey) {
      currentKey = null;
    }
  }}/>

<main>
  <div class="gameInformation">
    <span>Coins: {numberOfCoins} / {totalNumberOfCoins}</span>
    <span>Time left: {timeLeft}</span>
  </div>
  <div class="canvasContainer">
    <canvas
      bind:this={canvas}
      width={300}
      height={300}
    ></canvas>
    {#if gameState === 'won' && !isPlayerMoving}
      <div class="gameResult">
        <div>
          <span>You won!</span>
          <button on:click={newGame}>New game</button>
        </div>
      </div>
    {:else if gameState === 'lost' && !isPlayerMoving}
      <div class="gameResult">
        <div>
          <span>Time is up!</span>
          <button on:click={newGame}>New game</button>
        </div>
      </div>
    {/if}
  </div>
</main>

<style>
  main {
    width: min-content;
    padding: 0;
    margin: auto;
  }

  .gameInformation {
    display: flex;
    justify-content: space-between;
    padding: 10px 3px;
  }

  .canvasContainer {
    position: relative;
  }

  .gameResult {
    background-color: rgba(255, 255, 255, 0.3);
    position: absolute;
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .gameResult div {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 20px;
  }

  .gameResult span {
    font-size: x-large;
    font-weight: bold;
  }

  .gameResult button {
    padding: 7px;
    background-color: rgb(14, 121, 192);
    color: white;
    border: none;
    border-radius: 3px;
    cursor: pointer;
  }
</style>