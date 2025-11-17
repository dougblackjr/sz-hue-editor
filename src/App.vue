<script setup>
import { ref, reactive, onMounted, watch } from 'vue';

const baseCanvas = ref(null);   // image + recolored result
const uiCanvas   = ref(null);   // selection rectangle + handles
const fileInput  = ref(null);

const PALETTE = [
  { name: 'Red',    hex: '#e63946' },
  { name: 'Orange', hex: '#f77f00' },
  { name: 'Yellow', hex: '#ffd60a' },
  { name: 'Green',  hex: '#2a9d8f' },
  { name: 'Teal',   hex: '#00b3a4' },
  { name: 'Blue',   hex: '#4361ee' },
  { name: 'Purple', hex: '#8a4fff' },
  { name: 'Pink',   hex: '#ff4d88' },
  { name: 'Brown',  hex: '#8b5e34' },
];

const s = reactive({
  // image
  img: null,
  natW: 0,
  natH: 0,
  dpr: Math.max(1, window.devicePixelRatio || 1),
  scale: 1,
  originalImageData: null,

  // target color
  targetHex: '#4361ee',
  targetHue: 0, // 0..1
  targetSat: 1, // 0..1
  targetLight: 0.5, // we probably won't use this, but keep it

  // swatch rect (CSS pixels)
  rect: { x: 60, y: 60, w: 160, h: 120 },

  // dragging / resizing
  dragging: false,
  dragMode: null, // 'move' | 'nw' | 'ne' | 'se' | 'sw' | 'n' | 's' | 'e' | 'w'
  dragStart: { x: 0, y: 0 },
  rectStart: { x: 0, y: 0, w: 0, h: 0 },
  handleSize: 8,
});

function pickFile() {
  fileInput.value?.click();
}

function onFile(e) {
  const f = e.target.files?.[0];
  if (!f) return;
  loadImage(URL.createObjectURL(f));
}

function loadImage(src) {
  const img = new Image();
  img.crossOrigin = 'anonymous';
  img.onload = () => {
    s.img = img;
    s.natW = img.naturalWidth;
    s.natH = img.naturalHeight;

    fitToCanvas();
    drawImage();

    const cw = baseCanvas.value.width / s.dpr;
    const ch = baseCanvas.value.height / s.dpr;
    s.rect = {
      x: Math.round(cw * 0.1),
      y: Math.round(ch * 0.1),
      w: Math.round(cw * 0.25),
      h: Math.round(ch * 0.2),
    };

    // initial target color
    setTargetHex(s.targetHex, false);

    applyRecolorInRect();
    drawUI();
  };
  img.src = src;
}

function fitToCanvas() {
  if (!s.img) return;
  const maxW = Math.min(1200, window.innerWidth - 48);
  s.scale = Math.min(1, maxW / s.natW);

  const cw = Math.floor(s.natW * s.scale);
  const ch = Math.floor(s.natH * s.scale);

  for (const c of [baseCanvas.value, uiCanvas.value]) {
    c.width = Math.floor(cw * s.dpr);
    c.height = Math.floor(ch * s.dpr);
    c.style.width = `${cw}px`;
    c.style.height = `${ch}px`;
    const ctx = c.getContext('2d');
    ctx.setTransform(s.dpr, 0, 0, s.dpr, 0, 0);
    ctx.imageSmoothingEnabled = true;
  }
}

function drawImage() {
  const ctx = baseCanvas.value.getContext('2d');
  ctx.clearRect(0, 0, baseCanvas.value.width, baseCanvas.value.height);
  ctx.drawImage(s.img, 0, 0, s.natW * s.scale, s.natH * s.scale);

  s.originalImageData = ctx.getImageData(
    0,
    0,
    baseCanvas.value.width / s.dpr,
    baseCanvas.value.height / s.dpr
  );
}

/* =========================
   UI Overlay (selection)
   ========================= */
