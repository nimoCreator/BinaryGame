<template>
  <div id="BinaryGame">
    <div class="flexCenter" :class="inputModeEnum === 'tools' ? `cursor-${selectedTool}` : ''">
      <div id="playAreaGrid" :style="{'--playAreaSize': playArea.length}">
        <template v-for="(row, i) in playArea" :key="i">
          <div v-for="(cell, j) in row" :key="j">
            <span 
              @click="leftClickCell(i, j)" 
              @contextmenu.prevent="rightClickCell(i, j)" 
              @click.middle.exact="middleClickCell(i, j)"
              :class = "{
                'disabled': playArea[i][j].disabled,
                'empty': playArea[i][j].value === null,
                'zero': playArea[i][j].value === 0,
                'one': playArea[i][j].value === 1,
              }"
              type="text" >
              {{  playArea[i][j].value }}
            </span>
          </div>
          <!-- <span 
            @click="leftClickCell(i)" 
            @contextmenu.prevent="rightClickCell(i)" 
            @click.middle.exact="middleClickCell(i)"
            :class = "{
              'disabled': playArea[i%6][(i - i%6)/6].disabled,
              'empty': playArea[i%6][(i - i%6)/6].value === null,
              'zero': playArea[i%6][(i - i%6)/6].value === 0,
              'one': playArea[i%6][(i - i%6)/6].value === 1,
            }"
            type="text" >
            {{  playArea[i%6][(i - i%6)/6].value }}
          </span> -->
          
        </template>
        <template v-if="showErrors">
          <div 
            v-for="e in errors" 
            :key="e" 
            class="error" 
            :style="{
              '--xstart': e.x, 
              '--ystart': e.y,
              '--xend': e.xs,
              '--yend': e.ys,}
              ">
          </div>
        </template>
      </div>
    </div>
    <div id="tools">
      <input type="checkbox" id="showErrors" v-model="showErrors" />
      <label for="showErrors">Show Errors</label>
      <template v-if="showErrors">
        <div v-for="e in errors" :key="e" class="error">
          {{ e.message }}
        </div>
      </template>
    </div>

    <div id="buttons">
      <h2>Tools</h2>
      <div id="inputMode">
        <div class="flexRowRight">
          <input type="checkbox" id="inputModeInput" v-model="inputMode"/>
          <label for="inputModeInput">
            Selected input mode: {{ inputModeEnum }}
          </label>
        </div>
        <template v-if="inputModeEnum == `tools`">

          <div class="flexRowRight">
            <input type="radio" id="pen1" name="tool" value="pen1" v-model="selectedTool"/>
            <label for="pen1">1 Pen</label>
          </div>

          <div class="flexRowRight">
            <input type="radio" id="pen0" name="tool" value="pen0" v-model="selectedTool"/>
            <label for="pen0">0 Pen</label>
          </div>

          <div class="flexRowRight">
            <input type="radio" id="eraser" name="tool" value="eraser" v-model="selectedTool"/>
            <label for="eraser">Eraser</label>
          </div>
        </template>
      </div>

      <hr style="width: 100%;">
      <button @click="fillAreaWithRandom">random Fill</button>
      <div>
        <input type="range" id="fillness" v-model="fillness" min="0" max="100" step="1"/>
        <label for="fillness">{{ fillness/100 }}% Fillness</label>
      </div>
      <div>
        <input type="range" id="playAreaSize" v-model="playAreaSize" min="2" max="12" step="2"/>
        <label for="playAreaSize">{{ playAreaSize }} Play Area Size</label>
      </div>
      <button @click="solve">Solve</button>
      <span>valid: {{ valid }}</span>
      <span>solutions: {{ solutionsCount }}</span>
    </div>
    <div id="tooltip">
      <h2>How to Play</h2>
      <template v-if="inputModeEnum==`mouseClicks`">
        <p>Left Click on empty or 0 for 1</p>
        <p>Right Click on empty or 1 for 0</p>
        <p>Left Click on 1 for empty</p>
        <p>Right Click on 0 for empty</p>
        <p>Middle Click for empty</p>
      </template>
      <template v-else-if="inputModeEnum==`tools`">
        <p>Use the 1 pen to mark 1</p>
        <p>Use the 0 pen to mark 0</p>
        <p>Use the eraser to mark empty</p>
      </template>
    </div>
    <div>
      <h2>Rules</h2>
      <p>Fill the grid with 1s and 0s</p>
      <p>No more than half of fields in any row and column can be filled with the same digit</p>
      <p>No more than two 1s or 0s in a row or column</p>
    </div>
  </div>
  <div 
    id="customCursor" 
    v-if="inputModeEnum === 'tools'"
    :style="{ 
      left: mouseX + 'px', 
      top: mouseY + 'px' 
    }"
  >
    <span v-if="selectedTool === 'pen1'">1</span>
    <span v-if="selectedTool === 'pen0'">0</span>
    <span v-if="selectedTool === 'eraser'">🧽</span>
  </div>
