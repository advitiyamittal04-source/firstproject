<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Prepare VR – Emergency Response Trainer</title>
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
<div id="msg">Loading… (tip: keys 1, 2, 3 also pick the options)</div>
<a-scene background="color:#1b2230" renderer="antialias:true" cursor="rayOrigin:mouse" raycaster="objects:.clickable">
  <a-entity id="rig" position="0 0 0">
    <a-entity camera look-controls position="0 1.6 0">
      <a-entity id="panel" position="0 -.05 -1.8"></a-entity>
    </a-entity>
    <a-entity laser-controls="hand:left" raycaster="objects:.clickable"></a-entity>
    <a-entity laser-controls="hand:right" raycaster="objects:.clickable"></a-entity>
  </a-entity>
  <a-entity id="world"></a-entity>
</a-scene>

<script>
const msg = document.getElementById("msg");
window.onerror = (m, s, l) => { msg.style.display = "block"; msg.style.color = "#f87171"; msg.textContent = "Error: " + m + " (line " + l + ")"; };
if (!window.AFRAME) throw new Error("A-Frame failed to load. Check your internet connection.");

/* ---------- Text/box drawn on a canvas, so no external fonts are needed ---------- */
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

const world = document.getElementById("world"), panel = document.getElementById("panel"), rig = document.getElementById("rig");
const el = (tag, attrs, parent) => { const e = document.createElement(tag); for (const k in attrs) e.setAttribute(k, attrs[k]); (parent || world).appendChild(e); return e; };
const box = (parent, pos, o, rot) => { const e = el("a-entity", { position: pos, rotation: rot || "0 0 0" }, parent); e.setAttribute("ui-box", o); return e; };

/* ---------- SCENARIOS: add your own here. [label, isCorrect, feedback] ---------- */
const S = {
  fire: { name: "Office Fire", steps: [
    { q: "The fire alarm is ringing and you smell smoke. First action?", c: [
      ["Grab laptop and valuables", 0, "Belongings can be replaced. Every second in smoke counts."],
      ["Leave now and alert others", 1, "Correct. Get out first and warn people on the way."],
      ["Open a window and wait", 0, "Waiting traps you, and fresh air feeds the fire."]] },
    { q: "The corridor is filling with smoke. How do you move?", c: [
      ["Run upright as fast as possible", 0, "Hot, toxic smoke rises. Upright, you breathe the worst of it."],
      ["Stay low, cover nose and mouth", 1, "Correct. The cleanest air is close to the floor."],
      ["Hold your breath and sprint", 0, "You can’t hold it long enough, and may get disoriented."]] },
    { q: "You reach a lift and a stairwell. Which one?", c: [
      ["Take the lift, it’s faster", 0, "Lifts can stall or open onto burning floors."],
      ["Use the stairs", 1, "Correct. Stairwells are the designed escape route."],
      ["Wait by the lift for help", 0, "Staying put wastes the time you have."]] },
    { q: "You’re outside. A colleague says a friend is still inside.", c: [
      ["Go back in to find them", 0, "Never re-enter. Tell firefighters exactly where they were."],
      ["Go to the assembly point, inform the fire crew", 1, "Correct. Trained crews can rescue them far more safely."],
      ["Wait right next to the building", 0, "Stay clear of smoke and falling glass."]] }
  ]},
  quake: { name: "Earthquake", steps: [
    { q: "The ground starts shaking violently!", hook: () => { shake(4500); setTimeout(() => debris.setAttribute("visible", true), 1500); }, c: [
      ["Run for the exit", 0, "Moving during shaking causes many injuries. Falling objects are the main threat."],
      ["Drop, cover under the desk, hold on", 1, "Correct. Protect your head and neck until it stops."],
      ["Stand in a doorway", 0, "Outdated advice. Get under sturdy furniture instead."]] },
    { q: "The shaking stops. Aftershocks are possible. What next?", c: [
      ["Take the lift down", 0, "Never use lifts after a quake. Use the stairs."],
      ["Check for injuries and hazards, then move out", 1, "Correct. Be ready to drop and cover again."],
      ["Go back to work", 0, "Aftershocks and damaged structures make staying risky."]] },
    { q: "You smell gas near the kitchen.", c: [
      ["Light a match to find the leak", 0, "Never. Any spark can ignite gas."],
      ["Flip the lights on to see better", 0, "Switches can spark. Avoid all electrics."],
      ["Open windows, avoid switches, leave", 1, "Correct. Report it from a safe distance."]] },
    { q: "Outside, where do you stand?", c: [
      ["Beside the building wall", 0, "Walls and glass can still fall in aftershocks."],
      ["Under a tree or power pole", 0, "Lines and branches can fall."],
      ["Open ground, away from buildings and lines", 1, "Correct. Wait there for official instructions."]] }
  ]}
};

