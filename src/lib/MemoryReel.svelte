<script>
  import { onMount, onDestroy } from 'svelte';
  import MorphText from './MorphText.svelte';

  export let visible = false;

  // Public Drive folder: share it "Anyone with the link" and drop the
  // folder id + a browser API key (restricted to Drive API, HTTP referrer
  // locked to this site) into .env as VITE_DRIVE_FOLDER_ID / VITE_DRIVE_API_KEY.
  // With neither set, the reel falls back to placeholder cards so the
  // carousel still renders in dev/preview.
  const FOLDER_ID = import.meta.env.VITE_DRIVE_FOLDER_ID;
  const API_KEY = import.meta.env.VITE_DRIVE_API_KEY;

  const PLACEHOLDER_CARDS = Array.from({ length: 7 }, (_, i) => ({
    id: `placeholder-${i}`,
    name: `Memory ${i + 1}`,
    isVideo: i % 3 === 1,
    thumb: null,
  }));

  let cards = PLACEHOLDER_CARDS;
  let loadError = false;

  async function loadDriveFiles() {
    if (!FOLDER_ID || !API_KEY) return;
    try {
      const q = encodeURIComponent(`'${FOLDER_ID}' in parents and trashed = false`);
      const fields = encodeURIComponent('files(id,name,mimeType,thumbnailLink)');
      const url = `https://www.googleapis.com/drive/v3/files?q=${q}&fields=${fields}&key=${API_KEY}&pageSize=50`;
      const res = await fetch(url);
      if (!res.ok) throw new Error(`Drive API ${res.status}`);
      const data = await res.json();
      const files = (data.files || []).filter(
        (f) => f.mimeType?.startsWith('image/') || f.mimeType?.startsWith('video/')
      );
      if (files.length) {
        cards = files.map((f) => ({
          id: f.id,
          name: f.name,
          isVideo: f.mimeType.startsWith('video/'),
          // Drive's thumbnailLink is small by default (=s220); ask for bigger.
          thumb: f.thumbnailLink ? f.thumbnailLink.replace(/=s\d+$/, '=s800') : null,
          driveUrl: `https://drive.google.com/uc?id=${f.id}`,
        }));
      }
    } catch (err) {
      console.warn('MemoryReel: failed to load Drive folder, using placeholders', err);
      loadError = true;
    }
  }

  // --- Carousel state -------------------------------------------------
  let track;
  let currentIndex = 0; // fractional — lets drag/scroll feel continuous
  let dragging = false;
  let dragStartX = 0;
  let dragStartIndex = 0;
  let rafId = null;

  const ANGLE_STEP = 32; // deg between neighboring cards around the cylinder
  const RADIUS = 260; // px, how far cards sit from the center on the arc
  const MAX_VISIBLE_DISTANCE = 3; // cards further than this fade out fully

  function clampIndex(i) {
    const max = cards.length - 1;
    return Math.max(0, Math.min(max, i));
  }

  function goTo(i) {
    currentIndex = clampIndex(i);
  }
  function next() {
    goTo(Math.round(currentIndex) + 1);
  }
  function prev() {
    goTo(Math.round(currentIndex) - 1);
  }

  let wheelAccum = 0;
  let wheelRafId = null;
  function onWheel(e) {
    // Only hijack the scroll while the pointer is over the reel and the
    // gesture reads as horizontal-ish; otherwise let the page scroll.
    const horizontal = Math.abs(e.deltaX) > Math.abs(e.deltaY);
    if (!horizontal) return;
    e.preventDefault();
    wheelAccum += e.deltaX;
    if (!wheelRafId) {
      wheelRafId = requestAnimationFrame(() => {
        goTo(currentIndex + wheelAccum / 120);
        wheelAccum = 0;
        wheelRafId = null;
      });
    }
  }

  function onPointerDown(e) {
    dragging = true;
    dragStartX = e.clientX;
    dragStartIndex = currentIndex;
    track?.setPointerCapture?.(e.pointerId);
  }
  function onPointerMove(e) {
    if (!dragging) return;
    const dx = e.clientX - dragStartX;
    goTo(dragStartIndex - dx / 90);
  }
  function onPointerUp() {
    if (!dragging) return;
    dragging = false;
    goTo(Math.round(currentIndex));
  }

  function onKeydown(e) {
    if (e.key === 'ArrowRight') next();
    if (e.key === 'ArrowLeft') prev();
  }

  $: activeCardId = cards[Math.round(clampIndex(currentIndex))]?.id;

  onMount(() => {
    loadDriveFiles();
  });
  onDestroy(() => {
    if (rafId) cancelAnimationFrame(rafId);
    if (wheelRafId) cancelAnimationFrame(wheelRafId);
  });
