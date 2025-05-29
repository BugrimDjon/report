<template>
  <div class="okx-total-table-wrapper">
    <el-table
      :data="tableData"
      row-key="Параметр"
      border
      :tree-props="{ children: 'children' }"
      :row-style="rowStyle"
      style="width: 100%"
    >
      <el-table-column fixed prop="Параметр" label="Параметр" width="250" />
      <el-table-column
        v-for="(coin, index) in columns.slice(1)"
        :key="index"
        :prop="coin"
        :label="coin"
        min-width="100"
      />
    </el-table>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'

const tableData = ref([])
const columns = ref([])

// Функция для форматирования чисел
function formatNumber(numStr) {
  if (!numStr) return ''
  const n = parseFloat(numStr.toString().replace(',', '.'))
  if (isNaN(n)) return numStr
  return n.toFixed(4)
}

// Функция для осветления цвета
function lightenColor(color, percent) {
  // color в формате "#rrggbb"
  const num = parseInt(color.slice(1), 16);
  let r = (num >> 16) & 0xFF;
  let g = (num >> 8) & 0xFF;
  let b = num & 0xFF;

  r = Math.min(255, Math.floor(r + (255 - r) * percent));
  g = Math.min(255, Math.floor(g + (255 - g) * percent));
  b = Math.min(255, Math.floor(b + (255 - b) * percent));

  return `rgb(${r},${g},${b})`;
}


// Преобразуем исходные данные в дерево с вложенностью
function transformDataForTable(data) {
  const coinNames = data[0].Values;
  const rows = data.slice(1);

  const result = [];

  for (const row of rows) {
    // Разбиваем Label по ">"
    const levelsRaw = row.Label.split('>').map(s => s.trim());

    let currentLevelArray = result;
    let currentNode = null;

    for (let i = 0; i < levelsRaw.length; i++) {
      let levelLabel = levelsRaw[i];

      // Извлекаем цвет из уровня (например, [color:#7f44f5])
      const colorMatch = levelLabel.match(/\[color:(#[0-9a-fA-F]{6})\]/);
      let color = null;
      if (colorMatch) {
        color = colorMatch[1];
        // Убираем тег цвета из текста
        levelLabel = levelLabel.replace(/\[color:#[0-9a-fA-F]{6}\]/, '').trim();
      }

      // Ищем или создаём узел
      let node = currentLevelArray.find(item => item.Параметр === levelLabel);
      if (!node) {
        node = { Параметр: levelLabel };
        currentLevelArray.push(node);
      }

      // Если цвет найден, сохраняем его в узле
      if (color) {
        node.color = color;
      }

      // Если последний уровень — добавляем значения монет
      if (i === levelsRaw.length - 1) {
        coinNames.forEach((coin, idx) => {
          node[coin] = formatNumber(row.Values[idx]);
        });
      }

      // Переходим на children текущего узла
      if (!node.children) node.children = [];
      currentLevelArray = node.children;
      currentNode = node;
    }
  }

  return result;
}


// Распространяем цвет от родителей к детям с осветлением
function propagateColors(nodes, parentColor = null, level = 0) {
  for (const node of nodes) {
    if (node.color) {
      node._effectiveColor = node.color
    } else if (parentColor) {
      // Чем глубже вложенность, тем больше светлеем (максимум 0.5)
      const lightenPercent = Math.min(level * 50.15, 0.5)
      node._effectiveColor = lightenColor(parentColor, lightenPercent)
    } else {
      node._effectiveColor = null
    }

    if (node.children && node.children.length) {
      propagateColors(node.children, node._effectiveColor, level + 1)
    }
  }
}

// Функция для установки стиля строки — задаём фон
function rowStyle({ row }) {
  if (row._effectiveColor) {
    return { backgroundColor: row._effectiveColor }
  }
  return {}
}

onMounted(async () => {
  // Загружаем данные из json файла (пример, замени на свои данные)
  const response = await fetch('/Report.json')
  const json = await response.json()

  const treeData = transformDataForTable(json)
  propagateColors(treeData)
  tableData.value = treeData

  columns.value = ['Параметр', ...json[0].Values]
})
</script>

<style>
.okx-total-table-wrapper {
 
  width: 60vw;
  max-width: 900px;
  margin: 60px auto;
  border: 2px solid #40ff6d !important;
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(64, 158, 255, 0.3);
    font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
  color: #222;
  
}
.okx-total-table-wrapper .el-table__inner-wrapper{
  background: #70db19 !important;
}
.okx-total-table-wrapper .el-table{
   --el-table-border-color:rgb(0, 213, 255);
  border-color: #f20808 !important;
}

.okx-total-table-wrapper .el-table .cell{
  line-height: 13px;
}

.okx-total-table-wrapper .el-table th {
  background-color: #d1bdbd !important;
}



.okx-total-table-wrapper .el-table__cell {
  color: #000000;
    -webkit-font-smoothing: antialiased; /* или subpixel-antialiased */
  -moz-osx-font-smoothing: grayscale;
}


.okx-total-table-wrapper .el-table__row {
  height: 24px !important;
  line-height: 24px !important;
  transition: none !important;
}

.okx-total-table-wrapper .el-table th,
.okx-total-table-wrapper .el-table td {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  font-size: 13px;
  font-weight: 600;
  padding: 2px 4px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.okx-total-table-wrapper .el-table__row:nth-child(odd) {
  background-color: #f9f9f9;
}

.okx-total-table-wrapper .el-table__row:nth-child(even) {
  background-color: #ffffff;
}

.okx-total-table-wrapper .el-table__row:hover {
  background-color: #f20808!important; /* полупрозрачный голубой */
  transition: background-color 0.3s ease !important;
  /* background-color: #eee6ff !important; */
  cursor: pointer;
  height: 22px !important;
  line-height: 22px !important;
}

@media (max-width: 480px) {
  .okx-total-table-wrapper .el-table th,
  .okx-total-table-wrapper .el-table td {
    font-size: 11px !important;
    padding: 2px 4px !important;
    color: #000000 !important;         /* более тёмный цвет текста */
  }
  .okx-total-table-wrapper .el-table__row {
    height: 22px !important;
    line-height: 22px !important;
  }




  }
</style>
