<script setup>
import { ref, reactive, onMounted, watch } from 'vue';

const baseCanvas = ref(null);   // image + live effect
const uiCanvas   = ref(null);   // overlay UI for divider / lasso
const maskCanvas = ref(null);   // offscreen mask buffer (pixels = where to recolor)
const fileInput  = ref(null);

const SIDES = { LEFT: 'left', RIGHT: 'right' };

/**
 * OPTIONAL: Pre-mapped mask (normalized 0..1 coordinates).
 * Replace this with your polygon for the zipper tape or any shape you want.
 * Example makes a rough vertical region on the left half.
 */
const PRESET_MASK_NORMALIZED = [
  // x, y are 0..1 relative to image width/height
  { x: 0.12, y: 0.10 },
  { x: 0.28, y: 0.10 },
  { x: 0.30, y: 0.90 },
  { x: 0.10, y: 0.90 },
];

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
  img: null,
  natW: 0,
  natH: 0,
  dpr: Math.max(1, window.devicePixelRatio || 1),
  scale: 1,

  // target color & strength
  targetHex: '#4361ee',
  targetHue: 0,        // 0..1
  hueStrength: 150,    // 0..200 (%). >100 also boosts saturation

  // half-split controls (fallback when no mask)
  side: SIDES.LEFT,
  splitX: 0,
  draggingSplit: false,
  dragHitPad: 8,
  minSplit: 10,
  maxSplit: 10,

  // mask state
  useMask: true,           // if true and maskPoints exist, mask takes priority
  maskPoints: [],          // in canvas pixels
  maskPointsNorm: [],      // normalized 0..1 (for persistence/resizing)
  maskClosed: false,
  drawingMask: false,      // when true, clicks add polygon points

  // cached image
  originalImageData: null,
});

function pickFile() { fileInput.value?.click(); }
function onFile(e) {
  const file = e.target.files?.[0];
  if (!file) return;
  const url = URL.createObjectURL(file);
  loadImage(url);
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
    s.splitX = (baseCanvas.value.width / s.dpr) / 2;
    s.maxSplit = (baseCanvas.value.width / s.dpr) - s.minSplit;
    setTargetHex(s.targetHex, false);

    // If you want the preset mask active by default:
    setMaskFromNormalized(PRESET_MASK_NORMALIZED);

    redrawUI();
    applyRecolor();
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

  // create/resize mask canvas (offscreen)
  const m = document.createElement('canvas');
  m.width = Math.floor(cw * s.dpr);
  m.height = Math.floor(ch * s.dpr);
  maskCanvas.value = m;
}

function drawImage() {
  const ctx = baseCanvas.value.getContext('2d');
  ctx.clearRect(0, 0, baseCanvas.value.width, baseCanvas.value.height);
  ctx.drawImage(s.img, 0, 0, s.natW * s.scale, s.natH * s.scale);
  const full = ctx.getImageData(
    0, 0,
    baseCanvas.value.width / s.dpr,
    baseCanvas.value.height / s.dpr
  );
  s.originalImageData = full;
}

function redrawUI() {
  const ctx = uiCanvas.value.getContext('2d');
  const W = uiCanvas.value.width / s.dpr;
  const H = uiCanvas.value.height / s.dpr;
  ctx.clearRect(0, 0, W, H);

  // If mask exists, draw it
  if (s.maskPoints.length) {
    ctx.save();
    ctx.lineWidth = 2;
    ctx.strokeStyle = s.drawingMask ? 'rgba(0,0,0,0.8)' : 'rgba(0,0,0,0.6)';
    ctx.setLineDash([6, 4]);
    ctx.beginPath();
    ctx.moveTo(s.maskPoints[0].x, s.maskPoints[0].y);
    for (let i = 1; i < s.maskPoints.length; i++) ctx.lineTo(s.maskPoints[i].x, s.maskPoints[i].y);
    if (s.maskClosed) ctx.closePath();
    ctx.stroke();

    ctx.setLineDash([2, 3]);
    ctx.strokeStyle = 'rgba(255,255,255,0.9)';
    ctx.stroke();

    // vertices
    ctx.fillStyle = 'rgba(0,0,0,0.75)';
    s.maskPoints.forEach(p => {
      ctx.beginPath();
      ctx.arc(p.x, p.y, 2.5, 0, Math.PI * 2);
      ctx.fill();
    });
    ctx.restore();
  }

  // If no mask or useMask=false, show the split divider
  if (!s.useMask || !s.maskClosed) {
    const x = s.splitX;
    ctx.save();
    ctx.lineWidth = 2;
    ctx.strokeStyle = 'rgba(0,0,0,0.7)';
    ctx.beginPath();
    ctx.moveTo(x + 0.5, 0);
    ctx.lineTo(x + 0.5, H);
    ctx.stroke();

    ctx.setLineDash([4, 3]);
    ctx.strokeStyle = 'rgba(255,255,255,0.9)';
    ctx.beginPath();
    ctx.moveTo(x - 1.5, 0);
    ctx.lineTo(x - 1.5, H);
    ctx.stroke();

    ctx.fillStyle = 'rgba(0,0,0,0.8)';
    ctx.fillRect(x - 5, H / 2 - 16, 10, 32);
    ctx.restore();
  }
}

