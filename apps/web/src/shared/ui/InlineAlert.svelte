<script lang="ts">
  import type { Snippet } from "svelte";
  import { AlertTriangle, CircleCheck, Info } from "@lucide/svelte";
  import { cn } from "../../lib/utils";
  let {
    title,
    body,
    tone = "info",
    children,
    class: className = "",
  }: {
    title: string;
    body?: string;
    tone?: "info" | "warning" | "success";
    children?: Snippet;
    class?: string;
  } = $props();
  const AlertIcon = $derived(
    tone === "warning"
      ? AlertTriangle
      : tone === "success"
        ? CircleCheck
        : Info,
  );
</script>

<section
  class={cn(
    "flex items-start gap-3 rounded-xl border px-4 py-3",
    tone === "warning"
      ? "border-amber-200 bg-amber-50 text-amber-900"
      : tone === "success"
        ? "border-moss/20 bg-moss/5 text-moss"
        : "border-steel/20 bg-steel/5 text-ink",
    className,
  )}
>
  <AlertIcon class="mt-0.5 size-4 shrink-0" />
  <div class="min-w-0 flex-1">
    <p class="text-sm font-semibold">{title}</p>
    {#if body}<p class="mt-0.5 text-xs leading-relaxed opacity-75">
        {body}
      </p>{/if}
  </div>
  {@render children?.()}
</section>
