<script>
  import { onMount, onDestroy } from "svelte";
  import { scale } from "svelte/transition";
  import { cubicOut } from "svelte/easing";
  import RotatingCube from "./lib/RotatingCube.svelte";
  import HeaderBar from "./lib/HeaderBar.svelte";
  import FooterBar from "./lib/FooterBar.svelte";
  import BootSequence from "./lib/BootSequence.svelte";
  import Postcard from "./lib/Postcard.svelte";
  import MemoryReel from "./lib/MemoryReel.svelte";
  import EscapeMetrics from "./lib/EscapeMetrics.svelte";
  import Community from "./lib/Community.svelte";
  import MemoryTimeline from "./lib/MemoryTimeline.svelte";
  import Playlist from "./lib/Playlist.svelte";
  import Tickets from "./lib/Tickets.svelte";
  import DanfoBus from "./lib/DanfoBus.svelte";
  import ZineDecorations from "./lib/ZineDecorations.svelte";
  import ScrollReveal from "./lib/ScrollReveal.svelte";
  import Boat from "./lib/Boat.svelte";
  import ChaosLayer from "./lib/ChaosLayer.svelte";
  import StressMeter from "./lib/StressMeter.svelte";
  import AmbientSound from "./lib/AmbientSound.svelte";
  import AboutEvent from "./lib/AboutEvent.svelte";
  import ToastSystem from "./lib/ToastSystem.svelte";
  import RsvpDrawer from "./lib/RsvpDrawer.svelte";
  import CommandPalette from "./lib/CommandPalette.svelte";
  import OooGeneratorModal from "./lib/OooGeneratorModal.svelte";
  import ScheduleFAQ from "./lib/ScheduleFAQ.svelte";
  import EventTrail from "./lib/EventTrail.svelte";
  import CountdownTimer from "./lib/CountdownTimer.svelte";
  import ScrollToTop from "./lib/ScrollToTop.svelte";
  import { clearAllToasts } from "./lib/toastStore.js";
  import dropletBlue from "../docs/brand-reference/paint-droplet-blue.png";
  import dropletPink from "../docs/brand-reference/paint-droplet-pink.png";
  import { pageProgress } from "./lib/scrollProgress.js";

  let isDrawerOpen = false;
  let isCmdKOpen = false;
  let isOooGenOpen = false;

  function openDrawer() { isDrawerOpen = true; }
  function closeDrawer() { isDrawerOpen = false; }

  function openCmdK() { isCmdKOpen = true; }
  function closeCmdK() { isCmdKOpen = false; }

  function openOooGen() { isOooGenOpen = true; }
  function closeOooGen() { isOooGenOpen = false; }

  let currentRoute = typeof window !== 'undefined' ? window.location.hash || '#/' : '#/';
  function onHashChange() {
    currentRoute = window.location.hash || '#/';
  }

  let scrollTrack;
  let progress = 0;
  let smoothedProgress = 0;
  let ticking = false;
  let smoothRafId;
  let prefersReducedMotion = false;

  // Two-key Easter Egg (type 'oo')
  const KONAMI_CODE = ['o', 'o'];
  let konamiIndex = 0;
  let showStats = false;
  let statsTimer;

  function onKeyDown(e) {
    if (e.key === KONAMI_CODE[konamiIndex]) {
      konamiIndex++;
      if (konamiIndex === KONAMI_CODE.length) {
        showStats = true;
        konamiIndex = 0;
        clearTimeout(statsTimer);
        statsTimer = setTimeout(() => showStats = false, 6000);
      }
    } else {
      konamiIndex = 0;
    }
  }

  let lastSmoothTime = 0;
  function smoothTick(now) {
    const dt = lastSmoothTime ? now - lastSmoothTime : 16;
    lastSmoothTime = now;
    const delta = progress - smoothedProgress;
    const factor = 1 - Math.exp(-dt / SMOOTH_TAU_MS);
    if (Math.abs(delta) < 0.0005) {
      smoothedProgress = progress;
      smoothRafId = null;
      return;
    }
    smoothedProgress += delta * factor;
    smoothRafId = requestAnimationFrame(smoothTick);
  }

  let isScrolledPast80 = false;
  let isScrolledPast500 = false;
  let showStickyCta = false;
  let ticketsEl;

  function readProgress() {
    isScrolledPast80 = typeof window !== 'undefined' && window.scrollY > 80;
    isScrolledPast500 = typeof window !== 'undefined' && window.scrollY > 500;
    showStickyCta = typeof window !== 'undefined' && window.scrollY > window.innerHeight * 0.8;

    if (!scrollTrack) return;
    const rect = scrollTrack.getBoundingClientRect();
    const total = rect.height - window.innerHeight;
    progress = total <= 0 ? 1 : Math.max(0, Math.min(1, -rect.top / total));
    if (prefersReducedMotion) {
      smoothedProgress = progress;
    } else if (!smoothRafId) {
      lastSmoothTime = 0;
      smoothRafId = requestAnimationFrame(smoothTick);
    }

    // Whole-document scroll fraction, for the corporate->handwritten
    // typography morph (concept.txt) — separate from the hero-only
    // `progress` above, piggybacked on the same rAF batch.
    const docTotal = document.documentElement.scrollHeight - window.innerHeight;
    pageProgress.set(docTotal <= 0 ? 0 : Math.max(0, Math.min(1, window.scrollY / docTotal)));

    ticking = false;
  }

  function updateProgress() {
    if (ticking) return;
    ticking = true;
    requestAnimationFrame(readProgress);
  }

  const SMOOTH_TAU_MS = 90;

  onMount(() => {
    prefersReducedMotion = window.matchMedia("(prefers-reduced-motion: reduce)").matches;
    updateProgress();
    window.addEventListener("scroll", updateProgress, { passive: true });
    window.addEventListener("resize", updateProgress);
    window.addEventListener("keydown", onKeyDown);
    if (!prefersReducedMotion) {
      smoothRafId = requestAnimationFrame(smoothTick);
    }
  });
  onDestroy(() => {
    window.removeEventListener("scroll", updateProgress);
    window.removeEventListener("resize", updateProgress);
    window.removeEventListener("keydown", onKeyDown);
    clearTimeout(statsTimer);
    clearTimeout(shareToastTimer);
    if (smoothRafId) cancelAnimationFrame(smoothRafId);
  });

  // Custom eased scroll for the "Activate Auto Reply ->" CTA — deliberately
  // slower than native smooth-scroll so the cube-solve narrative is glimpsed
  // in transit rather than skipped in one jump. Falls back to an instant
  // jump under prefers-reduced-motion.
  function easeInOutCubic(t) {
    return t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2;
  }
  let scrollAnimId = 0;
  function easedScrollTo(targetY, duration = 1400) {
    if (prefersReducedMotion) {
      window.scrollTo(0, targetY);
      return;
    }
    const animId = ++scrollAnimId;
    const startY = window.scrollY;
    const distance = targetY - startY;
    const startTime = performance.now();
    function step(now) {
      if (animId !== scrollAnimId) return;
      const t = Math.min(1, (now - startTime) / duration);
      window.scrollTo(0, startY + distance * easeInOutCubic(t));
      if (t < 1) requestAnimationFrame(step);
    }
    requestAnimationFrame(step);
  }

  function onActivateClick() {
    const ticketsSection = document.querySelector('.tickets-section');
    if (ticketsSection) {
      ticketsSection.scrollIntoView({ behavior: 'smooth' });
    } else if (scrollTrack) {
      const rect = scrollTrack.getBoundingClientRect();
      easedScrollTo(window.scrollY + rect.top + window.innerHeight * 1.5);
    }
  }

  let shareConfirmed = false;
  let shareToastTimer;
  async function handleShare() {
    const shareData = {
      title: "Out of Office",
      text: "Auto-reply for real life. Leaving yellow Lagos, entering blue Lagos.",
      url: window.location.href,
    };
    if (navigator.share) {
      try { await navigator.share(shareData); } catch { /* user cancelled */ }
      return;
    }
    if (navigator.clipboard) {
      await navigator.clipboard.writeText(shareData.url);
      shareConfirmed = true;
      clearTimeout(shareToastTimer);
      shareToastTimer = setTimeout(() => { shareConfirmed = false; }, 2000);
    }
  }

  let isOnline = true;

  function handleStatusChange(onlineState) {
    isOnline = onlineState;
    if (isOnline) {
      try {
        sessionStorage.removeItem("oooChaosDismissed");
      } catch {}
      // Restore all chaos popups + toasts
      window.dispatchEvent(new CustomEvent('oooStatusOnline'));
    } else {
      // Going AWAY — kill every popup and toast immediately
      clearAllToasts();
      window.dispatchEvent(new CustomEvent('oooStatusAway'));
    }
  }

  function handleArrowClick() {
    const el = document.getElementById('open-canvas-event');
    if (el) {
      el.scrollIntoView({ behavior: 'smooth' });
    }
  }

  $: activated = progress >= 0.995;

  $: notificationCount = isOnline
    ? "999+"
    : smoothedProgress < 0.05
      ? "999+"
      : smoothedProgress >= 0.95
        ? 0
        : Math.floor(999 * Math.pow(1 - (smoothedProgress - 0.05) / 0.9, 3));

  // Chaos -> calm color arc for the notification pill
  $: notifStage = notificationCount === 0 ? "zero" : (smoothedProgress < 0.25 || isOnline) ? "high" : "mid";