/* =========================
   Mask: preset & draw modes
   ========================= */
function setMaskFromNormalized(pointsNorm) {
  if (!pointsNorm || !pointsNorm.length) {
    clearMask();
    return;
  }
  const W = uiCanvas.value.width / s.dpr;
  const H = uiCanvas.value.height / s.dpr;
  s.maskPoints = pointsNorm.map(p => ({ x: p.x * W, y: p.y * H }));
  s.maskPointsNorm = JSON.parse(JSON.stringify(pointsNorm));
  s.maskClosed = true;
  s.drawingMask = false;
  rasterizeMask(); // draw to maskCanvas
}

function clearMask() {
  s.maskPoints = [];
  s.maskPointsNorm = [];
  s.maskClosed = false;
  s.drawingMask = false;
  const mctx = maskCanvas.value.getContext('2d');
  mctx.clearRect(0, 0, maskCanvas.value.width, maskCanvas.value.height);
}

function startDrawingMask() {
  s.drawingMask = true;
  s.maskPoints = [];
  s.maskPointsNorm = [];
  s.maskClosed = false;
  redrawUI();
}

function closeMask() {
  if (s.maskPoints.length >= 3) {
    s.maskClosed = true;
    s.drawingMask = false;
    // save normalized coords for persistence across resize
    const W = uiCanvas.value.width / s.dpr;
    const H = uiCanvas.value.height / s.dpr;
    s.maskPointsNorm = s.maskPoints.map(p => ({ x: p.x / W, y: p.y / H }));
    rasterizeMask();
    redrawUI();
    applyRecolor();
  }
}

function rasterizeMask() {
  const mctx = maskCanvas.value.getContext('2d');
  const W = maskCanvas.value.width / s.dpr;
  const H = maskCanvas.value.height / s.dpr;
  mctx.setTransform(s.dpr, 0, 0, s.dpr, 0, 0);
  mctx.clearRect(0, 0, W, H);
  if (!s.maskPoints.length || !s.maskClosed) return;
  mctx.save();
  mctx.beginPath();
  mctx.moveTo(s.maskPoints[0].x, s.maskPoints[0].y);
  for (let i = 1; i < s.maskPoints.length; i++) mctx.lineTo(s.maskPoints[i].x, s.maskPoints[i].y);
  mctx.closePath();
  mctx.fillStyle = 'rgba(255,255,255,1)'; // white = inside mask
  mctx.fill();
  mctx.restore();
}

/* =========================
   UI Handlers
   ========================= */