/* ---------- State ---------- */
let sc = null, step = 0, score = 0, danger = 0, t0 = 0, audio, alarmTimer, shakeTimer, hud, actions = [], smoke = [], fireG, debris;

/* ---------- Audio (starts after the first click) ---------- */
function beep(f, d, type, v) { const o = audio.createOscillator(), g = audio.createGain(); o.type = type; o.frequency.value = f; g.gain.value = v; o.connect(g); g.connect(audio.destination); o.start(); o.stop(audio.currentTime + d); }
function stopAlarm() { clearInterval(alarmTimer); }
function startAlarm(k) {
  stopAlarm(); if (!audio) return; let n = 0;
  alarmTimer = setInterval(() => k === "fire" ? beep(n++ % 2 ? 660 : 880, .25, "square", .04) : beep(60 + Math.random() * 20, .3, "sawtooth", .15), k === "fire" ? 400 : 250);
}
function shake(ms) {
  clearInterval(shakeTimer); const end = Date.now() + ms;
  shakeTimer = setInterval(() => {
    if (Date.now() > end) { clearInterval(shakeTimer); rig.setAttribute("position", "0 0 0"); stopAlarm(); return; }
    rig.setAttribute("position", `${(Math.random() - .5) * .1} ${(Math.random() - .5) * .06} ${(Math.random() - .5) * .1}`);
  }, 40);
}

/* ---------- World: you start in the room facing the exit ---------- */
function build() {
  el("a-plane", { rotation: "-90 0 0", width: 60, height: 60, color: "#3a3f47" });
  const wall = (p, w, h, d) => el("a-box", { position: p, width: w, height: h, depth: d, color: "#8d939c" });
  wall("0 1.5 4", 8, 3, .2); wall("-4 1.5 0", .2, 3, 8); wall("4 1.5 0", .2, 3, 8);
  wall("-2.5 1.5 -4", 3, 3, .2); wall("2.5 1.5 -4", 3, 3, .2); wall("0 2.6 -4", 2, .8, .2);
  el("a-box", { position: "0 3.1 0", width: 8, height: .2, depth: 8, color: "#6b7078" });
  el("a-box", { position: "2 .4 -1.5", width: 1.6, height: .8, depth: .8, color: "#6b4f2f" });
  box(world, "0 2.95 -3.85", { w: 1.2, h: .3, text: "EXIT", bg: "#16a34a", size: 60 });
  box(world, "-3.85 1.4 0", { w: 1.6, h: .7, text: "LIFT – do not use", bg: "#555b63", color: "#ffb4b4", size: 44 }, "0 90 0");
  box(world, "0 1.6 -13.8", { w: 3, h: 1.4, text: "ASSEMBLY POINT", bg: "#16a34a", size: 80 });
  fireG = el("a-entity", { position: "-2.5 0 -2.5" });
  [[0, 0], [.4, .2], [-.3, .3]].forEach(([x, z], i) => el("a-cone", { position: `${x} .5 ${z}`, "radius-bottom": .35 - i * .05, "radius-top": 0, height: 1 + i * .3, material: `color:${i % 2 ? "#ff9800" : "#ff3d00"};emissive:#ff5a00;emissiveIntensity:1`, animation: `property:scale;to:1 1.4 1;dir:alternate;loop:true;dur:${250 + i * 80}` }, fireG));
  el("a-light", { type: "point", color: "#ff6a00", intensity: 1.4, distance: 8, position: "0 1 0", animation: "property:intensity;from:.8;to:1.8;dir:alternate;loop:true;dur:200" }, fireG);
  for (let i = 0; i < 16; i++) smoke.push(el("a-sphere", { position: `${(Math.random() - .5) * 7} ${2.2 + Math.random() * .7} ${(Math.random() - .5) * 7}`, radius: 1 + Math.random() * .6, material: "color:#444;opacity:0;transparent:true;depthWrite:false" }));
  debris = el("a-entity", { visible: false });
  for (let i = 0; i < 9; i++) el("a-box", { position: `${(Math.random() - .5) * 6} .1 ${(Math.random() - .5) * 6}`, rotation: `${Math.random() * 40} ${Math.random() * 180} ${Math.random() * 40}`, width: .3 + Math.random() * .4, height: .2, depth: .3 + Math.random() * .4, color: "#5c5f66" }, debris);
  el("a-light", { type: "ambient", intensity: .8 }); el("a-light", { type: "directional", position: "2 4 2", intensity: .6 });
}
function setDanger(n) { smoke.forEach(s => s.setAttribute("material", "opacity", sc === S.fire ? Math.min(.12 + n * .16, .7) : 0)); }
setInterval(() => { if (hud && sc) hud.setAttribute("ui-box", "text", `Score ${score}   Step ${Math.min(step + 1, sc.steps.length)}/${sc.steps.length}   Time ${Math.floor((Date.now() - t0) / 1000)}s`); }, 500);

