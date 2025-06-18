<script setup lang="ts">
import { onMounted, watch, ref, reactive } from "vue";

const dotSize = ref(8);
const gap = ref(2);
const cols = ref(96);
const rows = ref(16);
const shadowSize = ref(1.1); // シャドウサイズ倍率
const isCircleLED = ref(false); // 縦横比を固定するかどうか

// 2 次元配列（反応性）
const grid = reactive<boolean[][]>([]);
const canvas = ref({} as HTMLCanvasElement);

// テキストエリア用のref
const gridText = ref("");

// grid -> テキスト変換
const gridToText = () =>
  grid.map(row => row.map(cell => (cell ? "1" : "0")).join("")).join("\n");
// テキスト -> grid変換
const textToGrid = (text: string) => {
  const lines = text.split("\n");
  for (let y = 0; y < rows.value; y++) {
    const line = lines[y] || "";
    for (let x = 0; x < cols.value; x++) {
      grid[y][x] = line[x] === "1";
    }
  }
  scheduleRender();
};

// gridの変更をテキストに反映
watch(
  () => grid.map(row => row.join("")),
  () => {
    gridText.value = gridToText();
  },
  { deep: true, immediate: true }
);
// テキストエリアの変更をgridに反映
watch(
  gridText,
  (val) => {
    textToGrid(val);
  }
);

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
function roundedRect(ctx: CanvasRenderingContext2D, x: number, y: number, w: number, h: number, r: number) {
  ctx.beginPath();
  ctx.moveTo(x + r, y);
  ctx.lineTo(x + w - r, y);
  ctx.quadraticCurveTo(x + w, y, x + w, y + r);
  ctx.lineTo(x + w, y + h - r);
  ctx.quadraticCurveTo(x + w, y + h, x + w - r, y + h);
  ctx.lineTo(x + r, y + h);
  ctx.quadraticCurveTo(x, y + h, x, y + h - r);
  ctx.lineTo(x, y + r);
  ctx.quadraticCurveTo(x, y, x + r, y);
  ctx.closePath();
}
const makeDotImg = (lit: boolean) => {
  const s = skip();
  const r = dotSize.value / 2;
  const off = document.createElement('canvas');
  off.width = off.height = s;
  const ctx = off.getContext('2d');
  if (!ctx) throw new Error('Canvas context not available');
  const cx = s / 2;
  const cy = s / 2;
  const size = dotSize.value;
  const corner = dotSize.value * 0.2; // 角丸半径
  if (lit) {
    // グラデーションのみ（シャドウなし）
    const g = ctx.createRadialGradient(cx, cy, 0, cx, cy, r * 1.5);
    g.addColorStop(0, '#ff9830');
    g.addColorStop(0.7, '#d37e00');
    g.addColorStop(1, 'rgba(51,33,0,0.8)');
    ctx.fillStyle = g;
    if (isCircleLED.value) {
      // 円形LED
      ctx.beginPath();
      ctx.arc(cx, cy, r, 0, Math.PI * 2);
      ctx.fill();
    } else {
      // 四角形LED
      roundedRect(ctx, cx - size/2, cy - size/2, size, size, corner);
    }
    ctx.fill();
  } else {
    const g = ctx.createRadialGradient(cx, cy, 0, cx, cy, r * 1.5);
    g.addColorStop(0, '#222');
    g.addColorStop(1, '#000');
    ctx.fillStyle = g;
    if (isCircleLED.value) {
      // 円形LED
      ctx.beginPath();
      ctx.arc(cx, cy, r, 0, Math.PI * 2);
      ctx.fill();
    } else {
      // 四角形LED
      roundedRect(ctx, cx - size/2, cy - size/2, size, size, corner);
    }
    ctx.fill();
  }
  return off;
};

const updateDotImgs = () => { imgOff = makeDotImg(false); imgOn = makeDotImg(true); };