function onSideChange(e) {
  s.side = e.target.value;
  applyRecolor();
  redrawUI();
}
function onSplitSlider(e) {
  s.splitX = Number(e.target.value);
  applyRecolor();
  redrawUI();
}
function onStrengthInput(e) {
  s.hueStrength = Number(e.target.value);
  applyRecolor();
}
function toggleUseMask(e) {
  s.useMask = e.target.checked;
  applyRecolor();
  redrawUI();
}
function onCanvasDown(ev) {
  const { x, y } = getMouse(ev);

  // If we're drawing a mask, clicks add points
  if (s.drawingMask) {
    if (!s.maskPoints.length) {
      s.maskPoints.push({ x, y });
    } else {
      // If near first point and we already have >=2 points, close on click
      const first = s.maskPoints[0];
      const dx = x - first.x, dy = y - first.y;
      if (Math.hypot(dx, dy) < 10 && s.maskPoints.length >= 3) {
        closeMask();
        return;
      }
      s.maskPoints.push({ x, y });
    }
    redrawUI();
    return;
  }

  // Otherwise, allow dragging the split if mask is not active/closed
  if (!s.useMask || !s.maskClosed) {
    if (Math.abs(x - s.splitX) <= s.dragHitPad) s.draggingSplit = true;
  }
}
function onCanvasMove(ev) {
  const { x } = getMouse(ev);

  if (s.drawingMask) {
    // visual aid only; we draw static points; no preview line needed
    return;
  }

  if (!s.useMask || !s.maskClosed) {
    uiCanvas.value.style.cursor = Math.abs(x - s.splitX) <= s.dragHitPad ? 'col-resize' : 'default';
    if (!s.draggingSplit) return;
    const W = baseCanvas.value.width / s.dpr;
    s.splitX = clamp(x, s.minSplit, W - s.minSplit);
    applyRecolor();
    redrawUI();
  }
}
function onCanvasUp() { s.draggingSplit = false; }
function onCanvasDblClick() {
  if (s.drawingMask) closeMask();
}

function onKeydown(e) {
  if (e.key === 'Escape') {
    if (s.drawingMask) {
      s.drawingMask = false;
      s.maskPoints = [];
      s.maskClosed = false;
      redrawUI();
    }
  }
  if ((e.key === 'Enter' || e.key === 'Return') && s.drawingMask) {
    closeMask();
  }
}

function getMouse(e) {
  const rect = uiCanvas.value.getBoundingClientRect();
  return { x: e.clientX - rect.left, y: e.clientY - rect.top };
}

/* =========================
   Color & Recolor
   ========================= */
