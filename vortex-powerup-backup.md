# Vortex Powerup — Backup & Restoration Guide

This documents everything needed to restore the Vortex powerup to DODGE.
All code was working as of the removal date. To restore, add each section
back to the corresponding location in dodge.html.

---

## Overview
- Purple powerup ball with spinning Archimedean spiral
- When collected, spawns a vortex at the pickup location
- Pulls nearby enemies (and player) toward its center
- Objects shrink as they approach (visual + hitbox)
- Enemies destroyed at center (+50 pts each)
- Player dies if sucked into center
- 1.5s grace period after spawn (no player pull)
- Duration-based (8s classic, 9s relaxed, 7s expert)

---

## 1. DEFAULTS — add these properties

```js
powerupVortex: true,
vortexRateMult: 0.50,    // 0.50% spawn chance
vortexDuration: 8,        // seconds
```

## 2. PRESETS — add to easy and hard

### Easy preset:
```js
powerupVortex: true,
vortexRateMult: 0.60,
vortexDuration: 9,
```

### Hard preset:
```js
powerupVortex: true,
vortexRateMult: 0.35,
vortexDuration: 7,
```

## 3. POWERUP_INFO — add entry

```js
{ key: 'Vortex', label: 'vortex', color: '#8844cc', durationKey: 'vortexDuration' },
```

## 4. Game State Variables — add near other powerup state

```js
let vortexes = []; // {x, y, expiresAt, spawnedAt, graceUntil}
const VORTEX_RADIUS = 320;     // pull radius in px
const VORTEX_PULL = 3.0;       // base pull strength (px/frame at edge)
const VORTEX_KILL_R = 6;       // enemies destroyed within this radius
const VORTEX_DEATH_R = 8;      // player dies within this radius
```

## 5. Start/GameOver Resets

Add `vortexes = [];` to both `start()` and `gameOver()`.

## 6. Spawn Selection — add after gun spawn check

```js
if (kind === 'normal' && settings.powerupVortex) {
  cum += settings.vortexRateMult / 100;
  if (roll < cum) kind = 'vortex';
}
```

## 7. Player Pickup — add before coin pickup

```js
} else if (e.kind === 'vortex') {
  e.dead = true;
  scoreAccum += 250;
  pushScoreText(e.x, e.y, 250);
  playVortexPickupSound();
  vortexes.push({ x: e.x, y: e.y, spawnedAt: now, expiresAt: now + settings.vortexDuration * 1000, graceUntil: now + 1500 });
  for (let i = 0; i < 24; i++) {
    particles.push({ x: e.x, y: e.y, vx: (Math.random()-0.5)*6, vy: (Math.random()-0.5)*6, life: 35, color: '#8844cc' });
  }
```

## 8. Bullet Pickup — add before coin in bullet collision section

```js
} else if (e.kind === 'vortex') {
  scoreAccum += 250;
  pushScoreText(e.x, e.y, 250);
  playVortexPickupSound();
  vortexes.push({ x: e.x, y: e.y, spawnedAt: now, expiresAt: now + settings.vortexDuration * 1000, graceUntil: now + 1500 });
```

## 9. Physics — Vortex Pull on Enemies

Add inside the enemy movement loop, after repel force:

```js
// vortex pull on enemies
let minVortexScale = 1;
for (const v of vortexes) {
  const vdx = v.x - e.x, vdy = v.y - e.y;
  const vdist = Math.hypot(vdx, vdy);
  if (vdist > 0 && vdist < VORTEX_RADIUS) {
    const pull = VORTEX_PULL * (1 - vdist / VORTEX_RADIUS) * frameScale;
    e.x += (vdx / vdist) * pull;
    e.y += (vdy / vdist) * pull;
    minVortexScale = Math.min(minVortexScale, Math.max(0.1, vdist / VORTEX_RADIUS));
    if (vdist < VORTEX_KILL_R) {
      e.dead = true;
      if (e.kind === 'normal') {
        scoreAccum += 50;
        pushScoreText(e.x, e.y, 50);
      }
    }
  }
}
e.vortexScale = minVortexScale;
```

