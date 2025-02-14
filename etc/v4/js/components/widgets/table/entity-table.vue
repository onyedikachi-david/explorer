<template>
  <div class="entity-table">
    <table>
      <thead>
        <template v-for="(col, index) in tableHeaders">
          <th class="noselect" 
              @click="toggleOrderBy(col)" 
              :style="getColumnStyle(index)">
            {{ col.name }}
            <template v-if="orderBy.index === col.index">
              <template v-if="orderBy.mode === 'none'">
              </template>
              <template v-if="orderBy.mode === 'asc'">
                <icon src="arrow-down"></icon>
              </template>
              <template v-if="orderBy.mode === 'desc'">
                <icon src="arrow-up"></icon>
              </template>
            </template>
            <div v-if="index < tableHeaders.length - 1" 
                 class="column-resizer"
                 @mousedown.stop="startResize($event, index)"
                 @dblclick.stop="resetColumnWidth(index)">
            </div>
          </th>
        </template>
        <th class="squeeze"></th>
      </thead>
      <tbody>
        <tr v-for="(result, i) in results">
          <td v-for="(col, colIndex) in tableHeaders" 
              :class="tdCss(i)"
              :style="getColumnStyle(colIndex)">
            <template v-if="isEntity(col)">
              <template v-if="col.get(result) === '*'">
                <div class="entity-table-none">
                  None
                </div>
              </template>
              <template v-else>
                <div class="entity-table-name">
                  <entity-parent :path="col.get(result)"></entity-parent>
                  <entity-name :path="col.get(result)"></entity-name>
                </div>
              </template>
            </template>
            <template v-else>
              <template v-if="col.get(result)">
                <entity-inspector-preview
                  :value="col.get(result)"
                  :type="col.schema"
                  :expand="false"
                  :readonly="true"
                  :compact="true"
                  fieldClass="table-field">
                </entity-inspector-preview>
              </template>
              <template v-else>
                <div class="entity-table-none">
                  None
                </div>
              </template>
            </template>
          </td>
          <td :class="tdCss(i)"></td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
export default { name: "entity-table" }
</script>

<script setup>
import { defineProps, computed, ref, onMounted, onUnmounted, nextTick } from 'vue';

const orderByModes = ["none", "asc", "desc"];
const orderBy = ref({});
const columnWidths = ref([]);
const resizing = ref({ active: false, index: -1, startX: 0, startWidth: 0 });

const props = defineProps({
  result: {type: Object, required: true }
});

const tableHeaders = computed(() => {
  const result = props.result;
  const columns = [];
  let index = 0;

  // Append each variable as column
  const query_info = result.query_info;
  if (query_info && query_info.vars) {
    for (let i = 0; i < query_info.vars.length; i ++) {
      const varName = query_info.vars[i];
      if (varName === "this") {
        columns.push({
          name: "Entity",
          schema: ["entity"],
          index: index++,
          get: (result) => {
            if (result.parent) {
              return result.parent + "." + result.name;
            } else {
              return result.name;
            }
          }
        });
      } else {
        columns.push({
          name: varName,
          schema: ["entity"],
          index: index++,
          get: (result) => {
            return result.vars[varName];
          }
        });
      }
    }
  }

  // Append fields
  const fields = result.field_info;
  if (!fields) {
    return columns;
  }

  for (let i = 0; i < fields.length; i ++) {
    const field = fields[i];
    if (field.schema) {
      columns.push({
        name: field.id,
        schema: field.schema,
        index: index++,
        get: (result) => {
          if (!result.is_set || result.is_set[i]) {
            return result.fields[i].data;
          } else {
            return undefined;
          }
        }
      });
    }
  }

  return columns;
});

const orderByIndices = computed(() => {
  if (!orderBy.value.mode || orderBy.value.mode === 'none') {
    return undefined;
  }

  let orderByIndex = orderBy.value.index;
  let orderByValues = [];

  const result = props.result;
  if (!result.results) {
    return [];
  }

  let resultIndex = 0;
  for (let r of result.results) {
    const value = tableHeaders.value[orderByIndex].get(r);
    orderByValues.push({value: value, resultIndex: resultIndex});
    resultIndex ++;
  }

  orderByValues.sort((a, b) => {
    let aValue = a.value;
    let bValue = b.value;

    if (typeof aValue === 'number' && typeof bValue === 'number') {
      return (aValue - bValue) - (bValue - aValue);
    }

    if (typeof aValue === 'number') {
      aValue = "" + aValue;
    }
    if (typeof bValue === 'number') {
      bValue = "" + bValue;
    }

    if (typeof aValue === 'object') {
      aValue = JSON.stringify(aValue);
    }
    if (typeof bValue === 'object') {
      bValue = JSON.stringify(bValue);
    }

    let comp = aValue.localeCompare(bValue);
    if (orderBy.value.mode === 'desc') {
      comp *= -1;
    }

    return comp;
  });

  return orderByValues;
});

