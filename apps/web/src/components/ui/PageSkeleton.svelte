<script lang="ts">
  import Skeleton from "./Skeleton.svelte";
</script>

<div class="grid gap-5" aria-label="載入中" aria-busy="true">
  <div
    class="grid grid-cols-2 gap-3 rounded-xl border border-border bg-card p-5 md:grid-cols-4"
  >
    {#each Array(4) as _}
      <div class="grid gap-2">
        <Skeleton class="h-3 w-20" />
        <Skeleton class="h-7 w-32 max-w-full" />
        <Skeleton class="h-3 w-24" />
      </div>
    {/each}
  </div>
  <Skeleton class="h-72 w-full" />
  <Skeleton class="h-52 w-full" />
</div>