## 10. Physics — Vortex Pull on Player + Death Check

Add after enemies filter, before gun logic:

```js
// vortex: pull player + check death + expire
vortexes = vortexes.filter(v => v.expiresAt > now);
playerVortexScale = 1;
for (const v of vortexes) {
  const pastGrace = now >= v.graceUntil;
  const vdx = v.x - player.x, vdy = v.y - player.y;
  const vdist = Math.hypot(vdx, vdy);
  if (pastGrace && vdist > 0 && vdist < VORTEX_RADIUS) {
    const pull = VORTEX_PULL * 1.2 * (1 - vdist / VORTEX_RADIUS) * frameScale;
    if (!attractActive) {
      player.x += (vdx / vdist) * pull;
      player.y += (vdy / vdist) * pull;
    }
    playerVortexScale = Math.min(playerVortexScale, Math.max(0.15, vdist / VORTEX_RADIUS));
    if (vdist < VORTEX_DEATH_R) {
      playDeathSound();
      gameOver(null);
      break;
    }
  }
}
```

Requires `let playerVortexScale = 1;` declared in scope visible to both physics and drawing.

## 11. Drawing — Enemy vortexScale

In the collision check, use scaled radius:
```js
e.r * (e.vortexScale || 1) + effectivePlayerRadius()
```

In the enemy draw loop, add canvas scale transform:
```js
const vs = e.vortexScale || 1;
if (vs < 0.99) {
  ctx.save();
  ctx.translate(e.x, e.y);
  ctx.scale(vs, vs);
  ctx.translate(-e.x, -e.y);
}
// ... draw enemy ...
if (vs < 0.99) ctx.restore();
```

## 12. Drawing — Player vortexScale

Multiply player draw radius by `playerVortexScale`.
Also multiply shield ring radius: `(radius + 5 + i * 4) * playerVortexScale`

## 13. Drawing — Active Vortex Visual

```js
// vortex visuals
for (const v of vortexes) {
  const age = (drawNow - v.spawnedAt) / 1000;
  const rem = v.expiresAt - drawNow;
  const fadeIn = Math.min(1, age * 2);
  const fadeOut = rem < 1000 ? rem / 1000 : 1;
  const alpha = fadeIn * fadeOut;
  const rot = drawNow / 400;
  ctx.save();
  ctx.translate(v.x, v.y);
  // smooth Archimedean spiral arms
  for (let arm = 0; arm < 2; arm++) {
    ctx.beginPath();
    const armOffset = arm * Math.PI;
    const steps = 200;
    for (let s = 0; s <= steps; s++) {
      const t = s / steps;
      const angle = rot + armOffset + t * Math.PI * 7;
      const r = 6 + t * (VORTEX_RADIUS * 0.8);
      const px = Math.cos(angle) * r;
      const py = Math.sin(angle) * r;
      if (s === 0) ctx.moveTo(px, py); else ctx.lineTo(px, py);
    }
    ctx.strokeStyle = '#9955dd';
    ctx.lineWidth = 2.5;
    ctx.globalAlpha = alpha * 0.3;
    ctx.stroke();
  }
  // dark center void
  const grad = ctx.createRadialGradient(0, 0, 0, 0, 0, 50);
  grad.addColorStop(0, `rgba(5, 0, 15, ${0.97 * alpha})`);
  grad.addColorStop(0.3, `rgba(30, 10, 60, ${0.6 * alpha})`);
  grad.addColorStop(0.7, `rgba(20, 5, 40, ${0.2 * alpha})`);
  grad.addColorStop(1, 'rgba(0,0,0,0)');
  ctx.globalAlpha = 1;
  ctx.fillStyle = grad;
  ctx.beginPath();
  ctx.arc(0, 0, 50, 0, Math.PI * 2);
  ctx.fill();
  // bright center dot
  ctx.beginPath();
  ctx.fillStyle = '#cc88ff';
  ctx.globalAlpha = alpha * (0.5 + 0.3 * Math.sin(drawNow / 80));
  ctx.shadowColor = '#8844cc';
  ctx.shadowBlur = 15;
  ctx.arc(0, 0, 4, 0, Math.PI * 2);
  ctx.fill();
  ctx.restore();
  ctx.shadowBlur = 0;
  ctx.globalAlpha = 1;
}
```

