<script setup>
import { ref, reactive, onMounted, onBeforeUnmount } from 'vue'

const BOARD_WIDTH = 10
const BOARD_HEIGHT = 20

// 初始化游戏板
const board = reactive(Array.from({ length: BOARD_HEIGHT }, () => 
  Array(BOARD_WIDTH).fill(0)
))

// 当前下落中的方块
const currentPiece = ref(null)
const currentPos = ref({ x: 0, y: 0 })
const score = ref(0)
const gameOver = ref(false)

// 方块形状预设
const TETROMINOES = [
  [[1,1,1,1]], // I型
  [[1,1],[1,1]], // O型
  [[1,1,1],[0,1,0]], // T型
  [[1,1,1],[1,0,0]], // L型
  [[1,1,0],[0,1,1]] // S型
]

// 生成新方块
const spawnPiece = () => {
  const piece = TETROMINOES[Math.floor(Math.random() * TETROMINOES.length)]
  currentPiece.value = piece
  currentPos.value = {
    x: Math.floor(BOARD_WIDTH/2 - piece[0].length/2),
    y: 0
  }
}

// 碰撞检测
const checkCollision = (newX, newY, piece) => {
  if (!piece) return true
  
  for (let y = 0; y < piece.length; y++) {
    for (let x = 0; x < piece[y].length; x++) {
      if (piece[y][x]) {
        const boardX = newX + x
        const boardY = newY + y
        if (
          boardX < 0 ||
          boardX >= BOARD_WIDTH ||
          boardY >= BOARD_HEIGHT ||
          (boardY >= 0 && board[boardY][boardX])
        ) {
          return true
        }
      }
    }
  }
  return false
}

// 键盘控制
const isProcessing = ref(false)
const dropInterval = ref(null)

const handleKeyDown = async (e) => {
  if (gameOver.value || isProcessing.value) return

  isProcessing.value = true
  
  switch(e.key) {
    case 'ArrowLeft':
      move(-1)
      break
    case 'ArrowRight':
      move(1)
      break
    case 'ArrowDown':
      // 持续加速下落
      if (!dropInterval.value) {
        dropInterval.value = setInterval(() => {
          move(0, 1)
        }, 100)
      }
      break
    case 'ArrowUp':
      rotate()
      break
    case ' ':
      drop()
      break
  }
  
  setTimeout(() => {
    isProcessing.value = false
    if (dropInterval.value) {
      clearInterval(dropInterval.value)
      dropInterval.value = null
    }
  }, 100)
}

// 移动方块
const move = (dx, dy = 0) => {
  const newX = currentPos.value.x + dx
  const newY = currentPos.value.y + dy
  if (!checkCollision(newX, newY, currentPiece.value)) {
    currentPos.value.x = newX
    currentPos.value.y = newY
  } else if (dy > 0) {
    mergePiece()
  }
}

// 快速下落
const drop = () => {
  let newY = currentPos.value.y + 1
  while (!checkCollision(currentPos.value.x, newY, currentPiece.value)) {
    newY++
  }
  currentPos.value.y = newY - 1
  mergePiece()
}

// 旋转方块
const rotate = () => {
  const rotated = currentPiece.value[0].map((_, i) =>
    currentPiece.value.map(row => row[i]).reverse()
  )
  
  // 尝试不同的位置来适应旋转
  const kicks = [
    [0, 0],  // 原位置
    [-1, 0], // 左移
    [1, 0],  // 右移
    [0, -1], // 上移
    [-1, -1],// 左上
    [1, -1]  // 右上
  ]
  
  for (const [kickX, kickY] of kicks) {
    const newX = currentPos.value.x + kickX
    const newY = currentPos.value.y + kickY
    if (!checkCollision(newX, newY, rotated)) {
      currentPiece.value = rotated
      currentPos.value.x = newX
      currentPos.value.y = newY
      return
    }
  }
}

// 合并方块到底部
const mergePiece = () => {
  currentPiece.value.forEach((row, y) => {
    row.forEach((value, x) => {
      if (value) {
        const boardY = currentPos.value.y + y
        if (boardY >= 0) {
          board[boardY][currentPos.value.x + x] = 1
        }
      }
    })
  })
  checkLines()
  spawnPiece()
  if (checkCollision(currentPos.value.x, currentPos.value.y, currentPiece.value)) {
    gameOver.value = true
  }
}

// 消除满行
const checkLines = () => {
  let lines = 0
  for (let y = BOARD_HEIGHT - 1; y >= 0; y--) {
    if (board[y].every(cell => cell)) {
      board.splice(y, 1)
      board.unshift(Array(BOARD_WIDTH).fill(0))
      lines++
      y++ // 重新检查当前行
    }
  }
  score.value += lines * 100
}

// 游戏循环
let gameInterval
onMounted(() => {
  spawnPiece()
  gameInterval = setInterval(() => {
    if (!gameOver.value) {
      const newY = currentPos.value.y + 1
      if (checkCollision(currentPos.value.x, newY, currentPiece.value)) {
        mergePiece()
      } else {
        currentPos.value.y = newY
      }
    }
  }, 1000)
  window.addEventListener('keydown', handleKeyDown)
})

onBeforeUnmount(() => {
  clearInterval(gameInterval)
  window.removeEventListener('keydown', handleKeyDown)
})
</script>

<template>
  <div class="game-container">
    <div class="score">得分: {{ score }}</div>
    <div class="board">
      <div 
        v-for="(row, y) in board"
        :key="y"
        class="row"
      >
        <div
          v-for="(cell, x) in row"
          :key="x"
          class="cell"
          :class="{
            'active': currentPiece?.[y - currentPos.y]?.[x - currentPos.x] && (y - currentPos.y >= 0),
            'filled': board[y][x]
          }"
        />
      </div>
    </div>
    <div v-if="gameOver" class="game-over">游戏结束！</div>
  </div>
</template>

<style scoped>
.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  background: #2d2d2d;
  border-radius: 8px;
  width: 340px;
  margin: 20px auto;
}

.score {
  font-size: 24px;
  margin-bottom: 20px;
  color: #fff;
}

.board {
  width: calc(34px * 10);
  height: calc(34px * 20);
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
}

.row {
  display: flex;
  flex-direction: row;
}

.cell {
  width: 32px;
  height: 32px;
  border: 1px solid #333;
}

.cell.active {
  background: #2196f3 !important;
  border-color: #1565c0 !important;
}

.cell.filled {
  background: #ff9800;
  border-color: #ef6c00;
}

.game-over {
  position: absolute;
  font-size: 48px;
  color: #ff0000;
  text-shadow: 2px 2px #000;
  margin-top: 100px;
}
</style>