</template>

<script>
export default {
  data() {
    return {
      playArea: [],
      possibleSolution: [],
      playAreaSize: 4, // has to be even!
      valid: "?",
      errors: [],

      showErrors: true,
      fillness: 20,
      solutionsCount: 0,

      inputModeEnum: `tools`,
      inputMode: false,
      selectedTool: null,

      mouseX: 0,
      mouseY: 0,

    }
  },
  mounted() {
    this.fillAreaWithRandom();
    window.addEventListener("mousemove", this.updateMousePosition);
    window.addEventListener("keydown", this.handleKeyPress);
  },
  beforeUnmount() {
    window.removeEventListener("mousemove", this.updateMousePosition);
    window.removeEventListener("keydown", this.handleKeyPress);
  },
  computed: {
    playAreaFlat() {
      return this.playArea.flat();
    },
  },
  methods: {
    handleKeyPress(event) {
      if (this.inputModeEnum !== 'tools') return;

      switch (event.key.toLowerCase()) {
        case '1':
          this.selectedTool = 'pen1';
          break;
        case '0':
          this.selectedTool = 'pen0';
          break;
        case 'e':
        case 'delete':
        case 'backspace':
          this.selectedTool = 'eraser';
          break;
      }
    },
    updateMousePosition(e) {
      this.mouseX = e.clientX + 10; // slight offset so it doesn’t cover the real pointer
      this.mouseY = e.clientY + 10;
    },
    leftClickCell(i) {
      this.leftClickCell(Math.floor(i / this.playAreaSize), i % this.playAreaSize);
    },
    leftClickCell(x, y)
    {
      if(this.inputModeEnum == 'mouseClicks')
      {
        if (this.playArea[x][y].value === null || this.playArea[x][y].value === 0) 
        {
          this.playArea[x][y].value = 1;
        } 
        else if (this.playArea[x][y] === 1) 
        {
          this.playArea[x][y].value = null;
        }
      }
      else
      {
        console.log("tool click")
        switch(this.selectedTool) {
          case 'pen1':
            this.playArea[x][y].value = 1;
            break;
          case 'pen0':
            this.playArea[x][y].value = 0;
            break;
          case 'eraser':
            this.playArea[x][y].value = null;
            break;
          default:
            break;
        }
      }
      this.verifySolvability();
    },
    rightClickCell(i) {
      if(this.inputModeEnum == 'tools') return;

      this.rightClickCell(Math.floor(i / this.playAreaSize), i % this.playAreaSize);
    },
    rightClickCell(x, y) {
      if(this.inputModeEnum == 'tools') return;

      if (this.playArea[x][y].value === null || this.playArea[x][y].value === 1) {
        this.playArea[x][y].value = 0;
      } else if (this.playArea[x][y] === 0) {
        this.playArea[x][y].value = null;
      }
      this.verifySolvability();
    },
    middleClickCell(i) {
      if(this.inputModeEnum == 'tools') return;

      this.middleClickCell(Math.floor(i / this.playAreaSize), i % this.playAreaSize);
    },
    middleClickCell(x, y) {
      if(this.inputModeEnum == 'tools') return;

      this.playArea[x][y].value = null;
      this.verifySolvability();
    },



    fillAreaWithRandom() {
      this.playArea = []
      for(let i = 0; i < this.playAreaSize; i++)
      {
        this.playArea.push([]);
        for(let j = 0; j < this.playAreaSize; j++)
        {
          this.playArea[i].push({x: i, y: j, value: null, disabled: false});
          if(Math.random() < (this.fillness/100)) {
            if (Math.random() < 0.5) 
              this.playArea[i][j] = {x: i, y: j, value: 1, disabled: true};
            else 
              this.playArea[i][j] = {x: i, y: j, value: 0, disabled: true};
          } 
          else {
            this.playArea[i][j] = {x: i, y: j, value: null, disabled: false};;
          }
        }
      }
      if( !this.checkIfInstanceValid(this.playArea) || !this.recursiveCheckStart()) 
      {
        this.fillAreaWithRandom();
        return;
      }
      this.verifySolvability();
    },
    solve() {
      this.playArea = JSON.parse(JSON.stringify(this.possibleSolution));
      this.verifySolvability();
    },
    verifySolvability() {
      this.valid = "?";
      this.solutionsCount = 0;

      if(this.checkIfValid() ) {

        this.valid = "yes";

        if(this.isFull()) {
          this.valid = "🏆 FINISHED!";
          this.solutionsCount = 1;
          alert("You finished the game!");
          return;
        }

        if(!this.recursiveCheckStart())
        {
          this.valid = "IMPOSSIBLE!";
        }
      } 
      else 
      {
        this.valid = "NO!";
        return
      }
      
      
    },

    isFull() {
      for(let i = 0; i < this.playAreaSize; i++)
      {
        for(let j = 0; j < this.playAreaSize; j++)
        {
          if(this.playArea[i][j].value === null) 
          {
              return false;
          }
        }
      }
      return true;
    },

    checkIfValid() {
      let isValid = true;
      this.errors = [];
      // not more than 3 1s or 0s in a row or column
      for(let i = 0; i < this.playAreaSize; i++)
      {
        let count1r = 0;
        let count0r = 0;
        let count1c = 0;
        let count0c = 0;
        for(let j = 0; j < this.playAreaSize; j++)
        {
          if(this.playArea[i][j].value === 1) count1r++;
          if(this.playArea[i][j].value === 0) count0r++;
          if(this.playArea[j][i].value === 1) count1c++;
          if(this.playArea[j][i].value === 0) count0c++;
        }
        if(count1r > this.playAreaSize/2 || count0r > this.playAreaSize/2) 
        {
          isValid = false;
          if (!this.showErrors) return false;
          this.errors.push({x: 0, y: i, xs: this.playAreaSize, ys: 1, message: `too many repeated digits in row ${i}`});
        } 
        if (count1c > this.playAreaSize/2 || count0c > this.playAreaSize/2)
        {
          isValid = false;
          if (!this.showErrors) return false;
          this.errors.push({x: i, y: 0, xs: 1, ys: this.playAreaSize, message: `too many repeated digits in column ${i}`});
        }
      }

      // no more than two 1s or 0s in a row of a row or column
      for(let i = 0; i < this.playAreaSize; i++)
      {
        for(let j = 0; j < this.playAreaSize-2; j++)
        {
          if(this.playArea[i][j].value === 1 && this.playArea[i][j+1].value === 1 && this.playArea[i][j+2].value === 1) 
          {
            isValid = false;
            if (!this.showErrors) return false;
            this.errors.push({x: j, y: i, xs: 3, ys: 1, message: `there are three 1s in a row in the row ${i}`});
          }
          if(this.playArea[i][j].value === 0 && this.playArea[i][j+1].value === 0 && this.playArea[i][j+2].value === 0) 
          {
            isValid = false;
            if (!this.showErrors) return false;
            this.errors.push({x: j, y: i, xs: 3, ys: 1, message: `there are three 0s in a row in the row ${i}`});
          }
          if(this.playArea[j][i].value === 1 && this.playArea[j+1][i].value === 1 && this.playArea[j+2][i].value === 1) 
          {
            isValid = false;
            if (!this.showErrors) return false;
            this.errors.push({x: i, y: j, xs: 1, ys: 3, message: `there are three 1s in a row in the column ${i}`});
          }
          if(this.playArea[j][i].value === 0 && this.playArea[j+1][i].value === 0 && this.playArea[j+2][i].value === 0)
          {
            isValid = false;
            if (!this.showErrors) return false;
            this.errors.push({x: i, y: j, xs: 1, ys: 3, message: `there are three 0s in a row in the column ${i}`});
          }
        }
      }

      return isValid;
    },

    checkIfInstanceValid(copy) {
      for(let i = 0; i < copy.length; i++)
      {
        let count1r = 0;
        let count0r = 0;
        let count1c = 0;
        let count0c = 0;
        for(let j = 0; j < copy.length; j++)
        {
          if(copy[i][j].value === 1) count1r++;
          if(copy[i][j].value === 0) count0r++;
          if(copy[j][i].value === 1) count1c++;
          if(copy[j][i].value === 0) count0c++;
        }
        if(count1r > copy.length/2 || count0r > copy.length/2) 
        {
          return false;
        } 
        if (count1c > copy.length/2 || count0c > copy.length/2)
        {
          return false;
        }
      }

      // no more than two 1s or 0s in a row of a row or column
      for(let i = 0; i < copy.length; i++)
      {
        for(let j = 0; j < copy.length-2; j++)
        {
          if(copy[i][j].value === 1 && copy[i][j+1].value === 1 && copy[i][j+2].value === 1) 
          {
            return false;
          }
          if(copy[i][j].value === 0 && copy[i][j+1].value === 0 && copy[i][j+2].value === 0) 
          {
            return false;
          }
          if(copy[j][i].value === 1 && copy[j+1][i].value === 1 && copy[j+2][i].value === 1) 
          {
            return false;
          }
          if(copy[j][i].value === 0 && copy[j+1][i].value === 0 && copy[j+2][i].value === 0)
          {
            return false;
          }
        }
      }

      return true;
    },

    recursiveCheckStart()
    {
      this.possibleSolution = JSON.parse(JSON.stringify(this.playArea));
      this.solutionsCount = 0;
      let copy = JSON.parse(JSON.stringify(this.playArea));
      this.recursiveCheck(copy, 0, 0);
      return this.solutionsCount == "many" || this.solutionsCount > 0;
    },
    recursiveCheck(copy, i, j) {
      // console.log(`checking ${i}, ${j}`);
      // for(let k = 0; k < copy.length; k++) {
      //   console.log(copy[k].map(c => c.value).join(" "));
      // }

      if(this.solutionsCount == "many" || this.solutionsCount > 256) // max recursive checks
      {
        this.solutionsCount = "many";
        return;
      }

      if (i >= copy.length) {
        this.solutionsCount++;
        this.possibleSolution = JSON.parse(JSON.stringify(copy));
        //console.log(`solution ${this.solutionsCount}`);
        //this.consoleLogPlayArea(copy);
        return;
      }
      if (j >= copy.length) {
        this.recursiveCheck(copy, i + 1, 0);
        return;
      }
      if (copy[i][j].value !== null) {
        this.recursiveCheck(copy, i, j + 1);
        return;
      }

      copy[i][j].value = 0;
      if (this.checkIfInstanceValid(copy)) {
        this.recursiveCheck(copy, i, j + 1);
      }

      copy[i][j].value = 1;
      if (this.checkIfInstanceValid(copy)) {
        this.recursiveCheck(copy, i, j + 1);
      }

      copy[i][j].value = null;
    },
    consoleLogPlayArea() {
      console.log("playArea:");
      for(let i = 0; i < this.playArea.length; i++)
      {
        console.log(this.playArea[i].map(c => c.value).join(" "));
      }
    },
    consoleLogPlayArea(copy)
    {
      console.log("playArea:");
      for(let i = 0; i < copy.length; i++)
      {
        console.log(copy[i].map(c => c.value).join(" "));
      }
    },
  },
  watch: {
    inputMode: function(newValue, oldValue) {
      //console.log(`inputMode changed from ${oldValue} to ${newValue}`);
      if (newValue) {
        this.inputModeEnum = `mouseClicks`;
      } else {
        this.inputModeEnum = `tools`;
      }
      //console.log(this.inputModeEnum)
    },
  }
}
</script>