</script>

<section class="memory-reel">
  <div class="text-content">
    <p class="eyebrow" class:visible><MorphText text="The memory reel" /></p>
    <h2 class="heading" class:visible>Every escape, spun into a card.</h2>
    <p class="caption" class:visible>
      Drag, scroll, or use the arrows to spin through the shots — pulled straight from the shared Drive.
    </p>
  </div>

  <div
    class="stage"
    class:visible
    bind:this={track}
    on:wheel={onWheel}
    on:pointerdown={onPointerDown}
    on:pointermove={onPointerMove}
    on:pointerup={onPointerUp}
    on:pointercancel={onPointerUp}
    on:keydown={onKeydown}
    tabindex="0"
    role="listbox"
    aria-label="Photo and video carousel"
  >
    <div class="cylinder">
      {#each cards as card, i (card.id)}
        {@const distance = i - currentIndex}
        {@const absD = Math.abs(distance)}
        {@const angle = distance * ANGLE_STEP}
        {@const rad = (angle * Math.PI) / 180}
        {@const x = Math.sin(rad) * RADIUS}
        {@const z = Math.cos(rad) * RADIUS - RADIUS}
        {@const scale = Math.max(0.55, 1 - absD * 0.16)}
        {@const opacity = absD > MAX_VISIBLE_DISTANCE ? 0 : Math.max(0, 1 - absD * 0.3)}
        <button
          type="button"
          class="card"
          class:active={card.id === activeCardId}
          style="
            transform: translateX({x}px) translateZ({z}px) rotateY({-angle}deg) scale({scale});
            opacity: {opacity};
            z-index: {100 - Math.round(absD * 10)};
            pointer-events: {absD > MAX_VISIBLE_DISTANCE ? 'none' : 'auto'};
          "
          on:click={() => goTo(i)}
          role="option"
          aria-selected={card.id === activeCardId}
          aria-label={card.name}
        >
          <div class="card-face">
            {#if card.thumb}
              <img src={card.thumb} alt={card.name} loading="lazy" />
            {:else}
              <div class="placeholder" aria-hidden="true">
                <span>{card.isVideo ? '▶' : '🖼'}</span>
              </div>
            {/if}
            {#if card.isVideo}
              <span class="video-badge" aria-hidden="true">▶</span>
            {/if}
          </div>
          <span class="card-name">{card.name}</span>
        </button>
      {/each}
    </div>

    <button class="nav prev" type="button" on:click|stopPropagation={prev} aria-label="Previous">‹</button>
    <button class="nav next" type="button" on:click|stopPropagation={next} aria-label="Next">›</button>
  </div>

  {#if !FOLDER_ID || !API_KEY}
    <p class="hint">
      Showing placeholders — set <code>VITE_DRIVE_FOLDER_ID</code> and <code>VITE_DRIVE_API_KEY</code> in
      <code>.env</code> to pull real photos/videos from your shared Drive folder.
    </p>
  {:else if loadError}
    <p class="hint">Couldn't reach the Drive folder — showing placeholders instead.</p>
  {/if}
</section>

<style>
  .memory-reel {
    max-width: 1100px;
    margin: 0 auto;
    padding: clamp(3rem, 10vh, 6rem) 1.5rem;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 2.5rem;
    text-align: center;
  }

  .text-content {
    max-width: 520px;
  }

  .eyebrow,
  .heading,
  .caption {
    opacity: 0;
    transform: translateY(16px);
    transition: opacity 0.6s var(--ease-out-expo), transform 0.6s var(--ease-out-expo);
  }
  .eyebrow.visible,
  .heading.visible,
  .caption.visible {
    opacity: 1;
    transform: translateY(0);
  }
  .heading.visible { transition-delay: 80ms; }
  .caption.visible { transition-delay: 160ms; }

  .eyebrow {
    margin: 0 0 0.5rem;
    font-size: clamp(1.4rem, 3.5vw, 1.8rem);
    color: var(--pink-deep);
  }
  .heading {
    margin: 0 0 0.75rem;
    font-weight: 700;
    font-size: clamp(1.6rem, 4.5vw, 2.4rem);
    color: var(--ink);
    line-height: 1.15;
  }
  .caption {
    margin: 0;
    color: var(--muted);
    line-height: 1.5;
  }

  .stage {
    position: relative;
    width: 100%;
    height: clamp(260px, 36vw, 340px);
    perspective: 1200px;
    outline: none;
    opacity: 0;
    transform: translateY(24px);
    transition: opacity 0.7s var(--ease-out-expo) 220ms, transform 0.7s var(--ease-out-expo) 220ms;
    touch-action: pan-y;
    cursor: grab;
  }
  .stage.visible {
    opacity: 1;
    transform: translateY(0);
  }
  .stage:active {
    cursor: grabbing;
  }

  .cylinder {
    position: absolute;
    inset: 0;
    transform-style: preserve-3d;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .card {
    position: absolute;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.5rem;
    width: clamp(140px, 20vw, 190px);
    background: none;
    border: none;
    padding: 0;
    cursor: pointer;
    transition: transform 0.15s linear, opacity 0.15s linear;
    will-change: transform, opacity;
  }

  .card-face {
    position: relative;
    width: 100%;
    aspect-ratio: 3 / 4;
    border-radius: 12px;
    overflow: hidden;
    background: var(--card-surface);
    border: 1px solid var(--border-soft);
    box-shadow: 0 16px 34px rgba(0, 0, 0, 0.18);
    transition: box-shadow 0.25s var(--ease-out-expo), border-color 0.25s var(--ease-out-expo);
  }
  .card.active .card-face {
    border-color: var(--blue);
    box-shadow: 0 24px 48px rgba(0, 0, 0, 0.28);
  }

  .card-face img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
    pointer-events: none;
  }

  .placeholder {
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 2rem;
    color: var(--muted);
    background: linear-gradient(135deg, var(--border-soft) 0%, var(--card-surface) 100%);
  }

  .video-badge {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 1.4rem;
    color: #fff;
    background: rgba(0, 0, 0, 0.25);
    pointer-events: none;
  }

  .card-name {
    font-size: 0.75rem;
    font-weight: 600;
    color: var(--muted);
    max-width: 100%;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  .card.active .card-name {
    color: var(--ink);
  }

  .nav {
    position: absolute;
    top: 50%;
    transform: translateY(-50%);
    width: 2.5rem;
    height: 2.5rem;
    border-radius: 50%;
    border: 1px solid var(--border-soft-deep);
    background: var(--card-surface);
    color: var(--ink);
    font-size: 1.4rem;
    line-height: 1;
    cursor: pointer;
    z-index: 200;
    transition: background 0.2s var(--ease-standard), transform 0.2s var(--ease-standard);
  }
  .nav:hover {
    background: var(--blue);
    color: #fff;
  }
  .nav.prev { left: 0; }
  .nav.next { right: 0; }

  .hint {
    margin: 0;
    font-size: 0.78rem;
    color: var(--muted);
  }
  .hint code {
    background: var(--border-soft);
    padding: 0.1rem 0.35rem;
    border-radius: 4px;
  }

  @media (max-width: 600px) {
    .nav { display: none; }
  }
</style>
