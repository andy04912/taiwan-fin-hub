<script lang="ts">
  import { cn } from "../../lib/utils";
  let {
    class: className = "",
    value = $bindable(),
    type = "text",
    ...rest
  }: {
    class?: string;
    value?: string | number;
    type?: string;
    [key: string]: unknown;
  } = $props();
</script>

<input
  {...rest}
  {type}
  bind:value
  class={cn(
    "flex h-10 w-full rounded-lg border border-input bg-card px-3 py-2 text-sm transition-colors placeholder:text-muted-foreground focus-visible:border-primary focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring/20 disabled:cursor-not-allowed disabled:opacity-50",
    className,
  )}
/>
