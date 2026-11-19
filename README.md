<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Prepare VR – Fire Escape (Walkable)</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/aframe/1.4.2/aframe.min.js"></script>
<script>window.AFRAME||document.write('<script src="https://cdn.jsdelivr.net/npm/aframe@1.5.0/dist/aframe.min.js"><\/script>')</script>
<script>window.AFRAME||document.write('<script src="https://unpkg.com/aframe@1.5.0/dist/aframe.min.js"><\/script>')</script>
<style>
:root{color-scheme:light dark}
html,body{margin:0;height:100%;background:#0b0d10}
#msg{position:fixed;left:8px;right:8px;bottom:8px;z-index:9;font:13px sans-serif;color:#9fb3c8;pointer-events:none}
</style>
</head>
<body>
<div id="msg">Loading…</div>
<a-scene background="color:#0d1016" fog="type:exponential;color:#0d1016;density:.012" renderer="antialias:true" cursor="rayOrigin:mouse" raycaster="objects:.clickable">
  <a-entity id="rig" position="0 0 0">
    <a-entity id="cam" camera look-controls="pointerLockEnabled:false" position="0 1.6 0"></a-entity>
    <a-entity id="lh" laser-controls="hand:left" raycaster="objects:.clickable"></a-entity>
    <a-entity id="rh" laser-controls="hand:right" raycaster="objects:.clickable"></a-entity>
  </a-entity>
  <a-entity id="world"></a-entity>
</a-scene>

<script>
const note = document.getElementById("msg");
window.onerror = (m, s, l) => { note.style.display = "block"; note.style.color = "#f87171"; note.textContent = "Error: " + m + " (line " + l + ")"; };
if (!window.AFRAME) throw new Error("A-Frame failed to load. Check your internet connection.");

/* ---- UI box drawn on a canvas (no external fonts needed) ---- */
AFRAME.registerComponent("ui-box", {
  schema: { w: { default: 1 }, h: { default: .3 }, text: { default: "" }, bg: { default: "none" }, color: { default: "#ffffff" }, size: { default: 40 }, op: { default: 1 } },
  init() {
    this.cv = document.createElement("canvas");
    this.tex = new THREE.CanvasTexture(this.cv);
    this.mat = new THREE.MeshBasicMaterial({ map: this.tex, transparent: true, side: THREE.DoubleSide });
    this.mesh = new THREE.Mesh(new THREE.PlaneGeometry(1, 1), this.mat);
    this.el.setObject3D("mesh", this.mesh);
  },
  update() {
    const d = this.data, k = 500, cv = this.cv;
    cv.width = Math.round(d.w * k); cv.height = Math.round(d.h * k);
    this.mesh.scale.set(d.w, d.h, 1);
    const x = cv.getContext("2d");
    x.clearRect(0, 0, cv.width, cv.height);
    if (d.bg !== "none") { x.globalAlpha = d.op; x.fillStyle = d.bg; x.fillRect(0, 0, cv.width, cv.height); x.globalAlpha = 1; }
    if (d.text) {
      x.fillStyle = d.color; x.font = d.size + "px sans-serif"; x.textAlign = "center"; x.textBaseline = "middle";
      const lines = [];
      d.text.split("\n").forEach(p => { let l = ""; p.split(" ").forEach(w => { const t = l ? l + " " + w : w; if (x.measureText(t).width > cv.width - 30 && l) { lines.push(l); l = w; } else l = t; }); lines.push(l); });
      const lh = d.size * 1.25, y0 = cv.height / 2 - (lines.length - 1) * lh / 2;
      lines.forEach((l, i) => x.fillText(l, cv.width / 2, y0 + i * lh));
    }
    this.tex.needsUpdate = true;
  }
});

/* ---- Procedural textures + effects (no image files needed) ---- */
const _tx = {};
function TEX(k) {
  if (_tx[k]) return _tx[k];
  const c = document.createElement("canvas"), S = c.width = c.height = 256, x = c.getContext("2d"), R = Math.random;
  const base = { carpet: "#3b4252", tile: "#b9bcc0", wall: "#d8d3c8", ceil: "#cfd2d4", asphalt: "#2c2f34", facade: "#5d524b" }[k];
  if (base) { x.fillStyle = base; x.fillRect(0, 0, S, S); for (let i = 0; i < 4000; i++) { const v = R() < .5 ? 0 : 255; x.fillStyle = `rgba(${v},${v},${v},.06)`; x.fillRect(R() * S, R() * S, 2, 2); } }
  x.strokeStyle = "rgba(0,0,0,.4)"; x.lineWidth = 3;
  if (k === "tile") { x.strokeRect(0, 0, S, S); x.beginPath(); x.moveTo(S / 2, 0); x.lineTo(S / 2, S); x.moveTo(0, S / 2); x.lineTo(S, S / 2); x.stroke(); }
  if (k === "ceil") x.strokeRect(0, 0, S, S);
  if (k === "facade") for (let i = 0; i < 4; i++) for (let j = 0; j < 4; j++) { x.fillStyle = R() < .45 ? "#ff8a2a" : "#161a22"; x.fillRect(i * 64 + 12, j * 64 + 12, 40, 40); }
  const gr = { puff: [[0, "rgba(255,255,255,.9)"], [1, "rgba(255,255,255,0)"]], flame: [[0, "rgba(255,240,170,1)"], [.35, "rgba(255,140,30,.8)"], [1, "rgba(255,40,0,0)"]], vig: [[0, "rgba(160,0,0,0)"], [1, "rgba(160,0,0,.95)"]] }[k];
  if (gr) { const g = x.createRadialGradient(S / 2, S / 2, k === "vig" ? S * .25 : 0, S / 2, S / 2, k === "vig" ? S * .72 : S / 2); gr.forEach(s => g.addColorStop(s[0], s[1])); x.fillStyle = g; x.fillRect(0, 0, S, S); }
  const t = new THREE.CanvasTexture(c); t.wrapS = t.wrapT = THREE.RepeatWrapping; return _tx[k] = t;
}
AFRAME.registerComponent("ptex", {
  schema: { kind: { default: "wall" }, rx: { default: 1 }, ry: { default: 1 } },
  update() {
    if (!this.el.hasLoaded) { this.el.addEventListener("loaded", () => this.update(), { once: true }); return; }
    const m = this.el.getObject3D("mesh"); if (!m) return;
    const t = TEX(this.data.kind).clone(); t.needsUpdate = true; t.repeat.set(this.data.rx, this.data.ry); m.material.map = t; m.material.needsUpdate = true;
  }
});
AFRAME.registerComponent("billboard", { tick() { this.q = this.q || new THREE.Quaternion(); this.el.object3D.quaternion.copy(cam.object3D.getWorldQuaternion(this.q)); } });
AFRAME.registerComponent("flick", { schema: { s: { default: 1 } }, tick(t) { const o = this.el.object3D, i = o.id, k = this.data.s * (1 + .25 * Math.sin(t * .011 + i) + .15 * Math.sin(t * .027 + i * 2)); o.scale.set(k, k * (1.2 + .2 * Math.sin(t * .017 + i)), 1); } });
AFRAME.registerComponent("puff", { tick(t) { const o = this.el.object3D; if (!this.b) { this.b = o.position.clone(); this.ph = Math.random() * 6; this.q = new THREE.Quaternion(); } o.position.set(this.b.x + Math.sin(t * .0004 + this.ph) * .35, this.b.y + Math.sin(t * .0003 + this.ph * 2) * .15, this.b.z); o.quaternion.copy(cam.object3D.getWorldQuaternion(this.q)); o.rotateZ(this.ph); } });
function desk(x, z) {
  wall(x - .8, x + .8, z - .4, z + .4, .75, "#7a5a3a");
  el("a-box", { position: `${x} .95 ${z - .1}`, width: .55, height: .35, depth: .04, material: "color:#111;emissive:#3b82f6;emissiveIntensity:.6" });
  el("a-box", { position: `${x} .45 ${z + .9}`, width: .5, height: .06, depth: .5, color: "#222" });
  el("a-box", { position: `${x} .75 ${z + 1.12}`, width: .5, height: .6, depth: .06, color: "#222" });
}

const scene = document.querySelector("a-scene"), world = document.getElementById("world"), rigEl = document.getElementById("rig"), cam = document.getElementById("cam");
const el = (tag, attrs, parent) => { const e = document.createElement(tag); for (const k in attrs) e.setAttribute(k, attrs[k]); (parent || world).appendChild(e); return e; };
const box = (parent, pos, o, rot) => { const e = el("a-entity", { position: pos, rotation: rot || "0 0 0" }, parent); e.setAttribute("ui-box", o); return e; };

/* ---- Game state ---- */
const G = { phase: "menu", health: 100, bonus: 0, ext: false, t0: 0, msg: "", until: 0, hurtT: 0 };
const AX = [0, 0], rects = [], fires = [];
let audio, alarmTimer, hud, ov, ovFn, laptop, extEl, turnCool = 0, vig, crackleG, emer, noiseBuf;
const say = (t, s) => { G.msg = t; G.until = Date.now() + (s || 3) * 1000; };

/* ---- Alarm ---- */
function beep(f, d) { const o = audio.createOscillator(), g = audio.createGain(); o.type = "square"; o.frequency.value = f; g.gain.value = .04; o.connect(g); g.connect(audio.destination); o.start(); o.stop(audio.currentTime + d); }
const stopAlarm = () => clearInterval(alarmTimer);
function noise() { if (!noiseBuf) { noiseBuf = audio.createBuffer(1, audio.sampleRate, audio.sampleRate); const d = noiseBuf.getChannelData(0); for (let i = 0; i < d.length; i++) d[i] = Math.random() * 2 - 1; } return noiseBuf; }
function startCrackle() { const s = audio.createBufferSource(), f = audio.createBiquadFilter(), g = crackleG = audio.createGain(); s.buffer = noise(); s.loop = true; f.type = "lowpass"; f.frequency.value = 900; g.gain.value = 0; s.connect(f); f.connect(g); g.connect(audio.destination); s.start(); }
function step() { const s = audio.createBufferSource(), f = audio.createBiquadFilter(), g = audio.createGain(); s.buffer = noise(); f.type = "lowpass"; f.frequency.value = 400; g.gain.setValueAtTime(.25, audio.currentTime); g.gain.exponentialRampToValueAtTime(.001, audio.currentTime + .12); s.connect(f); f.connect(g); g.connect(audio.destination); s.start(0, Math.random()); s.stop(audio.currentTime + .15); }
function startAlarm() { stopAlarm(); if (!audio) return; let n = 0; alarmTimer = setInterval(() => beep(n++ % 2 ? 660 : 880, .25), 400); }

/* ---- Level (x right, z forward = negative). Collision boxes double as walls ---- */
function wall(x1, x2, z1, z2, h, color, kind, sc) {
  kind = kind || (color ? "" : "wall"); sc = sc || 2;
  rects.push([x1, x2, z1, z2]);
  el("a-box", { position: `${(x1 + x2) / 2} ${h / 2} ${(z1 + z2) / 2}`, width: x2 - x1, height: h, depth: z2 - z1, color: color || "#ffffff", ...(kind ? { ptex: `kind:${kind};rx:${Math.max(x2 - x1, z2 - z1) / sc};ry:${h / sc}` } : {}) });
}
function makeFire(x, z) {
  const F = { x, z, on: true }, g = el("a-entity", { position: `${x} 0 ${z}` });
  [[0, 0], [.3, .2], [-.25, .25], [.1, -.3], [-.2, -.1]].forEach(([dx, dz], i) => {
    el("a-plane", { class: "clickable", position: `${dx} .6 ${dz}`, width: 1, height: 1.3, material: "shader:flat;transparent:true;blending:additive;depthWrite:false;side:double", ptex: "kind:flame", billboard: "", flick: `s:${.8 + i * .08}` }, g);
  });
  F.light = el("a-light", { type: "point", color: "#ff6a00", intensity: 1.3, distance: 7, position: "0 1 0" }, g);
  g.addEventListener("click", () => {
    if (!F.on || G.phase !== "play") return;
    const p = rigEl.object3D.position, d = Math.hypot(p.x - F.x, p.z - F.z);
    if (!G.ext) say("You need an extinguisher to fight it. Or go around.");
    else if (d > 3.5) say("Move a little closer (but not too close).");
    else { F.on = false; g.setAttribute("visible", false); G.bonus += 15; say("Fire out! PASS: Pull, Aim, Squeeze, Sweep.", 4); }
  });
  F.g = g; fires.push(F);
}
function build() {
  el("a-plane", { rotation: "-90 0 0", width: 200, height: 200, ptex: "kind:asphalt;rx:60;ry:60" });
  el("a-plane", { rotation: "-90 0 0", position: "0 .01 0", width: 8, height: 8, ptex: "kind:carpet;rx:4;ry:4" });
  el("a-plane", { rotation: "-90 0 0", position: "0 .01 -14", width: 3, height: 20, ptex: "kind:tile;rx:5;ry:33" });
  /* office 8x8 + doorway in the front wall */
  wall(-4.1, 4.1, 3.9, 4.1, 3); wall(-4.1, -3.9, -4.1, 4.1, 3); wall(3.9, 4.1, -4.1, 4.1, 3);
  wall(-4.1, -1, -4.1, -3.9, 3); wall(1, 4.1, -4.1, -3.9, 3);
  el("a-box", { position: "0 2.6 -4", width: 2, height: .8, depth: .2, color: "#8d939c" });
  /* corridor 3 wide, 20 long, ends in two doors */
  wall(-1.6, -1.4, -24.1, -4, 3); wall(1.4, 1.6, -24.1, -4, 3); wall(-1.6, 1.6, -24.1, -23.9, 3);
  el("a-box", { position: "0 3.1 0", width: 8.2, height: .2, depth: 8.2, color: "#ffffff", ptex: "kind:ceil;rx:4;ry:4" });
  el("a-box", { position: "0 3.1 -14", width: 3.2, height: .2, depth: 20.2, color: "#ffffff", ptex: "kind:ceil;rx:2;ry:10" });
  desk(2, -1.5); desk(-2, -1.5); desk(-2.2, 1.8); desk(2.2, 1.8);                    /* desk */
  wall(-10, 10, -33, -27, 8, "#ffffff", "facade", 6);
  for (let i = 0; i < 7; i++) el("a-plane", { position: `${-8 + i * 2.7} ${5 + Math.random() * 3.5} -33.4`, width: 3, height: 4, material: "shader:flat;transparent:true;blending:additive;depthWrite:false;side:double", ptex: "kind:flame", billboard: "", flick: "s:1.4" });
  el("a-light", { type: "point", color: "#ff6a00", intensity: 1.5, distance: 30, position: "0 4 -37" });                        /* burning building (outside) */
  el("a-box", { position: "0 2.05 -14", width: 3, height: 1.9, depth: 20, material: "color:#2b2b2b;opacity:.35;transparent:true;side:double;depthWrite:false" });
  for (let i = 0; i < 40; i++) el("a-plane", { position: `${(Math.random() - .5) * 2.4} ${1.3 + Math.random() * 1.5} ${-4.5 - Math.random() * 19}`, width: 2.6, height: 2.6, material: "shader:flat;transparent:true;opacity:.3;depthWrite:false;side:double;color:#555", ptex: "kind:puff", puff: "" }); /* smoke layer: stay below 1.1 m */
  emer = el("a-light", { type: "point", color: "#ff2a2a", intensity: .5, distance: 14, position: "0 2.6 -12" });
  el("a-light", { type: "ambient", intensity: .55 }); el("a-light", { type: "directional", position: "2 4 2", intensity: .3 }); el("a-light", { type: "point", color: "#fff2d8", intensity: .5, distance: 9, position: "0 2.6 0" });
  for (const [x, z] of [[-2, -2], [2, -2], [-2, 2], [2, 2]]) el("a-box", { position: `${x} 2.98 ${z}`, width: 1.2, height: .05, depth: .6, material: "shader:flat;color:#fff8e8" });
  for (const z of [-2, 2]) box(world, `-3.88 1.7 ${z}`, { w: 1.5, h: 1.1, bg: "#ff7a2a", op: .85 }, "0 90 0");
  wall(-3.7, -2.9, 3.2, 3.8, 1.4, "#8a949e"); wall(2.9, 3.7, 3.2, 3.8, 1.4, "#8a949e");
  el("a-box", { position: "-.8 1.1 -23.85", width: 1.1, height: 2.2, depth: .05, color: "#9aa0a8" }); el("a-box", { position: ".8 1.1 -23.85", width: 1.1, height: 2.2, depth: .05, color: "#4b5563" });
  box(world, "0 2.95 -3.85", { w: 1.2, h: .3, text: "EXIT", bg: "#16a34a", size: 60 });
  box(world, "-.8 2.3 -23.85", { w: 1.2, h: .45, text: "LIFT ✖", bg: "#b91c1c", size: 60 });
  box(world, ".8 2.3 -23.85", { w: 1.2, h: .45, text: "STAIRS", bg: "#16a34a", size: 60 });
  box(world, "1.39 1.15 -6", { w: .8, h: .2, text: "EXTINGUISHER", bg: "#7f1d1d", size: 40 }, "0 -90 0");
  laptop = el("a-box", { class: "clickable", position: "2 .77 -1.5", width: .45, height: .04, depth: .3, color: "#222" });
  laptop.addEventListener("click", () => { if (G.phase !== "play" || !laptop.getAttribute("visible")) return; laptop.setAttribute("visible", false); G.bonus -= 10; say("−10: Never stop for belongings. Get out!", 4); });
  extEl = el("a-cylinder", { class: "clickable", position: "1.25 .45 -6", radius: .12, height: .5, color: "#dc2626" });
  extEl.addEventListener("click", () => {
    if (G.phase !== "play" || G.ext) return;
    const p = rigEl.object3D.position; if (Math.hypot(p.x - 1.25, p.z + 6) > 3.5) return say("Move closer to the extinguisher.");
    G.ext = true; extEl.setAttribute("visible", false); G.bonus += 5; say("Got it! Click a fire from close range to put it out.", 4);
  });
  makeFire(.7, -10); makeFire(-.7, -15);
  /* outside */
  el("a-circle", { rotation: "-90 0 0", position: "0 .02 -60", radius: 2.3, color: "#16a34a" });
  box(world, "0 1.8 -60", { w: 3, h: 1, text: "ASSEMBLY POINT", bg: "#16a34a", size: 70 });
  box(world, "0 2.5 -33.15", { w: 6, h: 1, text: "BUILDING ON FIRE – STAY AWAY", bg: "#b91c1c", size: 50 }, "0 180 0");
}
const hit = (x, z) => rects.some(r => x > r[0] - .3 && x < r[1] + .3 && z > r[2] - .3 && z < r[3] + .3);

/* ---- Movement + game rules ---- */
AFRAME.registerComponent("mover", {
  init() { this.k = {}; this.v = new THREE.Vector3(); addEventListener("keydown", e => this.k[e.code] = 1); addEventListener("keyup", e => this.k[e.code] = 0); },
  tick(t, dt) {
    dt = Math.min(dt, 50) / 1000; const k = this.k, rig = rigEl.object3D, p = rig.position;
    if (!scene.is("vr-mode")) cam.object3D.position.y = (k.KeyC || k.ControlLeft ? .8 : 1.6) + (this.mv ? Math.sin(t * .009) * .025 : 0);     /* desktop crouch */
    cam.object3D.getWorldPosition(this.v); const headY = this.v.y;
    const fg = scene.object3D.fog; if (fg) { const tgt = G.phase === "play" && p.z < -4.3 ? (headY > 1.15 ? .3 : .07) : .012; fg.density += (tgt - fg.density) * Math.min(1, dt * 2); }
    const vm = vig && vig.getObject3D("mesh"); if (vm) vm.material.opacity = Math.min(.9, (G.phase === "play" ? (1 - G.health / 100) * .7 : 0) + (Date.now() < G.hurtT ? .3 : 0));
    let near = 99; const grow = G.phase === "play" ? Math.min(1.4, 1 + (Date.now() - G.t0) / 90000) : 1;
    fires.forEach(F => { if (!F.on) return; F.g.object3D.scale.setScalar(grow); near = Math.min(near, Math.hypot(p.x - F.x, p.z - F.z)); const L = F.light.getObject3D("light"); if (L) L.intensity = 1.1 + .5 * Math.sin(t * .02 + F.x * 7) + .3 * Math.random(); });
    if (crackleG) crackleG.gain.value = G.phase === "play" ? Math.random() * .3 * Math.max(0, 1 - near / 9) : 0;
    if (G.phase !== "play" && G.phase !== "out") return;
    let f = (k.KeyW || k.ArrowUp ? 1 : 0) - (k.KeyS || k.ArrowDown ? 1 : 0) - AX[1], s = (k.KeyD || k.ArrowRight ? 1 : 0) - (k.KeyA || k.ArrowLeft ? 1 : 0) + AX[0];
    const m = Math.hypot(f, s); if (m > 1) { f /= m; s /= m; } this.mv = m > .05;
    const sp = (headY < 1.15 ? 1 : 2.2) * dt, yaw = rig.rotation.y + cam.object3D.rotation.y;
    const mx = (-Math.sin(yaw) * f + Math.cos(yaw) * s) * sp, mz = (-Math.cos(yaw) * f - Math.sin(yaw) * s) * sp;
    if (!hit(p.x + mx, p.z)) p.x += mx; if (!hit(p.x, p.z + mz)) p.z += mz;
    this.sd = (this.sd || 0) + Math.hypot(mx, mz); if (this.sd > .75) { this.sd = 0; if (audio) step(); }
    if (G.phase === "out") { if (Math.hypot(p.x, p.z + 60) < 2.3) end(true); return; }
    const hurt = (d, why) => { G.health -= d; G.hurtT = Date.now() + 250; say(why, 1.2); if (G.health <= 0) end(false, why); };
    if (p.z < -4.3 && headY > 1.15) hurt(14 * dt, "SMOKE! Get low: hold C (or crouch).");
    fires.forEach(F => { if (F.on && Math.hypot(p.x - F.x, p.z - F.z) < .85 * grow) hurt(35 * dt, "Too close to the fire! Go around it."); });
    if (p.z < -22.8) {
      if (p.x < 0) { G.bonus -= 20; p.x = 0; p.z = -20; say("−20: Never use a lift in a fire. It can stall or open on flames.", 5); }
      else { G.phase = "out"; stopAlarm(); p.x = 0; p.z = -35; say("Stairs ✔ You’re out! Head to the green assembly point.", 5); }
    }
  }
});

/* ---- Overlay + HUD ---- */
function clearOv() { while (ov.firstChild) ov.removeChild(ov.firstChild); ovFn = null; }
function overlay(title, body, label, fn) {
  clearOv(); box(ov, "0 0 0", { w: 1.8, h: 1.25, bg: "#0e1420", op: .93 });
  box(ov, "0 .5 .01", { w: 1.7, h: .16, text: title, color: "#ffd54a", size: 54 });
  box(ov, "0 .1 .01", { w: 1.7, h: .65, text: body, size: 34 });
  const b = box(ov, "0 -.5 .02", { w: 1.2, h: .2, text: label, bg: "#16a34a", size: 38 });
  b.classList.add("clickable"); b.addEventListener("click", fn); ovFn = fn;
}
addEventListener("keydown", e => { if (e.code === "Enter" && ovFn) ovFn(); });
const objective = () => G.phase === "out" ? "Reach the green ASSEMBLY POINT" : rigEl.object3D.position.z > -4 ? "Leave the room. Don’t stop for belongings!" : "Stay low, avoid fire, take the STAIRS";
setInterval(() => { if (hud) hud.setAttribute("ui-box", "text", G.phase === "menu" || G.phase === "done" ? "" : `❤ ${Math.max(0, Math.round(G.health))}%    ⏱ ${Math.floor((Date.now() - G.t0) / 1000)}s\n${objective()}\n${Date.now() < G.until ? G.msg : ""}`); }, 250);

function begin() {
  try { audio = audio || new (window.AudioContext || window.webkitAudioContext)(); } catch (e) {}
  Object.assign(G, { phase: "play", health: 100, bonus: 0, ext: false, t0: Date.now(), msg: "", until: 0 });
  rigEl.object3D.position.set(0, 0, 0); rigEl.object3D.rotation.set(0, 0, 0);
  fires.forEach(F => { F.on = true; F.g.setAttribute("visible", true); });
  laptop.setAttribute("visible", true); extEl.setAttribute("visible", true);
  clearOv(); startAlarm(); if (audio && !crackleG) startCrackle(); say("Alarm! Find the way out.", 4);
}
function end(win, why) {
  G.phase = "done"; stopAlarm(); const t = Math.floor((Date.now() - G.t0) / 1000);
  const score = win ? Math.max(0, Math.min(100, Math.round(40 + G.health * .4 + G.bonus + Math.max(0, 60 - t) * .3))) : 0;
  overlay(win ? "YOU ESCAPED" : "YOU DIDN’T MAKE IT",
    (win ? `Score ${score}/100   Health ${Math.round(G.health)}%   Time ${t}s\n` : `${why}\n`) +
    "Remember: leave at once, stay low under smoke, never use lifts, use stairs, go to the assembly point, never re-enter.", "Play again (Enter)", begin);
}

/* ---- Init ---- */
function init() {
  build(); rigEl.setAttribute("mover", "");
  hud = box(cam, "0 .62 -1.5", { w: 1.7, h: .34, text: "", bg: "#0e1420", op: .6, size: 32 });
  vig = el("a-plane", { position: "0 0 -.35", width: 1.6, height: 1.6, material: "shader:flat;transparent:true;opacity:0;depthWrite:false;depthTest:false;color:#fff", ptex: "kind:vig" }, cam);
  setInterval(() => { const L = emer && emer.getObject3D("light"); if (L) L.intensity = Math.random() < .25 ? .1 : .6; }, 180);
  ov = el("a-entity", { position: "0 0 -1.7" }, cam);
  document.getElementById("lh").addEventListener("thumbstickmoved", e => { AX[0] = Math.abs(e.detail.x) > .15 ? e.detail.x : 0; AX[1] = Math.abs(e.detail.y) > .15 ? e.detail.y : 0; });
  document.getElementById("rh").addEventListener("thumbstickmoved", e => { if (Math.abs(e.detail.x) > .7 && Date.now() > turnCool) { rigEl.object3D.rotation.y -= Math.sign(e.detail.x) * Math.PI / 6; turnCool = Date.now() + 350; } });
  overlay("PREPARE VR: FIRE ESCAPE", "A fire alarm is ringing. Escape the building alive.\nPC: WASD move, drag mouse to look, hold C to crouch, click objects.\nVR: left stick move, right stick turn, physically crouch.", "Start (Enter)", begin);
  note.style.display = "none";
}
scene.hasLoaded ? init() : scene.addEventListener("loaded", init);
</script>
</body>
</html>