const results = computed(() => {
  if (!orderByIndices.value) {
    return props.result.results;
  } else {
    let results = [];
    for (let elem of orderByIndices.value) {
      results.push(props.result.results[elem.resultIndex]);
    }
    return results;
  }
});

function isEntity(col) {
  if (col.schema && Array.isArray(col.schema)) {
    return col.schema[0] === "entity";
  } else {
    return false;
  }
}

function tdCss(i) {
  if (!(i % 2)) {
    return "cell-alt"
  } else {
    return "cell";
  }
}

function toggleOrderBy(col) {
  if (orderBy.value.index != col.index) {
    orderBy.value.index = col.index;
    orderBy.value.mode = orderByModes[1];
  } else {
    let orderByIndex = orderByModes.indexOf(orderBy.value.mode);
    if (orderByIndex == -1) {
      orderByIndex = 0;
    } else {
      orderByIndex = (orderByIndex + 1) % orderByModes.length;
    }

    orderBy.value.mode = orderByModes[orderByIndex];
  }
}

function getColumnStyle(index) {
  const width = columnWidths.value[index] || 150;
  return {
    width: `${width}px`,
    minWidth: '50px',
    maxWidth: 'none',
    position: 'relative',
    overflow: 'hidden'
  };
}

onMounted(() => {
  console.log('[EntityTable] Component mounted');
  // Initialize column widths based on actual number of columns
  let totalColumns = tableHeaders.value.length;
  columnWidths.value = new Array(totalColumns).fill(150);
  console.log('[EntityTable] Initialized column widths:', { totalColumns, widths: columnWidths.value });
  
  // Add window event listeners for resizing
  window.addEventListener('mousemove', handleResize, { passive: true });
  window.addEventListener('mouseup', stopResize);
});

onUnmounted(() => {
  console.log('[EntityTable] Component unmounting, removing event listeners');
  window.removeEventListener('mousemove', handleResize);
  window.removeEventListener('mouseup', stopResize);
});

function startResize(event, index) {
  console.log('[EntityTable] Starting resize:', { index, event: event.type });
  event.preventDefault();
  event.stopPropagation();
  
  // Force stop any ongoing resize
  if (resizing.value.active) {
    stopResize();
  }
  
  const target = event.target;
  const th = target.closest('th');
  if (!th) return;
  
  const initialWidth = th.offsetWidth;
  const initialX = event.pageX;
  
  resizing.value = {
    active: true,
    index,
    startX: initialX,
    startWidth: initialWidth,
    element: th
  };
  
  // Add visual feedback
  document.body.style.cursor = 'col-resize';
  target.classList.add('resizing');
  
  // Force style update
  nextTick(() => {
    if (th) {
      th.style.width = `${initialWidth}px`;
      th.style.minWidth = '50px';
      th.style.maxWidth = 'none';
    }
  });
}

function handleResize(event) {
  if (!resizing.value.active) return;
  
  requestAnimationFrame(() => {
    const delta = event.pageX - resizing.value.startX;
    const newWidth = Math.max(50, resizing.value.startWidth + delta);
    
    // Update width in our state
    columnWidths.value[resizing.value.index] = newWidth;
    
    // Force immediate DOM update
    if (resizing.value.element) {
      resizing.value.element.style.width = `${newWidth}px`;
    }
    
    // Update all cells in this column
    const table = resizing.value.element.closest('table');
    if (table) {
      const cells = table.querySelectorAll(`td:nth-child(${resizing.value.index + 1})`);
      cells.forEach(cell => {
        cell.style.width = `${newWidth}px`;
        cell.style.minWidth = '50px';
        cell.style.maxWidth = 'none';
      });
    }
  });
}

