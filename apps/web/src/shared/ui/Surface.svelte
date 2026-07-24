<script lang="ts">
  import type { Snippet } from "svelte";
  import { cn } from "../../lib/utils";

  let {
    children,
    class: className = "",
    tone = "default",
    as = "section",
  }: {
    children?: Snippet;
    class?: string;
    tone?: "default" | "subtle" | "strong";
    as?: "div" | "section" | "article";
  } = $props();

  const classes = $derived(
    cn(
      "c-surface-motion rounded-xl border",
      tone === "strong"
        ? "border-ink bg-ink text-white"
        : tone === "subtle"
          ? "border-transparent bg-secondary/65"
          : "border-border bg-card",
      className,
    ),
  );
</script>

{#if as === "div"}
  <div class={classes}>{@render children?.()}</div>
{:else if as === "article"}
  <article class={classes}>{@render children?.()}</article>
{:else}
  <section class={classes}>{@render children?.()}</section>
{/if}
