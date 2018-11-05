<template>
  <div class="board-container">
    <div>
      Select your boggle board size:
      <div class="top-controls">
        <v-btn :class="getBtnClass(4)" :color="getBtnColor(4)" v-on:click="setBoardSize(4)">
          4 x 4
        </v-btn>
        <v-btn :class="getBtnClass(5)" :color="getBtnColor(5)" v-on:click="setBoardSize(5)">
          5 x 5
        </v-btn>
        <v-btn :class="getBtnClass(6)" :color="getBtnColor(6)" v-on:click="setBoardSize(6)">
          6 x 6
        </v-btn>
      </div>
    </div>
    <v-form v-model="valid" ref="form" class="board">
      <div v-for="y in size" :key="y" class='board-row'>
        <v-text-field
          class="text-field"
          v-model="board[y - 1][x - 1]"
          v-for="x in size"
          placeholder="x"
          :key="x"
          :maxlength="1"
          :rules="[required, letterOnly]"
          box
          single-line
          hide-details/>
      </div>
    </v-form>
    <div class="msg-container">
      <div class='error white--text' v-show='!isFormValid()'>
        Please enter only one letter for each cube
      </div>
    </div>
    <div class="bottom-controls">
      <v-btn color="success" v-on:click="getResults()"
        :disabled="!isFormValid() || !dictionaryReady">Solve</v-btn>
      <v-btn color="info" v-on:click="reset()">Reset</v-btn>
      <v-btn color="info" v-on:click="random()" :disabled="!dictionaryReady">Random</v-btn>
    </div>

    <div v-if="hasResults()" class="results">
      <div v-for="col in resultColumns" :key="col" class="result-subset">
        <div v-for="index in resultSize" :key="index" v-if="isValidResultIndex(index, col)">
          {{getResultKey(index, col)}}: ({{results[getResultKey(index, col)]}})
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Util from '@/util/Trie';
import axios from 'axios';

