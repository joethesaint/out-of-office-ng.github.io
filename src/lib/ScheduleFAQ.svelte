<script>
  import { slide } from 'svelte/transition';
  import { cubicOut } from 'svelte/easing';
  import { dialogDuration } from './motion.js';
  import { NEXT_EVENT } from './sales.js';
  let open = null;
  const questions = [
    { question: 'What is Out of Office?', answer: 'Out of Office is for anyone who wants to step outside their usual routine, discover new interests, build genuine connections, express themselves freely, or enjoy life beyond the everyday.' },
    { question: 'When and where is the next one?', answer: `The next edition is planned for ${NEXT_EVENT.when}. The exact date and venue are still to be announced.` },
    { question: 'Can I buy a pass yet?', answer: 'Passes for 0x04 are not on sale yet. Ticket details and prices will be shared when they are confirmed.' },
    { question: 'What should I bring, and how do I get there?', answer: 'The programme, what to bring and any transport details will be shared once the venue is confirmed. Previous editions do not confirm the plans for 0x04.' },
  ];
</script>

<section id="next-event" class="chapter next-event" aria-labelledby="next-event-title">
  <div class="next-inner">
    <div class="next-intro">
      <p class="eyebrow">WHAT WE KNOW ABOUT 0x04</p>
      <h2 id="next-event-title">November is the plan.<br />The details are coming.</h2>
      <p class="case-link"><a href="./case.html">Can't wait? Try Case File #1, a small taste of Games Day &rarr;</a></p>
      <p class="event-date">{NEXT_EVENT.code} <span aria-hidden="true">·</span> {NEXT_EVENT.when}</p>
      <p class="intro">Date, venue, programme and passes will be announced here.</p>
    </div>
    <div class="faq">
      <h3>Before you switch off.</h3>
      {#each questions as item, i}
        <div class="faq-row" class:expanded={open === i}>
          <h4>
            <button id={`faq-question-${i}`} class="question" type="button" aria-expanded={open === i} aria-controls={`faq-answer-${i}`} on:click={() => open = open === i ? null : i}>
              <span class="number" aria-hidden="true">0{i + 1}</span>
              <span>{item.question}</span>
              <span class="indicator" aria-hidden="true">+</span>
            </button>
          </h4>
          {#if open === i}
            <div id={`faq-answer-${i}`} role="region" aria-labelledby={`faq-question-${i}`} transition:slide={{ duration: dialogDuration(250), easing: cubicOut }}>
              <p class="answer">{item.answer}</p>
            </div>
          {/if}
        </div>
      {/each}
    </div>
  </div>
</section>

<style>
  .next-inner { max-width: 1140px; margin: auto; display: grid; grid-template-columns: 1fr 1fr; gap: clamp(40px, 7vw, 100px); }
  .event-date { margin: 28px 0 16px; font-size: 17px; font-weight: 500; }
  .event-date span { margin: 0 6px; }
  .intro { max-width: 380px; line-height: 1.8; color: var(--muted); }
  h3 { margin: 0 0 24px; font: 400 25px/1.3 var(--serif); }
  h4 { margin: 0; }
  .faq-row { border-top: 1px solid var(--border-soft-deep); }
  .faq-row:last-child { border-bottom: 1px solid var(--border-soft-deep); }
  .question { display: grid; grid-template-columns: 22px 1fr 20px; gap: 12px; align-items: center; width: 100%; padding: 23px 0; background: none; border: 0; color: var(--ink); font: 500 15px/1.5 var(--sans); text-align: left; cursor: pointer; }
  .question:hover { color: var(--deep); }
  .number { color: var(--muted); font-size: 10px; }
  .indicator { font-size: 25px; font-weight: 400; text-align: center; transition: transform 250ms ease; }
  .expanded .indicator { transform: rotate(45deg); }
  .answer { margin: 0; padding: 0 30px 24px 34px; color: var(--muted); font-size: 14px; line-height: 1.8; }
  @media (max-width: 760px) { .next-inner { grid-template-columns: minmax(0, 1fr); gap: 44px; } }
  .case-link { margin-top: 12px; font-family: "Permanent Marker", cursive; }
  .case-link a { color: var(--pink-deep); }
</style>
