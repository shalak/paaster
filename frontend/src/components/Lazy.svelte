<script lang="ts">
  import { onMount } from "svelte";

  export let component: any;
  export let delayMs: number | null = null;

  let loadedComponent: any = null;
  let timeout: NodeJS.Timeout;
  let showFallback = !delayMs;

  let props: Record<any, any>;
  $: {
    // eslint-disable-next-line no-shadow
    const { component, delayMs, ...restProps } = $$props;
    props = restProps;
  }

  onMount(() => {
    if (delayMs) {
      timeout = setTimeout(() => {
        showFallback = true;
      }, delayMs);
    }
    component().then((module) => {
      loadedComponent = module.default;
    });
    return () => clearTimeout(timeout);
  });
</script>

{#if loadedComponent}
  <svelte:component this={loadedComponent} {...props} />
{:else if showFallback}
  <slot />
{/if}
