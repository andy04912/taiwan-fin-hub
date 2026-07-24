<script lang="ts">
  import { cn } from "../../lib/utils";

  let {
    label,
    value,
    detail,
    tone = "neutral",
    class: className = "",
  }: {
    label: string;
    value: string;
    detail?: string;
    tone?: "neutral" | "positive" | "negative" | "brand" | "warning";
    class?: string;
  } = $props();

  const toneClass = $derived(
    tone === "positive"
      ? "text-moss"
      : tone === "negative"
        ? "text-coral"
        : tone === "brand"
          ? "text-steel"
          : tone === "warning"
            ? "text-warning"
            : "text-ink",
  );
</script>

<div class={cn("min-w-0 py-1", className)}>
  <p class="text-xs font-semibold text-muted-foreground">{label}</p>
  <p
    class={cn(
      "c-metric-value mt-1.5 truncate text-xl font-bold tracking-tight tabular-nums md:text-2xl",
      toneClass,
    )}
  >
    {value}
  </p>
  {#if detail}
    <p class="mt-1 truncate text-xs text-muted-foreground">{detail}</p>
  {/if}
</div>
