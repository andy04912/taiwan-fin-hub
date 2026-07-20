<script lang="ts">
  import { cn } from "../../lib/utils";
  let {
    class: className = "",
    value = $bindable(),
    ...rest
  }: { class?: string; value?: string; [key: string]: unknown } = $props();
</script>

<textarea
  {...rest}
  bind:value
  class={cn(
    "flex min-h-20 w-full rounded-lg border border-input bg-card px-3 py-2 text-sm transition-colors placeholder:text-muted-foreground focus-visible:border-primary focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring/20 disabled:cursor-not-allowed disabled:opacity-50",
    className,
  )}></textarea>
