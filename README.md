<!DOCTYPE html>
<html lang="en">
<head>
  <!--
    Web Page with Animated Background
    - HTML5 + CSS3 + ES2021
    - Responsive, accessible, and performant
    - Animated gradient + GPU-friendly glow + Canvas "orbs" particles
    - Honors reduced-motion preferences and pauses when tab is hidden
    - Production-ready single-file example

    Performance Notes:
    - Uses transform/background-position animations (GPU-accelerated) for the gradient layer.
    - Canvas animation uses requestAnimationFrame with dynamic throttling and DPR-aware scaling.
    - Particle count scales with viewport size; animation auto-pauses on visibility change and when reduced motion is enabled.
    - Resize is debounced to avoid layout thrash.
  -->
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Animated Background Demo</title>

  <style>
    /* -----------------------------
       CSS Custom Properties & Reset
       ----------------------------- */
    :root {
      /* Base scale + fluid typography */
      --step-0: clamp(1rem, 0.95rem + 0.5vw, 1.25rem);
      --step-1: clamp(1.25rem, 1.1rem + 1vw, 2rem);
      --step-2: clamp(2rem, 1.6rem + 2vw, 3.25rem);

      /* Color system (auto adapts to color scheme) */
      color-scheme: light dark;
      --bg: #0e0f14;
      --fg: #e9eef5;
      --muted: color-mix(in oklab, var(--fg) 70%, transparent);

      /* Accent gradient stops */
      --c1: #6ea8fe;
      --c2: #9a6bff;
      --c3: #00d1b2;
      --c4: #ff6b6b;

      /* Layout */
      --radius: 14px;
      --shadow: 0 10px 30px hsl(0 0% 0% / 0.35);

      /* Animation controls */
      --gradient-speed: 36s;
      --glow-speed: 14s;
    }

    /* A modest modern reset */
    *, *::before, *::after { box-sizing: border-box; }
    html, body { height: 100%; }
    body {
      margin: 0;
      font: 400 var(--step-0)/1.5 system-ui, -apple-system, Segoe UI, Roboto, "Helvetica Neue", Arial, "Noto Sans", "Apple Color Emoji", "Segoe UI Emoji", "Segoe UI Symbol";
      color: var(--fg);
      background: var(--bg);
      overflow-x: hidden;
      -webkit-font-smoothing: antialiased;
      -moz-osx-font-smoothing: grayscale;
    }

    /* -----------------------------
       Layered Animated Background
       ----------------------------- */

    /* Gradient layer: big soft animated background */
    .bg-gradient {
      position: fixed;
      inset: 0;
      z-index: -3;
      /* Oversize background to allow smooth pan without hard edges */
      background-image:
        radial-gradient(60% 60% at 20% 20%, color-mix(in oklab, var(--c1) 35%, transparent) 0%, transparent 60%),
        radial-gradient(50% 50% at 80% 30%, color-mix(in oklab, var(--c2) 35%, transparent) 0%, transparent 60%),
        radial-gradient(70% 70% at 30% 80%, color-mix(in oklab, var(--c3) 35%, transparent) 0%, transparent 70%),
        radial-gradient(60% 60% at 80% 80%, color-mix(in oklab, var(--c4) 35%, transparent) 0%, transparent 65%),
        linear-gradient(135deg, #0a0b10, #11131a 40%, #0b0d12);
      background-size: 200% 200%, 220% 220%, 180% 180%, 200% 200%, 100% 100%;
      background-position: 0% 0%, 100% 0%, 0% 100%, 100% 100%, center;
      animation: drift var(--gradient-speed) linear infinite alternate;
      will-change: background-position, transform;
    }

    @keyframes drift {
      0%   { background-position: 0% 0%, 100% 0%, 0% 100%, 100% 100%, center; filter: saturate(1) hue-rotate(0deg); }
      50%  { background-position: 50% 50%, 60% 40%, 40% 60%, 70% 80%, center; filter: saturate(1.05) hue-rotate(10deg); }
      100% { background-position: 100% 100%, 0% 100%, 100% 0%, 0% 0%, center; filter: saturate(1) hue-rotate(-10deg); }
    }

    /* Glow layer: subtle moving bloom for depth */
    .bg-glow {
      position: fixed;
      inset: -10vmax; /* slightly larger to avoid clipping */
      z-index: -2;
      background:
        radial-gradient(closest-side, color-mix(in oklab, var(--c2) 20%, transparent), transparent 70%) 20% 30%/35vmax 35vmax no-repeat,
        radial-gradient(closest-side, color-mix(in oklab, var(--c3) 16%, transparent), transparent 70%) 80% 60%/45vmax 45vmax no-repeat,
        radial-gradient(closest-side, color-mix(in oklab, var(--c1) 14%, transparent), transparent 70%) 60% 20%/30vmax 30vmax no-repeat;
      filter: blur(40px) saturate(1.1);
      animation: floatGlow var(--glow-speed) ease-in-out infinite alternate;
      mix-blend-mode: screen;
      pointer-events: none;
    }

    @keyframes floatGlow {
      from { transform: translate3d(-1%, -1%, 0) scale(1); }
      to   { transform: translate3d(1%, 1%, 0) scale(1.02); }
    }

    /* Canvas particle layer sits above gradient/glow but below content */
    #bg-canvas {
      position: fixed;
      inset: 0;
      z-index: -1;
      display: block;
      pointer-events: none;
      /* Slight transparency to let gradient show through */
      opacity: 0.65;
    }

    /* -----------------------------
       Foreground Content
       ----------------------------- */
    .page {
      min-height: 100dvh;
      display: grid;
      place-items: center;
      padding: 6vmin;
    }

    .card {
      width: min(900px, 92vw);
      background: color-mix(in oklab, var(--bg) 70%, transparent);
      backdrop-filter: blur(10px) saturate(1.1);
      -webkit-backdrop-filter: blur(10px) saturate(1.1);
      border: 1px solid hsl(0 0% 100% / 0.06);
      box-shadow: var(--shadow);
      border-radius: var(--radius);
      padding: clamp(1rem, 3vmin, 2rem);
      outline: 0;
    }

    .eyebrow {
      font-size: 0.85em;
      letter-spacing: 0.18em;
      text-transform: uppercase;
      color: var(--muted);
      margin-bottom: 0.5rem;
    }

    h1 {
      margin: 0 0 0.5rem;
      font-size: var(--step-2);
      line-height: 1.1;
      letter-spacing: -0.01em;
      text-wrap: balance;
    }

    p.lead {
      margin: 0 0 1.25rem;
      font-size: var(--step-0);
      color: var(--muted);
      max-width: 70ch;
      text-wrap: pretty;
    }

    .cta-row {
      display: flex;
      flex-wrap: wrap;
      gap: 0.75rem;
      margin-top: 1rem;
    }

    .btn {
      --btn-bg: color-mix(in oklab, var(--c1) 25%, #1a1d26);
      --btn-fg: white;
      appearance: none;
      border: 0;
      color: var(--btn-fg);
      background: linear-gradient(180deg, color-mix(in oklab, var(--btn-bg) 70%, white 6%), var(--btn-bg)) padding-box,
                  linear-gradient(180deg, hsl(0 0% 100% / 0.2), hsl(0 0% 0% / 0.2)) border-box;
      border: 1px solid transparent;
      border-radius: calc(var(--radius) - 6px);
      padding: 0.8rem 1.1rem;
      font-weight: 600;
      letter-spacing: 0.02em;
      cursor: pointer;
      transition: transform 0.15s ease, filter 0.2s ease, background-position 0.2s ease;
      background-size: 200% 200%, 100% 100%;
      background-position: 0% 0%;
      text-decoration: none;
      display: inline-flex;
      align-items: center;
      gap: 0.6rem;
    }
    .btn:focus-visible {
      outline: 3px solid color-mix(in oklab, var(--c3) 60%, white 20%);
      outline-offset: 2px;
    }
    .btn:hover { transform: translateY(-1px); background-position: 0% 100%; }
    .btn:active { transform: translateY(0); filter: brightness(0.95); }

    .btn.secondary {
      --btn-bg: color-mix(in oklab, var(--c3) 22%, #1a1d26);
    }

    /* Footer meta */
    .meta {
      margin-top: 1.25rem;
      font-size: 0.9em;
      color: var(--muted);
    }

    /* -----------------------------
       Reduced Motion & Prefs
       ----------------------------- */
    @media (prefers-reduced-motion: reduce) {
      .bg-gradient,
      .bg-glow {
        animation: none;
      }
      #bg-canvas {
        display: none; /* Canvas not necessary when motion is reduced */
      }
    }

    /* Safe fallback for older browsers lacking color-mix/oklab */
    @supports not (color: oklab(0 0 0)) {
      :root {
        --bg: #101218;
        --fg: #e9eef5;
        --c1: #6ea8fe;
        --c2: #9a6bff;
        --c3: #00d1b2;
        --c4: #ff6b6b;
      }
    }
  </style>
</head>
<body>
  <!-- Decorative animated layers (aria-hidden to avoid noise for AT users) -->
  <div class="bg-gradient" aria-hidden="true"></div>
  <div class="bg-glow" aria-hidden="true"></div>
  <canvas id="bg-canvas" aria-hidden="true"></canvas>

  <!-- Foreground content -->
  <main class="page" id="main">
    <section class="card" role="region" aria-labelledby="hero-title">
      <p class="eyebrow">Demo</p>
      <h1 id="hero-title">Animated Background, Zero Fuss</h1>
      <p class="lead">
        This page showcases a layered animated background using a GPU-friendly gradient,
        soft glows, and a performance-tuned canvas particle system. It adapts to motion settings,
        color scheme, and viewport size.
      </p>
      <div class="cta-row">
        <a class="btn" href="#learn" aria-describedby="hero-title">
          <!-- Simple unicode icon to avoid extra assets -->
          <span aria-hidden="true">✨</span>
          Learn More
        </a>
        <button class="btn secondary" id="toggleParticlesBtn" type="button" aria-pressed="false">
          <span aria-hidden="true">🌀</span>
          Toggle Particles
        </button>
      </div>
      <p class="meta" id="learn">
        Tip: enable “Reduce Motion” in your OS/browser to pause the background animations automatically.
      </p>
    </section>
  </main>

  <script>
    /* -----------------------------------------------------------
       Canvas Particle System (Orbs) — ES2021, DPR-aware, throttled
       -----------------------------------------------------------
       - Particles gently wander with per-orb noise-like drift.
       - Number of particles scales with viewport area (capped).
       - Resizes are debounced; canvas scales to devicePixelRatio.
       - Animation pauses when tab is hidden or reduced motion is on.
       - Public toggle via button for user control.
    */

    (() => {
      const canvas = document.getElementById('bg-canvas');
      const ctx = canvas.getContext('2d', { alpha: true, desynchronized: true });

      // State
      let width = 0, height = 0, dpr = Math.max(1, Math.min(window.devicePixelRatio || 1, 2));
      let orbs = [];
      let rafId = null;
      let running = true;
      let lastT = 0;

      // Feature/media flags
      const mqReduced = window.matchMedia('(prefers-reduced-motion: reduce)');

      // Particle population scales with area; tuned to be gentle on CPU/GPU.
      function targetCount() {
        const area = width * height;
        // 1 orb per ~30k pixels, between 16 and 90 orbs.
        return Math.max(16, Math.min(90, Math.round(area / 30000)));
      }

      // Utility: random in [min, max)
      const rand = (min, max) => Math.random() * (max - min) + min;

      // Resize handler: DPR-aware canvas sizing (debounced)
      const resize = () => {
        const { innerWidth, innerHeight } = window;
        width = Math.max(1, innerWidth);
        height = Math.max(1, innerHeight);
        dpr = Math.max(1, Math.min(window.devicePixelRatio || 1, 2));
        canvas.width = Math.floor(width * dpr);
        canvas.height = Math.floor(height * dpr);
        canvas.style.width = width + 'px';
        canvas.style.height = height + 'px';
        ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
        // Re-tint onbs to match density after resize
        seedOrbs();
      };

      let resizeTimer = 0;
      const onResize = () => {
        clearTimeout(resizeTimer);
        resizeTimer = setTimeout(resize, 120);
      };

      // Orb factory
      function makeOrb() {
        const maxR = Math.max(20, Math.min(120, Math.hypot(width, height) * 0.025));
        const r = rand(8, maxR);
        const hueShift = rand(-12, 12);
        const palette = [
          { h: 215 + hueShift, s: 90, l: 65, a: 0.12 }, // var(--c1)-ish
          { h: 265 + hueShift, s: 88, l: 66, a: 0.12 }, // var(--c2)-ish
          { h: 170 + hueShift, s: 70, l: 58, a: 0.12 }, // var(--c3)-ish
          { h:  10 + hueShift, s: 90, l: 63, a: 0.12 }, // var(--c4)-ish
        ];
        const tint = palette[Math.floor(rand(0, palette.length))];
        const speed = rand(0.06, 0.28); // px/ms scaled later
        const wobble = rand(0.0006, 0.0018); // wobble frequency
        return {
          x: rand(-width * 0.15, width * 1.15),
          y: rand(-height * 0.15, height * 1.15),
          r,
          vx: rand(-0.03, 0.03),
          vy: rand(-0.03, 0.03),
          speed,
          wobble,
          phase: rand(0, Math.PI * 2),
          tint
        };
      }

      // Initialize or adjust orb list to desired count
      function seedOrbs() {
        const n = targetCount();
        if (orbs.length > n) {
          orbs.length = n;
        } else {
          while (orbs.length < n) orbs.push(makeOrb());
        }
      }

      // Draw one orb with soft edges (two-pass radial for glow)
      function drawOrb(o) {
        const grd = ctx.createRadialGradient(o.x, o.y, o.r * 0.2, o.x, o.y, o.r);
        const c = (alpha) => `hsla(${o.tint.h} ${o.tint.s}% ${o.tint.l}% / ${alpha})`;
        grd.addColorStop(0, c(o.tint.a * 1.3));
        grd.addColorStop(0.6, c(o.tint.a * 0.7));
        grd.addColorStop(1, 'hsla(0 0% 0% / 0)');
        ctx.fillStyle = grd;
        ctx.beginPath();
        ctx.arc(o.x, o.y, o.r, 0, Math.PI * 2);
        ctx.fill();

        // Core highlight for depth
        ctx.fillStyle = c(o.tint.a * 0.8);
        ctx.beginPath();
        ctx.arc(o.x - o.r * 0.18, o.y - o.r * 0.18, o.r * 0.35, 0, Math.PI * 2);
        ctx.fill();
      }

      // Advance orb physics
      function stepOrb(o, dt) {
        // Base drift
        o.x += (o.vx * 60 + Math.cos(o.phase) * 0.12) * dt * o.speed;
        o.y += (o.vy * 60 + Math.sin(o.phase) * 0.12) * dt * o.speed;

        // Slow wobble to avoid straight paths
        o.phase += o.wobble * dt * 60;

        // Wrap around edges softly to maintain density
        const pad = Math.max(50, o.r * 1.2);
        if (o.x < -pad) o.x = width + pad;
        if (o.x > width + pad) o.x = -pad;
        if (o.y < -pad) o.y = height + pad;
        if (o.y > height + pad) o.y = -pad;
      }

      // Dynamic frame throttling (aim ~60fps, allow dips to ~30fps on heavy load)
      let accum = 0;
      const minFrame = 1000 / 65; // ~15ms
      const maxFrame = 1000 / 28; // ~36ms throttle cap

      function frame(ts) {
        if (!running) return;
        if (!lastT) lastT = ts;
        let dt = ts - lastT;
        lastT = ts;

        // Clamp dt to avoid large jumps after tab is hidden
        dt = Math.max(minFrame, Math.min(dt, 100));

        accum += dt;
        // Throttle if we're running hot
        const stepNow = accum >= minFrame;
        if (!stepNow) {
          rafId = requestAnimationFrame(frame);
          return;
        }
        accum = 0;

        // Clear canvas with slight fade to create trails (lower alpha = longer trails)
        ctx.globalCompositeOperation = 'source-over';
        ctx.fillStyle = 'rgba(0,0,0,0.10)';
        ctx.fillRect(0, 0, width, height);

        // Additive blending for glow accumulation
        ctx.globalCompositeOperation = 'lighter';

        // Update and render
        for (let i = 0; i < orbs.length; i++) {
          stepOrb(orbs[i], Math.min(dt / 16.6667, 3)); // normalize dt against 60fps
          drawOrb(orbs[i]);
        }

        rafId = requestAnimationFrame(frame);
      }

      function start() {
        if (rafId || !running) return;
        running = true;
        lastT = 0;
        rafId = requestAnimationFrame(frame);
      }

      function stop() {
        running = false;
        if (rafId) cancelAnimationFrame(rafId);
        rafId = null;
      }

      // Visibility: pause when tab not visible
      document.addEventListener('visibilitychange', () => {
        if (document.hidden) stop(); else if (!mqReduced.matches) start();
      }, { passive: true });

      // Reduced motion: toggle canvas + clear
      function handleReducedMotion(e) {
        if (e.matches) {
          stop();
          // Hard clear to avoid leaving trails
          ctx.clearRect(0, 0, width, height);
          canvas.style.display = 'none';
        } else {
          canvas.style.display = 'block';
          start();
        }
      }
      mqReduced.addEventListener?.('change', handleReducedMotion);
      // Fallback for Safari < 14
      mqReduced.addListener?.(handleReducedMotion);

      // Init
      resize();
      if (!mqReduced.matches && !document.hidden) start();
      else canvas.style.display = 'none';

      // User-initiated toggle
      const toggleBtn = document.getElementById('toggleParticlesBtn');
      toggleBtn?.addEventListener('click', () => {
        const currentlyPressed = toggleBtn.getAttribute('aria-pressed') === 'true';
        if (currentlyPressed) {
          stop();
          ctx.clearRect(0, 0, width, height);
          toggleBtn.setAttribute('aria-pressed', 'false');
          toggleBtn.innerHTML = '<span aria-hidden="true">🌀</span> Toggle Particles';
        } else {
          if (!mqReduced.matches) {
            start();
            toggleBtn.setAttribute('aria-pressed', 'true');
            toggleBtn.innerHTML = '<span aria-hidden="true">🛑</span> Stop Particles';
          }
        }
      }, { passive: true });

      // Keep label accurate on load
      toggleBtn?.setAttribute('aria-pressed', (!mqReduced.matches && !document.hidden).toString());
      if (!mqReduced.matches && document.hidden) {
        toggleBtn.innerHTML = '<span aria-hidden="true">🌀</span> Toggle Particles';
      } else if (!mqReduced.matches) {
        toggleBtn.innerHTML = '<span aria-hidden="true">🛑</span> Stop Particles';
      }

      // Resize listener
      window.addEventListener('resize', onResize, { passive: true });

      // Re-seed orbs after initial load (in case fonts/layout change viewport slightly)
      setTimeout(seedOrbs, 200);
    })();
  </script>
</body>
</html>


