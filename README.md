# migue142006.github.io
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
  <title>Ramo 3D ASCII</title>

  <!-- Tailwind CSS -->
  <script src="https://cdn.tailwindcss.com"></script>

  <!-- Fuentes: Inter para la interfaz y JetBrains Mono para el estilo terminal -->
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&family=JetBrains+Mono:wght@400;600;700&display=swap" rel="stylesheet" />

  <style>
    :root {
      color-scheme: dark;
    }

    html, body {
      margin: 0;
      padding: 0;
      width: 100%;
      height: 100%;
      overflow: hidden;
      background-color: #08080c;
      font-family: 'Inter', system-ui, -apple-system, sans-serif;
      -webkit-tap-highlight-color: transparent;
      user-select: none;
      -webkit-user-select: none;
    }

    #asciiCanvas {
      display: block;
      width: 100%;
      height: 100%;
      touch-action: none; /* Permite arrastrar y rotar sin desplazar la página */
      cursor: grab;
    }

    #asciiCanvas:active {
      cursor: grabbing;
    }

    .font-mono-terminal {
      font-family: 'JetBrains Mono', ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
    }

    /* Sutil resplandor para el mensaje personalizado */
    .text-glow {
      text-shadow: 0 0 18px rgba(255, 110, 150, 0.35), 0 2px 6px rgba(0, 0, 0, 0.85);
    }

    /* Animación suave para el aviso inicial */
    @keyframes fadeHint {
      0% { opacity: 0; transform: translate(-50%, 6px); }
      15% { opacity: 0.85; transform: translate(-50%, 0); }
      80% { opacity: 0.85; transform: translate(-50%, 0); }
      100% { opacity: 0; transform: translate(-50%, -4px); }
    }

    .hint-animate {
      animation: fadeHint 4.5s ease forwards;
    }
  </style>
