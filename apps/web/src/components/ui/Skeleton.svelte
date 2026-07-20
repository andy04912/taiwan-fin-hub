<script lang="ts">
  import { cn } from "../../lib/utils";
  let { class: className = "" }: { class?: string } = $props();
</script>

<div
  aria-hidden="true"
  class={cn("animate-pulse rounded-lg bg-ink/8", className)}
></div>