function drawUI() {
  const ctx = uiCanvas.value.getContext('2d');
  const W = uiCanvas.value.width / s.dpr;
  const H = uiCanvas.value.height / s.dpr;
  ctx.clearRect(0, 0, W, H);

  const { x, y, w, h } = s.rect;

  // optional dim outside
  ctx.save();
  ctx.fillStyle = 'rgba(0,0,0,0.12)';
  ctx.fillRect(0, 0, W, H);
  ctx.clearRect(x, y, w, h);
  ctx.restore();

  // border
  ctx.save();
  ctx.strokeStyle = 'rgba(0,0,0,0.85)';
  ctx.lineWidth = 1.5;
  ctx.setLineDash([5, 3]);
  ctx.strokeRect(x + 0.5, y + 0.5, w, h);

  ctx.setLineDash([]);
  ctx.strokeStyle = 'rgba(255,255,255,0.9)';
  ctx.strokeRect(x + 0.5, y + 0.5, w, h);

  // handles
  ctx.fillStyle = 'rgba(0,0,0,0.85)';
  for (const p of handlePoints(s.rect, s.handleSize)) {
    ctx.fillRect(p.x, p.y, s.handleSize, s.handleSize);
  }
  ctx.restore();
}

function handlePoints(rect, size) {
  const { x, y, w, h } = rect;
  const hs = size;
  const midX = x + w / 2 - hs / 2;
  const midY = y + h / 2 - hs / 2;
  return [
    { x: x - hs / 2,     y: y - hs / 2,     mode: 'nw', cursor: 'nwse-resize' },
    { x: x + w - hs / 2, y: y - hs / 2,     mode: 'ne', cursor: 'nesw-resize' },
    { x: x + w - hs / 2, y: y + h - hs / 2, mode: 'se', cursor: 'nwse-resize' },
    { x: x - hs / 2,     y: y + h - hs / 2, mode: 'sw', cursor: 'nesw-resize' },
    { x: midX,           y: y - hs / 2,     mode: 'n',  cursor: 'ns-resize'   },
    { x: midX,           y: y + h - hs / 2, mode: 's',  cursor: 'ns-resize'   },
    { x: x - hs / 2,     y: midY,           mode: 'w',  cursor: 'ew-resize'   },
    { x: x + w - hs / 2, y: midY,           mode: 'e',  cursor: 'ew-resize'   },
  ];
}

function hitTest(x, y) {
  // handles first
  for (const h of handlePoints(s.rect, s.handleSize)) {
    if (
      x >= h.x &&
      x <= h.x + s.handleSize &&
      y >= h.y &&
      y <= h.y + s.handleSize
    ) {
      return h.mode;
    }
  }
  // inside rect?
  const { x: rx, y: ry, w, h } = s.rect;
  if (x >= rx && x <= rx + w && y >= ry && y <= ry + h) return 'move';
  return null;
}

function onCanvasDown(ev) {
  const { x, y } = getMouse(ev);
  const hit = hitTest(x, y);
  if (!hit) return;

  s.dragging = true;
  s.dragMode = hit;
  s.dragStart = { x, y };
  s.rectStart = { ...s.rect };
}

function onCanvasMove(ev) {
  const { x, y } = getMouse(ev);
  const hit = hitTest(x, y);
  uiCanvas.value.style.cursor = cursorForMode(hit);

  if (!s.dragging) return;

  const dx = x - s.dragStart.x;
  const dy = y - s.dragStart.y;

  let { x: rx, y: ry, w, h } = s.rectStart;

  switch (s.dragMode) {
    case 'move':
      rx += dx;
      ry += dy;
      break;
    case 'nw':
      rx += dx;
      ry += dy;
      w -= dx;
      h -= dy;
      break;
    case 'ne':
      ry += dy;
      w += dx;
      h -= dy;
      break;
    case 'se':
      w += dx;
      h += dy;
      break;
    case 'sw':
      rx += dx;
      w -= dx;
      h += dy;
      break;
    case 'n':
      ry += dy;
      h -= dy;
      break;
    case 's':
      h += dy;
      break;
    case 'e':
      w += dx;
      break;
    case 'w':
      rx += dx;
      w -= dx;
      break;
  }

  // normalize negative sizes
  if (w < 1) {
    rx = rx + w;
    w = Math.abs(w);
  }
  if (h < 1) {
    ry = ry + h;
    h = Math.abs(h);
  }

  // constrain to canvas
  const W = baseCanvas.value.width / s.dpr;
  const H = baseCanvas.value.height / s.dpr;
  rx = Math.max(0, Math.min(W - w, rx));
  ry = Math.max(0, Math.min(H - h, ry));

  s.rect = {
    x: Math.round(rx),
    y: Math.round(ry),
    w: Math.round(w),
    h: Math.round(h),
  };

  applyRecolorInRect();
  drawUI();
}

