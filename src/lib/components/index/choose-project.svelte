<script lang="ts">
  import { onMount } from "svelte";
  import { reducedMotion } from "$lib/stores/reduce-motion";
  import Section from "../section.svelte";
  let disableAnimation = false;

  let animatedTexts = {
    desktop: [
      {
        isVisible: false,
        text: "check dependencies, checkout branch, view",
      },
      {
        isVisible: false,
        text: "readme.txt, install tools, run build, run test,",
      },
    ],
    mobile: [
      {
        isVisible: false,
        text: "check dependencies,",
      },
      {
        isVisible: false,
        text: "checkout branch, view",
      },
      {
        isVisible: false,
        text: "readme.txt, install tools,",
      },
      {
        isVisible: false,
        text: "run build, run test,",
      },
    ],
  };

  /*
   * In order to trigger the animated text at the right time on both desktop and mobile,
   * we kick off the animation when the nav element starts to move out of the viewport
   * and the next section starts to enter the viewport. At that time, the animated Gitpod
   * benefits text is placed roughly at the center of the browser's viewport.
   */
  let isTopHidden = false;
  let isBottomShown = false;
  let hasAnimated = false;

  const startAnimation = () => {
    let t = 0;
    Object.entries(animatedTexts).forEach(([, texts]) =>
      texts.forEach((text) => {
        setTimeout(() => {
          text.isVisible = true;
          animatedTexts = animatedTexts; // This triggers Svelte's reactivity
        }, t);
        t = t + 400;
      })
    );
    hasAnimated = true;
  };

  const manageAnimation = () => {
    if (isTopHidden && isBottomShown && !hasAnimated) {
      startAnimation();
    }
  };

  onMount(() => {
    if ($reducedMotion) {
      disableAnimation = true;
      return;
    }
    const callbackTop = (entries: IntersectionObserverEntry[]) => {
      entries.forEach((entry) => {
        isTopHidden = !entry.isIntersecting;
      });
      manageAnimation();
    };

    const callbackBottom = (entries: IntersectionObserverEntry[]) => {
      entries.forEach((entry) => {
        isBottomShown = entry.isIntersecting;
      });
      manageAnimation();
    };

    const observerTop = new IntersectionObserver(callbackTop, {
      threshold: [0.2],
    });
    const observerBottom = new IntersectionObserver(callbackBottom, {
      threshold: [0],
    });
    const targetTop = document.querySelector(
      "#choose-project-observer-target-top"
    );
    const targetBottom = document.querySelector(
      "#choose-project-observer-target-bottom"
    );
    observerTop.observe(targetTop);
    observerBottom.observe(targetBottom);

    return () => {
      observerTop.disconnect();
      observerBottom.disconnect();
    };
  });
</script>

<style lang="postcss">
  .row {
    width: 100%;
  }

  del {
    text-decoration: none;
  }

  .desktop {
    display: none;
  }

  @media (min-width: 683px) {
    .desktop {
      display: initial;
    }

    .mobile {
      display: none;
    }
  }

  span {
    transition: all 0.2s;
    display: inline-block;
    position: relative;
    color: inherit;
  }

  .strikethrough {
    transition: all 0.5s cubic-bezier(0.55, 0, 0.1, 1);
    color: var(--slight-grey);
  }

  .strikethrough::after {
    content: "";
    position: absolute;
    display: block;
    width: 100%;
    height: 3px;
    margin-top: -0.5em;
    @media (max-width: 640px) {
      margin-top: -0.6em;
    }
    transform-origin: center left;
    animation: strikethrough 0.6s 0.4s cubic-bezier(0.55, 0, 0.1, 1) 1 forwards;
    transition: 0.4s cubic-bezier(0.55, 0, 0.1, 1);
  }

  .disabled {
    @apply transition-none;
    &::after {
      @apply transition-none animate-none bg-sub scale-100;
    }
  }

  @keyframes strikethrough {
    from {
      transform: scaleX(0);
      background: var(--slight-grey);
    }
    to {
      transform: scaleX(1);
      background: var(--slight-grey);
    }
  }
</style>

<div class="row">
  <Section>
    <div
      class="text-3xl sm:text-5xl text-black font-bold text-center dark:text-important"
    >
      Select project,
      <br />
      {#each Object.entries(animatedTexts) as [device, texts]}
        <del class={device}>
          {#each texts as { isVisible, text }}
            <span
              class:strikethrough={isVisible || disableAnimation}
              class:disabled={disableAnimation}>{text}</span
            >
          {/each}
        </del>
      {/each}
      <br />
      start coding.
    </div>
  </Section>
</div>
