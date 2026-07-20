<script lang="ts">
  import { createQuery } from "@tanstack/svelte-query";
  import { ChevronDown, Search, X } from "@lucide/svelte";
  import Button from "../../components/ui/Button.svelte";
  import EmptyState from "../../components/ui/EmptyState.svelte";
  import Input from "../../components/ui/Input.svelte";
  import Metric from "../../components/ui/Metric.svelte";
  import PageSkeleton from "../../components/ui/PageSkeleton.svelte";
  import Select from "../../components/ui/Select.svelte";
  import SummaryStrip from "../../components/ui/SummaryStrip.svelte";
  import Surface from "../../components/ui/Surface.svelte";
  import type { ApiClient } from "../../lib/api";
  import { invoicesQuery } from "../../lib/queries";
  import { formatCurrency, formatDate } from "../../lib/format.svelte";

  let { api }: { api: ApiClient } = $props();
  const invoices = createQuery(invoicesQuery(() => api));
  let search = $state("");
  let selectedMonth = $state("all");
  let selectedSeller = $state("all");
  let amountMin = $state("");
  let amountMax = $state("");
  let expanded = $state<Record<string, boolean>>({});
  const all = $derived($invoices.data ?? []);
  const monthOptions = $derived(
    [...new Set(all.map((invoice) => invoice.invoiceDate.slice(0, 7)))].sort(
      (a, b) => b.localeCompare(a),
    ),
  );
  const sellerOptions = $derived(
    [
      ...new Set(
        all
          .map((invoice) => invoice.sellerName)
          .filter((name): name is string => Boolean(name)),
      ),
    ].sort((a, b) => a.localeCompare(b, "zh-TW")),
  );
  const filtered = $derived(
    all
      .filter((invoice) => {
        const matchesSearch =
          `${invoice.sellerName ?? ""} ${invoice.invoiceNumber ?? ""} ${invoice.items.map((item) => item.description).join(" ")}`
            .toLowerCase()
            .includes(search.trim().toLowerCase());
        return (
          matchesSearch &&
          (selectedMonth === "all" ||
            invoice.invoiceDate.startsWith(selectedMonth)) &&
          (selectedSeller === "all" || invoice.sellerName === selectedSeller) &&
          (!amountMin || invoice.amount >= Number(amountMin)) &&
          (!amountMax || invoice.amount <= Number(amountMax))
        );
      })
      .sort((a, b) => b.invoiceDate.localeCompare(a.invoiceDate)),
  );
  const months = $derived(
    [
      ...new Set(filtered.map((invoice) => invoice.invoiceDate.slice(0, 7))),
    ].sort((a, b) => b.localeCompare(a)),
  );
  const thisMonth = new Date().toISOString().slice(0, 7);
  const thisMonthInvoices = $derived(
    all.filter((invoice) => invoice.invoiceDate.startsWith(thisMonth)),
  );
  const thisMonthTotal = $derived(
    thisMonthInvoices.reduce((sum, invoice) => sum + invoice.amount, 0),
  );
  const allExpanded = $derived(
    filtered.length > 0 && filtered.every((invoice) => expanded[invoice.id]),
  );
  function toggleAll() {
    if (allExpanded) expanded = {};
    else
      expanded = Object.fromEntries(
        filtered.map((invoice) => [invoice.id, true]),
      );
  }
  const hasFilters = $derived(
    Boolean(
      search ||
      selectedMonth !== "all" ||
      selectedSeller !== "all" ||
      amountMin ||
      amountMax,
    ),
  );
  function clearFilters() {
    search = "";
    selectedMonth = "all";
    selectedSeller = "all";
    amountMin = "";
    amountMax = "";
  }
  function matchReason(invoice: (typeof all)[number]) {
    const query = search.trim().toLowerCase();
    if (!query) return "";
    if ((invoice.sellerName ?? "").toLowerCase().includes(query))
      return "符合商家";
    if ((invoice.invoiceNumber ?? "").toLowerCase().includes(query))
      return "符合發票號碼";
    if (
      invoice.items.some((item) =>
        item.description.toLowerCase().includes(query),
      )
    )
      return "符合品項";
    return "";
  }