function hexToRgb(hex) {
  const m = /^#?([A-Fa-f0-9]{6})$/.exec(hex);
  if (!m) return null;
  const intVal = parseInt(m[1], 16);
  return { r: (intVal >> 16) & 255, g: (intVal >> 8) & 255, b: intVal & 255 };
}
function rgbToHsl(r, g, b) {
  r /= 255; g /= 255; b /= 255;
  const max = Math.max(r, g, b), min = Math.min(r, g, b);
  let h = 0, s_ = 0, l = (max + min) / 2;
  const d = max - min;
  if (d !== 0) {
    s_ = l > 0.5 ? d / (2 - max - min) : d / (max + min);
    switch (max) {
      case r: h = (g - b) / d + (g < b ? 6 : 0); break;
      case g: h = (b - r) / d + 2; break;
      case b: h = (r - g) / d + 4; break;
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
      if (t < 1/6) return p + (q - p) * 6 * t;
      if (t < 1/2) return q;
      if (t < 2/3) return p + (q - p) * (2/3 - t) * 6;
      return p;
    };
    const q = l < 0.5 ? l * (1 + s_) : l + s_ - l * s_;
    const p = 2 * l - q;
    r = hue2rgb(p, q, h + 1/3);
    g = hue2rgb(p, q, h);
    b = hue2rgb(p, q, h - 1/3);
  }
  return [Math.round(r * 255), Math.round(g * 255), Math.round(b * 255)];
}
function setTargetHex(hex, reapply = true) {
  const rgb = hexToRgb(hex);
  if (!rgb) return;
  const [h] = rgbToHsl(rgb.r, rgb.g, rgb.b);
  s.targetHex = hex;
  s.targetHue = h;
  if (reapply) applyRecolor();
}
function onHexInput(e) {
  const val = e.target.value.trim();
  const norm = val.startsWith('#') ? val : `#${val}`;
  if (/^#([A-Fa-f0-9]{6})$/.test(norm)) setTargetHex(norm);
}
function pickPreset(hex) { setTargetHex(hex); }
function clamp(v, a, b) { return Math.max(a, Math.min(b, v)); }
function wrap01(x) { x = x % 1; return x < 0 ? x + 1 : x; }
function clamp01(v) { return Math.max(0, Math.min(1, v)); }
function lerpHue(h0, h1, t) {
  let dh = h1 - h0;
  if (dh > 0.5) dh -= 1;
  if (dh < -0.5) dh += 1;
  return wrap01(h0 + dh * t);
}

/* Core recolor:
   - If useMask && maskClosed: apply only where mask alpha > 0
   - Else: apply to selected half (left/right of split)
*/
function applyRecolor() {
  if (!s.img || !s.originalImageData) return;
  const ctx = baseCanvas.value.getContext('2d');
  ctx.putImageData(s.originalImageData, 0, 0);

  const W = baseCanvas.value.width / s.dpr;
  const H = baseCanvas.value.height / s.dpr;

  const useMaskNow = s.useMask && s.maskClosed && s.maskPoints.length;

  // Region to process: for mask, use its bounding box; otherwise the chosen half
  let region = { x: 0, y: 0, w: W, h: H };
  let maskData = null;

  if (useMaskNow) {
    const bb = boundsOfPoints(s.maskPoints);
    region = {
      x: Math.max(0, Math.floor(bb.minX)),
      y: Math.max(0, Math.floor(bb.minY)),
      w: Math.max(0, Math.ceil(bb.maxX) - Math.floor(bb.minX)),
      h: Math.max(0, Math.ceil(bb.maxY) - Math.floor(bb.minY)),
    };
    // get mask alpha (same DPI/scaling)
    const mctx = maskCanvas.value.getContext('2d');
    const maskImg = mctx.getImageData(region.x, region.y, region.w, region.h);
    maskData = maskImg.data; // use alpha to gate recolor
  } else {
    const leftRegion  = { x: 0, y: 0, w: Math.floor(s.splitX), h: H };
    const rightRegion = { x: Math.floor(s.splitX), y: 0, w: W - Math.floor(s.splitX), h: H };
    region = (s.side === SIDES.LEFT) ? leftRegion : rightRegion;
    if (region.w <= 0 || region.h <= 0) return;
  }

  const imgData = ctx.getImageData(region.x, region.y, region.w, region.h);
  const data = imgData.data;
  const targetHue = s.targetHue;

  const tHue  = Math.min(1, Math.max(0, s.hueStrength / 100));
  const extra = Math.min(1, Math.max(0, (s.hueStrength - 100) / 100));

  for (let j = 0; j < region.h; j++) {
    for (let i = 0; i < region.w; i++) {
      const idx = (j * region.w + i) * 4;
      const a = data[idx + 3];
      if (a === 0) continue;

      // If masking, skip pixels outside polygon
      if (useMaskNow) {
        const ma = maskData[idx + 3]; // alpha from mask
        if (ma === 0) continue;
      }

      const r = data[idx], g = data[idx + 1], b = data[idx + 2];
      let [h0, s0, l0] = rgbToHsl(r, g, b);

      const h1 = (tHue < 1) ? lerpHue(h0, targetHue, tHue) : targetHue;
      const s1 = (extra > 0) ? clamp01(s0 + (1 - s0) * extra) : s0;

      const [rr, gg, bb] = hslToRgb(h1, s1, l0);
      data[idx]     = rr;
      data[idx + 1] = gg;
      data[idx + 2] = bb;
    }
  }

  ctx.putImageData(imgData, region.x, region.y);
}

function boundsOfPoints(pts) {
  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
  for (const p of pts) {
    if (p.x < minX) minX = p.x;
    if (p.y < minY) minY = p.y;
    if (p.x > maxX) maxX = p.x;
    if (p.y > maxY) maxY = p.y;
  }
  return { minX, minY, maxX, maxY };
}

/* =========================
   Lifecycle / Watchers
   ========================= */
onMounted(() => {
  loadImage('/zipper.jpg'); // default
  window.addEventListener('keydown', onKeydown);
  window.addEventListener('resize', () => {
    if (!s.img) return;
    const saved = {
      splitX: s.splitX,
      side: s.side,
      targetHex: s.targetHex,
      hueStrength: s.hueStrength,
      useMask: s.useMask,
      maskNorm: s.maskPointsNorm ? JSON.parse(JSON.stringify(s.maskPointsNorm)) : [],
    };
    fitToCanvas();
    drawImage();

    const W = baseCanvas.value.width / s.dpr;
    s.splitX = clamp(saved.splitX, s.minSplit, W - s.minSplit);
    s.side = saved.side;
    setTargetHex(saved.targetHex, false);
    s.hueStrength = saved.hueStrength;
    s.maxSplit = W - s.minSplit;
    s.useMask = saved.useMask;

    if (saved.maskNorm && saved.maskNorm.length) {
      setMaskFromNormalized(saved.maskNorm);
    } else {
      clearMask();
    }

    redrawUI();
    applyRecolor();
  });
});

watch(() => s.side, applyRecolor);
watch(() => s.targetHex, applyRecolor);
watch(() => s.hueStrength, applyRecolor);
watch(() => s.useMask, applyRecolor);
</script>

<template>
  <div style="padding:16px; font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial;">
    <h2 style="margin:0 0 12px;">Selective Recolor: Preset Mask or Drawn Mask (falls back to Half Split)</h2>

    <div style="display:flex; flex-wrap:wrap; gap:12px; align-items:center; margin-bottom:12px;">
      <input ref="fileInput" type="file" accept="image/png,image/jpeg" @change="onFile" style="display:none" />
      <button @click="pickFile">Load Image (PNG/JPG)</button>

      <label style="display:flex; align-items:center; gap:8px;">
        <span>Hue Strength</span>
        <input type="range" min="0" max="200" step="1" :value="s.hueStrength" @input="onStrengthInput" />
        <span style="width:42px; text-align:right;">{{ s.hueStrength }}%</span>
      </label>

      <label style="display:flex; align-items:center; gap:8px;">
        <input type="checkbox" :checked="s.useMask" @change="toggleUseMask" />
        <span>Use Mask (if available)</span>
      </label>

      <button @click="startDrawingMask">Draw Mask</button>
      <button @click="clearMask" :disabled="!s.maskPoints.length">Clear Mask</button>

      <label style="display:flex; align-items:center; gap:6px;">
        <span>Side</span>
        <select :value="s.side" @change="onSideChange" :disabled="s.useMask && s.maskClosed">
          <option :value="`left`">Left side recolored</option>
          <option :value="`right`">Right side recolored</option>
        </select>
      </label>

      <div style="display:flex; align-items:center; gap:8px; white-space:nowrap;" v-if="!s.useMask || !s.maskClosed">
        <span>Split</span>
        <input
          type="range"
          :min="s.minSplit"
          :max="s.maxSplit"
          step="1"
          :value="s.splitX"
          @input="onSplitSlider"
          style="width:200px;"
        />
        <span style="width:44px; text-align:right;">{{ Math.round(s.splitX) }}px</span>
      </div>

      <div style="height:1px; background:#e3e3e3; width:100%;"></div>

      <div style="display:flex; gap:8px; flex-wrap:wrap; align-items:center;">
        <span style="margin-right:4px;">Quick colors:</span>
        <button
          v-for="c in PALETTE"
          :key="c.hex"
          @click="pickPreset(c.hex)"
          :title="c.name"
          :style="{
            width:'28px', height:'28px', border:'1px solid #bbb', borderRadius:'6px',
            background:c.hex, outline: s.targetHex.toLowerCase() === c.hex.toLowerCase() ? '2px solid #333' : 'none'
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
    </div>

    <div style="position:relative; display:inline-block; border:1px solid #ddd; background:#f6f6f6;">
      <canvas ref="baseCanvas"></canvas>
      <canvas ref="uiCanvas"
              style="position:absolute; left:0; top:0; cursor:crosshair;"
              @mousedown="onCanvasDown"
              @mousemove="onCanvasMove"
              @mouseup="onCanvasUp"
              @mouseleave="onCanvasUp"
              @dblclick="onCanvasDblClick"></canvas>
    </div>

    <p style="color:#555; margin-top:8px;">
      Mask drawing: Click to add points around your area. Double-click or press Enter to close.  
      When a mask is closed and “Use Mask” is checked, recolor applies only inside the mask.
    </p>
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
select, input[type="range"], input[type="text"] { cursor: pointer; }
</style>
