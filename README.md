# 🌸 Maticupo's Pomodoro

A fully self-contained anime-aesthetic Pomodoro timer built in a single HTML file — no frameworks, no build tools, no dependencies. Just open it in a browser.

**[▶ Live Demo](https://YOUR-USERNAME.github.io/maticupos-pomodoro)** &nbsp;·&nbsp; **[SoundCloud](https://soundcloud.com)**

![Preview](preview.png)

---

## ✨ Features

- **4 real photo backgrounds** — rural Japan rain, Tokyo neon night, Kyoto shrine lanterns, Japanese countryside — all with a slow Ken Burns zoom and cinematic vignette
- **Animated rain overlay** on the two rainy scenes
- **Full Pomodoro cycle** — Focus → Short Break → Long Break, auto-advancing on session end
- **16 SoundCloud tracks** — Animal Crossing lofi, Studio Ghibli lofi, Your Name, Demon Slayer lofi — choose from a track picker panel
- **Volume slider** — live volume control for the music stream
- **Japanese school bell** (キーンコーンカーンコーン) built with Web Audio API — plays on every session end and skip
- **Soft toast chime** — a lighter single bell ding when the notification appears
- **Japanese voice announcements** — randomized train station lines spoken via Web Speech API after the bell
- **改善 (Kaizen)** — displayed below the timer as the guiding philosophy
- **Fully customizable** — adjust Focus, Short Break, Long Break durations and sessions before long break

---

## 🚀 Deploy to GitHub Pages in 3 steps

```bash
# 1. Create a new repo on github.com called: maticupos-pomodoro

# 2. Push the file
git init
git add index.html README.md
git commit -m "🌸 initial commit"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/maticupos-pomodoro.git
git push -u origin main

# 3. Enable GitHub Pages
# Go to repo Settings → Pages → Source: Deploy from branch → main → / (root) → Save
# Your app will be live at: https://YOUR-USERNAME.github.io/maticupos-pomodoro
```

> **Note:** rename `sakura-study.html` to `index.html` before pushing so GitHub Pages serves it as the root page.

---

## 🧠 How it's built

Everything is vanilla — one file, ~978 lines, zero npm installs.

### Timer engine — `requestAnimationFrame` not `setInterval`

```javascript
function timerTick() {
  if (!running) return;
  requestAnimationFrame(timerTick);
  // Derive remaining time from elapsed wall-clock time
  // so the display is always accurate even if the tab is backgrounded
  const elapsed = (performance.now() - startTs) / 1000;
  const s = Math.ceil(Math.max(0, startRem - elapsed));
  if (s === lastSec) return; // only redraw when the second actually changes
  lastSec = s; rem = s;
  redrawRing(); redrawTime();
  if (rem <= 0) onSessionEnd();
}
```

Using `performance.now()` instead of counting ticks means the timer stays accurate even if the browser throttles the tab.

### SVG progress ring

```javascript
const CIRC = 2 * Math.PI * 84; // circumference of the ring (r=84)

function redrawRing() {
  const el = document.getElementById('ring');
  // stroke-dashoffset controls how much of the ring is "filled"
  el.style.strokeDashoffset = CIRC * (1 - rem / total);
  el.style.stroke = MODES[mode].color;
}
```

### Japanese school bell — Web Audio API

```javascript
// キーンコーンカーンコーン — the 4-note Westminster chime used in Japanese schools
// Notes: E5 (659Hz), C5 (523Hz), D5 (587Hz), G4 (392Hz)
const notes  = [659.25, 523.25, 587.33, 392.00];
const gaps   = [0, 0.7, 1.4, 2.1]; // seconds between each note

notes.forEach((freq, i) => {
  const t0 = ctx.currentTime + gaps[i];
  // Bell timbre = fundamental + inharmonic partials with exponential decay
  [[freq, 0.55], [freq * 2.756, 0.13], [freq * 5.404, 0.04]].forEach(([f, pk]) => {
    const o = ctx.createOscillator(), g = ctx.createGain();
    o.type = 'sine'; o.frequency.value = f;
    g.gain.setValueAtTime(0, t0);
    g.gain.linearRampToValueAtTime(pk, t0 + 0.007); // instant attack
    g.gain.exponentialRampToValueAtTime(0.0001, t0 + 2.0); // natural decay
    o.connect(g); g.connect(reverb);
    o.start(t0); o.stop(t0 + 2.2);
  });
});
```

### SoundCloud Widget API

```javascript
function loadTrack(idx, play) {
  const frame = document.getElementById('sc-frame');
  const t = TRACKS[idx];
  // Swap the iframe src to load a new track
  frame.src = `https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/${t.type}/${t.id}` +
    `&color=%23f0a0c0&auto_play=false&hide_related=true&visual=false`;
  // SC.Widget wraps the iframe and exposes play/pause/setVolume
  scWidget = SC.Widget(frame);
  scWidget.bind(SC.Widget.Events.READY, () => {
    scReady = true;
    scWidget.setVolume(currentVolume);
    if (play) scWidget.play();
  });
}
```

### Photo backgrounds — Unsplash CDN

```javascript
// Each scene is a fixed div with background-image, opacity toggled by class
// Ken Burns effect is pure CSS animation
const SCENES = [
  { bg:'bg0', rain:true,  credit:'Ben George / Unsplash — Rainy rural Japan'         },
  { bg:'bg1', rain:true,  credit:'BREAKIFY / Unsplash — Tokyo rainy night'           },
  { bg:'bg2', rain:false, credit:'ayumi kubo / Unsplash — Kyoto lanterns'            },
  { bg:'bg3', rain:false, credit:'Ryoji Hayasaka / Unsplash — Japanese countryside'  },
];
```

```css
.bg-layer {
  position: fixed; inset: 0;
  background-size: cover; background-position: center;
  opacity: 0; transition: opacity 1.6s ease;
  pointer-events: none; /* never blocks UI clicks */
  animation: kbzoom 35s ease-in-out infinite alternate;
}
@keyframes kbzoom {
  from { transform: scale(1); }
  to   { transform: scale(1.06); }
}
```

### Rain canvas

```javascript
// 220 drops, each with randomized speed, length, and opacity
const drops = Array.from({length: 220}, () => ({
  x: Math.random() * 1800,
  y: Math.random() * 1000,
  len: Math.random() * 18 + 7,
  speed: Math.random() * 7 + 4,
  opacity: Math.random() * 0.38 + 0.07,
}));

(function rainLoop() {
  requestAnimationFrame(rainLoop);
  rctx.clearRect(0, 0, rainCanvas.width, rainCanvas.height);
  if (!SCENES[sceneIdx].rain) return; // only draw on rainy scenes
  drops.forEach(d => {
    d.y += d.speed; d.x -= d.speed * 0.14; // slight diagonal
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

## 🎵 Music credits

All tracks streamed via SoundCloud. Artists: Closed on Sunday, Cozy, Aso, Dreamhop, jonzz, Bagas Satriyotomo, Hetalia, Ambition, joevio, Scoryx, AJota Beats, kisa, Samuel Kim, Enjoy Studying.

## 📸 Photo credits

All photos from [Unsplash](https://unsplash.com) — free to use under the [Unsplash License](https://unsplash.com/license).
Ben George · BREAKIFY · ayumi kubo · Ryoji Hayasaka

---

## 📁 File structure

```
maticupos-pomodoro/
├── index.html      ← the entire app (rename from sakura-study.html)
└── README.md
```

That's it. No node_modules. No webpack. No React. Just HTML, CSS, and JavaScript.

---

*Built with 🌸 and too many late nights.*