export default {
  name: 'Board',
  data() {
    return {
      size: 4,
      board: this.getEmptyBoard(4),
      dictionary: null,
      results: {},
      enabled: false,
      dictionaryReady: false,
      valid: false,
      letters: [],
      resultKeys: [],
      resultSize: 0,
      resultColumns: 3,
    };
  },
  created() {
    axios
      .get('https://raw.githubusercontent.com/jmlewis/valett/master/scrabble/sowpods.txt')
      .then(response => this.populateDictionary(response.data.split('\n')));
  },
  methods: {
    getResultIndex(index, col) {
      return index - 1 + ((col - 1) * this.resultSize);
    },
    getResultKey(index, col) {
      return this.resultKeys[this.getResultIndex(index, col)];
    },
    isValidResultIndex(index, col) {
      return this.getResultIndex(index, col) < this.resultKeys.length;
    },
    getEmptyBoard(size) {
      return Array(size).fill(null).map(() => Array(size).fill(null).map(() => ('')));
    },
    random() {
      this.board = this.getEmptyBoard(this.size);

      this.board.forEach((yVal, y) => {
        yVal.forEach((xVal, x) => {
          const randomIndex = Math.floor(Math.random() * this.letters.length);
          this.board[y][x] = this.letters[randomIndex];
        });
      });
    },
    reset() {
      this.$refs.form.reset();
      this.results = {};
      this.resultKeys = [];
      this.resultSize = 0;
    },
    isFormValid() {
      return this.valid;
    },
    hasResults() {
      return Object.keys(this.results).length;
    },
    required(value) {
      return !!value || 'Required';
    },
    letterOnly(value) {
      return /^[a-zA-Z]+$/.test(value) || 'Letters only';
    },
    getBtnColor(btnSize) {
      if (this.size === btnSize) {
        return 'success';
      }
      return 'blue-grey';
    },
    getBtnClass(btnSize) {
      if (this.size !== btnSize) {
        return 'white--text';
      }
      return '';
    },
    getResults() {
      this.results = {};

      this.board.forEach((yVal, y) => {
        yVal.forEach((xVal, x) => {
          this.board[y][x] = xVal.toUpperCase();
        });
      });

      this.calculateResults();

      this.resultKeys = Object.keys(this.results).sort();
      const rows = this.resultKeys.length / this.resultColumns;
      this.resultSize = this.resultKeys.length % this.resultColumns === 0 ? rows : Math.ceil(rows);
    },
    populateDictionary(data) {
      this.dictionary = new Util.Trie();

      data.forEach((word) => {
        const trimmedWord = word.trim();
        if (trimmedWord.length > 0) {
          this.dictionary.add(trimmedWord);
        }
      });

      const allWords = this.dictionary.getWords();
      const allLetters = {};

      allWords.forEach((word) => {
        if (!allLetters[word.charAt(0)]) {
          allLetters[word.charAt(0)] = 1;
          this.letters.push(word.charAt(0));
        }
      });

      this.dictionaryReady = true;
    },
    setBoardSize(newSize) {
      this.reset();
      this.board = this.getEmptyBoard(newSize);
      this.size = newSize;
    },
    calculateResults() {
      this.board.forEach((yVal, y) => {
        yVal.forEach((xVal, x) => {
          const visited = { [`${y}${x}`]: true };
          this.getAdjacentLetters(xVal, y, x, visited);
        });
      });
    },
    getPosition(pos) {
      return pos > -1 && pos < this.size ? pos : null;
    },
    getAdjacentLetters(word, y, x, visited) {
      const left = this.getPosition(x - 1);
      const right = this.getPosition(x + 1);
      const up = this.getPosition(y - 1);
      const down = this.getPosition(y + 1);

      if (word && word.length > 2 && this.dictionary.contains(word)) {
        this.results[word] = word.length - 2;
      }

      if (left !== null && !visited[`${y}${left}`]) {
        this.getNextWord(word, y, left, visited);
      }
      if (right !== null && !visited[`${y}${right}`]) {
        this.getNextWord(word, y, right, visited);
      }
      if (up !== null && !visited[`${up}${x}`]) {
        this.getNextWord(word, up, x, visited);
      }
      if (down !== null && !visited[`${down}${x}`]) {
        this.getNextWord(word, down, x, visited);
      }
      if (up !== null && left !== null && !visited[`${up}${left}`]) {
        this.getNextWord(word, up, left, visited);
      }
      if (up !== null && right !== null && !visited[`${up}${right}`]) {
        this.getNextWord(word, up, right, visited);
      }
      if (down !== null && left !== null && !visited[`${down}${left}`]) {
        this.getNextWord(word, down, left, visited);
      }
      if (down !== null && right !== null && !visited[`${down}${right}`]) {
        this.getNextWord(word, down, right, visited);
      }
    },
    getNextWord(word, y, x, visited) {
      const nextWord = `${word}${this.board[y][x]}`;
      if (this.dictionary.search(nextWord)) {
        const nextVisited = { ...visited, [`${y}${x}`]: true };
        this.getAdjacentLetters(nextWord, y, x, nextVisited);
      }
    },
  },
};
</script>

<style scoped lang="scss">
.board-container {
  margin-top: 20px;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.msg-container {
  height: 25px;

  .error {
    margin-top: 10px;
    padding: 0 10px;
  }
}

.results {
  display: flex;
  margin-bottom: 10px;

  .result-subset {
    margin: 0 10px;
    text-align: left;
  }
}

.top-controls, .bottom-controls, .results {
  margin-top: 10px;
}

.board {
  display: flex;
  flex-direction: column;
  margin-top: 20px;

  .board-row {
    display: flex;
    justify-content: center;

    .text-field {
      max-height: 60px;
      max-width: 60px;
      margin: 5px;
    }
  }
}
</style>

<style lang="scss">
input {
  text-align: center;
  font-size: 1.5em;
  text-transform: capitalize;
}
.v-text-field--outline .v-input__slot {
  min-height: 60px;
  min-width: 60px;
}
</style>
