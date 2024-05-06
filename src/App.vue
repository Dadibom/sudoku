<template>
  <div class="sudoku-grid">
    <div v-for="i in 9 * 9" :key="i" class="cell" :class="getClass(i)" @click="select(i)">
      <div class="cell-idx">{{ i - 1 }}</div>
      <span v-if="grid[i-1]">{{ grid[i - 1] }}</span>
      <div v-else class="marks">
        <div v-for="j in 9" :key="j" class="mark">
          <span v-if="marks[i-1][j-1]">{{ j }}</span>
        </div>
      </div>
    </div>
  </div>
  <button @click="solveIteration">Solve Iteration</button>
  <div v-if="isValid" style="color: green">Sudoku is valid</div>
  <div v-else style="color: red">Sudoku is invalid</div>
  <div>Unfilled cells: {{ unfilledCells }}</div>
  <table>
    <thead>
      <tr>
        <th>Technique</th>
        <th>Enabled</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>Fill Single Possible Location In Row/Col/Box</td>
        <td><input type="checkbox" v-model="tech.single_possibility_in_section"></td>
      </tr>
      <tr>
        <td>Isolate Pairs</td>
        <td><input type="checkbox" v-model="tech.isolate_pairs"></td>
      </tr>
      <tr>
        <td>Forced Column Or Row In Box</td>
        <td><input type="checkbox" v-model="tech.forced_column_or_row_in_box"></td>
      </tr>
      <tr>
        <td>Isolate Triads</td>
        <td><input type="checkbox" v-model="tech.isolate_triads"></td>
      </tr>
    </tbody>
  </table>
</template>

<script>
//const getEmptyGrid = () => Array.from({ length: 9 * 9 }, () => null);
function getMarkGrid() {
  return Array.from({ length: 9 * 9 }, () => Array.from({ length: 9 }, () => true));
}

function eliminateMarks(grid, marks, getIndices) {
  for (let index = 0; index < 9; index++) {
    const values = new Set();
    const indices = getIndices(index);
    // Collect values present in the grid at the given indices
    indices.forEach(i => {
      const value = grid[i];
      if (value) values.add(value);
    });
    // Eliminate marks based on the collected values
    indices.forEach(i => {
      values.forEach(v => {
        marks[i][v - 1] = false;
      });
    });
  }
}

function getRowIndices(row) {
  const indices = [];
  for (let col = 0; col < 9; col++) {
    indices.push(row * 9 + col);
  }
  return indices;
}

function getColumnIndices(col) {
  const indices = [];
  for (let row = 0; row < 9; row++) {
    indices.push(row * 9 + col);
  }
  return indices;
}

function getBoxIndices(box) {
  const indices = [];
  for (let row = 0; row < 3; row++) {
    for (let col = 0; col < 3; col++) {
      const i = Math.floor(box / 3) * 27 + (box % 3) * 3 + row * 9 + col;
      indices.push(i);
    }
  }
  return indices;
}

function arraysEq (a1, a2) {
  if (a1.length !== a2.length) {
    return false;
  }
  for (let i = 0; i < a1.length; i++) {
    if (a1[i] !== a2[i]) {
      return false;
    }
  }
  return true;
}

function isolatePairs (grid, marks, getIndices) {
  for (let row = 0; row < 9; row++) {
    const indices = getIndices(row);
    
    // find pairs of two cells with the same two marks
    for (let i1 = 0; i1 < indices.length; i1++) {
      // if cell is already solved, continue
      if (grid[indices[i1]]) {
        continue;
      }
      const mark1Values = marks[indices[i1]].map((m, i) => m ? i + 1 : 0).filter(v => v);

      // @TODO we could check triplets too
      if (mark1Values.length !== 2) {
        continue;
      }

      for (let i2 = i1+1; i2 < indices.length; i2++) {
        // if cell is already solved, continue
        if (grid[indices[i2]]) {
          continue;
        }
        const mark2Values = marks[indices[i2]].map((m, i) => m ? i + 1 : 0).filter(v => v);

        // if values dont match, continue
        if (!arraysEq(mark1Values, mark2Values)) {
          continue;
        }
        //console.log('Found pair on row:', row, i1, i2, mark1Values, mark2Values);

        for (let i3 = 0; i3 < indices.length; i3++) {
          if (i3 === i1 || i3 === i2) continue;
          const mark = marks[indices[i3]];
          mark[mark1Values[0] - 1] = false;
          mark[mark1Values[1] - 1] = false;
        }
      }
    }
  }
}