</script>

{#if $invoices.isPending}
  <PageSkeleton />
{:else if $invoices.isError}
  <EmptyState title="無法載入發票" body="請稍後再試。" />
{:else}
  <section class="c-page-grid">
    <SummaryStrip class="sm:grid-cols-3 sm:[&>*]:pl-5">
      <Metric
        label="本月消費"
        value={formatCurrency(thisMonthTotal)}
        detail={`${thisMonthInvoices.length} 張發票`}
        tone="negative"
      />
      <Metric
        label="發票總數"
        value={all.length.toLocaleString()}
        detail="已同步記錄"
      />
      <Metric
        label="本月均消"
        value={thisMonthInvoices.length > 0
          ? formatCurrency(thisMonthTotal / thisMonthInvoices.length)
          : "—"}
        detail="每張平均"
      />
    </SummaryStrip>

    <Surface
      class="sticky top-[68px] z-10 p-3 shadow-[0_8px_22px_rgba(24,38,45,0.05)] md:top-2"
    >
      <div
        class="grid gap-2 md:grid-cols-[minmax(220px,1.5fr)_minmax(140px,0.7fr)_minmax(150px,0.8fr)_110px_110px_auto]"
      >
        <div class="relative min-w-0">
          <Search
            class="pointer-events-none absolute left-3 top-1/2 z-10 size-4 -translate-y-1/2 text-muted-foreground"
          />
          <Input
            class="h-10 pl-9"
            placeholder="搜尋商家、發票號碼或品項"
            aria-label="搜尋發票"
            bind:value={search}
          />
        </div>
        <Select aria-label="篩選月份" bind:value={selectedMonth}>
          <option value="all">全部月份</option>
          {#each monthOptions as month (month)}<option value={month}
              >{month.slice(0, 4)} 年 {Number(month.slice(5))} 月</option
            >{/each}
        </Select>
        <Select aria-label="篩選商家" bind:value={selectedSeller}>
          <option value="all">全部商家</option>
          {#each sellerOptions as seller (seller)}<option value={seller}
              >{seller}</option
            >{/each}
        </Select>
        <Input
          type="number"
          min="0"
          placeholder="最低金額"
          aria-label="最低金額"
          bind:value={amountMin}
        />
        <Input
          type="number"
          min="0"
          placeholder="最高金額"
          aria-label="最高金額"
          bind:value={amountMax}
        />
        <div class="flex gap-2">
          {#if hasFilters}<Button
              size="icon"
              variant="ghost"
              aria-label="清除篩選"
              title="清除篩選"
              onclick={clearFilters}><X class="size-4" /></Button
            >{/if}
          {#if filtered.length > 0}<Button
              class="flex-1 md:flex-none"
              variant="outline"
              onclick={toggleAll}>{allExpanded ? "收合" : "展開"}</Button
            >{/if}
        </div>
      </div>
      <p class="mt-2 px-1 text-xs text-muted-foreground">
        顯示 {filtered.length.toLocaleString()} / {all.length.toLocaleString()} 張發票
      </p>
    </Surface>

    {#if months.length === 0}
      <EmptyState
        title={search.trim() ? "無符合結果" : "尚無發票記錄"}
        body={search.trim() ? "請調整搜尋條件。" : "同步電子發票連接器後顯示。"}
      />
    {:else}
      <Surface class="overflow-hidden">
        {#each months as month (month)}
          {@const group = filtered.filter((invoice) =>
            invoice.invoiceDate.startsWith(month),
          )}
          <div>
            <div
              class="sticky top-[176px] z-[1] flex items-center justify-between border-b border-border bg-secondary/95 px-4 py-2 backdrop-blur md:top-[94px]"
            >
              <span class="text-xs font-semibold text-ink/55"
                >{month.slice(0, 4)} 年 {Number(month.slice(5))} 月</span
              ><span class="text-xs font-semibold tabular-nums text-ink/55"
                >{formatCurrency(
                  group.reduce((sum, invoice) => sum + invoice.amount, 0),
                )}</span
              >
            </div>
            <div class="divide-y divide-border">
              {#each group as invoice (invoice.id)}
                <div>
                  <button
                    class={`c-data-row grid w-full grid-cols-[minmax(0,1fr)_auto] items-center gap-3 px-4 py-3 text-left md:grid-cols-[120px_minmax(0,1fr)_140px_90px_24px] ${expanded[invoice.id] ? "bg-accent/55" : ""}`}
                    onclick={() =>
                      (expanded[invoice.id] = !expanded[invoice.id])}
                  >
                    <p class="hidden text-sm text-muted-foreground md:block">
                      {formatDate(invoice.invoiceDate)}
                    </p>
                    <div class="min-w-0">
                      <p class="truncate text-sm font-medium">
                        {invoice.sellerName ?? "未知商家"}
                      </p>
                      <p class="mt-0.5 text-xs text-muted-foreground">
                        <span class="md:hidden"
                          >{formatDate(
                            invoice.invoiceDate,
                          )}{invoice.invoiceNumber
                            ? ` · ${invoice.invoiceNumber}`
                            : ""}</span
                        >
                        <span class="hidden md:inline"
                          >{matchReason(invoice) ||
                            invoice.invoiceNumber ||
                            "無發票號碼"}</span
                        >
                      </p>
                    </div>
                    <p
                      class="hidden truncate text-xs text-muted-foreground md:block"
                    >
                      {invoice.invoiceNumber ?? "—"}
                    </p>
                    <div class="shrink-0 text-right">
                      <p class="text-sm font-bold tabular-nums">
                        {formatCurrency(invoice.amount)}
                      </p>
                      {#if invoice.items.length > 0}<p
                          class="text-xs text-muted-foreground md:hidden"
                        >
                          {invoice.items.length} 項
                        </p>{/if}
                    </div>
                    <ChevronDown
                      class={`hidden size-4 text-muted-foreground transition-transform md:block ${expanded[invoice.id] ? "rotate-180" : ""}`}
                    />
                  </button>
                  {#if expanded[invoice.id]}
                    <div class="border-t border-border bg-secondary/35">
                      {#if invoice.items.length > 0}
                        <div class="divide-y divide-ink/6">
                          {#each invoice.items as item (item.id)}<div
                              class="flex items-start gap-3 px-5 py-2.5 md:pl-[152px]"
                            >
                              <div class="min-w-0 flex-1">
                                <p class="text-sm text-ink/80">
                                  {item.description}
                                </p>
                                {#if item.quantity != null || item.unitPrice != null}<p
                                    class="mt-0.5 text-xs text-ink/45"
                                  >
                                    {item.quantity != null
                                      ? `${item.quantity.toLocaleString()} × `
                                      : ""}{item.unitPrice != null
                                      ? formatCurrency(item.unitPrice)
                                      : ""}
                                  </p>{/if}
                              </div>
                              <p
                                class="shrink-0 text-sm font-medium tabular-nums"
                              >
                                {formatCurrency(item.amount)}
                              </p>
                            </div>{/each}
                        </div>
                      {:else}<p class="px-5 py-3 text-xs text-ink/40">
                          無品項記錄
                        </p>{/if}
                      <div
                        class="flex items-center justify-between border-t border-border px-5 py-2.5 md:pl-[152px]"
                      >
                        <p class="text-xs font-semibold text-ink/50">合計</p>
                        <p class="text-sm font-bold tabular-nums">
                          {formatCurrency(invoice.amount)}
                        </p>
                      </div>
                    </div>
                  {/if}
                </div>
              {/each}
            </div>
          </div>
        {/each}
      </Surface>
    {/if}
  </section>
{/if}