/* ---------- UI panel (head-locked; click with mouse, VR controller ray, or keys 1-3) ---------- */
function base(title, color) {
  while (panel.firstChild) panel.removeChild(panel.firstChild);
  actions = [];
  box(panel, "0 0 0", { w: 1.8, h: 1.45, bg: "#0e1420", op: .9 });
  box(panel, "0 .56 .01", { w: 1.7, h: .16, text: title, color: color || "#ffd54a", size: 60 });
  hud = box(panel, "0 -.66 .01", { w: 1.7, h: .08, text: "", color: "#9fb3c8", size: 26 });
}
const say = t => box(panel, "0 .2 .01", { w: 1.7, h: .5, text: t, size: 44 });
function button(label, y, fn, color) {
  color = color || "#2a5bd7";
  const b = box(panel, `0 ${y} .02`, { w: 1.7, h: .19, text: label, bg: color, size: 34 });
  b.classList.add("clickable");
  b.addEventListener("click", fn);
  b.addEventListener("mouseenter", () => b.setAttribute("ui-box", "bg", "#4f86ff"));
  b.addEventListener("mouseleave", () => b.setAttribute("ui-box", "bg", color));
  actions.push(fn);
}
window.addEventListener("keydown", e => { const f = actions[parseInt(e.key) - 1]; if (f) f(); });

/* ---------- Flow ---------- */
function menu() {
  stopAlarm(); sc = null; fireG.setAttribute("visible", false); debris.setAttribute("visible", false); setDanger(0);
  base("PREPARE VR");
  say("We cannot predict the unexpected, but we can prepare for it.\nChoose a scenario:");
  button("1. Office Fire", -.1, () => start("fire"));
  button("2. Earthquake", -.31, () => start("quake"));
}
function start(k) {
  try { audio = audio || new (window.AudioContext || window.webkitAudioContext)(); } catch (e) {}
  sc = S[k]; step = 0; score = 0; danger = 0; t0 = Date.now();
  fireG.setAttribute("visible", k === "fire"); debris.setAttribute("visible", false); setDanger(0);
  startAlarm(k); show();
}
function show() {
  const s = sc.steps[step]; if (s.hook) s.hook();
  base(sc.name); say(s.q);
  [...s.c].sort(() => Math.random() - .5).forEach((c, i) => button(`${i + 1}. ${c[0]}`, -.1 - i * .21, () => answer(c)));
}
function answer(c) {
  if (c[1]) score += 25; else setDanger(++danger);
  base(c[1] ? "✔ Correct" : "✖ Not quite", c[1] ? "#4ade80" : "#f87171");
  say(c[2]);
  button(step + 1 < sc.steps.length ? "1. Continue" : "1. See results", -.3, () => { step++; step < sc.steps.length ? show() : finish(); }, "#16a34a");
}
function finish() {
  stopAlarm(); const pct = Math.round(score / (sc.steps.length * 25) * 100);
  base("Debrief: " + sc.name);
  say(`Score: ${score}/${sc.steps.length * 25} (${pct}%)   Time: ${Math.floor((Date.now() - t0) / 1000)}s\n` +
    (pct === 100 ? "Excellent. You stayed calm and made the right call every time." : pct >= 50 ? "Good effort. Review the missed steps and try again." : "Keep practising. Repetition builds calm, automatic responses."));
  button("1. Play again", -.3, menu, "#16a34a");
}

const scene = document.querySelector("a-scene");
const init = () => { build(); menu(); msg.style.display = "none"; };
scene.hasLoaded ? init() : scene.addEventListener("loaded", init);
</script>
</body>
</html>