function generateNumberCombinations(n, max) {
    let combinations = [];

    function backtrack(start, currentCombo) {
        if (currentCombo.length === n) {
            combinations.push(currentCombo.slice());
            return;
        }

        for (let i = start; i <= max; i++) {
            currentCombo.push(i);
            backtrack(i + 1, currentCombo);
            currentCombo.pop();
        }
    }

    backtrack(1, []);

    return combinations;
}

function isolateExclusiveGroups (grid, marks, getIndices, n = 3) {
  const combinations = generateNumberCombinations(n, 9);

  for (let row = 0; row < 9; row++) {
    const indices = getIndices(row).filter(i => !grid[i]);
    
    // find n cells that do not contain any other marks than the same n marks
    // for example 1,2,3 in three cells
    // or 1,2 2,3 1,3 in three cells
    // if n cells contain only n marks, remove those marks from other cells in the section

    for (const combination of combinations) {
      let found = [];

      for (let i of indices) {
        const mark = marks[i];
        // if cell contains any other marks than the combination, continue
        if (mark.some((m, j) => m && !combination.includes(j + 1))) {
          continue;
        }
        found.push(i);
      }

      if (found.length >= n) {
        console.log('Found exclusive group:', row, combination, found);
        for (let i of indices) {
          if (!found.includes(i)) {
            const mark = marks[i];
            combination.forEach(v => mark[v - 1] = false);
          }
        }
      }
    }
  }
}

function sudokuIsValid (grid) {
  for (let row = 0; row < 9; row++) {
    const rowValues = new Set();
    const colValues = new Set();
    const boxValues = new Set();
    for (let col = 0; col < 9; col++) {
      const rowValue = grid[row * 9 + col];
      const colValue = grid[col * 9 + row];
      const boxValue = grid[Math.floor(row / 3) * 27 + Math.floor(col / 3) * 3 + (row % 3) * 9 + col % 3];
      if (rowValue) {
        if (rowValues.has(rowValue)) {
          return false;
        }
        rowValues.add(rowValue);
      }
      if (colValue) {
        if (colValues.has(colValue)) {
          return false;
        }
        colValues.add(colValue);
      }
      if (boxValue) {
        if (boxValues.has(boxValue)) {
          return false;
        }
        boxValues.add(boxValue);
      }
    }
  }
  return true;
}

function checkForcedRowOrColumnInBox (grid, marks) {
  for (let box = 0; box < 9; box++) {
    const boxIndices = getBoxIndices(box);

    for (let num = 0; num < 9; num++) {
      const countInCol = Array.from({ length: 3 }, () => 0);
      const countInRow = Array.from({ length: 3 }, () => 0);

      for (let i = 0; i < boxIndices.length; i++) {
        if (grid[boxIndices[i]]) {
          continue;
        }
        const mark = marks[boxIndices[i]];
        if (mark[num]) {
          const row = Math.floor(i / 3);
          const col = i % 3;
          countInRow[row]++;
          countInCol[col]++;
        }
      }

      // if a number appears only in one row or column in the box, set it
      const inColumns = countInCol.map((count, i) => count > 0 ? i : -1).filter(i => i !== -1);
      const inRows = countInRow.map((count, i) => count > 0 ? i : -1).filter(i => i !== -1);

      if (inColumns.length === 1) {
        const realCol = inColumns[0] + (box % 3) * 3;
        const columnIndices = getColumnIndices(realCol).filter(i => !boxIndices.includes(i));
        let changed = false;
        for (let i = 0; i < columnIndices.length; i++) {
          if (grid[columnIndices[i]]) {
            continue;
          }
          const mark = marks[columnIndices[i]];
          if (mark[num]) {
            mark[num] = false;
            changed = true;
          }
        }
        if (changed)
          console.log('Ruled out column', realCol, 'for', num, '(indices)', columnIndices);
      }

      if (inRows.length === 1) {
        const realRow = inRows[0] + Math.floor(box/3) * 3;
        const rowIndices = getRowIndices(realRow).filter(i => !boxIndices.includes(i));
        let changed = false;
        for (let i = 0; i < rowIndices.length; i++) {
          if (grid[rowIndices[i]]) {
            continue;
          }
          const mark = marks[rowIndices[i]];
          if (mark[num]) {
            mark[num] = false;
            changed = true;
          }
        }
        if (changed)
          console.log('Ruled out row', realRow, 'for', num, '(indices)', rowIndices);
      }
    }
  }
  return 0;

}

