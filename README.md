<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Prepare VR – Emergency Response Trainer</title>
<script src="https://cdn.jsdelivr.net/npm/aframe@1.5.0/dist/aframe.min.js"></script>
<style>:root{color-scheme:light dark}html,body{margin:0;height:100%;background:#0b0d10}</style>
</head>
<body>
<a-scene background="color:#1b2230" renderer="antialias:true" cursor="rayOrigin:mouse" raycaster="objects:.clickable">
  <a-entity id="rig" position="0 0 0">
    <a-entity camera look-controls position="0 1.6 0">
      <a-entity id="panel" position="0 0 -1.7"></a-entity>
    </a-entity>
    <a-entity laser-controls="hand:left" raycaster="objects:.clickable"></a-entity>
    <a-entity laser-controls="hand:right" raycaster="objects:.clickable"></a-entity>
  </a-entity>
  <a-entity id="world"></a-entity>
</a-scene>

<script>
const panel = document.getElementById("panel"), world = document.getElementById("world"), rig = document.getElementById("rig");
const el = (tag, attrs, parent) => { const e = document.createElement(tag); for (const k in attrs) e.setAttribute(k, attrs[k]); (parent || world).appendChild(e); return e; };

/* ---------- SCENARIOS: add your own here ---------- */
const S = {
  fire: { name: "Office Fire", steps: [
    { q: "The fire alarm is ringing and you smell smoke. First action?", c: [
      ["Grab laptop and valuables", 0, "Belongings can be replaced. Every second in smoke counts."],
      ["Leave now and alert others", 1, "Correct. Get out first and warn people on the way."],
      ["Open a window and wait", 0, "Waiting traps you, and fresh air feeds the fire."]] },
    { q: "The corridor is filling with smoke. How do you move?", c: [
      ["Run upright as fast as possible", 0, "Hot, toxic smoke rises. Upright, you breathe the worst of it."],
      ["Stay low, cover nose and mouth", 1, "Correct. The cleanest air is close to the floor."],
      ["Hold your breath and sprint", 0, "You can’t hold it long enough, and you may get disoriented."]] },
    { q: "You reach a lift and a stairwell. Which one?", c: [
      ["Take the lift, it’s faster", 0, "Lifts can stall or open onto burning floors."],
      ["Use the stairs", 1, "Correct. Stairwells are the safe, designed escape route."],
      ["Wait by the lift for help", 0, "Staying put wastes the time you have."]] },
    { q: "You’re outside. A colleague says a friend is still inside.", c: [
      ["Go back in to find them", 0, "Never re-enter. Tell firefighters exactly where they were."],
      ["Go to the assembly point and inform the fire crew", 1, "Correct. Trained crews can rescue them far more safely."],
      ["Wait right next to the building", 0, "Stay clear of smoke and falling glass."]] }
  ]},
  quake: { name: "Earthquake", steps: [
    { q: "The ground starts shaking violently!", hook: () => { shake(4500); setTimeout(() => debris.setAttribute("visible", true), 1500); }, c: [
      ["Run for the exit", 0, "Moving during shaking is how many injuries happen. Falling objects are the main threat."],
      ["Drop, cover under the desk, hold on", 1, "Correct. Protect your head and neck and stay put until it stops."],
      ["Stand in a doorway", 0, "Outdated advice. Get under sturdy furniture instead."]] },
    { q: "The shaking stops. Aftershocks are possible. What next?", c: [
      ["Take the lift down", 0, "Never use lifts after a quake. Use the stairs."],
      ["Check for injuries and hazards, then move out", 1, "Correct. Be ready to drop and cover again."],
      ["Go back to work", 0, "Aftershocks and damaged structure make staying risky."]] },
    { q: "You smell gas near the kitchen.", c: [
      ["Light a match to find the leak", 0, "Never! Any spark can ignite gas."],
      ["Flip the lights on to see better", 0, "Switches can spark. Avoid all electrics."],
      ["Open windows, don’t touch switches, leave", 1, "Correct. Report it from a safe distance."]] },
    { q: "Outside, where do you stand?", c: [
      ["Beside the building wall", 0, "Walls and glass can still fall in aftershocks."],
      ["Under a tree or power pole", 0, "Lines and branches can fall."],
      ["Open ground, away from buildings and lines", 1, "Correct. Wait there for official instructions."]] }
  ]}
};

/* ---------- STATE ---------- */
let sc, step = 0, score = 0, danger = 0, t0 = 0, audio, alarmTimer, shakeTimer, hud, smoke = [], fireG, debris;

/* ---------- AUDIO ---------- */
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
    rig.setAttribute("position", `${(Math.random() - .5) * .12} ${(Math.random() - .5) * .08} ${(Math.random() - .5) * .12}`);
  }, 40);
}

