<script setup lang="ts">
import { ref, computed } from 'vue'
import { useEventListener } from '@vueuse/core'

interface CellData {
  value: string;
  id: string;
}

interface UndoRedoState {
  cells: Record<string, string>;
  selectedCell: string | null;
  selectedRange: string[];
}

interface CellRange {
  startRow: number;
  startCol: number;
  endRow: number;
  endCol: number;
}

const rows = 10
const cols = 8

// Initialize table data
const generateInitialData = () => {
  const data: Record<string, CellData> = {}
  for (let i = 0; i < rows; i++) {
    for (let j = 0; j < cols; j++) {
      const id = `${i}-${j}`
      data[id] = { value: '', id }
    }
  }
  return data
}

const cells = ref(generateInitialData())
const selectedCell = ref<string | null>(null)
const selectedRange = ref<string[]>([])
const isDragging = ref(false)
const dragStart = ref<{ row: number; col: number } | null>(null)
const undoStack = ref<UndoRedoState[]>([])
const redoStack = ref<UndoRedoState[]>([])

// Save current state for undo/redo
const saveState = () => {
  const currentState: UndoRedoState = {
    cells: Object.fromEntries(
      Object.entries(cells.value).map(([key, cell]) => [key, cell.value])
    ),
    selectedCell: selectedCell.value,
    selectedRange: selectedRange.value
  }
  undoStack.value.push(currentState)
  redoStack.value = []
}

// Get cells within a range
const getCellsInRange = (range: CellRange) => {
  const cellsInRange: string[] = []
  for (let i = range.startRow; i <= range.endRow; i++) {
    for (let j = range.startCol; j <= range.endCol; j++) {
      cellsInRange.push(`${i}-${j}`)
    }
  }
  return cellsInRange
}

// Handle cell selection
const selectCell = (id: string) => {
  selectedCell.value = id
  selectedRange.value = [id]
  // Focus the input element after a short delay to ensure the DOM is updated
  setTimeout(() => {
    const input = document.querySelector(`input[data-id="${id}"]`) as HTMLInputElement
    if (input) {
      input.focus()
    }
  }, 0)
}

// Handle mouse down for drag selection
const handleMouseDown = (row: number, col: number) => {
  isDragging.value = true
  dragStart.value = { row, col }
  selectCell(`${row}-${col}`)
}

// Handle mouse move for drag selection
const handleMouseMove = (row: number, col: number) => {
  if (!isDragging.value || !dragStart.value) return

  const range: CellRange = {
    startRow: Math.min(dragStart.value.row, row),
    startCol: Math.min(dragStart.value.col, col),
    endRow: Math.max(dragStart.value.row, row),
    endCol: Math.max(dragStart.value.col, col)
  }

  selectedRange.value = getCellsInRange(range)
}

// Handle mouse up to end drag selection
const handleMouseUp = () => {
  isDragging.value = false
}

// Handle cell value change
const updateCell = (id: string, value: string) => {
  cells.value[id].value = value
  saveState()
}

// Copy selected range to clipboard
const copySelectedRange = () => {
  if (selectedRange.value.length === 0) return

  // Sort cells by row and column
  const sortedCells = selectedRange.value.sort((a, b) => {
    const [aRow, aCol] = a.split('-').map(Number)
    const [bRow, bCol] = b.split('-').map(Number)
    return aRow === bRow ? aCol - bCol : aRow - bRow
  })

  // Get the range dimensions
  const [firstRow] = sortedCells[0].split('-').map(Number)
  let currentRow = firstRow
  let rowValues: string[] = []
  const allRowValues: string[] = []

  // Build CSV format
  sortedCells.forEach(cellId => {
    const [row] = cellId.split('-').map(Number)
    if (row !== currentRow) {
      allRowValues.push(rowValues.join('\t'))
      rowValues = []
      currentRow = row
    }
    rowValues.push(cells.value[cellId].value)
  })
  allRowValues.push(rowValues.join('\t'))

  return allRowValues.join('\n')
}

// Calculate dimensions of copied data
const getDataDimensions = (content: string) => {
  const rows = content.split('\n')
  const rowCount = rows.length
  const colCount = Math.max(...rows.map(row => row.split('\t').length))
  return { rowCount, colCount }
}

// Paste content into selected cells
const pasteContent = async (content: string) => {
  if (!selectedCell.value) return

  const [startRow, startCol] = selectedCell.value.split('-').map(Number)
  const { rowCount, colCount } = getDataDimensions(content)
  
  // Create a new range for the paste target
  const targetRange: CellRange = {
    startRow,
    startCol,
    endRow: Math.min(startRow + rowCount - 1, rows - 1),
    endCol: Math.min(startCol + colCount - 1, cols - 1)
  }

  // Update the selected range to show the paste area
  selectedRange.value = getCellsInRange(targetRange)

  const rows = content.split('\n')
  rows.forEach((row, rowIndex) => {
    const values = row.split('\t')
    values.forEach((value, colIndex) => {
      const targetRow = startRow + rowIndex
      const targetCol = startCol + colIndex
      if (targetRow < 10 && targetCol < 8) {
        const targetId = `${targetRow}-${targetCol}`
        cells.value[targetId].value = value
      }
    })
  })
  saveState()
}