function solveSinglePossibilities (grid, marks, getIndices) {
  for (let row = 0; row < 9; row++) {
    const indices = getIndices(row);
    const valueCounts = Array.from({ length: 9 }, () => 0);
    
    // for each mark, count how many times it appears in the row
    for (let i = 0; i < indices.length; i++) {
      if (grid[indices[i]]) {
        continue;
      }
      const mark = marks[indices[i]];
      for (let j = 0; j < 9; j++) {
        if (mark[j]) {
          valueCounts[j]++;
        }
      }
    }

    // if a value appears only once, set it in the grid
    for (let i = 0; i < indices.length; i++) {
      if (grid[indices[i]]) {
        continue;
      }
      const mark = marks[indices[i]];
      for (let j = 0; j < 9; j++) {
        if (mark[j] && valueCounts[j] === 1) {
          grid[indices[i]] = j + 1;
          return 1;
        }
      }
    }
  }
  return 0;
}

function solveSingleMarks (grid, marks) {
  let solved = 0;
  for (let i = 0; i < 9 * 9; i++) {
    if (grid[i]) {
      continue;
    }
    const mark = marks[i];
    const markCount = mark.reduce((acc, m) => acc + (m ? 1 : 0), 0);
    if (markCount === 1) {
      const value = mark.findIndex(m => m) + 1;
      grid[i] = value;
      solved++;
    }
  }
  return solved;
}

const testGrid = [
  0, 9, 1,  0, 4, 0,  7, 5, 0,
  0, 0, 0,  0, 5, 0,  0, 0, 3,
  5, 0, 0,  0, 0, 2,  0, 6, 1,

  0, 0, 0,  0, 9, 6,  1, 0, 8,
  1, 8, 0,  0, 0, 0,  0, 3, 9,
  2, 0, 9,  0, 1, 0,  0, 0, 0,

  0, 2, 0,  4, 0, 5,  0, 0, 0,
  6, 1, 8,  0, 0, 0,  2, 4, 5,
  0, 5, 0,  0, 0, 0,  0, 0, 0,
];

const testGrid2 = '500009030002010070000000000000000604040001700700008050900030010400060000050020063'.split('').map(Number);
const testGrid3 = '000003009000060400106000002080000000050090030004000850000538000420900000700400000'.split('').map(Number);
const testGrid4 = '304006000060000000000070250000209700000685000020000090901000400008040015000000003'.split('').map(Number);
const testGrid5 = '000000000001030600400500900002700000009400100003005209800600500000000027100007060'.split('').map(Number);
const testGrid6 = '040005600906003010070090000000500070004000000603020500000001008205030700090000000'.split('').map(Number);
const testGrid7 = '370001000000902000000006510002000640190000000040009020004000150000800000600500700'.split('').map(Number);
const testGrid8 = '860000013090000000200109600000001400037900000002007508020710005000003726004500000'.split('').map(Number);