## 14. Drawing — Powerup Ball

```js
if (e.kind === 'vortex') {
  const gp = 0.5 + 0.5 * Math.sin(drawNow / 120);
  // core first (underneath)
  ctx.beginPath();
  ctx.fillStyle = '#6633aa';
  ctx.shadowColor = '#8844cc';
  ctx.shadowBlur = 18 + 8 * gp;
  ctx.arc(e.x, e.y, e.r * (1 + 0.06 * gp), 0, Math.PI*2);
  ctx.fill();
  ctx.shadowBlur = 0;
  // spiral on top
  ctx.save();
  ctx.translate(e.x, e.y);
  ctx.beginPath();
  ctx.strokeStyle = '#cc88ff';
  ctx.lineWidth = 1.3;
  ctx.globalAlpha = 0.8 + 0.2 * gp;
  const maxR = e.r * 0.9;
  const turns = 2.5;
  const steps = 80;
  for (let s = 0; s <= steps; s++) {
    const t = s / steps;
    const angle = drawNow / 200 + t * turns * Math.PI * 2;
    const r = 0.3 + t * maxR;
    const px = Math.cos(angle) * r;
    const py = Math.sin(angle) * r;
    if (s === 0) ctx.moveTo(px, py); else ctx.lineTo(px, py);
  }
  ctx.stroke();
  ctx.globalAlpha = 1;
  ctx.restore();
  continue;
}
```

## 15. Sound — playVortexPickupSound

```js
function playVortexPickupSound() {
  const ac = ensureAudio(); if (!ac) return;
  const t = ac.currentTime;
  // deep warping vortex — descending wobble
  const o = ac.createOscillator(), g = ac.createGain();
  o.type = 'sawtooth';
  o.frequency.setValueAtTime(400, t);
  o.frequency.exponentialRampToValueAtTime(40, t + 0.6);
  g.gain.setValueAtTime(0.08, t);
  g.gain.exponentialRampToValueAtTime(0.01, t + 0.6);
  // wobble via LFO
  const lfo = ac.createOscillator(), lfoG = ac.createGain();
  lfo.frequency.value = 12; lfoG.gain.value = 30;
  lfo.connect(lfoG); lfoG.connect(o.frequency);
  lfo.start(t); lfo.stop(t + 0.6);
  o.connect(g); g.connect(sfxOut);
  o.start(t); o.stop(t + 0.6);
  // low rumble
  const o2 = ac.createOscillator(), g2 = ac.createGain();
  o2.type = 'sine'; o2.frequency.value = 50;
  g2.gain.setValueAtTime(0.1, t);
  g2.gain.exponentialRampToValueAtTime(0.01, t + 0.5);
  o2.connect(g2); g2.connect(sfxOut);
  o2.start(t); o2.stop(t + 0.5);
}
```

## 16. Legend HTML

```html
<div class="row"><svg class="dot-svg" viewBox="0 0 36 20" width="36" height="20">
  <path d="M18.5,10.0 L18.6,10.1 ... L25.5,10.0" fill="none" stroke="#bb66ff" stroke-width="1.4" stroke-linecap="round">
    <animateTransform attributeName="transform" type="rotate" values="0 18 10;360 18 10" dur="3s" repeatCount="indefinite"/>
  </path>
</svg>vortex</div>
```

(Full path data is the mathematically generated Archimedean spiral — see the legend entry in the git history)

---

## Design Notes / Ideas for Making It More Fun

- The mechanic felt passive — you collect it and watch. Needs more player agency.
- Possible tweak: let the player PLACE it with a tap/click instead of auto-spawning at pickup
- Possible tweak: make it follow the player slowly (like a pet) so you can aim it
- The "mountain" (inverse vortex that pushes) could pair with it for strategic play
- Consider making it rarer but more powerful, or more common but shorter duration
- The visual (spiral + shrinking objects) was good — keep that if restoring
