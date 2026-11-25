<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
  <title> Pa la muchacha Naomi♥</title>
  <style>
    /* Full-screen dark terminal aesthetic with zero distractions */
    *, *::before, *::after {
      box-sizing: border-box;
      margin: 0;
      padding: 0;
    }

    html, body {
      width: 100%;
      height: 100%;
      overflow: hidden;
      background-color: #09090b;
      user-select: none;
      -webkit-user-select: none;
    }

    canvas {
      display: block;
      width: 100vw;
      height: 100vh;
      background-color: #09090b;
    }
  </style>
</head>
<body>
  <!-- Full-viewport HTML5 Canvas for crisp monospace ASCII rendering -->
  <canvas id="ascii-canvas" aria-label="3D ASCII Flower Bouquet Animation"></canvas>

  <script>
    (function () {
      "use strict";

      // =========================================================================
      // 1. CONSTANTS & COLOR PALETTES (Faithful port from original Python script)
      // =========================================================================
      const SPIN_SPEED = 0.6;
      const TILT = 0.30;
      const CAM_DIST = 4.0;
      const MESSAGE = "para la muchacha Naomi ♥";
      const HINT = "Con cariño del conejo";
      const NUM_HEARTS = 18;
      const PIVOT_Y = 0.2;

      const WHITE = [244, 248, 251, 254, 255, 231];
      const PINK = [125, 162, 168, 205, 211, 218, 225];
      const RED = [52, 88, 124, 160, 196, 203];
      const PURPLE = [54, 91, 97, 134, 140, 183];
      const YELLOW = [136, 178, 214, 220, 226, 229];
      const ORANGE = [130, 166, 202, 208, 214, 216];
      const GOLD_CENTER = [94, 136, 178, 214, 220];
      const DARK_CENTER = [52, 58, 94, 130, 136];
      const GREENS = [22, 28, 28, 34, 35, 71, 77, 114];
      const PAPER = [101, 144, 187, 223, 230, 231];
      const RIBBON = [88, 125, 161, 197, 204, 211];
      const HEART_COLORS = [161, 197, 204, 205, 211, 218];
      const MSG_COLORS = [161, 168, 205, 211, 218, 225, 218, 211, 205, 168];

      const SHADE = ".:-=+*%@";
      const PETAL_SHADE = "+*#%";

      // Directional light vector L = normalize(-0.45, 0.6, 0.66)
      const _L = [-0.45, 0.6, 0.66];
      const _Ln = Math.sqrt(_L[0] * _L[0] + _L[1] * _L[1] + _L[2] * _L[2]);
      const LX = _L[0] / _Ln;
      const LY = _L[1] / _Ln;
      const LZ = _L[2] / _Ln;

      const DISC_TYPES = [
        { petals: WHITE, center: GOLD_CENTER, nPet: 13 },
        { petals: PINK, center: GOLD_CENTER, nPet: 8 },
        { petals: RED, center: DARK_CENTER, nPet: 6 },
        { petals: PURPLE, center: GOLD_CENTER, nPet: 7 },
        { petals: YELLOW, center: DARK_CENTER, nPet: 12 },
        { petals: ORANGE, center: DARK_CENTER, nPet: 9 },
      ];
      const TULIP_COLORS = [RED, PINK, YELLOW];

      // =========================================================================
      // 2. XTERM 256-COLOR ANSI INDEX TO CSS RGB CONVERTER
      // =========================================================================
      const ANSI_PALETTE = new Array(256);

      (function initAnsiPalette() {
        // 0-15: Standard 16 xterm system colors
        const standard16 = [
          [0, 0, 0],       [128, 0, 0],     [0, 128, 0],     [128, 128, 0],
          [0, 0, 128],     [128, 0, 128],   [0, 128, 128],   [192, 192, 192],
          [128, 128, 128], [255, 0, 0],     [0, 255, 0],     [255, 255, 0],
          [0, 0, 255],     [255, 0, 255],   [0, 255, 255],   [255, 255, 255]
        ];
        for (let i = 0; i < 16; i++) {
          const [r, g, b] = standard16[i];
          ANSI_PALETTE[i] = `rgb(${r},${g},${b})`;
        }

        // 16-231: 6x6x6 RGB color cube
        const levels = [0, 95, 135, 175, 215, 255];
        for (let i = 16; i <= 231; i++) {
          const idx = i - 16;
          const r = levels[Math.floor(idx / 36)];
          const g = levels[Math.floor((idx % 36) / 6)];
          const b = levels[idx % 6];
          ANSI_PALETTE[i] = `rgb(${r},${g},${b})`;
        }

        // 232-255: 24-step grayscale ramp
        for (let i = 232; i <= 255; i++) {
          const v = 8 + 10 * (i - 232);
          ANSI_PALETTE[i] = `rgb(${v},${v},${v})`;
        }
      })();

      // =========================================================================
      // 3. SEEDED PRNG (Mulberry32) & 3D VECTOR HELPERS
      // =========================================================================
      function createSeededRandom(seed) {
        let a = seed >>> 0;
        return {
          random() {
            a = (a + 0x6D2B79F5) >>> 0;
            let t = Math.imul(a ^ (a >>> 15), 1 | a);
            t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
            return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
          },
          uniform(min, max) {
            return min + (max - min) * this.random();
          }
        };
      }

      function norm(v) {
        const m = Math.sqrt(v[0] * v[0] + v[1] * v[1] + v[2] * v[2]) || 1.0;
        return [v[0] / m, v[1] / m, v[2] / m];
      }

      function cross(a, b) {
        return [
          a[1] * b[2] - a[2] * b[1],
          a[2] * b[0] - a[0] * b[2],
          a[0] * b[1] - a[1] * b[0]
        ];
      }

      function add(p, d, k) {
        return [p[0] + d[0] * k, p[1] + d[1] * k, p[2] + d[2] * k];
      }

      // =========================================================================
      // 4. 3D SCENE BUILDER (Deterministic geometry ported from build_scene)
      // =========================================================================
      function buildScene() {
        const rnd = createSeededRandom(21);
        const pts = [];
        const neck = [0.0, -0.12, 0.0];

        const C = [0.0, 0.12, 0.0];
        const D = 0.64;
        const N = 10;
        const golden = Math.PI * (3 - Math.sqrt(5));
        const heads = [];

        // 10 flower heads arranged in a golden angle spiral dome
        for (let i = 0; i < N; i++) {
          const el = (88 - 60 * Math.sqrt(i / (N - 1))) * (Math.PI / 180);
          const az = i * golden;
          const d = [
            Math.cos(el) * Math.cos(az),
            Math.sin(el),
            Math.cos(el) * Math.sin(az)
          ];
          const pos = add(C, d, D * rnd.uniform(0.92, 1.05));
          heads.push({ pos, axis: norm(d) });
        }

        const tulipSlots = new Set([3, 8]);
        const flowers = [];
        let discI = 0;

        // Stems and flower head descriptors
        for (let idx = 0; idx < heads.length; idx++) {
          const { pos, axis } = heads[idx];
          const base = add(pos, axis, -0.03);
          for (let j = 0; j < 70; j++) {
            const t = j / 69;
            const px = base[0] + (neck[0] - base[0]) * t;
            const py = base[1] + (neck[1] - base[1]) * t;
            const pz = base[2] + (neck[2] - base[2]) * t;
            pts.push({
              x: px, y: py, z: pz,
              nx: axis[0], ny: 0.0, nz: axis[2],
              pal: GREENS, tone: -0.15, twoSided: true, override: null
            });
          }

          if (tulipSlots.has(idx)) {
            let nT = 0;
            for (const k of tulipSlots) {
              if (k < idx) nT++;
            }
            flowers.push({
              pos,
              kind: "tulip",
              pal: TULIP_COLORS[nT % TULIP_COLORS.length],
              cpal: null,
              nPet: 0,
              R: 0.11,
              seed: rnd.uniform(0, 6.28)
            });
          } else {
            const spec = DISC_TYPES[discI % DISC_TYPES.length];
            discI++;
            flowers.push({
              pos,
              kind: "disc",
              pal: spec.petals,
              cpal: spec.center,
              nPet: spec.nPet,
              R: rnd.uniform(0.12, 0.14),
              seed: rnd.uniform(0, 6.28)
            });
          }
        }

        // 8 curved leaves around the bouquet collar with central rib shading
        for (let i = 0; i < 8; i++) {
          const a = ((i + 0.4) / 8) * 2 * Math.PI;
          const c = Math.cos(a);
          const s = Math.sin(a);
          const base = [0.26 * c, 0.30, 0.26 * s];
          const d = norm([c, 0.15, s]);
          const side = [-s, 0.0, c];
          const n = norm(cross(side, d));
          const L = 0.30;
          const W = 0.07;

          for (let ti = 0; ti < 32; ti++) {
            const t = ti / 31;
            const sinPiT = Math.sin(Math.PI * t);
            for (let wi = -5; wi <= 5; wi++) {
              const w = wi / 5;
              const curl = 0.025 * w * w;
              const px = base[0] + d[0] * L * t + side[0] * W * sinPiT * w + n[0] * curl;
              const py = base[1] + d[1] * L * t - 0.10 * t * t + n[1] * curl;
              const pz = base[2] + d[2] * L * t + side[2] * W * sinPiT * w + n[2] * curl;
              const rib = wi === 0 ? -0.25 : 0.0;
              pts.push({
                x: px, y: py, z: pz,
                nx: n[0], ny: n[1], nz: n[2],
                pal: GREENS, tone: rib, twoSided: true, override: null
              });
            }
          }
        }

        // Wrapping paper cone with top ruffle and folds
        const y0 = -0.20, y1 = 0.30, r0 = 0.06, r1 = 0.36;
        const kSlope = (r1 - r0) / (y1 - y0);
        const rings = 50;
        for (let i = 0; i < rings; i++) {
          const t = i / (rings - 1);
          const r = r0 + (r1 - r0) * t;
          const count = Math.max(8, Math.floor(170 * r));
          for (let j = 0; j < count; j++) {
            const ph = ((j + rnd.random()) / count) * 2 * Math.PI;
            const c = Math.cos(ph);
            const s = Math.sin(ph);
            let y = y0 + (y1 - y0) * t;
            if (t > 0.9) {
              y += 0.03 * Math.sin(9 * ph);
            }
            const fold = 0.3 * (Math.abs(Math.sin(5 * ph)) - 0.6);
            const nn = norm([c, -kSlope, s]);
            pts.push({
              x: r * c, y, z: r * s,
              nx: nn[0], ny: nn[1], nz: nn[2],
              pal: PAPER, tone: fold, twoSided: false, override: null
            });
          }
        }

        // 7 bottom stems emerging beneath the wrap
        for (let i = 0; i < 7; i++) {
          const a = (i / 7) * 2 * Math.PI;
          const ca = Math.cos(a);
          const sa = Math.sin(a);
          const sx = 0.025 * ca;
          const sz = 0.025 * sa;
          for (let j = 0; j < 40; j++) {
            const t = j / 39;
            pts.push({
              x: sx * (1 + 1.2 * t),
              y: -0.20 - 0.30 * t,
              z: sz * (1 + 1.2 * t),
              nx: ca, ny: 0.0, nz: sa,
              pal: GREENS, tone: -0.1, twoSided: true, override: null
            });
          }
        }

        // Ribbon band, bow loops, tails, and center knot
        const by = -0.12;
        const br = r0 + kSlope * (by - y0) + 0.01;
        for (let j = 0; j < 90; j++) {
          const ph = (j / 90) * 2 * Math.PI;
          const c = Math.cos(ph);
          const s = Math.sin(ph);
          for (const dy of [-0.012, 0.0, 0.012]) {
            pts.push({
              x: br * c, y: by + dy, z: br * s,
              nx: c, ny: 0.0, nz: s,
              pal: RIBBON, tone: 0.0, twoSided: false, override: null
            });
          }
        }

        const knot = [0.0, by, br + 0.01];
        for (const side of [-1, 1]) {
          for (let j = 0; j < 70; j++) {
            const a = (j / 70) * 2 * Math.PI;
            const lx = side * (0.07 + 0.07 * Math.cos(a));
            const ly = 0.04 * Math.sin(a) + 0.02 * (0.07 + 0.07 * Math.cos(a)) / 0.14;
            for (const fill of [1.0, 0.7]) {
              pts.push({
                x: knot[0] + lx * fill,
                y: knot[1] + ly * fill,
                z: knot[2] + 0.01,
                nx: side * 0.3, ny: 0.2, nz: 0.93,
                pal: RIBBON, tone: 0.1, twoSided: true, override: null
              });
            }
          }
          for (let j = 0; j < 30; j++) {
            const t = j / 29;
            pts.push({
              x: knot[0] + side * 0.06 * t,
              y: knot[1] - 0.15 * t,
              z: knot[2] + 0.015,
              nx: side * 0.2, ny: 0.0, nz: 0.98,
              pal: RIBBON, tone: -0.05 + 0.05 * Math.sin(t * 9), twoSided: true, override: null
            });
          }
        }

        for (const dx of [-0.01, 0.0, 0.01]) {
          pts.push({
            x: dx, y: by, z: knot[2] + 0.02,
            nx: 0, ny: 0, nz: 1,
            pal: RIBBON, tone: 0.2, twoSided: true, override: "@"
          });
        }

        // 90 baby's breath filler blossoms
        const babies = [];
        for (let i = 0; i < 90; i++) {
          const a = rnd.random() * 2 * Math.PI;
          const el = rnd.uniform(0.3, 1.4);
          const rr = rnd.uniform(0.40, 0.58);
          babies.push({
            x: rr * Math.cos(a) * Math.cos(el),
            y: C[1] + rr * Math.sin(el),
            z: rr * Math.sin(a) * Math.cos(el),
            phase: rnd.uniform(0, 6.28)
          });
        }

        // 30 orbiting ambient sparkles
        const sparkles = [];
        for (let i = 0; i < 30; i++) {
          const a = rnd.random() * 2 * Math.PI;
          const rr = rnd.uniform(0.75, 0.95);
          sparkles.push({
            x: rr * Math.cos(a),
            y: rnd.uniform(-0.3, 1.0),
            z: rr * Math.sin(a),
            phase: rnd.uniform(0, 6.28),
            speed: rnd.uniform(1.5, 4.0)
          });
        }

        return { pts, flowers, babies, sparkles };
      }

      // =========================================================================
      // 5. FLOATING HEARTS SIMULATION
      // =========================================================================
      function randomChoice(arr) {
        return arr[Math.floor(Math.random() * arr.length)];
      }

      function updateHearts(hearts, dt) {
        while (hearts.length < NUM_HEARTS) {
          hearts.push({
            x: Math.random(),
            y: 1.0 + Math.random(),
            speed: 0.03 + Math.random() * 0.05,
            phase: Math.random() * 6.28,
            color: randomChoice(HEART_COLORS)
          });
        }
        for (let i = 0; i < hearts.length; i++) {
          const h = hearts[i];
          h.y -= h.speed * dt;
          if (h.y < 0) {
            h.x = Math.random();
            h.y = 1.0;
            h.color = randomChoice(HEART_COLORS);
          }
        }
      }

      // =========================================================================
      // 6. CANVAS GRID & RENDERING ENGINE
      // =========================================================================
      const canvas = document.getElementById("ascii-canvas");
      const ctx = canvas.getContext("2d", { alpha: false });

      const FONT_STACK = "'JetBrains Mono', 'Fira Code', 'Cascadia Code', 'SFMono-Regular', Menlo, Monaco, Consolas, 'Liberation Mono', 'Courier New', monospace";

      let cols = 80;
      let rows = 36;
      let cellW = 8;
      let cellH = 16;
      let offsetX = 0;
      let offsetY = 0;
      let fontSize = 14;

      // Flat reusable buffers to avoid per-frame allocations
      let chars = [];
      let colors = new Int16Array(0);
      let zbuf = new Float32Array(0);

      function resize() {
        const dpr = Math.min(window.devicePixelRatio || 1, 2);
        const vw = window.innerWidth;
        const vh = window.innerHeight;

        canvas.width = Math.floor(vw * dpr);
        canvas.height = Math.floor(vh * dpr);
        ctx.setTransform(dpr, 0, 0, dpr, 0, 0);

        // Choose an ideal monospace font size for ~80-130 cols and ~38-55 rows
        fontSize = Math.max(8, Math.min(16, Math.min(vw / 56, vh / 50)));
        ctx.font = `${fontSize}px ${FONT_STACK}`;
        ctx.textBaseline = "top";

        const measuredW = ctx.measureText("M").width;
        cellW = Math.max(4.5, measuredW);
        // Classic 2:1 terminal cell aspect ratio so sx_scale = scale * 2.0 stays true
        cellH = cellW * 2.0;

        cols = Math.max(24, Math.floor(vw / cellW));
        rows = Math.max(16, Math.floor(vh / cellH));

        // Center the character grid seamlessly in the viewport
        offsetX = (vw - cols * cellW) * 0.5;
        offsetY = (vh - rows * cellH) * 0.5;

        const total = cols * rows;
        chars = new Array(total);
        colors = new Int16Array(total);
        zbuf = new Float32Array(total);
      }

      function renderFrame(scene, t, hearts) {
        const { pts, flowers, babies, sparkles } = scene;
        const total = cols * rows;

        // Clear character & depth buffers
        for (let i = 0; i < total; i++) {
          chars[i] = " ";
          colors[i] = 0;
          zbuf[i] = -1e9;
        }

        const viewRows = rows - 2;
        const scale = Math.max(1.0, Math.min((viewRows - 2) / 1.45, (cols - 2) / 2.7));
        const sxScale = scale * 2.0;
        const cx = cols * 0.5;
        // Vertically center nicely on tall portrait viewports while matching Python on landscape
        const bouquetHeight = 1.45 * scale;
        const topPad = Math.max(1.0, (viewRows - bouquetHeight) * 0.5);
        const cy = topPad + 0.78 * scale;

        const ang = t * SPIN_SPEED;
        const ca = Math.cos(ang);
        const sa = Math.sin(ang);
        const ct = Math.cos(TILT);
        const st = Math.sin(TILT);

        function plot(col, row, z, ch, color) {
          if (row >= 0 && row < viewRows && col >= 0 && col < cols) {
            const idx = row * cols + col;
            if (z > zbuf[idx]) {
              zbuf[idx] = z;
              chars[idx] = ch;
              colors[idx] = color;
            }
          }
        }

        // 1. Background floating hearts
        for (let i = 0; i < hearts.length; i++) {
          const h = hearts[i];
          const rawCol = Math.trunc((h.x + 0.015 * Math.sin(t * 1.2 + h.phase)) * cols);
          const col = ((rawCol % cols) + cols) % cols;
          const row = Math.trunc(h.y * viewRows);
          plot(col, row, -50, "♥", h.color);
        }

        // 2. 3D Point cloud (stems, leaves, wrapping paper, ribbon, bow)
        const shadeLen = SHADE.length;
        const petalShadeLen = PETAL_SHADE.length;

        for (let i = 0; i < pts.length; i++) {
          const p = pts[i];
          const xr = p.x * ca - p.z * sa;
          const zr = p.x * sa + p.z * ca;
          const yp = p.y - PIVOT_Y;
          const yt = yp * ct - zr * st;
          const zt = yp * st + zr * ct;
          const f = CAM_DIST / (CAM_DIST - zt);

          const col = Math.trunc(cx + xr * f * sxScale);
          const row = Math.trunc(cy - yt * f * scale);

          const nxr = p.nx * ca - p.nz * sa;
          const nzr = p.nx * sa + p.nz * ca;
          const nyt = p.ny * ct - nzr * st;
          const nzt = p.ny * st + nzr * ct;

          let dot = nxr * LX + nyt * LY + nzt * LZ;
          if (p.twoSided && dot < 0) {
            dot = -dot * 0.55;
          }

          let b = 0.25 + 0.75 * Math.max(0.0, dot) + p.tone;
          if (b > 0.999) b = 0.999;
          else if (b < 0.0) b = 0.0;

          let ch;
          if (p.override === "P") {
            ch = PETAL_SHADE[Math.floor(b * petalShadeLen)];
          } else if (p.override !== null) {
            ch = p.override;
          } else {
            ch = SHADE[Math.floor(b * shadeLen)];
          }

          const palColor = p.pal[Math.floor(b * p.pal.length)];
          plot(col, row, zt, ch, palColor);
        }

        // 3. Procedural billboard flowers (disc daisies & tulips)
        for (let i = 0; i < flowers.length; i++) {
          const fl = flowers[i];
          const x = fl.pos[0], y = fl.pos[1], z = fl.pos[2];

          const xr = x * ca - z * sa;
          const zr = x * sa + z * ca;
          const yp = y - PIVOT_Y;
          const yt = yp * ct - zr * st;
          const zt = yp * st + zr * ct;
          const f = CAM_DIST / (CAM_DIST - zt);

          const fx = cx + xr * f * sxScale;
          const fy = cy - yt * f * scale;
          const rx = fl.R * f * sxScale;
          const ry = fl.R * f * scale;

          const dim = -0.4 * (1 - Math.min(1.0, Math.max(0.0, (zt + 0.6) / 1.2)));
          const rot = fl.seed + ang * 0.7;

          const rowStart = Math.trunc(fy - ry) - 1;
          const rowEnd = Math.trunc(fy + ry) + 1;
          const colStart = Math.trunc(fx - rx) - 1;
          const colEnd = Math.trunc(fx + rx) + 1;

          for (let row = rowStart; row <= rowEnd; row++) {
            const dy = (row + 0.5 - fy) / ry;
            for (let col = colStart; col <= colEnd; col++) {
              const dx = (col + 0.5 - fx) / rx;

              if (fl.kind === "disc") {
                const s = Math.sqrt(dx * dx + dy * dy);
                if (s > 1.0) continue;

                if (s < 0.3) {
                  let b = 0.55 - 0.35 * dx - 0.35 * dy + dim;
                  if (b > 0.999) b = 0.999;
                  else if (b < 0.0) b = 0.0;
                  const ch = s < 0.17 ? "@" : "o";
                  const colCode = fl.cpal[Math.floor(b * fl.cpal.length)];
                  plot(col, row, zt + 0.06, ch, colCode);
                  continue;
                }

                const ph = Math.atan2(dy, dx);
                const reach = 0.32 + 0.68 * Math.pow(Math.abs(Math.cos((fl.nPet * (ph + rot)) * 0.5)), 0.7);
                if (s > reach) continue;

                const edge = s > reach - 0.14 ? -0.22 : 0.0;
                let b = 0.62 - 0.22 * dx - 0.28 * dy + 0.12 * s + edge + dim;
                if (b > 0.999) b = 0.999;
                else if (b < 0.0) b = 0.0;

                const ch = PETAL_SHADE[Math.floor(b * petalShadeLen)];
                const colCode = fl.pal[Math.floor(b * fl.pal.length)];
                plot(col, row, zt + 0.04 * (1 - s), ch, colCode);
              } else {
                // Tulip cup silhouette
                const v = (1 - dy) * 0.5;
                if (v < 0 || v > 1) continue;

                const hw = 0.62 * Math.sin(Math.PI * 0.5 * Math.min(1.0, v * 1.5 + 0.05));
                if (hw <= 0 || Math.abs(dx) > hw) continue;

                const xn = dx / hw;
                const cosTop = Math.cos(1.5 * Math.PI * xn);
                if (v > 0.72 + 0.28 * cosTop * cosTop) continue;

                const seam = (Math.abs(Math.abs(xn) - 0.33) < 0.12 && v > 0.3) ? -0.25 : 0.0;
                let b = 0.55 - 0.3 * xn + 0.1 * v + seam + dim;
                if (b > 0.999) b = 0.999;
                else if (b < 0.0) b = 0.0;

                const ch = PETAL_SHADE[Math.floor(b * petalShadeLen)];
                const colCode = fl.pal[Math.floor(b * fl.pal.length)];
                plot(col, row, zt + 0.03, ch, colCode);
              }
            }
          }
        }

        // 4. Baby's breath blossoms
        for (let i = 0; i < babies.length; i++) {
          const b = babies[i];
          const xr = b.x * ca - b.z * sa;
          const zr = b.x * sa + b.z * ca;
          const yp = b.y - PIVOT_Y;
          const yt = yp * ct - zr * st;
          const zt = yp * st + zr * ct;
          const f = CAM_DIST / (CAM_DIST - zt);

          const col = Math.trunc(cx + xr * f * sxScale);
          const row = Math.trunc(cy - yt * f * scale);
          const star = Math.sin(t * 2 + b.phase) > 0.6;
          plot(col, row, zt + 0.02, star ? "*" : "·", star ? 231 : 255);
        }

        // 5. Twinkling ambient sparkles
        for (let i = 0; i < sparkles.length; i++) {
          const sp = sparkles[i];
          const xr = sp.x * ca - sp.z * sa;
          const zr = sp.x * sa + sp.z * ca;
          const yp = sp.y - PIVOT_Y;
          const yt = yp * ct - zr * st;
          const zt = yp * st + zr * ct;
          const f = CAM_DIST / (CAM_DIST - zt);

          const col = Math.trunc(cx + xr * f * sxScale);
          const row = Math.trunc(cy - yt * f * scale);
          const tw = Math.sin(t * sp.speed + sp.phase);

          if (tw > 0.85) {
            plot(col, row, zt, "*", 231);
          } else if (tw > 0.4) {
            plot(col, row, zt, "+", 225);
          } else if (tw > 0.0) {
            plot(col, row, zt, "·", 218);
          }
        }

        // 6. Centered animated bottom message ("for you ♥")
        const mrow = rows - 2;
        const msgStart = Math.max(0, Math.floor((cols - MESSAGE.length) / 2));
        const shift = Math.trunc(t * 6);
        for (let i = 0; i < MESSAGE.length; i++) {
          const c = msgStart + i;
          if (c < cols && mrow >= 0) {
            const idx = mrow * cols + c;
            chars[idx] = MESSAGE[i];
            colors[idx] = MSG_COLORS[(i + shift) % MSG_COLORS.length];
          }
        }

        // 7. Subtle bottom credit in ANSI color 238
        const hrow = rows - 1;
        const hintStart = Math.max(0, Math.floor((cols - HINT.length) / 2));
        for (let i = 0; i < HINT.length; i++) {
          const c = hintStart + i;
          if (c < cols && hrow >= 0) {
            const idx = hrow * cols + c;
            chars[idx] = HINT[i];
            colors[idx] = 238;
          }
        }

        // 8. Paint onto Canvas
        ctx.fillStyle = "#09090b";
        ctx.fillRect(0, 0, window.innerWidth, window.innerHeight);

        let lastColor = -1;
        for (let r = 0; r < rows; r++) {
          const rowOffset = r * cols;
          const py = offsetY + r * cellH;
          for (let c = 0; c < cols; c++) {
            const idx = rowOffset + c;
            const ch = chars[idx];
            if (ch === " ") continue;

            const colCode = colors[idx];
            if (colCode !== lastColor) {
              lastColor = colCode;
              ctx.fillStyle = ANSI_PALETTE[colCode] || "#ffffff";
            }
            ctx.fillText(ch, offsetX + c * cellW, py);
          }
        }
      }

      // =========================================================================
      // 7. INITIALIZATION & ANIMATION LOOP
      // =========================================================================
      window.onload = function () {
        try {
          const scene = buildScene();
          const hearts = [];

          resize();
          window.addEventListener("resize", resize, { passive: true });

          const startTime = performance.now();
          let lastTime = startTime;

          function animate(now) {
            const dt = Math.min((now - lastTime) / 1000, 0.1);
            lastTime = now;
            const elapsed = (now - startTime) / 1000;

            updateHearts(hearts, dt);
            renderFrame(scene, elapsed, hearts);
            requestAnimationFrame(animate);
          }

          requestAnimationFrame(animate);
        } catch (err) {
          console.error("Error initializing ASCII bouquet:", err);
        }
      };
    })();
  </script>
</body>
</html>