export default {
  name: "App",
  data() {
    return {
      grid: testGrid8,
      marks: getMarkGrid(),
      selectedIndex: null,
      tech: {
        single_possibility_in_section: true,
        isolate_pairs: true,
        forced_column_or_row_in_box: true,
        isolate_triads: true,
      },
    };
  },
  async mounted () {
    eliminateMarks(this.grid, this.marks, getRowIndices);
    eliminateMarks(this.grid, this.marks, getColumnIndices);
    eliminateMarks(this.grid, this.marks, getBoxIndices);
    document.addEventListener('keydown', this.onKeyDown);
  },
  computed: {
    isValid () {
      return sudokuIsValid(this.grid);
    },
    unfilledCells () {
      return this.grid.filter(v => !v).length;
    },
  },
  methods: {
    select (i) {
      this.selectedIndex = i;
    },
    onKeyDown (e) {
      if (this.selectedIndex === null) {
        return;
      }
      if (e.key === 'Backspace') {
        this.grid[this.selectedIndex - 1] = null;
      } else if (e.key >= '1' && e.key <= '9') {
        this.grid[this.selectedIndex - 1] = Number(e.key);
      }
      eliminateMarks(this.grid, this.marks, getRowIndices);
      eliminateMarks(this.grid, this.marks, getColumnIndices);
      eliminateMarks(this.grid, this.marks, getBoxIndices);
    },
    solveIteration () {
      let solved = 0;
      if (this.tech.single_possibility_in_section) {
        solved += solveSinglePossibilities(this.grid, this.marks, getBoxIndices) || solveSinglePossibilities(this.grid, this.marks, getRowIndices) || solveSinglePossibilities(this.grid, this.marks, getColumnIndices);
      }
      solved += solveSingleMarks(this.grid, this.marks);

      console.log('Solved:', solved);

      eliminateMarks(this.grid, this.marks, getRowIndices);
      eliminateMarks(this.grid, this.marks, getColumnIndices);
      eliminateMarks(this.grid, this.marks, getBoxIndices);

      if (this.tech.isolate_pairs) {
        isolatePairs(this.grid, this.marks, getRowIndices);
        isolatePairs(this.grid, this.marks, getColumnIndices);
        isolatePairs(this.grid, this.marks, getBoxIndices);
      }
      if (this.tech.forced_column_or_row_in_box) {
        checkForcedRowOrColumnInBox(this.grid, this.marks);
      }

      if (this.tech.isolate_triads) {
        isolateExclusiveGroups(this.grid, this.marks, getRowIndices, 3);
        isolateExclusiveGroups(this.grid, this.marks, getColumnIndices, 3);
        isolateExclusiveGroups(this.grid, this.marks, getBoxIndices, 3);
      }

      //isolateExclusiveGroups(this.grid, this.marks, getRowIndices, 4);
      //isolateExclusiveGroups(this.grid, this.marks, getColumnIndices, 4);
      //isolateExclusiveGroups(this.grid, this.marks, getBoxIndices, 4);
    },
    getClass(i) {
      const row = Math.floor((i - 1) / 9);
      const col = (i - 1) % 9;
      const box = Math.floor(row / 3) * 3 + Math.floor(col / 3);
      let classes = `row-${row} col-${col} box-${box}`;
      if (i === this.selectedIndex) {
        classes += ' selected';
      }

      return classes;
    },
  },
};
</script>

<style>
.sudoku-grid {
  display: grid;
  grid-template-columns: repeat(9, 1fr);
  aspect-ratio: 1;
}
.cell {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 32px;
  box-sizing: border-box;
  border: 1px solid #444;
  cursor: pointer;
  user-select: none;
  position: relative;
}
.row-2,
.row-5 {
  border-bottom: 2px solid black;
}
.col-2,
.col-5 {
  border-right: 2px solid black;
}
.row-3,
.row-6 {
  border-top: 2px solid black;
}
.col-3,
.col-6 {
  border-left: 2px solid black;
}

.marks {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: repeat(3, 1fr);
  width: 100%;
  height: 100%;
}
.mark {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 16px;
  color: #aaa;
}

.selected {
  background-color: lightblue;
}

.cell-idx {
  position: absolute;
  top: 1px;
  left: 2px;
  font-size: 8px;
  color: #ccc;
}

</style>
