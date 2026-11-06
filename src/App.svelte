<script>
  import { onMount, tick } from 'svelte';
  import HeaderBar from './lib/HeaderBar.svelte';
  import Shallows from './lib/Shallows.svelte';
  import Postcard from './lib/Postcard.svelte';
  import RisingPhotos from './lib/RisingPhotos.svelte';
  import EditionArchive from './lib/EditionArchive.svelte';
  import PhotoShowcaseCarousel from './lib/PhotoShowcaseCarousel.svelte';
  import Playlist from './lib/Playlist.svelte';
  import PageFooter from './lib/PageFooter.svelte';
  import Tickets from './lib/Tickets.svelte';
  import ScheduleFAQ from './lib/ScheduleFAQ.svelte';
  import AboutEvent from './lib/AboutEvent.svelte';
  import EventTrail from './lib/EventTrail.svelte';
  import RsvpDrawer from './lib/RsvpDrawer.svelte';
  import OooGeneratorModal from './lib/OooGeneratorModal.svelte';
  import ScrollReveal from './lib/ScrollReveal.svelte';
  // Games Day props load after first paint so three.js stays off the critical path.
  const loadGameProps = () => import('./lib/GameProps.svelte');

  let isDrawerOpen = false;
  let isOooGenOpen = false;
  let currentRoute = typeof window !== 'undefined' ? window.location.hash || '#/' : '#/';
  
  const openDrawer = () => isDrawerOpen = true;
  const openOooGen = () => isOooGenOpen = true;
  $: trailMatch = /^#\/trail(?:\/(0x0[1-4]))?$/.exec(currentRoute);
  // Each view gets its own title (WCAG 2.4.2); home keeps the one in index.html.
  const homeTitle = typeof document !== 'undefined' ? document.title : '';
  $: if (typeof document !== 'undefined') {
    document.title = currentRoute === '#/about' ? 'Our story — Out of Office Lagos'
      : trailMatch ? 'Event trail — Out of Office Lagos'
      : homeTitle;
  }

  async function positionRoute(focus = true) {
    await tick();
    const match = /^#\/trail\/(0x0[1-4])$/.exec(currentRoute);
    if (match) {
      const target = document.getElementById(`edition-${match[1]}`);
      target?.scrollIntoView({ behavior: 'instant', block: 'start' });
      if (focus) target?.focus({ preventScroll: true });
    } else if (currentRoute.startsWith('#/')) {
      window.scrollTo({ top: 0, behavior: 'instant' });
      if (focus) document.querySelector('h1')?.focus({ preventScroll: true });
    } else {
      const target = document.getElementById(currentRoute.slice(1));
      target?.scrollIntoView({ behavior: 'instant', block: 'start' });
    }
  }
  onMount(() => {
    const previousRestoration = history.scrollRestoration;
    history.scrollRestoration = 'manual';
    let mounted = true;
    // Font loading can change card heights. Position direct links after it
    // settles so the selected edition stays below the sticky navigation.
    Promise.resolve(document.fonts?.ready).then(() => {
      if (mounted) positionRoute(false);
    });
    return () => { mounted = false; history.scrollRestoration = previousRestoration; };
  });

  async function onHashChange() {
    const previousRoute = currentRoute;
    currentRoute = window.location.hash || '#/';
    isDrawerOpen = false;
    isOooGenOpen = false;
    if (currentRoute.startsWith('#/') || previousRoute.startsWith('#/')) await positionRoute();
  }

  function scrollToContent() {
    const reduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
    document.getElementById('content-start')?.scrollIntoView({ behavior: reduced ? 'instant' : 'smooth' });
  }
</script>

<svelte:window on:hashchange={onHashChange} />

{#if currentRoute === '#/about'}
  <AboutEvent />
{:else if trailMatch}
  <EventTrail activeEdition={trailMatch[1] || null} onOpenDrawer={openDrawer} />
{:else}
  <a class="skip-link" href="#content-start">Skip to content</a>
  <HeaderBar onOpenDrawer={openDrawer} />
  
  <main id="top">
    <Shallows onOpenOooGen={openOooGen} onScrollToContent={scrollToContent} onScrollToArchive={() => document.getElementById('past-editions')?.scrollIntoView({ behavior: window.matchMedia('(prefers-reduced-motion: reduce)').matches ? 'instant' : 'smooth' })} />

    <div id="content-start" tabindex="-1">
      <ScrollReveal animate={false} let:visible><Postcard {visible} /></ScrollReveal>
      <RisingPhotos />
      <ScrollReveal><EditionArchive /></ScrollReveal>
      <ScrollReveal><PhotoShowcaseCarousel /></ScrollReveal>
      <ScheduleFAQ />
      <ScrollReveal let:visible><Tickets {visible} onOpenDrawer={openDrawer} /></ScrollReveal>
      <ScrollReveal let:visible><Playlist {visible} /></ScrollReveal>
    </div>
  </main>
  <PageFooter />
  {#await loadGameProps() then { default: GameProps }}<GameProps />{/await}

{/if}

<RsvpDrawer isOpen={isDrawerOpen} onClose={() => isDrawerOpen = false} />
<OooGeneratorModal isOpen={isOooGenOpen} onClose={() => isOooGenOpen = false} />
