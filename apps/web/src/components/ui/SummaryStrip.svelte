<script lang="ts">
  import type { Snippet } from "svelte";
  import { cn } from "../../lib/utils";
  let {
    children,
    class: className = "",
  }: { children?: Snippet; class?: string } = $props();
</script>

<section
  aria-label="摘要"
  class={cn(
    "grid grid-cols-2 gap-x-4 gap-y-3 rounded-xl border border-border bg-card px-4 py-3 sm:px-5 md:grid-cols-4 md:divide-x md:divide-border md:[&>*]:pl-5 md:[&>*:first-child]:pl-0",
    className,
  )}
>
  {@render children?.()}
</section>