</script>

<svelte:window on:hashchange={onHashChange} on:keydown={onKeyDown} />

{#if currentRoute === '#/about'}
  <AboutEvent />
{:else if currentRoute === '#/trail'}
  <EventTrail />
{:else}

<BootSequence />

<HeaderBar
  scrollState={isScrolledPast80 ? 'frosted' : 'transparent'}
  onOpenDrawer={openDrawer}
  onOpenOooGen={openOooGen}
  onStatusChange={handleStatusChange}
/>

<div class="scroll-track" bind:this={scrollTrack}>
  <div class="pinned">
    <div class="stage-wrap">
      <div class="grain"></div>
      <!-- Chaos Layer chat bubbles outside of the main postcard card to frame the digital noise around our escape -->
      <ChaosLayer progress={smoothedProgress} forceOnline={isOnline} />
      <main class="frame">
        <ZineDecorations />

        <div class="hero">
          <DanfoBus progress={smoothedProgress} />

          <button class="icon-btn icon-share" aria-label="Scroll to Open Canvas event" on:click={handleArrowClick} title="Scroll to Open Canvas event (May 30)">
            <svg viewBox="0 0 24 24"
              ><path
                fill="none"
                stroke="#181818"
                stroke-width="2.2"
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M4 15c0-5 3-8 8-8m0 0V3l6 5-6 5V9"
              /></svg
            >
          </button>
          {#if shareConfirmed}
            <span class="share-toast" role="status">Link copied</span>
          {/if}

          <div class="hero-row">
            <div class="headline">
              <div class="notification-pill" data-stage={notifStage}>
                <span class="bell-icon">🔔</span>
                <span class="notif-label">Pending Notifications:</span>
                <span class="notif-count">{notificationCount === 0 ? "0 (Muted ✓)" : notificationCount}</span>
              </div>
              <span class="eyebrow">
                <img class="paint-splat paint-splat-blue" src={dropletBlue} alt="" aria-hidden="true" width="489" height="404" />
                <img class="paint-splat paint-splat-pink" src={dropletPink} alt="" aria-hidden="true" width="394" height="407" />
                Auto-reply for real life</span
              >
              <h1 class="stack">
                <span class="word">OUT</span>
                <span class="of">of</span>
                <span class="word">OFFICE</span>
              </h1>
              <div class="subhead">
                <span class="stamp">OOO 0x03</span>
                <span class="venue">Tarkwa Bay · Aug 15</span>
              </div>
              <CountdownTimer targetDateStr="August 15, 2026 12:00:00" />
              <div class="tagline">
                <span>Release. Unwind. Reconnect.</span>
                <span class="sub">Auto replies enabled. Stress disabled.</span>
              </div>
              <div class="hero-cta-wrap">
                <button class="activate-cta" on:click={onActivateClick}>
                  Activate Auto Reply →
                </button>
              </div>
            </div>
            <div class="cube-slot">
              <RotatingCube {progress} />
            </div>
          </div>
          <Boat progress={smoothedProgress} />
        </div>

        <FooterBar />
      </main>
    </div>
  </div>
</div>

{#if activated}
  <div class="cube-companion" in:scale={{ duration: 550, start: 0.4, opacity: 0, easing: cubicOut }}>
    <RotatingCube progress={1} randomSolve={true} />
  </div>
{/if}

<StressMeter progress={smoothedProgress} {activated} />
<AmbientSound />

<section class="activated" class:visible={activated}>
  <p class="activated-eyebrow">Status update</p>
  <h2 class="activated-title">Out of Office Activated</h2>
  <p class="activated-sub">You've found a little order and peace.</p>
  <p class="activated-note">
    Life is messy. Like a scrambled cube. Out of Office is where we stop
    trying to solve everything for a moment.
  </p>
</section>

<ScrollReveal let:visible><Postcard {visible} /></ScrollReveal>
<ScrollReveal let:visible><MemoryReel {visible} /></ScrollReveal>
<ScrollReveal let:visible><EscapeMetrics {visible} /></ScrollReveal>
<ScrollReveal let:visible><Community {visible} /></ScrollReveal>
<ScrollReveal let:visible><MemoryTimeline {visible} /></ScrollReveal>
<ScrollReveal let:visible><Playlist {visible} /></ScrollReveal>
<ScrollReveal let:visible><Tickets {visible} showSticky={showStickyCta} onOpenDrawer={openDrawer} /></ScrollReveal>

<ScheduleFAQ />

<RsvpDrawer isOpen={isDrawerOpen} onClose={closeDrawer} />
<CommandPalette isOpen={isCmdKOpen} onClose={closeCmdK} onOpenDrawer={openDrawer} onOpenOooGen={openOooGen} />
<OooGeneratorModal isOpen={isOooGenOpen} onClose={closeOooGen} />
<ScrollToTop />
<ToastSystem />

<div class="stats-toast" class:visible={showStats} role="status">
  <p class="stats-title">Lagos Survival Stats</p>
  <ul>
    <li>Traffic avoided: <strong>3 hours</strong></li>
    <li>Emails ignored: <strong>17</strong></li>
    <li>Stress reduced: <strong>68%</strong></li>
  </ul>
</div>

{/if}

<style>
  .scroll-track {
    position: relative;
    height: 150vh;
  }

  .pinned {
    position: sticky;
    top: 0;
    height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
  }

  .stage-wrap {
    position: relative;
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 2rem 1rem;
    overflow: hidden;
  }

  .grain {
    position: absolute;
    inset: 0;
    pointer-events: none;
    opacity: 0.04;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='160' height='160'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.85' numOctaves='2' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)'/%3E%3C/svg%3E");
    will-change: transform;
  }

  .frame {
    position: relative;
    width: min(92vw, 1400px);
    height: min(90vh, 850px);
    background: var(--bg);
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    overflow: hidden;
    border-radius: 20px;
    box-shadow: 0 40px 100px rgba(0, 0, 0, 0.55);
  }

  @media (max-width: 700px) {
    .stage-wrap {
      padding: 0;
    }
    .frame {
      width: 100vw;
      height: 100vh;
      aspect-ratio: auto;
      border-radius: 0;
      box-shadow: none;
    }
  }

  .hero {
    position: relative;
    z-index: 1;
    flex: 1 1 auto;
    display: flex;
    /* Top-aligned, not centered: centering let tall hero-row content
       overflow evenly top/bottom, clipping into the header bar above
       (the notification pill was getting cut off at the header's edge). */
    align-items: flex-start;
    padding: clamp(0.75rem, 2vh, 1.25rem) clamp(1.25rem, 5vw, 3rem) 0;
    overflow: hidden;
  }

  .hero-row {
    position: relative;
    z-index: 1;
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;
    gap: 4rem;
    width: 100%;
    padding: 0 2rem;
  }

  @media (max-width: 700px) {
    .hero-row {
      flex-direction: column;
      gap: 0.5rem;
      padding: 0;
    }
  }

  .icon-btn {
    position: absolute;
    top: clamp(0.75rem, 2.5vh, 1.5rem);
    right: clamp(0.75rem, 4vw, 1.5rem);
    background: none;
    border: none;
    padding: 0;
    cursor: pointer;
    width: clamp(26px, 6vw, 34px);
    height: clamp(26px, 6vw, 34px);
    border-radius: 50%;
  }
  .icon-btn svg {
    width: 100%;
    height: 100%;
  }
  .icon-btn:focus-visible {
    outline: 2px solid var(--blue);
    outline-offset: 4px;
  }
  .share-toast {
    position: absolute;
    top: clamp(3.2rem, 6.5vh, 4rem);
    right: clamp(0.75rem, 4vw, 1.5rem);
    background: var(--ink);
    color: #fff;
    font-size: 0.7rem;
    font-weight: 600;
    padding: 0.35rem 0.7rem;
    border-radius: 999px;
    z-index: 3;
  }

  .headline {
    display: flex;
    flex-direction: column;
    gap: clamp(0.5rem, 1.5vh, 0.9rem);
  }

  .eyebrow {
    position: relative;
    font-weight: 600;
    font-size: clamp(0.6rem, 1.8vw, 0.8rem);
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: var(--accent);
  }

  .paint-splat {
    position: absolute;
    z-index: -1;
    pointer-events: none;
    height: auto;
  }

  .paint-splat-blue {
    top: -3.9em;
    left: -10.8em;
    width: clamp(66px, 15vw, 90px);
    opacity: 0.95;
  }

  .paint-splat-pink {
    top: 1.05em;
    left: -3.6em;
    width: clamp(45px, 10.2vw, 60px);
    opacity: 0.92;
  }

  .stack {
    position: relative;
    display: flex;
    flex-direction: column;
    line-height: 0.95;
  }
  .stack .word {
    font-weight: normal;
    font-size: clamp(3.2rem, 14vw, 5.4rem);
    color: var(--blue);
    letter-spacing: 0.02em;
    /* All-caps display heads need line-height >= 1.0 */
    line-height: 1.04;
  }
  /* "of" breaks from the wordmark's Back Wild/blue pairing into Permanent
     Marker on a danfo-yellow highlight — a hand-scrawled connector taped
     between two big painted words, the way a route detail gets marker'd
     onto a danfo alongside the painted destination name. */
  .stack .of {
    align-self: flex-start;
    font-weight: normal;
    font-size: clamp(1.3rem, 4.2vw, 1.9rem);
    color: var(--ink);
    margin: 0.15em 0 0.05em 0.12em;
    transform: rotate(-6deg);
    position: relative;
    z-index: 1;
  }
  .stack .of::after {
    content: "";
    position: absolute;
    left: -8%;
    right: -8%;
    bottom: 0.08em;
    height: 0.34em;
    background: var(--chaos-yellow);
    border-radius: 2px;
    z-index: -1;
    opacity: 0.9;
  }

  .subhead {
    display: flex;
    align-items: baseline;
    gap: 0.6rem;
  }
  .subhead .stamp {
    font-weight: 700;
    font-size: clamp(1rem, 4vw, 1.4rem);
    color: var(--ink);
  }
  .subhead .venue {
    font-weight: 500;
    font-size: clamp(0.75rem, 3vw, 1rem);
    color: #5a5a5a;
  }

  .tagline {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
    font-size: clamp(0.7rem, 2vw, 0.9rem);
    color: var(--ink);
  }
  .tagline .sub {
    text-transform: uppercase;
    letter-spacing: 0.06em;
    font-size: clamp(0.5rem, 1.6vw, 0.65rem);
    font-weight: 600;
    color: #6b6b6b;
  }

  .notification-pill {
    position: relative;
    display: inline-flex;
    align-items: center;
    gap: 0.45rem;
    /* Solid glass, not the barely-there tint it used to be: a properly
       frosted, mostly-opaque panel (unlike the chaos layer's translucent
       glass popups) so it reads as a fixed piece of the card's UI. */
    background: rgba(255, 255, 255, 0.6);
    backdrop-filter: blur(16px) saturate(180%);
    -webkit-backdrop-filter: blur(16px) saturate(180%);
    border: 1px solid rgba(229, 56, 58, 0.32);
    box-shadow: 0 8px 20px rgba(0, 0, 0, 0.08), inset 0 1px 0 rgba(255, 255, 255, 0.6);
    padding: 0.35rem 0.75rem 0.35rem 0.9rem;
    border-radius: 999px;
    font-size: clamp(0.65rem, 1.8vw, 0.75rem);
    width: fit-content;
    overflow: hidden;
    transition: background var(--dur-base) var(--ease-standard),
                border-color var(--dur-base) var(--ease-standard),
                transform var(--dur-base) var(--ease-standard);
    pointer-events: none;
  }
  /* Danfo zone: while the cube is still scrambled (stage "high"), the pill
     carries a yellow/black hazard-stripe edge instead of a flat red tint —
     the one moment on the page that's allowed to look like mainland chaos. */
  .notification-pill[data-stage="high"]::before {
    content: "";
    position: absolute;
    inset: 0 auto 0 0;
    width: 6px;
    background: repeating-linear-gradient(
      135deg,
      var(--ink) 0 4px,
      var(--chaos-yellow) 4px 8px
    );
  }
  .notification-pill[data-stage="mid"] {
    border-color: rgba(255, 199, 44, 0.45);
  }
  .notification-pill[data-stage="zero"] {
    border-color: rgba(0, 191, 255, 0.4);
  }
  .notif-label {
    color: var(--ink);
    font-weight: 500;
  }
  .notif-count {
    font-weight: 700;
    color: var(--chaos-red);
    /* Digit widths vary per-glyph in Fredoka; without tabular-nums the
       counter's own width jitters on every tick as it counts 999 -> 0. */
    font-variant-numeric: tabular-nums;
    transition: color var(--dur-base) var(--ease-standard);
  }
  .notification-pill[data-stage="mid"] .notif-count {
    color: #b8860b;
  }
  .notification-pill[data-stage="zero"] .notif-count {
    color: var(--blue, #00bfff);
  }

  .hero-cta-wrap {
    margin-top: clamp(0.4rem, 1.2vh, 0.8rem);
  }
  .activate-cta {
    font-weight: 700;
    font-size: clamp(0.85rem, 2.4vw, 1.05rem);
    background: var(--blue, #00bfff);
    color: #fff;
    border: none;
    padding: clamp(0.65rem, 2vh, 0.85rem) clamp(1.2rem, 4vw, 1.8rem);
    border-radius: 999px;
    cursor: pointer;
    box-shadow: 0 6px 20px rgba(0, 191, 255, 0.32);
    transition: transform var(--dur-fast) var(--ease-standard),
                box-shadow var(--dur-fast) var(--ease-standard),
                background var(--dur-fast) var(--ease-standard);
  }
  .activate-cta:hover {
    transform: translateY(-2px);
    box-shadow: 0 8px 24px rgba(0, 191, 255, 0.45);
    background: #00aceb;
  }
  .activate-cta:active {
    transform: translateY(0);
  }
  .activate-cta:focus-visible {
    outline: 3px solid var(--pink-deep, #fc9ce0);
    outline-offset: 3px;
  }

  .cube-slot {
    position: relative;
    z-index: 2;
    flex: none;
    pointer-events: auto;
    width: clamp(160px, min(27vw, 30vh), 340px);
    height: clamp(160px, min(27vw, 30vh), 340px);
  }
  /* Grounding glow — the cube is meant to be the hero object, but its
     bright/paper-cream seams sat directly on a near-identical cream page
     background with nothing to separate the two, so it visually thinned
     out instead of popping. A soft halo gives it an edge regardless of
     which sticker colors happen to be facing the camera. */
  .cube-slot::before {
    content: "";
    position: absolute;
    inset: -20%;
    z-index: -1;
    border-radius: 50%;
    background: radial-gradient(circle, rgba(0, 191, 255, 0.22), rgba(252, 156, 224, 0.14) 45%, transparent 72%);
    filter: blur(28px);
  }

  /* The hero cube stays put in .cube-slot for the whole scroll-pin —
     it does not relocate. Once activated, a second, independent cube
     (.cube-companion below) pops into play instead, docked in the
     corner for the rest of the scroll. */
  .cube-companion {
    position: fixed;
    bottom: clamp(1.5rem, 5vh, 3rem);
    right: clamp(1.5rem, 5vw, 3rem);
    width: clamp(70px, 10vw, 100px);
    height: clamp(70px, 10vw, 100px);
    z-index: 100;
    pointer-events: auto;
    animation: floatAssistant 4s ease-in-out infinite;
    filter: drop-shadow(0 10px 20px rgba(0,0,0,0.15));
  }
  /* Glassmorphism dock — the moodboard calls for it and the site has none
     yet; the transparent-canvas cube (renderer alpha: true) sitting over a
     frosted panel is the one place it reads as a UI affordance rather than
     decoration. */
  .cube-companion::before {
    content: "";
    position: absolute;
    inset: -18%;
    border-radius: 50%;
    background: rgba(255, 255, 255, 0.32);
    backdrop-filter: blur(14px) saturate(160%);
    -webkit-backdrop-filter: blur(14px) saturate(160%);
    border: 1px solid rgba(255, 255, 255, 0.55);
    box-shadow: 0 8px 28px rgba(0, 0, 0, 0.1);
    z-index: -1;
  }

  @keyframes floatAssistant {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(-8px); }
  }

  .activated {
    position: relative;
    min-height: 50vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 0.75rem;
    text-align: center;
    padding: 2rem;
    overflow: hidden;
    opacity: 0;
    transform: translateY(24px);
    transition: opacity 0.6s var(--ease-standard), transform 0.6s var(--ease-standard);
    /* Sea zone: this is the "you've found calm" moment, so blue gets to read
       as water here instead of as a flat UI accent — a soft foam wash from
       the top and a horizon line, not a saturated block, so the existing
       ink/blue/pink-deep text on top stays readable. */
    background:
      radial-gradient(140% 65% at 50% -10%, rgba(0, 191, 255, 0.16), transparent 60%),
      linear-gradient(180deg, transparent 0%, rgba(0, 191, 255, 0.05) 60%, rgba(0, 191, 255, 0.12) 100%);
  }
  .activated::before {
    content: "";
    position: absolute;
    left: 0;
    right: 0;
    bottom: 28%;
    height: 1px;
    background: linear-gradient(90deg, transparent, rgba(0, 191, 255, 0.5), transparent);
    box-shadow: 0 22px 0 -1px rgba(0, 191, 255, 0.22),
                0 46px 0 -1px rgba(0, 191, 255, 0.12);
    pointer-events: none;
  }
  .activated > * {
    position: relative;
    z-index: 1;
  }
  .activated.visible {
    opacity: 1;
    transform: translateY(0);
  }
  .activated-eyebrow {
    margin: 0;
    font-weight: 600;
    font-size: 0.85rem;
    letter-spacing: 0.18em;
    text-transform: uppercase;
    color: var(--accent);
  }
  .activated-title {
    margin: 0;
    font-weight: 700;
    font-size: clamp(2rem, 7vw, 3.2rem);
    color: var(--blue);
  }
  .activated-sub {
    margin: 0;
    color: var(--ink);
  }
  .activated-note {
    margin: 0.75rem 0 0;
    max-width: 32ch;
    font-size: clamp(0.95rem, 2.4vw, 1.15rem);
    line-height: 1.4;
    color: var(--pink-deep);
    transform: rotate(-1deg);
  }

  .stats-toast {
    position: fixed;
    top: 2rem;
    left: 50%;
    transform: translate(-50%, -20px);
    background: var(--card-surface);
    backdrop-filter: blur(8px);
    border: 2px solid var(--blue);
    padding: 1.25rem 1.75rem;
    border-radius: 14px;
    box-shadow: 0 20px 40px rgba(0, 0, 0, 0.15);
    z-index: 1000;
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.5s var(--ease-out-expo), transform 0.5s var(--ease-out-expo);
  }
  .stats-toast.visible {
    opacity: 1;
    transform: translate(-50%, 0);
  }
  .stats-title {
    margin: 0 0 0.75rem;
    font-weight: 700;
    font-size: 1.1rem;
    color: var(--blue);
  }
  .stats-toast ul {
    list-style: none;
    padding: 0;
    margin: 0;
    font-size: 0.85rem;
    color: var(--ink);
  }
  .stats-toast li {
    margin-bottom: 0.4rem;
    display: flex;
    justify-content: space-between;
    gap: 1.5rem;
  }
  .stats-toast li strong {
    color: var(--accent);
  }
</style>
