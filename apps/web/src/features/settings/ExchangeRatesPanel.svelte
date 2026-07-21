<script lang="ts">
  import { onMount } from "svelte";
  import { createMutation, createQuery, useQueryClient } from "@tanstack/svelte-query";
  import { Coins, Save } from "@lucide/svelte";
  import Card from "../../components/ui/Card.svelte";
  import Button from "../../components/ui/Button.svelte";
  import Input from "../../components/ui/Input.svelte";
  import type { ApiClient } from "../../lib/api";
  import { queryKeys } from "../../lib/api";
  import { bankQuery, exchangeRatesQuery } from "../../lib/queries";

  let { api }: { api: ApiClient } = $props();
  const rates = createQuery(exchangeRatesQuery(() => api));
  const bank = createQuery(bankQuery(() => api));
  const qc = useQueryClient();
  let values = $state<Record<string, string>>({});
  const currencies = $derived([
    ...new Set(($bank.data?.accounts ?? []).map((account) => account.currency).filter((currency) => currency && currency !== "TWD")),
  ]);
  const save = createMutation({
    mutationFn: (payload: Record<string, number>) => api.put("/api/exchange-rates", { rates: payload }),
    onSuccess: () => qc.invalidateQueries({ queryKey: queryKeys.exchangeRates }),
  });

  onMount(() =>
    rates.subscribe((result) => {
      for (const rate of result.data ?? []) {
        if (values[rate.currency] === undefined) values[rate.currency] = String(rate.rateTwd);
      }
    }),
  );
</script>

<Card class="overflow-hidden shadow-sm">
  <div class="flex flex-col gap-3 border-b border-border px-5 py-4 sm:flex-row sm:items-start sm:justify-between">
    <div class="flex items-start gap-3">
      <span class="grid size-9 shrink-0 place-items-center rounded-lg bg-steel/8 text-steel"><Coins class="size-4.5" /></span>
      <div>
        <h2 class="font-semibold">匯率</h2>
        <p class="mt-1 text-xs text-muted-foreground">設定外幣資產換算為新台幣時使用的參考匯率。</p>
      </div>
    </div>
    {#if currencies.length > 0}<span class="text-xs text-muted-foreground">{currencies.length} 種外幣</span>{/if}
  </div>

  {#if currencies.length === 0}
    <div class="px-5 py-8 text-center">
      <p class="text-sm font-medium text-ink/70">目前沒有外幣帳戶</p>
      <p class="mt-1 text-xs text-muted-foreground">新增外幣帳戶後，即可在這裡管理換算匯率。</p>
    </div>
  {:else}
    <div class="divide-y divide-border">
      {#each currencies as currency (currency)}
        <label class="grid gap-3 px-5 py-3.5 sm:grid-cols-[minmax(0,1fr)_180px] sm:items-center">
          <div>
            <span class="text-sm font-semibold">{currency}</span>
            <p class="mt-0.5 text-xs text-muted-foreground">1 {currency} 可換算的新台幣金額</p>
          </div>
          <Input class="w-full text-right tabular-nums" type="number" step="0.0001" bind:value={values[currency]} />
        </label>
      {/each}
    </div>
    <div class="flex flex-col gap-3 border-t border-border bg-muted/25 px-5 py-4 sm:flex-row sm:items-center sm:justify-between">
      <p class="text-xs text-muted-foreground">儲存後，系統會使用新匯率重新計算資產總額。</p>
      <Button class="w-full sm:w-auto" variant="primary" disabled={$save.isPending} onclick={() => $save.mutate(Object.fromEntries(Object.entries(values).map(([key, value]) => [key, Number(value)])))}>
        <Save class="size-4" />{$save.isPending ? "儲存中…" : "儲存匯率"}
      </Button>
    </div>
  {/if}
</Card>
