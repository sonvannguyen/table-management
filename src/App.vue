<script setup lang="ts">
import ExcelTable from './components/ExcelTable.vue'
</script>

<template>
  <div class="container">
    <h1>Excel-like Table</h1>
    <p class="instructions">
      Use arrow keys to navigate between cells<br>
      Click and drag to select multiple cells<br>
      Ctrl+C to copy selected cells<br>
      Ctrl+V to paste content<br>
      Ctrl+Z to undo<br>
      Ctrl+Y to redo
    </p>
    <ExcelTable />
  </div>
</template>

<style>
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  color: #000000;
}

.instructions {
  background-color: #f5f5f5;
  padding: 15px;
  border-radius: 8px;
  margin-bottom: 20px;
  line-height: 1.6;
  color: #000000;
}

h1 {
  margin-bottom: 20px;
  color: #000000;
}
</style>