# Maticupo's Pomodoro

A Pomodoro timer built as a single self-contained HTML file. No frameworks, no build tools, no dependencies — pure HTML, CSS, and JavaScript.

[Live Demo](https://maticupo.github.io/Maticupos-s-Pomodoro-Timer)

---

## Overview

This project started as an exercise in building something visually polished without reaching for a framework. The entire application — UI, animations, audio synthesis, and state management — lives in one `index.html` file under 1,000 lines of vanilla code.

---

## Features

- Pomodoro cycle with auto-advancing modes (Focus, Short Break, Long Break)
- Four real photo backgrounds (Unsplash) with Ken Burns animation and cinematic vignette
- Animated rain canvas overlay on rainy scenes
- 16 SoundCloud tracks across four categories: Animal Crossing lofi, Studio Ghibli lofi, anime lofi, Japanese lofi
- Track picker with live volume control via the SoundCloud Widget API
- Japanese school bell (the four-note Westminster chime) synthesized with Web Audio API
- Randomized Japanese voice announcements via Web Speech API after each session
- Fully configurable timer durations and session count

---

## Technical notes

### Timer accuracy

The timer uses `requestAnimationFrame` and derives remaining time from `performance.now()` rather than counting `setInterval` ticks. This keeps the display accurate even when the browser throttles background tabs.

```javascript
function timerTick() {
  if (!running) return;
  requestAnimationFrame(timerTick);
  const elapsed = (performance.now() - startTs) / 1000;
  const s = Math.ceil(Math.max(0, startRem - elapsed));
  if (s === lastSec) return;
  lastSec = s; rem = s;
  redrawRing(); redrawTime();
  if (rem <= 0) onSessionEnd();
}
```

### SVG progress ring

```javascript
const CIRC = 2 * Math.PI * 84;

function redrawRing() {
  const el = document.getElementById('ring');
  el.style.strokeDashoffset = CIRC * (1 - rem / total);
  el.style.stroke = MODES[mode].color;
}
```

### Bell synthesis  Web Audio API

The school bell is synthesized from a fundamental frequency plus two inharmonic partials, each with an exponential decay envelope, fed through a convolution reverb.

```javascript
// E5 -> C5 -> D5 -> G4  (the standard Japanese school chime)
const notes = [659.25, 523.25, 587.33, 392.00];

notes.forEach((freq, i) => {
  const t0 = ctx.currentTime + gaps[i];
  [[freq, 0.55], [freq * 2.756, 0.13], [freq * 5.404, 0.04]].forEach(([f, pk]) => {
    const o = ctx.createOscillator(), g = ctx.createGain();
    o.type = 'sine'; o.frequency.value = f;
    g.gain.setValueAtTime(0, t0);
    g.gain.linearRampToValueAtTime(pk, t0 + 0.007);
    g.gain.exponentialRampToValueAtTime(0.0001, t0 + 2.0);
    o.connect(g); g.connect(reverb);
    o.start(t0); o.stop(t0 + 2.2);
  });
});
```

### Photo backgrounds

Each scene is a `position: fixed` div with `pointer-events: none`. Opacity is toggled via a CSS class transition. The Ken Burns effect is a pure CSS animation no JavaScript involved.

```css
.bg-layer {
  position: fixed;
  inset: 0;
  background-size: cover;
  opacity: 0;
  transition: opacity 1.6s ease;
  pointer-events: none;
  animation: kbzoom 35s ease-in-out infinite alternate;
}

@keyframes kbzoom {
  from { transform: scale(1);    }
  to   { transform: scale(1.06); }
}
```

### Rain canvas

```javascript
const drops = Array.from({ length: 220 }, () => ({
  x: Math.random() * 1800,
  y: Math.random() * 1000,
  len: Math.random() * 18 + 7,
  speed: Math.random() * 7 + 4,
  opacity: Math.random() * 0.38 + 0.07,
}));

(function rainLoop() {
  requestAnimationFrame(rainLoop);
  rctx.clearRect(0, 0, rainCanvas.width, rainCanvas.height);
  if (!SCENES[sceneIdx].rain) return;
  drops.forEach(d => {
    d.y += d.speed;
    d.x -= d.speed * 0.14;
    if (d.y > rainCanvas.height) { d.y = -10; d.x = Math.random() * rainCanvas.width; }
    rctx.beginPath();
    rctx.moveTo(d.x, d.y);
    rctx.lineTo(d.x - d.len * 0.12, d.y + d.len);
    rctx.strokeStyle = `rgba(185,215,255,${d.opacity})`;
    rctx.stroke();
  });
})();
```

---

## File structure

```
maticupos-pomodoro/
├── index.html
└── README.md
```

---

## Credits

**Photos** — Unsplash (Ben George, BREAKIFY, ayumi kubo, Ryoji Hayasaka). Licensed under the [Unsplash License](https://unsplash.com/license).

**Music** — Streamed via SoundCloud. Artists: Closed on Sunday, Cozy, Aso, Dreamhop, jonzz, Bagas Satriyotomo, Ambition, joevio, Scoryx, AJota Beats, kisa, Samuel Kim, Enjoy Studying.