</head>
<body class="bg-[#08080c] text-zinc-100 h-[100dvh] w-screen relative overflow-hidden">

  <!-- Lienzo principal donde se renderiza el ramo 3D en ASCII -->
  <canvas id="asciiCanvas" aria-label="Ramo de flores 3D interactivo en arte ASCII"></canvas>

  <!-- Aviso discreto inicial de interacción -->
  <div id="dragHint"
       class="hint-animate pointer-events-none fixed top-5 left-1/2 -translate-x-1/2 z-10 px-3.5 py-1.5 rounded-full bg-white/5 border border-white/10 backdrop-blur-md text-xs text-zinc-300 tracking-wide whitespace-nowrap">
    Arrastra para girar el ramo
  </div>

  <!-- Mensaje personalizado debajo del ramo -->
  <div id="messageContainer"
       class="pointer-events-none fixed inset-x-0 bottom-24 sm:bottom-24 z-10 flex justify-center px-6 transition-all duration-300">
    <p id="messageDisplay"
       class="font-mono-terminal text-center text-base sm:text-xl font-semibold tracking-wider text-rose-100/95 text-glow max-w-md break-words">
      Para ti
    </p>
  </div>

  <!-- Barra inferior discreta y semitransparente de controles -->
  <div id="controlsBar"
       class="fixed bottom-4 left-1/2 -translate-x-1/2 z-20 flex items-center gap-1.5 sm:gap-2 px-2.5 py-2 rounded-2xl bg-zinc-900/65 border border-white/10 backdrop-blur-md shadow-2xl transition-all duration-300 max-w-[94vw]">

    <!-- Botón Pausar / Reanudar -->
    <button id="btnTogglePlay"
            type="button"
            class="flex items-center gap-1.5 px-3 py-2.5 sm:py-2 rounded-xl text-xs sm:text-sm font-medium text-zinc-100 bg-white/5 hover:bg-white/15 active:scale-95 transition"
            aria-label="Pausar o reanudar giro">
      <svg id="iconPause" class="w-4 h-4 shrink-0" viewBox="0 0 24 24" fill="currentColor">
        <rect x="6" y="4" width="4" height="16" rx="1.2"></rect>
        <rect x="14" y="4" width="4" height="16" rx="1.2"></rect>
      </svg>
      <svg id="iconPlay" class="w-4 h-4 shrink-0 hidden" viewBox="0 0 24 24" fill="currentColor">
        <path d="M8 5.14v14l11-7-11-7z"></path>
      </svg>
      <span id="labelTogglePlay">Pausar</span>
    </button>

    <!-- Botón Velocidad (1x / 2x / 0.5x) -->
    <button id="btnSpeed"
            type="button"
            class="flex items-center gap-1.5 px-3 py-2.5 sm:py-2 rounded-xl text-xs sm:text-sm font-medium text-zinc-200 bg-white/5 hover:bg-white/15 active:scale-95 transition font-mono-terminal"
            aria-label="Cambiar velocidad de rotación">
      <svg class="w-4 h-4 shrink-0 text-rose-400" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <circle cx="12" cy="12" r="9"></circle>
        <polyline points="12 7 12 12 15.5 14"></polyline>
      </svg>
      <span id="labelSpeed">1x</span>
    </button>

    <!-- Botón Editar Mensaje -->
    <button id="btnEditMessage"
            type="button"
            class="flex items-center gap-1.5 px-3 py-2.5 sm:py-2 rounded-xl text-xs sm:text-sm font-medium text-zinc-200 bg-white/5 hover:bg-white/15 active:scale-95 transition"
            aria-label="Editar mensaje personalizado">
      <svg class="w-4 h-4 shrink-0 text-amber-300" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <path d="M12 20h9"></path>
        <path d="M16.5 3.5a2.121 2.121 0 0 1 3 3L7 19l-4 1 1-4 12.5-12.5z"></path>
      </svg>
      <span class="hidden xs:inline sm:inline">Mensaje</span>
    </button>

    <div class="w-px h-5 bg-white/10 mx-0.5"></div>

    <!-- Botón Ocultar controles (pantalla limpia) -->
    <button id="btnHideControls"
            type="button"
            class="p-2.5 sm:p-2 rounded-xl text-zinc-400 hover:text-zinc-100 hover:bg-white/10 active:scale-95 transition"
            title="Ocultar controles"
            aria-label="Ocultar controles">
      <svg class="w-4 h-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <path d="M17.94 17.94A10.07 10.07 0 0 1 12 20c-7 0-11-8-11-8a18.45 18.45 0 0 1 5.06-5.94"></path>
        <path d="M9.9 4.24A9.12 9.12 0 0 1 12 4c7 0 11 8 11 8a18.5 18.5 0 0 1-2.16 3.19"></path>
        <line x1="1" y1="1" x2="23" y2="23"></line>
      </svg>
    </button>
  </div>

  <!-- Botón flotante ultra discreto para volver a mostrar los controles -->
  <button id="btnShowControls"
          type="button"
          class="hidden fixed bottom-4 right-4 z-20 p-3 rounded-full bg-white/5 hover:bg-white/15 border border-white/10 backdrop-blur-md text-zinc-400 hover:text-zinc-100 active:scale-95 transition shadow-lg"
          title="Mostrar controles"
          aria-label="Mostrar controles">
    <svg class="w-4 h-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
      <path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"></path>
      <circle cx="12" cy="12" r="3"></circle>
    </svg>
  </button>

  <!-- Modal minimalista para editar el mensaje -->
  <div id="messageModal"
       class="hidden fixed inset-0 z-30 flex items-center justify-center p-4 bg-black/70 backdrop-blur-sm">
    <div class="w-full max-w-sm rounded-2xl bg-zinc-900/95 border border-white/10 p-5 shadow-2xl">
      <h2 class="text-base font-semibold text-zinc-100 mb-1">Mensaje del ramo</h2>
      <p class="text-xs text-zinc-400 mb-4">Escribe una dedicatoria corta o déjalo vacío para ocultarlo.</p>

      <input id="messageInput"
             type="text"
             maxlength="60"
             placeholder="Ej. Para ti ✨"
             class="w-full px-3.5 py-2.5 rounded-xl bg-black/50 border border-white/15 text-sm text-zinc-100 placeholder-zinc-500 focus:outline-none focus:border-rose-400/70 font-mono-terminal mb-4" />

      <div class="flex items-center justify-end gap-2">
        <button id="btnCancelModal"
                type="button"
                class="px-3.5 py-2 rounded-xl text-xs font-medium text-zinc-400 hover:text-zinc-100 hover:bg-white/5 transition">
          Cancelar
        </button>
        <button id="btnClearMessage"
                type="button"
                class="px-3.5 py-2 rounded-xl text-xs font-medium text-rose-300 hover:bg-rose-500/15 transition">
          Quitar
        </button>
        <button id="btnSaveMessage"
                type="button"
                class="px-4 py-2 rounded-xl text-xs font-semibold text-white bg-rose-500 hover:bg-rose-400 active:scale-95 transition">
          Guardar
        </button>
      </div>
    </div>
  </div>

  <script>
    (() => {
      "use strict";

      /* =====================================================================
         1. UTILIDADES MATEMÁTICAS Y PRNG DETERMINISTA (SEMILLA 21)
         ===================================================================== */
      function createRng(seed) {
        let a = seed >>> 0;
        return function () {
          a |= 0;
          a = (a + 0x6D2B79F5) | 0;
          let t = Math.imul(a ^ (a >>> 15), 1 | a);
          t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
          return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
        };
      }

      function randUniform(rng, min, max) {
        return min + (max - min) * rng();
      }

      function addVec(p, d, k = 1.0) {
        return [p[0] + d[0] * k, p[1] + d[1] * k, p[2] + d[2] * k];
      }

      function crossVec(a, b) {
        return [
          a[1] * b[2] - a[2] * b[1],
          a[2] * b[0] - a[0] * b[2],
          a[0] * b[1] - a[1] * b[0],
        ];
      }

      function normVec(v) {
        const m = Math.hypot(v[0], v[1], v[2]) || 1e-9;
        return [v[0] / m, v[1] / m, v[2] / m];
      }

      function buildBasis(axis) {
        const w = normVec(axis);
        const helper = Math.abs(w[1]) < 0.9 ? [0.0, 1.0, 0.0] : [1.0, 0.0, 0.0];
        const u = normVec(crossVec(helper, w));
        const v = crossVec(w, u);
        return { u, v, w };
      }

      /* =====================================================================
         2. CONSTRUCCIÓN DE LA NUBE DE PUNTOS 3D DEL RAMO
         ===================================================================== */
      function buildBouquetScene() {
        const rng = createRng(21);
        const rawPoints = []; // Acumulador temporal: [x, y, z, nx, ny, nz, r, g, b]

        const pushPoint = (p, n, r, g, b) => {
          rawPoints.push(
            p[0], p[1], p[2],
            n[0], n[1], n[2],
            Math.min(255, Math.max(0, r | 0)),
            Math.min(255, Math.max(0, g | 0)),
            Math.min(255, Math.max(0, b | 0))
          );
        };

        // --- Rosa: centro dorado + 4 capas de pétalos en espiral ---
        function addRose(center, axis, radius, color) {
          const { u, v, w } = buildBasis(axis);

          // Centro dorado
          for (let i = 0; i < 24; i++) {
            const a = randUniform(rng, 0, Math.PI * 2);
            const r = randUniform(rng, 0, radius * 0.24);
            const p = addVec(addVec(center, u, r * Math.cos(a)), v, r * Math.sin(a));
            pushPoint(p, w, 255, 212, 92);
          }

          // 4 capas de pétalos en espiral
          const layers = 4;
          for (let layer = 0; layer < layers; layer++) {
            const t = (layer + 1) / layers;
            const nPetals = 5 + layer * 2;
            const rLayer = radius * (0.34 + 0.66 * t);
            const tilt = 0.24 + 0.56 * t;

            for (let k = 0; k < nPetals; k++) {
              const baseAng = (Math.PI * 2 * k) / nPetals + layer * 0.45;
              for (let s = 1; s <= 7; s++) {
                const frac = s / 7.0;
                for (let sp = -2; sp <= 2; sp++) {
                  const spread = sp * 0.16;
                  const ang = baseAng + spread * (1.0 - 0.3 * frac);
                  const rad = rLayer * frac;
                  const height = Math.sin(frac * Math.PI * 0.85) * radius * 0.46 * (1.15 - t * 0.4);

                  const ca = Math.cos(ang);
                  const sa = Math.sin(ang);
                  const dirR = [
                    u[0] * ca + v[0] * sa,
                    u[1] * ca + v[1] * sa,
                    u[2] * ca + v[2] * sa,
                  ];

                  const p = addVec(addVec(center, dirR, rad), w, height - tilt * rad * 0.2);
                  const n = normVec(addVec(dirR, w, 0.82));
                  const shade = 0.72 + 0.28 * frac;
                  pushPoint(p, n, color[0] * shade, color[1] * shade, color[2] * shade);
                }
              }
            }
          }
        }

        // --- Tulipán: 6 pétalos en forma de copa ---
        function addTulip(center, axis, radius, color) {
          const { u, v, w } = buildBasis(axis);

          for (let k = 0; k < 6; k++) {
            const baseAng = (Math.PI * 2 * k) / 6.0;
            for (let hStep = 1; hStep <= 9; hStep++) {
              const hf = hStep / 9.0;
              const cupR = radius * 0.72 * Math.sin(hf * Math.PI * 0.78 + 0.2);
              const height = (hf - 0.2) * radius * 1.48;

              for (let sp = -2; sp <= 2; sp++) {
                const spread = sp * 0.175;
                const ang = baseAng + spread * Math.sin(hf * Math.PI);
                const ca = Math.cos(ang);
                const sa = Math.sin(ang);
                const dirR = [
                  u[0] * ca + v[0] * sa,
                  u[1] * ca + v[1] * sa,
                  u[2] * ca + v[2] * sa,
                ];

                const p = addVec(addVec(center, dirR, cupR), w, height);
                const n = normVec(addVec(dirR, w, 0.36));
                const shade = 0.7 + 0.3 * hf;
                pushPoint(p, n, color[0] * shade, color[1] * shade, color[2] * shade);
              }
            }
          }
        }

        // --- Tallo verde con curva de Bézier cuadrática ---
        function addStem(start, neck, end) {
          const steps = 46;
          for (let i = 0; i <= steps; i++) {
            const t = i / steps;
            const inv = 1.0 - t;
            const x = inv * inv * start[0] + 2 * inv * t * neck[0] + t * t * end[0];
            const y = inv * inv * start[1] + 2 * inv * t * neck[1] + t * t * end[1];
            const z = inv * inv * start[2] + 2 * inv * t * neck[2] + t * t * end[2];
            const n = normVec([x, 0.22, z]);
            const col = t < 0.65 ? [48, 185, 82] : [34, 138, 62];
            pushPoint([x, y, z], n, col[0], col[1], col[2]);

            // Pequeño grosor adicional para nitidez en pantallas móviles
            pushPoint([x + 0.012, y, z], n, col[0] * 0.92, col[1] * 0.92, col[2] * 0.92);
          }
        }

        // --- Hoja verde arqueada ---
        function addLeaf(origin, direction, length, width) {
          const { u, v, w } = buildBasis(direction);
          for (let i = 1; i <= 15; i++) {
            const t = i / 15.0;
            const halfW = width * Math.sin(t * Math.PI);
            const arch = -0.18 * length * (t * t);

            for (let j = -3; j <= 3; j++) {
              const s = j / 3.0;
              let p = addVec(origin, w, t * length);
              p = addVec(p, u, s * halfW);
              p = addVec(p, v, arch);
              const n = normVec(addVec(v, w, 0.32));
              const shade = 0.76 + 0.24 * (1.0 - Math.abs(s));
              pushPoint(p, n, 42 * shade, 200 * shade, 88 * shade);
            }
          }
        }

        // --- Cinta roja/fucsia envolviendo el cuello + lazos sutiles ---
        function addRibbon(neck, radius = 0.135) {
          for (let layer = -2; layer <= 2; layer++) {
            const y = neck[1] + layer * 0.024;
            for (let i = 0; i < 36; i++) {
              const ang = (Math.PI * 2 * i) / 36.0;
              const ca = Math.cos(ang);
              const sa = Math.sin(ang);
              const p = [neck[0] + radius * ca, y, neck[2] + radius * sa];
              const n = normVec([ca, 0.15, sa]);
              pushPoint(p, n, 240, 58, 102);
            }
          }

          // Dos colas cortas de cinta cayendo suavemente desde el frente del nudo
          for (const side of [-1, 1]) {
            for (let i = 1; i <= 16; i++) {
              const t = i / 16.0;
              const x = neck[0] + side * (0.04 + 0.14 * t);
              const y = neck[1] - 0.04 - 0.26 * t;
              const z = neck[2] + radius + 0.03 * Math.sin(t * Math.PI);
              const n = normVec([side * 0.2, 0.2, 1.0]);
              pushPoint([x, y, z], n, 245, 70, 112);
              pushPoint([x + side * 0.016, y, z], n, 215, 48, 88);
            }
          }
        }

        // Configuración del ramo
        const neck = [0.0, -0.18, 0.0];
        const centerDome = [0.0, 0.16, 0.0];
        const domeR = 0.62;
        const goldenAngle = Math.PI * (3.0 - Math.sqrt(5.0));

        // Paleta vibrante especificada
        const palette = [
          [255, 70, 105],   // #ff4669 Rosa fuerte
          [255, 110, 60],   // #ff6e3c Naranja coral
          [230, 60, 70],    // #e63c46 Rojo
          [255, 195, 65],   // #ffc341 Amarillo dorado
          [195, 105, 255],  // #c369ff Lavanda
          [255, 130, 185],  // #ff82b9 Rosa pastel
          [255, 85, 130],   // #ff5582 Fucsia
          [255, 150, 75],   // #ff964b Durazno
          [240, 70, 90],    // #f0465a Carmesí
        ];

        const tulipSlots = new Set([1, 4, 7]);
        const nFlowers = 9;

        // 9 flores distribuidas en cúpula con el ángulo áureo
        for (let i = 0; i < nFlowers; i++) {
          const el = ((82 - 52 * Math.sqrt(i / (nFlowers - 1))) * Math.PI) / 180;
          const az = i * goldenAngle;
          const dir = [
            Math.cos(el) * Math.cos(az),
            Math.sin(el),
            Math.cos(el) * Math.sin(az),
          ];

          const pos = addVec(centerDome, dir, domeR * randUniform(rng, 0.92, 1.06));
          const axis = normVec(dir);
          const color = palette[i % palette.length];
          const radius = randUniform(rng, 0.21, 0.265);

          if (tulipSlots.has(i)) {
            addTulip(pos, axis, radius, color);
          } else {
            addRose(pos, axis, radius, color);
          }

          // Tallo desde la flor pasando por el cuello y abriéndose en la base
          const end = [
            neck[0] - dir[0] * 0.24,
            neck[1] - 0.54,
            neck[2] - dir[2] * 0.24,
          ];
          addStem(pos, neck, end);
        }

        // 10 hojas verdes arqueadas alrededor del ramo
        for (let i = 0; i < 10; i++) {
          const ang = (Math.PI * 2 * i) / 10.0 + 0.2;
          const leafDir = [
            Math.cos(ang) * 0.85,
            0.35 + (i % 2) * 0.2,
            Math.sin(ang) * 0.85,
          ];
          const origin = [neck[0], neck[1] + 0.12, neck[2]];
          addLeaf(origin, leafDir, 0.68, 0.145);
        }

        // Cinta envolviendo el cuello
        addRibbon(neck, 0.135);

        // Empaquetamos en TypedArrays para máximo rendimiento (60 FPS en móvil)
        const count = rawPoints.length / 9;
        const px = new Float32Array(count);
        const py = new Float32Array(count);
        const pz = new Float32Array(count);
        const nx = new Float32Array(count);
        const ny = new Float32Array(count);
        const nz = new Float32Array(count);
        const cr = new Uint8Array(count);
        const cg = new Uint8Array(count);
        const cb = new Uint8Array(count);

        for (let i = 0; i < count; i++) {
          const o = i * 9;
          px[i] = rawPoints[o];
          py[i] = rawPoints[o + 1];
          pz[i] = rawPoints[o + 2];
          nx[i] = rawPoints[o + 3];
          ny[i] = rawPoints[o + 4];
          nz[i] = rawPoints[o + 5];
          cr[i] = rawPoints[o + 6];
          cg[i] = rawPoints[o + 7];
          cb[i] = rawPoints[o + 8];
        }

        return { count, px, py, pz, nx, ny, nz, cr, cg, cb };
      }

      /* =====================================================================
         3. MOTOR DE RENDERIZADO ASCII 3D EN CANVAS
         ===================================================================== */
      const canvas = document.getElementById("asciiCanvas");
      const ctx = canvas.getContext("2d", { alpha: false });
      const scene = buildBouquetScene();

      const RAMP = " .:-=+*#%@";
      const RAMP_MAX = RAMP.length - 1;
      const LIGHT = normVec([0.45, 0.75, -0.65]);

      // Estado de pantalla y grilla ASCII
      let viewW = window.innerWidth;
      let viewH = window.innerHeight;
      let dpr = Math.min(window.devicePixelRatio || 1, 2);
      let cellW = 7;
      let cellH = 12;
      let fontSize = 11;
      let cols = 80;
      let rows = 50;
      let zbuf = new Float32Array(1);
      let charBuf = new Int8Array(1);
      let rBuf = new Uint8Array(1);
      let gBuf = new Uint8Array(1);
      let bBuf = new Uint8Array(1);

      // Estado de cámara y rotación interactiva
      let angleY = 0.0;
      let angleX = 0.28; // Inclinación inicial para apreciar las flores
      let velY = 0.0;
      let velX = 0.0;
      let isDragging = false;
      let lastPtrX = 0;
      let lastPtrY = 0;
      let isPlaying = true;

      const SPEEDS = [
        { label: "1x", value: 0.75 },
        { label: "2x", value: 1.55 },
        { label: "0.5x", value: 0.35 },
      ];
      let speedIdx = 0;

      // Recalcula dimensiones del canvas y tamaño de celda ASCII según el dispositivo
      function handleResize() {
        viewW = window.innerWidth;
        viewH = window.innerHeight;
        dpr = Math.min(window.devicePixelRatio || 1, 2);

        canvas.width = Math.floor(viewW * dpr);
        canvas.height = Math.floor(viewH * dpr);
        canvas.style.width = viewW + "px";
        canvas.style.height = viewH + "px";

        ctx.setTransform(dpr, 0, 0, dpr, 0, 0);

        // Tamaño de fuente responsivo: fino y nítido en celular, generoso en escritorio
        const minSide = Math.min(viewW, viewH);
        fontSize = Math.max(9, Math.min(13, Math.round(minSide / 58)));
        cellH = Math.round(fontSize * 1.12);
        cellW = Math.max(5.5, fontSize * 0.62);

        cols = Math.max(36, Math.floor(viewW / cellW));
        rows = Math.max(24, Math.floor(viewH / cellH));

        const total = cols * rows;
        if (zbuf.length !== total) {
          zbuf = new Float32Array(total);
          charBuf = new Int8Array(total);
          rBuf = new Uint8Array(total);
          gBuf = new Uint8Array(total);
          bBuf = new Uint8Array(total);
        }

        ctx.font = `600 ${fontSize}px 'JetBrains Mono', ui-monospace, monospace`;
        ctx.textBaseline = "top";
        ctx.textAlign = "left";
      }

      window.addEventListener("resize", handleResize);
      handleResize();

      /* =====================================================================
         4. INTERACCIÓN TÁCTIL Y DE MOUSE (CON INERCIA SUAVE)
         ===================================================================== */
      const dragHint = document.getElementById("dragHint");

      function hideHintNow() {
        if (dragHint && dragHint.style.display !== "none") {
          dragHint.style.opacity = "0";
          dragHint.style.display = "none";
        }
      }

      canvas.addEventListener("pointerdown", (e) => {
        isDragging = true;
        lastPtrX = e.clientX;
        lastPtrY = e.clientY;
        velY = 0;
        velX = 0;
        canvas.setPointerCapture(e.pointerId);
        hideHintNow();
      });

      canvas.addEventListener("pointermove", (e) => {
        if (!isDragging) return;
        const dx = e.clientX - lastPtrX;
        const dy = e.clientY - lastPtrY;
        lastPtrX = e.clientX;
        lastPtrY = e.clientY;

        const sens = 0.0085;
        angleY += dx * sens;
        angleX = Math.max(-0.85, Math.min(1.15, angleX + dy * sens));

        // Guardamos velocidad para la inercia al soltar
        velY = dx * sens * 0.85;
        velX = dy * sens * 0.85;
      });

      function endDrag(e) {
        if (!isDragging) return;
        isDragging = false;
        try {
          canvas.releasePointerCapture(e.pointerId);
        } catch (_) {
          // Ignorar si ya fue liberado
        }
      }

      canvas.addEventListener("pointerup", endDrag);
      canvas.addEventListener("pointercancel", endDrag);

      /* =====================================================================
         5. BUCLE PRINCIPAL DE ANIMACIÓN (60 FPS)
         ===================================================================== */
      let lastTime = performance.now();

      function renderFrame(now) {
        const dt = Math.min(0.05, (now - lastTime) / 1000);
        lastTime = now;

        // Actualizar ángulos con inercia y rotación automática
        if (!isDragging) {
          angleY += velY;
          angleX = Math.max(-0.85, Math.min(1.15, angleX + velX));

          // Fricción de inercia
          const damping = Math.pow(0.92, dt * 60);
          velY *= damping;
          velX *= damping;
          if (Math.abs(velY) < 0.0002) velY = 0;
          if (Math.abs(velX) < 0.0002) velX = 0;

          if (isPlaying) {
            angleY += SPEEDS[speedIdx].value * dt;
          }
        }

        // Limpiar fondo negro estilo terminal
        ctx.fillStyle = "#08080c";
        ctx.fillRect(0, 0, viewW, viewH);

        // Limpiar buffers
        zbuf.fill(-1e9);
        charBuf.fill(-1);

        const cosY = Math.cos(angleY);
        const sinY = Math.sin(angleY);
        const cosX = Math.cos(angleX);
        const sinX = Math.sin(angleX);

        // Escala focal responsiva para centrar y ajustar bien en móvil y escritorio
        const usableH = viewH * 0.84;
        const focalPx = Math.min(viewW * 1.18, usableH * 1.12);
        const camDist = 2.65;

        const centerCol = cols * 0.5;
        const centerRow = rows * 0.48; // Ligeramente elevado para dejar espacio al mensaje y controles
        const invCellW = 1.0 / cellW;
        const invCellH = 1.0 / cellH;

        const { count, px, py, pz, nx, ny, nz, cr, cg, cb } = scene;
        const lx = LIGHT[0];
        const ly = LIGHT[1];
        const lz = LIGHT[2];

        // Proyección 3D + Z-Buffer + Iluminación direccional
        for (let i = 0; i < count; i++) {
          const x = px[i];
          const y = py[i];
          const z = pz[i];

          // Rotación en Y
          const rx = x * cosY + z * sinY;
          const rz = -x * sinY + z * cosY;

          // Inclinación en X
          const ry2 = y * cosX - rz * sinX;
          const rz2 = y * sinX + rz * cosX;

          const dist = camDist - rz2;
          if (dist <= 0.25) continue;
          const invZ = 1.0 / dist;

          const sx = (centerCol + rx * invZ * focalPx * invCellW) | 0;
          const sy = (centerRow - ry2 * invZ * focalPx * invCellH) | 0;

          if (sx >= 0 && sx < cols && sy >= 0 && sy < rows) {
            const idx = sy * cols + sx;
            if (invZ > zbuf[idx]) {
              zbuf[idx] = invZ;

              // Rotar normal para calcular iluminación
              const xN = nx[i];
              const yN = ny[i];
              const zN = nz[i];
              const rnx = xN * cosY + zN * sinY;
              const rnz = -xN * sinY + zN * cosY;
              const rny2 = yN * cosX - rnz * sinX;
              const rnz2 = yN * sinX + rnz * cosX;

              let lum = (rnx * lx + rny2 * ly + rnz2 * lz) * 0.72 + 0.38;
              if (lum < 0.2) lum = 0.2;
              else if (lum > 1.0) lum = 1.0;

              charBuf[idx] = (lum * RAMP_MAX) | 0;
              rBuf[idx] = (cr[i] * lum) | 0;
              gBuf[idx] = (cg[i] * lum) | 0;
              bBuf[idx] = (cb[i] * lum) | 0;
            }
          }
        }

        // Dibujar caracteres ASCII en el canvas
        const offsetX = (viewW - cols * cellW) * 0.5;
        const offsetY = (viewH - rows * cellH) * 0.5;

        for (let r = 0; r < rows; r++) {
          const rowStart = r * cols;
          const pyDraw = offsetY + r * cellH;

          for (let c = 0; c < cols; c++) {
            const idx = rowStart + c;
            const chIdx = charBuf[idx];
            if (chIdx <= 0) continue; // Saltar espacios vacíos

            ctx.fillStyle = `rgb(${rBuf[idx]},${gBuf[idx]},${bBuf[idx]})`;
            ctx.fillText(RAMP[chIdx], offsetX + c * cellW, pyDraw);
          }
        }

        requestAnimationFrame(renderFrame);
      }

      /* =====================================================================
         6. CONTROLES MINIMALISTAS Y EDITOR DE MENSAJE
         ===================================================================== */
      const btnTogglePlay = document.getElementById("btnTogglePlay");
      const iconPause = document.getElementById("iconPause");
      const iconPlay = document.getElementById("iconPlay");
      const labelTogglePlay = document.getElementById("labelTogglePlay");

      const btnSpeed = document.getElementById("btnSpeed");
      const labelSpeed = document.getElementById("labelSpeed");

      const btnEditMessage = document.getElementById("btnEditMessage");
      const btnHideControls = document.getElementById("btnHideControls");
      const btnShowControls = document.getElementById("btnShowControls");
      const controlsBar = document.getElementById("controlsBar");

      const messageContainer = document.getElementById("messageContainer");
      const messageDisplay = document.getElementById("messageDisplay");
      const messageModal = document.getElementById("messageModal");
      const messageInput = document.getElementById("messageInput");
      const btnCancelModal = document.getElementById("btnCancelModal");
      const btnClearMessage = document.getElementById("btnClearMessage");
      const btnSaveMessage = document.getElementById("btnSaveMessage");

      // Leer mensaje inicial opcional desde parámetro URL (?msg=Hola)
      try {
        const params = new URLSearchParams(window.location.search);
        const urlMsg = params.get("msg");
        if (urlMsg !== null) {
          applyMessage(urlMsg.slice(0, 60));
        }
      } catch (_) {
        // Ignorar si el entorno restringe URLSearchParams
      }

      function applyMessage(text) {
        const clean = text.trim();
        messageDisplay.textContent = clean;
        if (clean.length === 0) {
          messageContainer.classList.add("opacity-0");
        } else {
          messageContainer.classList.remove("opacity-0");
        }
      }

      // Pausar / Reanudar
      btnTogglePlay.addEventListener("click", () => {
        isPlaying = !isPlaying;
        iconPause.classList.toggle("hidden", !isPlaying);
        iconPlay.classList.toggle("hidden", isPlaying);
        labelTogglePlay.textContent = isPlaying ? "Pausar" : "Girar";
      });

      // Cambiar velocidad (1x / 2x / 0.5x)
      btnSpeed.addEventListener("click", () => {
        speedIdx = (speedIdx + 1) % SPEEDS.length;
        labelSpeed.textContent = SPEEDS[speedIdx].label;
      });

      // Ocultar / Mostrar controles para vista a pantalla completa
      btnHideControls.addEventListener("click", () => {
        controlsBar.classList.add("opacity-0", "pointer-events-none", "translate-y-4");
        btnShowControls.classList.remove("hidden");
        messageContainer.classList.remove("bottom-24", "sm:bottom-24");
        messageContainer.classList.add("bottom-10");
      });

      btnShowControls.addEventListener("click", () => {
        controlsBar.classList.remove("opacity-0", "pointer-events-none", "translate-y-4");
        btnShowControls.classList.add("hidden");
        messageContainer.classList.remove("bottom-10");
        messageContainer.classList.add("bottom-24", "sm:bottom-24");
      });

      // Abrir y cerrar modal de mensaje
      function openModal() {
        messageInput.value = messageDisplay.textContent;
        messageModal.classList.remove("hidden");
        setTimeout(() => messageInput.focus(), 50);
      }

      function closeModal() {
        messageModal.classList.add("hidden");
      }

      btnEditMessage.addEventListener("click", openModal);
      btnCancelModal.addEventListener("click", closeModal);

      btnClearMessage.addEventListener("click", () => {
        applyMessage("");
        closeModal();
      });

      btnSaveMessage.addEventListener("click", () => {
        applyMessage(messageInput.value);
        closeModal();
      });

      messageInput.addEventListener("keydown", (e) => {
        if (e.key === "Enter") {
          applyMessage(messageInput.value);
          closeModal();
        } else if (e.key === "Escape") {
          closeModal();
        }
      });

      messageModal.addEventListener("click", (e) => {
        if (e.target === messageModal) closeModal();
      });

      // Iniciar animación al cargar la ventana
      window.onload = function () {
        handleResize();
        lastTime = performance.now();
        requestAnimationFrame(renderFrame);
      };
    })();
  </script>
</body>
</html>
