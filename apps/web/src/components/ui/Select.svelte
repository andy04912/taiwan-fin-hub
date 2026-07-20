<script lang="ts">
  import type { Snippet } from "svelte";
  import { cn } from "../../lib/utils";
  let {
    children,
    class: className = "",
    value = $bindable(),
    ...rest
  }: {
    children?: Snippet;
    class?: string;
    value?: string | number;
    [key: string]: unknown;
  } = $props();
</script>

<select
  {...rest}
  bind:value
  class={cn(
    "flex h-10 w-full rounded-lg border border-input bg-card px-3 py-2 text-sm transition-colors focus-visible:border-primary focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring/20 disabled:cursor-not-allowed disabled:opacity-50",
    className,
  )}>{@render children?.()}</select
>
