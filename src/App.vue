<script setup lang="ts">
import { onMounted, watch, ref, reactive } from "vue";

const dotSize = ref(8);
const gap = ref(2);
const cols = ref(64);
const rows = ref(16);

// 2 次元配列（反応性）
const grid = reactive<boolean[][]>([]);
const canvas = ref({} as HTMLCanvasElement);

// ---- ユーティリティ ----
const ensureGridSize = () => {
  // 行調整
  while (grid.length < rows.value) grid.push(Array(cols.value).fill(false));
  if (grid.length > rows.value) grid.length = rows.value;
  // 列調整
  for (let y = 0; y < rows.value; y++) {
    const row = grid[y] || (grid[y] = []);
    while (row.length < cols.value) row.push(false);
    if (row.length > cols.value) row.length = cols.value;
  }
};

// 初期化
ensureGridSize();

// ---- ドット画像のプリレンダ ----
let imgOn: HTMLCanvasElement, imgOff: HTMLCanvasElement;
const makeDotImg = (lit: boolean) => {
  const r = dotSize.value / 2;
  const s = dotSize.value;
  const off = document.createElement('canvas');
  off.width = off.height = s;
  const ctx = off.getContext('2d');
  if (!ctx) throw new Error('Canvas context not available');
  const g = ctx.createRadialGradient(r, r, 0, r, r, r);
  if (lit) {
    g.addColorStop(0, '#ffb030');
    g.addColorStop(0.7, '#d37e00');
    g.addColorStop(1, '#332100');
    ctx.fillStyle = g;
    ctx.shadowColor = '#ff8c00';
    ctx.shadowBlur = r * 1.3;
  } else {
    g.addColorStop(0, '#333');
    g.addColorStop(1, '#000');
    ctx.fillStyle = g;
  }
  ctx.beginPath();
  ctx.arc(r, r, r, 0, Math.PI * 2);
  ctx.fill();
  return off;
};
const updateDotImgs = () => { imgOff = makeDotImg(false); imgOn = makeDotImg(true); };

// ---- 描画 ----
const ctx = () => canvas.value.getContext('2d');
const skip = () => dotSize.value + gap.value;
let renderQueued = false;
const render = () => {
  const c = ctx();
  const s = skip();
  canvas.value.width = cols.value * s;
  canvas.value.height = rows.value * s;
  if (!c) return;
  c.clearRect(0, 0, canvas.value.width, canvas.value.height);
  for (let y = 0; y < rows.value; y++) {
    for (let x = 0; x < cols.value; x++) {
      c.drawImage(grid[y][x] ? imgOn : imgOff, x * s, y * s);
    }
  }
};
const scheduleRender = () => {
  if (renderQueued) return;
  renderQueued = true;
  requestAnimationFrame(() => {
    renderQueued = false;
    render();
  });
};

// ---- 座標→セル ----
const cellFromEvent = e => {
  const rect = canvas.value.getBoundingClientRect();
  const s = skip();
  const x = Math.floor((e.clientX - rect.left) / s);
  const y = Math.floor((e.clientY - rect.top) / s);
  if (x >= 0 && x < cols.value && y >= 0 && y < rows.value) return { x, y };
  return null;
};

// ---- ドラッグ描画 ----
let drawing = false;
let drawState = false;

const pointerDown = e => {
  e.preventDefault();
  const cell = cellFromEvent(e);
  if (!cell) return;
  drawState = !grid[cell.y][cell.x];
  grid[cell.y][cell.x] = drawState;
  drawing = true;
  scheduleRender();
};
const pointerMove = e => {
  if (!drawing) return;
  const cell = cellFromEvent(e);
  if (!cell) return;
  if (grid[cell.y][cell.x] !== drawState) {
    grid[cell.y][cell.x] = drawState;
    scheduleRender();
  }
};
const pointerUp = () => { drawing = false; };

// ---- クリア ----
const clear = () => {
  for (let y = 0; y < rows.value; y++) grid[y].fill(false);
  scheduleRender();
};

// ---- ライフサイクル ----
onMounted(() => {
  updateDotImgs();
  render();
  canvas.value.addEventListener('pointerdown', pointerDown);
  canvas.value.addEventListener('pointermove', pointerMove);
  window.addEventListener('pointerup', pointerUp);
});

// ---- ウォッチャ ----
watch(dotSize, () => { updateDotImgs(); scheduleRender(); });
watch([rows, cols], () => { ensureGridSize(); scheduleRender(); });

</script>

<template>

    <h1 class="text-xl font-bold">電車行き先LED表示シミュレーター</h1>

    <div class="controls">
      <label>ドット直径
        <input type="number" v-model.number="dotSize" min="2" max="20" />
      </label>
      <label>ドット間隔
        <input type="number" v-model.number="gap" min="2" max="20" />
      </label>
      <label>列数
        <input type="number" v-model.number="cols" min="8" max="128" />
      </label>
      <label>行数
        <input type="number" v-model.number="rows" min="4" max="64" />
      </label>
      <button @click="clear">クリア</button>
    </div>
    <canvas ref="canvas"></canvas>
</template>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