function onCanvasUp() {
  s.dragging = false;
  s.dragMode = null;
}

function cursorForMode(mode) {
  switch (mode) {
    case 'move':
      return 'move';
    case 'nw':
    case 'se':
      return 'nwse-resize';
    case 'ne':
    case 'sw':
      return 'nesw-resize';
    case 'n':
    case 's':
      return 'ns-resize';
    case 'e':
    case 'w':
      return 'ew-resize';
    default:
      return 'default';
  }
}

function getMouse(e) {
  const r = uiCanvas.value.getBoundingClientRect();
  return { x: e.clientX - r.left, y: e.clientY - r.top };
}

/* =========================
   Color helpers
   ========================= */
function hexToRgb(hex) {
  const m = /^#?([A-Fa-f0-9]{6})$/.exec(hex);
  if (!m) return null;
  const v = parseInt(m[1], 16);
  return {
    r: (v >> 16) & 255,
    g: (v >> 8) & 255,
    b: v & 255,
  };
}

function rgbToHsl(r, g, b) {
  r /= 255;
  g /= 255;
  b /= 255;
  const max = Math.max(r, g, b);
  const min = Math.min(r, g, b);
  let h = 0,
    s_ = 0;
  const l = (max + min) / 2;
  const d = max - min;
  if (d !== 0) {
    s_ = l > 0.5 ? d / (2 - max - min) : d / (max + min);
    switch (max) {
      case r:
        h = (g - b) / d + (g < b ? 6 : 0);
        break;
      case g:
        h = (b - r) / d + 2;
        break;
      case b:
        h = (r - g) / d + 4;
        break;
    }
    h /= 6;
  }
  return [h, s_, l];
}

function hslToRgb(h, s_, l) {
  let r, g, b;
  if (s_ === 0) {
    r = g = b = l;
  } else {
    const hue2rgb = (p, q, t) => {
      if (t < 0) t += 1;
      if (t > 1) t -= 1;
      if (t < 1 / 6) return p + (q - p) * 6 * t;
      if (t < 1 / 2) return q;
      if (t < 2 / 3) return p + (q - p) * (2 / 3 - t) * 6;
      return p;
    };
    const q = l < 0.5 ? l * (1 + s_) : l + s_ - l * s_;
    const p = 2 * l - q;
    r = hue2rgb(p, q, h + 1 / 3);
    g = hue2rgb(p, q, h);
    b = hue2rgb(p, q, h - 1 / 3);
  }
  return [Math.round(r * 255), Math.round(g * 255), Math.round(b * 255)];
}

/**
 * Set the target color, and cache its HSL so we can map
 * pixels to that color while keeping their own lightness.
 */
function setTargetHex(hex, reapply = true) {
  const rgb = hexToRgb(hex);
  if (!rgb) return;
  const [h, s_, l] = rgbToHsl(rgb.r, rgb.g, rgb.b);

  s.targetHex = hex.toUpperCase();
  s.targetHue = h;
  s.targetSat = s_;
  s.targetLight = l;

  if (reapply) applyRecolorInRect();
}

function onHexInput(e) {
  const val = e.target.value.trim();
  const norm = val.startsWith('#') ? val : `#${val}`;
  if (/^#([A-Fa-f0-9]{6})$/.test(norm)) {
    setTargetHex(norm);
  }
}

function pickPreset(hex) {
  setTargetHex(hex);
}

/* =========================
   Core recolor in swatch
   ========================= */
/**
 * Recolors ONLY the swatch rect by:
 *   - Converting pixel to HSL
 *   - Using target Hue & Saturation
 *   - Keeping original Lightness
 *
 * So you get full color replacement, but the original
 * brightness/texture stays, which makes it seamless.
 */