function stopResize() {
  if (!resizing.value.active) return;
  
  console.log('[EntityTable] Stopping resize:', {
    index: resizing.value.index,
    finalWidth: columnWidths.value[resizing.value.index]
  });
  
  // Remove visual feedback
  document.body.style.cursor = '';
  if (resizing.value.element) {
    const resizer = resizing.value.element.querySelector('.resizing');
    if (resizer) {
      resizer.classList.remove('resizing');
    }
  }
  
  resizing.value = { active: false, index: -1, startX: 0, startWidth: 0, element: null };
}

function resetColumnWidth(index) {
  columnWidths.value[index] = 150;
}

</script>

<style scoped>
.entity-table {
  position: relative;
  height: 100%;
  overflow-y: auto;
  width: 100%;
}

table {
  table-layout: fixed;
  border-collapse: collapse;
  width: 100%;
  text-align: left;
  font-variant: inherit;
  font-size: inherit;
}

thead {
  position: sticky;
  top: 0;
  z-index: 2;
  background-color: var(--bg-cell);
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.35);
}

th {
  position: relative;
  padding: var(--table-padding);
  height: 1.5rem;
  min-width: 50px;
  color: var(--primary-text);
  background-color: var(--bg-cell);
  cursor: pointer;
  user-select: none;
}

.column-resizer {
  position: absolute;
  right: -3px;
  top: 0;
  height: 100%;
  width: 6px;
  background-color: transparent;
  cursor: col-resize !important;
  user-select: none;
  touch-action: none;
  z-index: 10;
  transition: background-color 0.2s ease;
}

.column-resizer:hover,
.column-resizer.resizing {
  background-color: var(--primary-color) !important;
  opacity: 0.5;
}

/* Add a visual indicator for the resizer */
.column-resizer::after {
  content: '';
  position: absolute;
  right: 2px;
  top: 0;
  height: 100%;
  width: 2px;
  background-color: var(--primary-color);
  opacity: 0;
  transition: opacity 0.2s ease;
}

.column-resizer:hover::after,
.column-resizer.resizing::after {
  opacity: 0.5;
}

th, td {
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  transition: width 0.1s ease;
  min-width: 50px;
  max-width: none;
}

div.entity-table {
  position: relative;
  top: -0.5rem; /* table header has padding */
  height: 100%;
  overflow-y: auto;
}

table {
  border-collapse: collapse;
  width: 100%;
  text-align: left;
  font-variant: inherit;
  font-size: inherit;
  white-space: nowrap;
}

table thead {
  position: sticky;
  top: 0px;
  z-index: 2;
  box-shadow: 0px 2px 10px rgba(0, 0, 0, 0.35);
}

th:hover {
  background-color: var(--bg-cell-hover);
}

th.squeeze:hover {
  background-color: var(--bg-cell);
}

tr {
  border-style: solid;
  border-width: 0px;
  border-bottom-width: 1.5px;
  border-bottom-color: var(--tab-separator-color);
}

td {
  padding: var(--table-padding);
  padding-top: calc(var(--table-padding) * 0.5);
  padding-bottom: calc(var(--table-padding) * 0.5);
  background-color: var(--bg-cell);
}

td.cell-alt {
  background-color: var(--bg-cell-alt);
}

div.entity-table-name {
  display: flex;
  flex-direction: column;
  color: var(--primary-text);
}

div.entity-table-none {
  color: var(--secondary-text);
}

th.squeeze, td.squeeze {
  width: auto;
  min-width: 0;
  padding: 0;
  margin: 0;
}

.entity-table table {
  table-layout: fixed !important;
  width: 100% !important;
}

th, td {
  position: relative !important;
  overflow: hidden !important;
  white-space: nowrap !important;
  text-overflow: ellipsis !important;
  transition: none !important; /* Remove transition for smoother resizing */
}

.column-resizer {
  position: absolute !important;
  right: -3px !important;
  top: 0 !important;
  height: 100% !important;
  width: 6px !important;
  background-color: transparent !important;
  cursor: col-resize !important;
  user-select: none !important;
  touch-action: none !important;
  z-index: 1000 !important; /* Ensure it's above other elements */
}

.column-resizer:hover,
.column-resizer.resizing {
  background-color: var(--primary-color) !important;
  opacity: 0.5 !important;
}

/* ... rest of existing styles ... */
</style>