<style>

  body {
    font-family: 'Avenir', Helvetica, Arial, sans-serif;
    background-color: #101214;
    margin: 0;

    padding: 1rem;
    color: #fff;
    display: flex;
    justify-content: center;
    align-items: flex-start;
  }

  * {
    box-sizing: border-box;
  }

  #customCursor {
    position: fixed;
    pointer-events: none;

    z-index: 9999;

    font-size: 1.5rem;

    width: 2rem;
    height: 2rem;
    border-radius: 0 50% 50% 50%;
    color: #00aaff;
    background-color: #fff;

    display: flex;
    justify-content: center;
    align-items: center;

    font-weight: bold;
    user-select: none;

  }


  .cursor-pen1 {
    cursor: url('/cursors/pen1.cur'), pointer;
  }

  .cursor-pen0 {
    cursor: url('/cursors/pen0.cur'), pointer;
  }

  .cursor-eraser {
    cursor: url('/cursors/eraser.cur'), pointer;
  }

  #BinaryGame {
    display: flex;
    flex-direction: column;
    justify-content: flex-start;
    align-items: stretch;
    width: 100%;
    height: 100%;
    gap: 0.25rem;
  }
  #BinaryGame > * {
    background-color: #202224;
    padding: 0.25rem;
    border-radius: 0.25rem;
  }
  #playAreaGrid {
    --cellSize: 5rem;
    --cellGap: 0.25rem;
    --padding: 0.25rem;
    --xstart: 0;
    --ystart: 0;
    --xend: 0;
    --yend: 0;

    position: relative;

    display: grid;
    justify-content: center;
    align-content: center;
    align-items: center;
    justify-items: center;

    padding: var(--padding);

    grid-template-columns: repeat(var(--playAreaSize), 1fr);
    grid-template-rows: repeat(var(--playAreaSize), 1fr);

    gap: 0.25rem;

    width: calc( var(--cellSize) * var(--playAreaSize) + var(--cellGap) * (var(--playAreaSize) - 1) + var(--padding) * 2);
    height: calc( var(--cellSize) * var(--playAreaSize) + var(--cellGap) * (var(--playAreaSize) - 1) + var(--padding) * 2);
  }
  #playAreaGrid span {

    width: 5rem;
    height: 5rem;

    border-radius: 0.25rem;

    background-color: #eee;
    color: #111;

    text-align: center;
    display: flex;
    justify-content: center;
    align-items: center;

    font-size: 2rem;

    user-select: none;
    cursor: pointer;
  }
  #playAreaGrid span.disabled {
    background-color: #aaa;
    color: #444;
    pointer-events: none;
    cursor: not-allowed;
  }
  #playAreaGrid span:hover {
    outline: #00aaff 0.125rem solid;
  }
  #playAreaGrid div.error {
    position: absolute;
    outline: #ff000088 0.25rem solid;
    background-color: #ff000022;
    border-radius: 0.25rem;
    width: calc(var(--cellSize) * var(--xend) + var(--cellGap) * (var(--xend) - 1));
    height: calc(var(--cellSize) * var(--yend) + var(--cellGap) * (var(--yend) - 1));
    top: calc(var(--cellSize) * var(--ystart) + var(--cellGap) * var(--ystart) + var(--padding));
    left: calc(var(--cellSize) * var(--xstart) + var(--cellGap) * var(--xstart) + var(--padding));
    z-index: 1;
    pointer-events: none;
  }

  #buttons {
    display: flex;
    flex-direction: column;
    justify-content: flex-start;
    align-items: flex-start;
    gap: 1rem;
  }

  .flexCenter {
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 1rem !important;
  }

  #inputMode {
    display: flex;
    flex-direction: column;
    justify-content: flex-start;
    align-items: stretch;
    gap: 0.5rem;
    width: 100%;
  }
  .flexRowRight {
    display: flex;
    justify-content: flex-start;
    align-items: center;
    gap: 0.5rem;
    width: 100%;
  }
  .flexRowRight label {
    flex-grow: 1;
    text-align: left;
  }

  </style>