/* ---------- WORLD ---------- */
function build() {
  el("a-plane", { rotation: "-90 0 0", width: 40, height: 40, color: "#3a3f47" });
  const wall = (p, w, h, d) => el("a-box", { position: p, width: w, height: h, depth: d, color: "#8d939c" });
  wall("0 1.5 -4", 8, 3, .2); wall("-4 1.5 0", .2, 3, 8); wall("4 1.5 0", .2, 3, 8);
  wall("-2.5 1.5 4", 3, 3, .2); wall("2.5 1.5 4", 3, 3, .2); wall("0 2.6 4", 2, .8, .2);
  el("a-box", { position: "0 3.1 0", width: 8, height: .2, depth: 8, color: "#6b7078" });
  el("a-box", { position: "-2 .4 -2", width: 1.6, height: .8, depth: .8, color: "#6b4f2f" });
  const sign = el("a-box", { position: "0 2.95 3.85", width: 1.2, height: .3, depth: .05, color: "#16a34a", material: "emissive:#16a34a;emissiveIntensity:.8" });
  el("a-text", { value: "EXIT", align: "center", width: 3, position: "0 0 .04", color: "#fff" }, sign);
  const lift = el("a-box", { position: "3.85 1.2 0", width: .05, height: 2.4, depth: 1.4, color: "#555b63" });
  el("a-text", { value: "LIFT\nDo not use", align: "center", width: 2, position: "-.04 0 0", rotation: "0 -90 0", color: "#ff8a8a" }, lift);
  const asm = el("a-box", { position: "0 1.2 14", width: 3, height: 2.4, depth: .1, color: "#16a34a" });
  el("a-text", { value: "ASSEMBLY\nPOINT", align: "center", width: 4, position: "0 0 -.06", rotation: "0 180 0", color: "#fff" }, asm);
  /* fire */
  fireG = el("a-entity", { position: "2.5 0 -2.5" });
  [[0, 0], [.4, .2], [-.3, .3]].forEach(([x, z], i) => el("a-cone", { position: `${x} .5 ${z}`, radius: .35 - i * .05, height: 1 + i * .3, color: i % 2 ? "#ff9800" : "#ff3d00", material: "emissive:#ff5a00;emissiveIntensity:1", animation: `property:scale;to:1 1.4 1;dir:alternate;loop:true;dur:${250 + i * 80}` }, fireG));
  el("a-light", { type: "point", color: "#ff6a00", intensity: 1.4, distance: 8, position: "0 1 0", animation: "property:intensity;from:.8;to:1.8;dir:alternate;loop:true;dur:200" }, fireG);
  /* smoke near ceiling */
  for (let i = 0; i < 16; i++) smoke.push(el("a-sphere", { position: `${(Math.random() - .5) * 7} ${2.2 + Math.random() * .7} ${(Math.random() - .5) * 7}`, radius: 1 + Math.random() * .6, color: "#444", material: "opacity:0;transparent:true;depthWrite:false" }));
  /* quake debris */
  debris = el("a-entity", { visible: false });
  for (let i = 0; i < 9; i++) el("a-box", { position: `${(Math.random() - .5) * 6} .1 ${(Math.random() - .5) * 6}`, rotation: `${Math.random() * 40} ${Math.random() * 180} ${Math.random() * 40}`, width: .3 + Math.random() * .4, height: .2, depth: .3 + Math.random() * .4, color: "#5c5f66" }, debris);
  el("a-light", { type: "ambient", intensity: .7 }); el("a-light", { type: "directional", position: "2 4 2", intensity: .5 });
}
function setDanger(n) { smoke.forEach(s => s.setAttribute("material", "opacity", sc && sc === S.fire ? Math.min(.12 + n * .16, .7) : 0)); }
setInterval(() => { if (hud && sc) hud.setAttribute("value", `Score ${score}   Step ${Math.min(step + 1, sc.steps.length)}/${sc.steps.length}   Time ${Math.floor((Date.now() - t0) / 1000)}s`); }, 500);