function applyRecolorInRect() {
  if (!s.img || !s.originalImageData) return;

  const ctx = baseCanvas.value.getContext('2d');
  // restore full original frame
  ctx.putImageData(s.originalImageData, 0, 0);

  const { x, y, w, h } = s.rect;
  if (w <= 0 || h <= 0) return;

  const imgData = ctx.getImageData(x, y, w, h);
  const data = imgData.data;
  const targetH = s.targetHue;
  const targetS = s.targetSat;

  for (let i = 0; i < data.length; i += 4) {
    const a = data[i + 3];
    if (a === 0) continue; // ignore fully transparent pixels

    const r = data[i];
    const g = data[i + 1];
    const b = data[i + 2];

    const [h0, s0, l0] = rgbToHsl(r, g, b);

    // full color replacement:
    //   - completely override Hue with targetH
    //   - completely override Saturation with targetS
    //   - keep original Lightness l0 (shading/texture)
    const [rr, gg, bb] = hslToRgb(targetH, targetS, l0);

    data[i] = rr;
    data[i + 1] = gg;
    data[i + 2] = bb;
    // alpha unchanged
  }

  ctx.putImageData(imgData, x, y);
}

/* =========================
   Lifecycle
   ========================= */
onMounted(() => {
  setTargetHex(s.targetHex, false);
  loadImage('/zipper.jpg'); // default image in public/

  window.addEventListener('resize', () => {
    if (!s.img) return;

    const saved = {
      rect: { ...s.rect },
      targetHex: s.targetHex,
    };

    fitToCanvas();
    drawImage();

    const W = baseCanvas.value.width / s.dpr;
    const H = baseCanvas.value.height / s.dpr;

    let { x, y, w, h } = saved.rect;
    if (w > W) w = W;
    if (h > H) h = H;
    x = Math.max(0, Math.min(W - w, x));
    y = Math.max(0, Math.min(H - h, y));

    s.rect = { x, y, w, h };

    setTargetHex(saved.targetHex, false);
    applyRecolorInRect();
    drawUI();
  });
});

watch(() => s.targetHex, applyRecolorInRect);
</script>

<template>
  <div style="padding:16px; font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial;">
    <h2 style="margin:0 0 12px;">Swatch Recolor (True Color Swap with Shading)</h2>

    <div style="display:flex; flex-wrap:wrap; gap:12px; align-items:center; margin-bottom:12px;">
      <input
        ref="fileInput"
        type="file"
        accept="image/png,image/jpeg"
        @change="onFile"
        style="display:none"
      />
      <button @click="pickFile">Load Image (PNG/JPG)</button>

      <div style="display:flex; gap:8px; flex-wrap:wrap; align-items:center;">
        <span style="margin-right:4px;">Quick colors:</span>
        <button
          v-for="c in PALETTE"
          :key="c.hex"
          @click="pickPreset(c.hex)"
          :title="c.name"
          :style="{
            width:'28px', height:'28px',
            border:'1px solid #bbb',
            borderRadius:'6px',
            background:c.hex,
            outline: s.targetHex.toLowerCase() === c.hex.toLowerCase()
              ? '2px solid #333'
              : 'none'
          }"
        />
        <label style="display:flex; align-items:center; gap:6px; margin-left:6px;">
          <span>HEX</span>
          <input
            type="text"
            :value="s.targetHex"
            @input="onHexInput"
            placeholder="#RRGGBB"
            style="width:100px; padding:4px 6px; border:1px solid #bbb; border-radius:6px;"
          />
        </label>
      </div>

      <span style="color:#666;">Drag the rectangle to move; drag the handles to resize the recolor swatch.</span>
    </div>

    <div style="position:relative; display:inline-block; border:1px solid #ddd; background:#f6f6f6;">
      <canvas ref="baseCanvas"></canvas>
      <canvas
        ref="uiCanvas"
        style="position:absolute; left:0; top:0;"
        @mousedown="onCanvasDown"
        @mousemove="onCanvasMove"
        @mouseup="onCanvasUp"
        @mouseleave="onCanvasUp"
      ></canvas>
    </div>
  </div>
</template>

<style>
button {
  padding: 6px 10px;
  border: 1px solid #bbb;
  border-radius: 6px;
  background: #fff;
  cursor: pointer;
}
input[type='text'] {
  cursor: text;
}
</style>
