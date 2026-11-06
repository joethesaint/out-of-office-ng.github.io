<script>
  // OOO 0x04 Games Day: small polyfork props in the page margins.
  // Lean-scope note: this is NOT an always-running scene. It draws only while
  // the visitor scrolls (and ~1.2s after), obeys the site's motion switch and
  // reduced motion, and never takes pointer events. Models: Polyfork, CC BY 4.0.
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
  import * as SkeletonUtils from 'three/addons/utils/SkeletonUtils.js';
  import { motionPaused } from './motionPause.js';

  // Where each prop sits: a section, a side, and a vertical offset (fraction of
  // the section's height). Sizes are CSS pixels on desktop (d) / phone (m, 0 = hidden).
  const SPOTS = [
    { sel: '#about', side: 'right', y: 0.18, my: 0.05, kind: 'cards', d: 120, m: 56 },
    { sel: '#past-editions', side: 'left', y: 0.08, my: 0.0, kind: 'd20', d: 96, m: 44 },
    { sel: '#past-editions', side: 'right', y: 0.55, my: 0.6, kind: 'chip', d: 80, m: 40 },
    { sel: '#next-event', side: 'right', y: 0.1, my: -0.01, kind: 'hourglass', d: 110, m: 48 },
    { sel: '#next-event', side: 'right', y: 0.62, my: 0, kind: 'magnifier', d: 120, m: 0 },
    { sel: '#passes', side: 'left', y: 0.12, my: 0.02, kind: 'trophy', d: 100, m: 44 },
    { sel: '#passes', side: 'right', y: 0.7, my: 0.9, kind: 'dice', d: 110, m: 52 },
    { sel: '.playlist-section', side: 'right', y: 0.2, my: 0.02, kind: 'glass', d: 96, m: 0 },
    { sel: '.page-footer', side: 'right', y: -1.25, my: 0.08, kind: 'walkers', d: 150, m: 90 }
  ];
  const MODELS = {
    card: 'playing-card-f2f787', d20: 'd20-twenty-sided-die-038f10', chip: 'casino-chip-c0e3b6',
    hourglass: 'an-hourglass-with-sand-ec9f11', goblet: 'goblet-87c3ba', d6: 'd6-numeral-die-315b3a',
    cdie: 'casino-die-b25753', martini: 'martini-glass-c78705', student: 'graduate-student-945ef0', artist: 'graffiti-artist-2deb3d'
  };

  let canvas;

  onMount(() => {
    const reduce = matchMedia('(prefers-reduced-motion: reduce)');
    let paused = false;
    let raf = 0, awakeUntil = 0, lastY = scrollY, vel = 0, t = 0, last = performance.now();
    let renderer = null;

    renderer = new THREE.WebGLRenderer({ canvas, alpha: true, antialias: true });
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    const scene = new THREE.Scene();
    scene.add(new THREE.HemisphereLight(0xfff6e6, 0x5f7f7a, 1.7));
    const sun = new THREE.DirectionalLight(0xffeccc, 2.1); sun.position.set(3, 6, 5); scene.add(sun);
    const camera = new THREE.PerspectiveCamera(30, 1, 0.1, 100); camera.position.z = 20;
    const size = () => { renderer.setSize(innerWidth, innerHeight, false); camera.aspect = innerWidth / innerHeight; camera.updateProjectionMatrix(); wake(); };
    size();

    const loader = new GLTFLoader(); const cache = {};
    const get = async key => SkeletonUtils.clone(await (cache[key] ??= loader.loadAsync(`${import.meta.env.BASE_URL}models/${MODELS[key]}.glb`).then(g => g.scene)));
    const fit = (obj, s) => { const b = new THREE.Box3().setFromObject(obj), v = b.getSize(new THREE.Vector3()); const k = s / Math.max(v.x, v.y, v.z); obj.scale.setScalar(k); obj.position.sub(b.getCenter(new THREE.Vector3()).multiplyScalar(k)); const g = new THREE.Group(); g.add(obj); return g; };
    const gold = obj => { const m = new THREE.MeshStandardMaterial({ color: 0xe3ad48, metalness: .8, roughness: .32, flatShading: true }); obj.traverse(o => { if (o.isMesh) o.material = m; }); };
    const magnifier = () => { const g = new THREE.Group(); const brass = new THREE.MeshStandardMaterial({ color: 0xc8963a, metalness: .8, roughness: .35, flatShading: true });
      g.add(new THREE.Mesh(new THREE.TorusGeometry(.5, .07, 8, 24), brass));
      g.add(new THREE.Mesh(new THREE.CircleGeometry(.47, 24), new THREE.MeshStandardMaterial({ color: 0xbfe3ff, transparent: true, opacity: .35, side: THREE.DoubleSide })));
      const h = new THREE.Mesh(new THREE.CylinderGeometry(.08, .1, .75, 8), new THREE.MeshStandardMaterial({ color: 0x4a2c1a, flatShading: true })); h.position.y = -.98; g.add(h);
      g.rotation.z = .5; const w = new THREE.Group(); w.add(g); return w; };
    const bones = obj => { const b = {}; obj.traverse(o => { if (/^(Spine1|Head|LeftUpLeg|RightUpLeg|LeftLeg|RightLeg|LeftArm|RightArm)$/.test(o.name) && !b[o.name]) b[o.name] = { o, rest: o.rotation.clone() }; });
      if (b.LeftArm) { b.LeftArm.rest.z -= 1.25; } if (b.RightArm) { b.RightArm.rest.z += 1.25; } return b; };
    const walk = (b, t) => { const s = Math.sin(t), c = Math.cos(t); const set = (n, ax, v) => { const x = b[n]; if (x) x.o.rotation[ax] = x.rest[ax] + v; };
      set('LeftUpLeg', 'x', s * .45); set('RightUpLeg', 'x', -s * .45); set('LeftLeg', 'x', Math.max(0, -c) * .5); set('RightLeg', 'x', Math.max(0, c) * .5);
      set('LeftArm', 'x', -s * .35); set('RightArm', 'x', s * .35); };

    const items = [];
    (async () => {
      for (const spot of SPOTS) {
        const g = new THREE.Group(); const P = {};
        if (spot.kind === 'cards') for (let i = 0; i < 4; i++) { const p = new THREE.Group(); const c = fit(await get('card'), 1); c.position.y += .45; p.add(c); g.add(p); P['c' + i] = p; }
        if (spot.kind === 'd20') { P.a = fit(await get('d20'), 1); g.add(P.a); }
        if (spot.kind === 'chip') { P.a = fit(await get('chip'), .85); g.add(P.a); }
        if (spot.kind === 'hourglass') { P.a = fit(await get('hourglass'), 1); g.add(P.a); }
        if (spot.kind === 'magnifier') { P.a = fit(magnifier(), 1); g.add(P.a); }
        if (spot.kind === 'trophy') { const t = await get('goblet'); gold(t); P.a = fit(t, 1); g.add(P.a); }
        if (spot.kind === 'dice') { P.a = fit(await get('d6'), .5); P.b = fit(await get('cdie'), .5); P.a.position.x = -.3; P.b.position.x = .3; g.add(P.a, P.b); }
        if (spot.kind === 'glass') { P.a = fit(await get('martini'), 1); g.add(P.a); }
        if (spot.kind === 'walkers') for (const [i, k] of ['student', 'artist'].entries()) { const m = await get(k); const p = new THREE.Group(); p.add(fit(m, 1)); p.rotation.y = Math.PI / 2; g.add(p); P['w' + i] = { g: p, b: bones(m), off: i * 1.7 }; walk(P['w' + i].b, 0); }
        scene.add(g); items.push({ spot, g, P, spin: 0 });
      }
      wake();
    })();

    function frame(now) {
      raf = 0; const dt = Math.min(.05, (now - last) / 1000); last = now;
      const still = paused || reduce.matches;
      if (!still) t += dt;
      const dy = scrollY - lastY; lastY = scrollY; vel = vel * .85 + dy * .15; const v = still ? 0 : vel;
      const unit = (2 * Math.tan(THREE.MathUtils.degToRad(15)) * 20) / innerHeight; const phone = innerWidth < 720;
      for (const it of items) {
        const el = document.querySelector(it.spot.sel); if (!el) { it.g.visible = false; continue; }
        const r = el.getBoundingClientRect(); const px = phone ? it.spot.m : it.spot.d; if (!px) { it.g.visible = false; continue; }
        const gutter = Math.max(px * .6, (innerWidth - Math.min(innerWidth, 1120)) / 4 + px * .55);
        const cx = it.spot.side === 'left' ? (phone ? px * .3 : gutter) : innerWidth - (phone ? px * .3 : gutter);
        const cy = r.top + r.height * (phone ? it.spot.my : it.spot.y) + px / 2;
        it.g.visible = cy > -px && cy < innerHeight + px; if (!it.g.visible) continue;
        it.g.position.set((cx - innerWidth / 2) * unit, -(cy - innerHeight / 2) * unit, 0); it.g.scale.setScalar(px * unit);
        const prog = THREE.MathUtils.clamp(1 - cy / innerHeight, 0, 1); const P = it.P;
        switch (it.spot.kind) {
          case 'cards': for (let i = 0; i < 4; i++) { const c = P['c' + i]; if (!c) break; c.rotation.z = -(i - 1.5) * (.15 + prog * .35); c.rotation.y = Math.sin(t * .8 + i) * .25 + (i === 3 ? prog * Math.PI : 0); c.position.z = i * .02; } it.g.rotation.x = -.25; break;
          case 'd20': it.spin += .003 + Math.abs(v) * .004; P.a.rotation.set(it.spin * 1.3, it.spin, it.spin * .7); P.a.position.y = Math.abs(Math.sin(t * 1.6)) * .12; break;
          case 'chip': it.spin += .006 + Math.abs(v) * .006; P.a.rotation.set(Math.PI / 2.4, it.spin * 2, 0); break;
          case 'hourglass': P.a.rotation.z = prog * Math.PI; P.a.rotation.y = t * .3; break;
          case 'magnifier': P.a.position.set(Math.sin(t * .9) * .25, Math.sin(t * 1.8) * .12, 0); P.a.rotation.set(Math.sin(t * .7) * .25, Math.sin(t * .9) * .4, 0); break;
          case 'trophy': { const b = Math.abs(Math.sin(t * 2.4)); P.a.position.y = b * .28; P.a.scale.set(1 + (1 - b) * .06, 1 - (1 - b) * .08, 1 + (1 - b) * .06); P.a.rotation.y = t * .6; } break;
          case 'dice': it.spin += v * .012; P.a.rotation.set(it.spin * 1.2, it.spin * .8, it.spin); P.b.rotation.set(-it.spin, it.spin * 1.1, -it.spin * .6); P.a.position.y = Math.abs(Math.sin(t * 2.2)) * .1; P.b.position.y = Math.abs(Math.sin(t * 2.2 + 1)) * .1; break;
          case 'glass': P.a.rotation.z = Math.sin(t * 1.1) * .12; P.a.rotation.y = t * .25; break;
          case 'walkers': for (let i = 0; i < 2; i++) { const w = P['w' + i]; if (!w) break; w.g.position.x = ((t * .12 + w.off * .25) % 1.4) - .7; if (!still) walk(w.b, t * 5 + w.off); } break;
        }
      }
      renderer.render(scene, camera);
      if (!still && now < awakeUntil) raf = requestAnimationFrame(frame);
    }
    function wake() { awakeUntil = performance.now() + 1200; if (!raf && renderer) raf = requestAnimationFrame(frame); }
    const unsub = motionPaused.subscribe(v => { paused = v; wake(); });
    addEventListener('scroll', wake, { passive: true }); addEventListener('resize', size);
    return () => { unsub(); cancelAnimationFrame(raf); removeEventListener('scroll', wake); removeEventListener('resize', size); renderer.dispose(); };
  });
</script>

<canvas bind:this={canvas} class="game-props" aria-hidden="true"></canvas>

<style>
  .game-props { position: fixed; inset: 0; width: 100%; height: 100%; pointer-events: none; z-index: 40; }
</style>