// ---- 描画 ----
const ctx = () => canvas.value.getContext('2d');
const skip = () => dotSize.value + gap.value;
let renderQueued = false;
const render = () => {
  const CANVAS_PADDING = dotSize.value;
  const c = ctx();
  const s = skip();
  canvas.value.width = cols.value * s + CANVAS_PADDING * 2;
  canvas.value.height = rows.value * s + CANVAS_PADDING * 2;
  if (!c) return;
  c.clearRect(0, 0, canvas.value.width, canvas.value.height);
  c.fillStyle = "#000";
  c.fillRect(0, 0, canvas.value.width, canvas.value.height);
  // まず全ての消灯LEDを描画
  for (let y = 0; y < rows.value; y++) {
    for (let x = 0; x < cols.value; x++) {
      c.drawImage(imgOff, x * s + CANVAS_PADDING, y * s + CANVAS_PADDING);
    }
  }
  // 点灯LEDをグラデーション合成で重ねる
  for (let y = 0; y < rows.value; y++) {
    for (let x = 0; x < cols.value; x++) {
      if (grid[y][x]) {
        c.drawImage(imgOn, x * s + CANVAS_PADDING, y * s + CANVAS_PADDING);
      }
    }
  }
  c.globalCompositeOperation = "hard-light";
  // さらに点灯LEDのシャドウを直接描画
  for (let y = 0; y < rows.value; y++) {
    for (let x = 0; x < cols.value; x++) {
      if (grid[y][x]) {
        const cx = x * s + s / 2 + CANVAS_PADDING;
        const cy = y * s + s / 2 + CANVAS_PADDING;
        const r = dotSize.value * shadowSize.value;
        const grad = c.createRadialGradient(cx, cy, 0, cx, cy, r);
        grad.addColorStop(0, 'rgba(255, 176, 48, 0.5)');
        grad.addColorStop(0.7, 'rgba(255, 176, 48, 0.2)');
        grad.addColorStop(1, 'rgba(255, 176, 48, 0)');
        c.save();
        c.globalAlpha = 1.0;
        c.beginPath();
        c.arc(cx, cy, r, 0, Math.PI * 2);
        c.fillStyle = grad;
        c.fill();
        c.restore();
      }
    }
  }
  c.globalCompositeOperation = "source-over";
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
const cellFromEvent = (e: PointerEvent) => {
  const rect = canvas.value.getBoundingClientRect();
  const s = skip();
  const x = Math.floor((e.clientX - rect.left - dotSize.value) / s);
  const y = Math.floor((e.clientY - rect.top - dotSize.value) / s);
  if (x >= 0 && x < cols.value && y >= 0 && y < rows.value) return { x, y };
  return null;
};

// ---- ドラッグ描画 ----
let drawing = false;
let drawState = false;

const pointerDown = (e: PointerEvent) => {
  e.preventDefault();
  const cell = cellFromEvent(e);
  if (!cell) return;
  drawState = !grid[cell.y][cell.x];
  grid[cell.y][cell.x] = drawState;
  drawing = true;
  scheduleRender();
};
const pointerMove = (e: PointerEvent) => {
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

// ---- 画像保存 ----
const downloadImage = () => {
  const el = canvas.value;
  const url = el.toDataURL("image/png");
  const a = document.createElement("a");
  a.href = url;
  a.download = "led-matrix.png";
  a.click();
};

// --- 16進数圧縮・展開 ---
function gridToHex(): string {
  let bits = '';
  for (let y = 0; y < rows.value; y++) {
    for (let x = 0; x < cols.value; x++) {
      bits += grid[y][x] ? '1' : '0';
    }
  }
  // 4bitごとに分割して16進数化
  let hex = '';
  for (let i = 0; i < bits.length; i += 4) {
    const chunk = bits.slice(i, i + 4).padEnd(4, '0');
    hex += parseInt(chunk, 2).toString(16);
  }
  return hex;
}
function hexToGrid(hex: string) {
  let bits = '';
  for (let i = 0; i < hex.length; i++) {
    bits += parseInt(hex[i], 16).toString(2).padStart(4, '0');
  }
  let idx = 0;
  for (let y = 0; y < rows.value; y++) {
    for (let x = 0; x < cols.value; x++) {
      grid[y][x] = bits[idx++] === '1';
    }
  }
}

// --- URLクエリ共有 ---
const shareUrl = ref('');
function updateShareUrl() {
  const hex = gridToHex();
  const url = new URL(window.location.href);
  url.searchParams.set('data', hex);
  url.searchParams.set('cols', String(cols.value));
  url.searchParams.set('rows', String(rows.value));
  shareUrl.value = url.toString();
}
function loadFromQuery() {
  const params = new URLSearchParams(window.location.search);
  const hex = params.get('data');
  const qCols = params.get('cols');
  const qRows = params.get('rows');
  if (hex && qCols && qRows) {
    cols.value = parseInt(qCols);
    rows.value = parseInt(qRows);
    ensureGridSize();
    hexToGrid(hex);
    scheduleRender();
  }
}
onMounted(() => {
  loadFromQuery();
  updateDotImgs();
  render();
  canvas.value.addEventListener('pointerdown', pointerDown);
  canvas.value.addEventListener('pointermove', pointerMove);
  window.addEventListener('pointerup', pointerUp);
});

// ---- ウォッチャ ----
watch([dotSize, gap, shadowSize, isCircleLED], () => { updateDotImgs(); scheduleRender(); });
watch([rows, cols], () => { ensureGridSize(); scheduleRender(); });

</script>

<template>
  <div class="container">
    <div class="row">
      <div class="col my-4">
        <h1 class="fw-bold mb-3">
          単色LED方向幕っぽいジェネレータ
        </h1>
        <div class="row row-cols-lg-auto g-3 align-items-center">
          <div class="col-12">
            <div class="input-group">
              <div class="input-group-text">ドット直径</div>
              <input type="number" v-model.number="dotSize" min="2" max="20" class="form-control" />
            </div>
          </div>
          <div class="col-12">
            <div class="input-group">
              <div class="input-group-text">ドット間隔</div>
              <input type="number" v-model.number="gap" min="0" max="20" class="form-control">
            </div>
          </div>
          <div class="col-12">
            <div class="input-group">
              <div class="input-group-text">列数</div>
              <input type="number" v-model.number="cols" min="8" max="128" class="form-control"/>
            </div>
          </div>
          <div class="col-12">
            <div clasS="input-group">
              <div class="input-group-text">行数</div>
              <input type="number" v-model.number="rows" min="4" max="64" class="form-control"/>
            </div>
          </div>
          <div class="col-12">
            <div class="input-group">
              <div class="input-group-text">発光</div>
              <input type="range" v-model.number="shadowSize" min="0.5" max="2.0" step="0.01" class="form-control" />
              <div class="input-group-text">{{ shadowSize.toFixed(2) }}</div>
            </div>
          </div>
          <div class="col-12">
            <div class="form-check">
              <input
                type="checkbox"
                v-model="isCircleLED"
                class="form-check-input"
                id="circleLED"
              />
              <label class="form-check-label" for="circleLED">円形LED</label>
            </div>
          </div>
          <div class="col-12">
            <button @click="clear" class="btn btn-secondary">クリア</button>
          </div>
          <div class="col-12">
            <button @click="downloadImage" class="btn btn-primary">画像を保存</button>
          </div>
          <div class="col-12">
            <button @click="updateShareUrl" class="btn btn-success">共有用URL生成</button>
          </div>
          <div class="col-12">
            <input type="text" class="form-control font-monospace" v-model="shareUrl" readonly @focus="(e) => e.target.select()" />
          </div>
        </div>
        <canvas ref="canvas"></canvas>
        <div class="mt-3">
          <label>共有用LEDマトリックスデータ（0,1）</label>
          <textarea
            v-model="gridText"
            rows="10"
            class="form-control font-monospace"
          ></textarea>
        </div>
      </div>
    </div>
  </div>
</template>
