<template>
  <div>
    <div
      v-if="!playing"
      class="content-inline">
      <h2 v-if="gameState === 'lost'">YOU LOSE</h2>
      <h2 v-else-if="gameState === 'won'">YOU WIN!!!</h2>
      <button @click="resetGame()">Play Again</button>

      height
      <input
        v-model.number="height"
        type="number">
      width
      <input
        v-model.number="width"
        type="number">
      mines
      <input
        v-model.number="mines"
        type="number">
    </div>
    <div
      :style="gridStyle"
      class="ms-grid">
      <div
        v-for="(row, i) in grid"
        :key="i"
        class="ms-row">
        <div
          v-for="(cell, j) in row"
          :key="j"
          :class="cellClass(cell)"
          class="ms-cell"
          @click="onCellClick(cell)"
          @contextmenu="onCellRightClick($event, cell)">
          &nbsp;
        </div>
      </div>
    </div>
  </div>
</template>

<script>

const buildGrid = ({ width, height, mines }) => {
  const grid = []
  const choices = []
  for (let i = 0; i < height; i++) {
    const row = []
    for (let j = 0; j < width; j++) {
      row.push({
        flagged: false,
        question: false,
        mine: false,
        hit: false,
        revealed: false
      })
      choices.push({i, j})
    }
    grid.push(row)
  }

  for (let m = 0; m < mines; m++) {
    const idx = Math.floor(Math.random() * choices.length)
    const {i, j} = choices[idx]
    choices.splice(idx, 1)
    grid[i][j].mine = true
  }

  for (let i = 0; i < height; i++) {
    for (let j = 0; j < width; j++) {
      const neighbors = []
      if (i - 1 >= 0) {
        if (j - 1 >= 0) {
          neighbors.push(grid[i - 1][j - 1])
        }
        neighbors.push(grid[i - 1][j])
        if (j + 1 < width) {
          neighbors.push(grid[i - 1][j + 1])
        }
      }
      if (i + 1 < height) {
        if (j - 1 >= 0) {
          neighbors.push(grid[i + 1][j - 1])
        }
        neighbors.push(grid[i + 1][j])
        if (j + 1 < width) {
          neighbors.push(grid[i + 1][j + 1])
        }
      }
      if (j - 1 >= 0) {
        neighbors.push(grid[i][j - 1])
      }
      if (j + 1 < width) {
        neighbors.push(grid[i][j + 1])
      }
      grid[i][j].neighbors = neighbors
    }
  }

  return grid
}

export default {
  data () {
    return {
      gameState: 'idle',
      height: 16,
      width: 30,
      mines: 40,
      grid: []
    }
  },
  computed: {
    playing () {
      return this.gameState === 'playing'
    },
    gridStyle () {
      return {
        width: this.grid[0].length * 16 + 'px'
      }
    }
  },
  created () {
    if (!this.playing) {
      this.resetGame()
    }
  },
  methods: {
    resetGame () {
      this.gameState = 'playing'
      this.grid = buildGrid({ height: this.height, width: this.width, mines: this.mines })
    },
    onCellClick (cell) {
      if (!this.playing || cell.flagged || cell.question) {
        return
      }
      if (cell.revealed) {
        if (!cell.mine && this.countAdjacentFlags(cell) === this.countAdjacentMines(cell)) {
          cell.neighbors.forEach(n => {
            if (!n.flagged && !n.question) {
              this.checkMine(n)
            }
            this.revealSafe(n)
          })
        }
      } else {
        this.checkMine(cell)
        this.revealSafe(cell)
      }
      this.checkWin()
    },
    onCellRightClick (e, cell) {
      e.preventDefault()

      if (!this.playing || cell.revealed) {
        return
      }
      if (cell.flagged) {
        cell.flagged = false
        cell.question = true
      } else if (cell.question) {
        cell.flagged = false
        cell.question = false
      } else {
        cell.flagged = true
        cell.question = false
      }
      this.checkWin()
    },
    checkMine (cell) {
      if (cell.mine) {
        cell.revealed = true
        cell.hit = true
        this.gameState = 'lost'
        this.iterateCells(c => {
          if (c.mine) {
            c.revealed = true
          }
        })
      }
    },
    revealSafe (cell) {
      if (cell.mine || cell.revealed) {
        return
      }
      cell.revealed = true
      if (this.countAdjacentMines(cell) === 0) {
        cell.neighbors.forEach(n => {
          this.revealSafe(n)
        })
      }
    },
    checkWin () {
      let allMinesFlagged = true
      let allNonMinesRevealed = true
      this.iterateCells(cell => {
        if (cell.mine && !cell.flagged) {
          allMinesFlagged = false
        }
        if (!cell.mine && !cell.revealed) {
          allNonMinesRevealed = false
        }
      })
      const win = allMinesFlagged || allNonMinesRevealed
      if (win) {
        this.iterateCells(cell => {
          cell.revealed = true
          if (cell.mine) {
            cell.flagged = true
          }
        })
        this.gameState = 'won'
      }
    },
    countAdjacentMines (cell) {
      return cell.neighbors.reduce((count, cell) => cell.mine ? count + 1 : count, 0)
    },
    countAdjacentFlags (cell) {
      return cell.neighbors.reduce((count, cell) => cell.flagged ? count + 1 : count, 0)
    },
    cellClass (cell) {
      if (!this.playing && cell.flagged && !cell.mine) {
        return 'mine-misflagged'
      }
      if (cell.revealed) {
        if (cell.mine) {
          if (cell.hit) {
            return 'mine-hit'
          } else {
            if (this.gameState === 'won') {
              return 'flag'
            } else {
              return 'mine'
            }
          }
        } else {
          return 'mines' + this.countAdjacentMines(cell)
        }
      } else {
        if (cell.flagged) {
          return 'flag'
        } else if (cell.question) {
          return 'question'
        } else {
          return 'covered'
        }
      }
    },
    iterateCells (f) {
      for (let i = 0; i < this.grid.length; i++) {
        const row = this.grid[i]
        for (let j = 0; j < row.length; j++) {
          f(row[j])
        }
      }
    }
  }
}
</script>

<style>
.ms-cell {
  display: inline-block;
  width: 16px;
  height: 16px;
  background: url("../assets/minesweeper.png") no-repeat;
  background-position: 0 -39px;

  -moz-user-select: none;
  -khtml-user-select: none;
  -webkit-user-select: none;
  -ms-user-select: none;
  user-select: none;
}

.ms-row {
  height: 16px;
}

.ms-grid {
  overflow: hidden;
  border: 3px solid;
  border-color: #7d7d7d #fff #fff #7d7d7d;
}

.content-inline > * {
  display: inline-block;
}

.covered {
  background-position: 0 -39px;
}
.mines0 {
  background-position: 0 -23px;
}
.mines1 {
  background-position: -16px -23px;
}
.mines2 {
  background-position: -32px -23px;
}
.mines3 {
  background-position: -48px -23px;
}
.mines4 {
  background-position: -64px -23px;
}
.mines5 {
  background-position: -80px -23px;
}
.mines6 {
  background-position: -96px -23px;
}
.mines7 {
  background-position: -112px -23px;
}
.mines8 {
  background-position: -128px -23px;
}
.mine-hit {
  background-position: -32px -39px;
}
.mine-misflagged {
  background-position: -48px -39px;
}
.mine {
  background-position: -64px -39px;
}
.flag {
  background-position: -16px -39px;
}
.question {
  background-position: -80px -39px;
}
</style>
