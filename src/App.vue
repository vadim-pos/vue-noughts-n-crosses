<template>
    <main id="noughts-n-crosses">
        <header class="game-info">
            <div class="game-info-message"
                v-if="gameStatus === 'is over'">
                {{ gameOverMessage }}
            </div>
        </header>

        <div class="flip-container">
            <div class="flip-container__flipper" :class="{active: gameStatus !== 'before start'}">

                <div class="controls">
                    <p class="controls__title">Choose AI level</p>
                    <ul class="controls__list">
                        <li class="controls__item" v-for="(value, key) in aiLevels">
                            <button class="controls__btn" 
                                @click="startGame"
                                :data-ai-coeff="value">
                                {{ key.charAt(0).toUpperCase() + key.slice(1) }}
                            </button>
                        </li>
                    </ul>
                </div>

                <div class="board">
                    <div class="board__cell"
                        v-for="(cell, i) in gameState.board" 
                        @click="makeMove(i)"
                        :ref="'cell' + i">
                    </div>
                </div>

            </div>
        </div>

        <footer class="footer">
            <button class="restart" @click="restartGame">Restart</button>
        </footer>
    </main>
</template>

<script>
export default {
    data: () => ({
        gameStatus: 'before start',

        /* For each move, the state can be represented as current board position
        and current turn (X for Human and O for Computer) */
        gameState: {

            board: [
                'empty', 'empty', 'empty',
                'empty', 'empty', 'empty',
                'empty', 'empty', 'empty'
            ],

            turn: 'X'
        },
        
        /* Win combination of indexes */
        winIndexCombinations: [
            // rows
            [0, 1, 2], [3, 4, 5], [6, 7, 8],
            // columns
            [0, 3, 6], [1, 4, 7], [2, 5, 8],
            // diagonals
            [0, 4, 8], [2, 4, 6]

        ],
        
        /* List of possible AI levels that the player can choose */
        aiLevels: {
            newbie: 80,
            amateur: 40,
            expert: 0
        },
        
        /* Probability that AI will choose sub-optimal move istead of optimal */
        aiStupidityCoefficient: 0,
        
        /* Displayed when game is over */
        gameOverMessage: ''
    }),

    methods: {
        /* set aiStupidityCoefficient for choosen ai level, runs the game */
        startGame(e) {
            this.aiStupidityCoefficient = e.target.dataset.aiCoeff;
            this.gameStatus = 'is running';
        },

        restartGame() {
            window.location.reload();
        },

        /* returns next state which is based on current */
        getNextGameState(currentState, boardIndex) {
            let newGameState = {
                board: currentState.board.slice(), /* make a copy of current board */
                turn: currentState.turn === 'X' ? 'O' : 'X' /* switch turn */
            };
            newGameState.board[boardIndex] = currentState.turn; /* add current turrn to new board */

            return newGameState;
        },
        
        /* returns array of empty cells indexes */
        getEmptyCells(board) {
            let emptyCells = [];
            for (let i = 0; i < 9; i++) {
                if (board[i] === 'empty') {
                    emptyCells.push(i);
                }
            }
            return emptyCells;
        },
        
        /* if this is a winning situation, returns winning combination of indexes */
        isWin(board, turn) {
            let winCombination = null;

            this.winIndexCombinations.forEach(combination => {
                if (combination.every(index => board[index] === turn)) {
                    return winCombination = combination;
                }
            });

            return winCombination;
        },
        
        /* recursively computes minimax (http://neverstopbuilding.com/minimax) value for each of empty cells  */
        getMinimaxValue(gameState, depth = 0) {
            const extreme = gameState.turn === 'X' ? 'max' : 'min';
            const emptyCells = this.getEmptyCells(gameState.board);
            
            /* player receives 100 points for the win - max value, each recursion step (depth) subtracts 1 from this points */
            if (this.isWin(gameState.board, 'X')) { return 100 - depth; }
            /* computer receives -100 points for the win - min value, each recursion step (depth) adds 1 to this points */
            if (this.isWin(gameState.board, 'O')) { return -100 + depth; }
            /* draw, nobody gets any points */
            if (emptyCells.length === 0) { return 0; }
            
            /* recursively return extreme (max or min - depends on player) value for each of emptyCells */
            return Math[extreme].apply(this, emptyCells.map(index =>{
                let newGameState = this.getNextGameState(gameState, index);
                return this.getMinimaxValue(newGameState, depth + 1);
            }));
        },

        calculateMove(gameState) {
            const emptyCells = this.getEmptyCells(gameState.board);
            
            let minimaxMap = {}; /* object: key - index of empty cell, value - its minimax value */
            let minimaxKeys = []; /* array of board indexes */
            let minimaxArr = []; /* array of minimax values, sorted in ascending or descending order (depends on current turn)  */
            
            /* get minimax value for each empty cell index, add index and its minimax value to minimaxMap */
            emptyCells.forEach(index => {
                let newState = this.getNextGameState(gameState, index);
                minimaxMap[index] = this.getMinimaxValue(newState);
            });

            minimaxKeys = Object.keys(minimaxMap);
            
            if (gameState.turn === 'X') {
                minimaxArr = minimaxKeys.sort((a,b) => minimaxMap[b] - minimaxMap[a]);
            }
            if (gameState.turn === 'O') {
                minimaxArr = minimaxKeys.sort((a,b) => minimaxMap[a] - minimaxMap[b]);
            }
            
            /* take the optimal action (100 - aiStupidityCoefficient)% of the time, ohterwise take the 1st suboptimal action */
            if (this.aiStupidityCoefficient && Math.random()*100 <= this.aiStupidityCoefficient) {
                return minimaxArr.length >= 2 ? minimaxArr[1] : minimaxArr[0];
            } else {
                return minimaxArr[0];
            }
        },
        
        /* runs after clicking on the board cell */
        makeMove(index) {
            /* return if this cell is already clicked  */
            if (this.$refs['cell' + index][0].dataset.dirty) { return; }

            /* make players move */
            this.move(index);
            /* make AI move */
            this.move(this.calculateMove(this.gameState));
        },

        move(index) {
            if (this.gameStatus !== 'is running') { return; }
            
            /* update UI */
            this.$refs['cell' + index][0].innerHTML = `<span>${this.gameState.turn}</span>`;
            this.$refs['cell' + index][0].dataset.dirty = true; /* mark active cell as 'dirty' */
            this.gameState = this.getNextGameState(this.gameState, index); /* update game state */
            this.checkGameOver();
        },
        
        /* checks did the game end after last move */
        checkGameOver() {
            const isDraw = this.getEmptyCells(this.gameState.board).length === 0; /* if no empty cell - draw */
            const lastTurn = this.gameState.turn === 'X' ? 'O' : 'X'; /* last turn */
            const winCombination = this.isWin(this.gameState.board, lastTurn); /* winning combination of indexes */

            if (winCombination) {
                this.gameStatus = 'is over';
                winCombination.forEach(index => this.$refs['cell' + index][0].classList.add('strike'));
                this.gameOverMessage = (lastTurn === 'X' ? 'You' : 'Computer ') + ' won!';
                return;
            }

            if (isDraw) {
                this.gameStatus = 'is over';
                this.gameOverMessage = 'It\'s a draw!';
                return;
            }
        }
    }
}
</script>

<style lang="scss">
    @import 'scss/main.scss';
</style>