/* ---------- UI (head-locked panel, clickable with mouse or VR controller ray) ---------- */
const clear = () => { while (panel.firstChild) panel.removeChild(panel.firstChild); };
const text = (v, pos, w, color) => el("a-text", { value: v, position: pos, width: w, align: "center", color: color || "#fff", "wrap-count": 38, anchor: "center" }, panel);
function base(title, color) {
  clear(); el("a-plane", { width: 1.9, height: 1.5, color: "#0e1420", opacity: .92 }, panel);
  text(title, "0 .6 .01", 1.7, color || "#ffd54a");
  hud = text("", "0 -.68 .01", 1.5, "#9fb3c8");
}
function button(label, y, fn, color) {
  color = color || "#2a5bd7";
  const b = el("a-plane", { class: "clickable", position: `0 ${y} .02`, width: 1.7, height: .2, color }, panel);
  el("a-text", { value: label, width: 1.5, align: "center", color: "#fff", "wrap-count": 44, position: "0 0 .01" }, b);
  b.addEventListener("click", fn);
  b.addEventListener("mouseenter", () => b.setAttribute("color", "#4f86ff"));
  b.addEventListener("mouseleave", () => b.setAttribute("color", color));
}

/* ---------- FLOW ---------- */
function menu() {
  stopAlarm(); sc = null; fireG.setAttribute("visible", false); debris.setAttribute("visible", false); setDanger(0);
  base("PREPARE VR");
  text("We cannot predict the unexpected,\nbut we can prepare for it.\n\nChoose a scenario:", "0 .2 .01", 1.6);
  button("Office Fire", -.15, () => start("fire"));
  button("Earthquake", -.4, () => start("quake"));
}
function start(k) {
  if (!audio) audio = new (window.AudioContext || window.webkitAudioContext)();
  sc = S[k]; step = 0; score = 0; danger = 0; t0 = Date.now();
  fireG.setAttribute("visible", k === "fire"); debris.setAttribute("visible", false); setDanger(0);
  startAlarm(k); show();
}
function show() {
  const s = sc.steps[step]; if (s.hook) s.hook();
  base(sc.name);
  text(s.q, "0 .25 .01", 1.6);
  [...s.c].sort(() => Math.random() - .5).forEach((c, i) => button(c[0], -.05 - i * .24, () => answer(c)));
}
function answer(c) {
  if (c[1]) score += 25; else setDanger(++danger);
  base(c[1] ? "✔ Correct" : "✖ Not quite", c[1] ? "#4ade80" : "#f87171");
  text(c[2], "0 .2 .01", 1.6);
  button(step + 1 < sc.steps.length ? "Continue" : "See results", -.3, () => { step++; step < sc.steps.length ? show() : finish(); }, "#16a34a");
}
function finish() {
  stopAlarm(); const pct = Math.round(score / (sc.steps.length * 25) * 100);
  base("Debrief: " + sc.name);
  text(`Score: ${score}/${sc.steps.length * 25} (${pct}%)\nTime: ${Math.floor((Date.now() - t0) / 1000)}s\n\n` +
    (pct === 100 ? "Excellent. You stayed calm and made the right call every time." : pct >= 50 ? "Good effort. Review the missed steps and try again." : "Keep practising. Repetition builds calm, automatic responses."), "0 .2 .01", 1.6);
  button("Play again", -.3, menu, "#16a34a");
}

const scene = document.querySelector("a-scene");
const init = () => { build(); menu(); };
scene.hasLoaded ? init() : scene.addEventListener("loaded", init);
</script>
</body>
</html>
