<script lang="ts">
  let { title, body }: { title: string; body: string } = $props();
</script>

<section
  class="rounded-xl border border-dashed border-border bg-white/55 p-8 text-center"
>
  <h2 class="text-lg font-semibold">{title}</h2>
  <p
    class="mx-auto mt-2 max-w-md text-sm leading-relaxed text-muted-foreground"
  >
    {body}
  </p>
</section>