// Handle keyboard navigation
const handleKeyDown = (e: KeyboardEvent) => {
  if (!selectedCell.value) return

  const [currentRow, currentCol] = selectedCell.value.split('-').map(Number)

  const moveMap: Record<string, [number, number]> = {
    'ArrowUp': [-1, 0],
    'ArrowDown': [1, 0],
    'ArrowLeft': [0, -1],
    'ArrowRight': [0, 1],
    'Tab': [0, 1],
    'Enter': [1, 0]
  }

  // Handle Ctrl+C (Copy)
  if (e.ctrlKey && e.key === 'c') {
    const content = copySelectedRange()
    if (content) {
      navigator.clipboard.writeText(content)
    }
    return
  }

  // Handle Ctrl+V (Paste)
  if (e.ctrlKey && e.key === 'v') {
    navigator.clipboard.readText().then(text => {
      if (selectedCell.value) {
        pasteContent(text)
      }
    })
    return
  }

  // Handle Ctrl+Z (Undo)
  if (e.ctrlKey && e.key === 'z') {
    if (undoStack.value.length > 0) {
      const previousState = undoStack.value.pop()!
      const currentState: UndoRedoState = {
        cells: Object.fromEntries(
          Object.entries(cells.value).map(([key, cell]) => [key, cell.value])
        ),
        selectedCell: selectedCell.value,
        selectedRange: selectedRange.value
      }
      redoStack.value.push(currentState)

      // Restore previous state
      Object.entries(previousState.cells).forEach(([key, value]) => {
        cells.value[key].value = value
      })
      selectedCell.value = previousState.selectedCell
      selectedRange.value = previousState.selectedRange
      if (previousState.selectedCell) {
        selectCell(previousState.selectedCell)
      }
    }
    return
  }

  // Handle Ctrl+Y (Redo)
  if (e.ctrlKey && e.key === 'y') {
    if (redoStack.value.length > 0) {
      const nextState = redoStack.value.pop()!
      const currentState: UndoRedoState = {
        cells: Object.fromEntries(
          Object.entries(cells.value).map(([key, cell]) => [key, cell.value])
        ),
        selectedCell: selectedCell.value,
        selectedRange: selectedRange.value
      }
      undoStack.value.push(currentState)

      // Restore next state
      Object.entries(nextState.cells).forEach(([key, value]) => {
        cells.value[key].value = value
      })
      selectedCell.value = nextState.selectedCell
      selectedRange.value = nextState.selectedRange
      if (nextState.selectedCell) {
        selectCell(nextState.selectedCell)
      }
    }
    return
  }

  const move = moveMap[e.key]
  if (move) {
    e.preventDefault()
    const [rowDiff, colDiff] = move
    const newRow = currentRow + rowDiff
    const newCol = currentCol + colDiff

    if (
      newRow >= 0 && newRow < rows &&
      newCol >= 0 && newCol < cols
    ) {
      selectCell(`${newRow}-${newCol}`)
    }
  }
}

// Add keyboard event listener
useEventListener(window, 'keydown', handleKeyDown)
// Add mouse up listener to window to handle drag end outside table
useEventListener(window, 'mouseup', handleMouseUp)
</script>

<template>
  <div class="excel-table">
    <table>
      <tbody>
        <tr v-for="i in rows" :key="i">
          <td
            v-for="j in cols"
            :key="j"
            :class="{
              selected: selectedRange.includes(`${i-1}-${j-1}`),
              'primary-selected': selectedCell === `${i-1}-${j-1}`
            }"
            @mousedown="handleMouseDown(i-1, j-1)"
            @mousemove="handleMouseMove(i-1, j-1)"
            @mouseup="handleMouseUp"
          >
            <input
              type="text"
              v-model="cells[`${i-1}-${j-1}`].value"
              :data-id="`${i-1}-${j-1}`"
              @input="updateCell(`${i-1}-${j-1}`, $event.target.value)"
            >
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<style scoped>
.excel-table {
  margin: 20px;
  overflow: auto;
  user-select: none;
}

table {
  border-collapse: collapse;
  width: 100%;
  background: white;
}

td {
  border: 1px solid #ddd;
  padding: 0;
  position: relative;
}

input {
  width: 100%;
  height: 100%;
  padding: 8px;
  border: none;
  outline: none;
  background: transparent;
  color: #000000;
}

.selected {
  background-color: rgba(33, 150, 243, 0.1);
}

.primary-selected {
  background-color: #e3f2fd;
}

.selected input, .primary-selected input {
  background-color: transparent;
}
</style>