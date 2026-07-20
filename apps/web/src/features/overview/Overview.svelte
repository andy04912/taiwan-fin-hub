<script lang="ts">
  import { createQuery } from "@tanstack/svelte-query";
  import Button from "../../components/ui/Button.svelte";
  import EmptyState from "../../components/ui/EmptyState.svelte";
  import InlineAlert from "../../components/ui/InlineAlert.svelte";
  import Metric from "../../components/ui/Metric.svelte";
  import PageSkeleton from "../../components/ui/PageSkeleton.svelte";
  import SummaryStrip from "../../components/ui/SummaryStrip.svelte";
  import Surface from "../../components/ui/Surface.svelte";
  import type { ApiClient } from "../../lib/api";
  import {
    bankQuery,
    exchangeRatesQuery,
    investmentsQuery,
    investmentTransactionsQuery,
    invoicesQuery,
    manualAssetsQuery,
    netWorthHistoryQuery,
    syncJobsQuery,
  } from "../../lib/queries";
  import type { View } from "../../lib/types";
  import {
    formatBankAccountName,
    formatCompactTwd,
    formatCurrency,
    formatDate,
    normalizeFinancialDate,
    rateMap,
    transactionValueTwd,
  } from "../../lib/format.svelte";
  import NetWorthHistoryChart from "./NetWorthHistoryChart.svelte";

  let { api, navigate }: { api: ApiClient; navigate: (view: View) => void } =
    $props();

  const bank = createQuery(bankQuery(() => api));
  const investments = createQuery(investmentsQuery(() => api));
  const invoices = createQuery(invoicesQuery(() => api));
  const trades = createQuery(investmentTransactionsQuery(() => api));
  const manualAssets = createQuery(manualAssetsQuery(() => api));
  const rates = createQuery(exchangeRatesQuery(() => api));
  const jobs = createQuery(syncJobsQuery(() => api));
  const history = createQuery(netWorthHistoryQuery(() => api));

  const bankData = $derived($bank.data ?? { accounts: [], transactions: [] });
  const rateValues = $derived(rateMap($rates.data));
  const toTwd = (value: number, currency: string) =>
    currency === "TWD" ? value : value * (rateValues[currency] ?? 0);
  const deposits = $derived(
    bankData.accounts.filter((account) => account.accountType !== "credit"),
  );
  const cards = $derived(
    bankData.accounts.filter((account) => account.accountType === "credit"),
  );
  const depositTotal = $derived(
    deposits.reduce(
      (sum, account) => sum + toTwd(account.balance ?? 0, account.currency),
      0,
    ),
  );
  const cardDebt = $derived(
    cards.reduce(
      (sum, account) =>
        sum + Math.abs(toTwd(account.balance ?? 0, account.currency)),
      0,
    ),
  );
  const investmentTotal = $derived(
    ($investments.data ?? []).reduce(
      (sum, item) =>
        sum +
        toTwd((item.marketValue ?? 0) + (item.cashBalance ?? 0), item.currency),
      0,
    ),
  );
  const manualTotal = $derived(
    ($manualAssets.data ?? []).reduce(
      (sum, item) => sum + (item.value ?? 0),
      0,
    ),
  );
  const gross = $derived(depositTotal + investmentTotal + manualTotal);
  const netWorth = $derived(gross - cardDebt);
  const pct = (value: number) =>
    gross > 0 ? Math.round((value / gross) * 100) : 0;
  const allocation = $derived([
    {
      label: "投資",
      value: investmentTotal,
      bar: "bg-steel",
      text: "text-steel",
      detail: `${$investments.data?.length ?? 0} 個持倉`,
    },
    {
      label: "存款",
      value: depositTotal,
      bar: "bg-moss",
      text: "text-moss",
      detail: `${deposits.length} 個帳戶`,
    },
    {
      label: "其他",
      value: manualTotal,
      bar: "bg-coral",
      text: "text-coral",
      detail: "保險、房產",
    },
  ]);
  const monthKey = new Date().toISOString().slice(0, 7);
  const monthlyBank = $derived(
    bankData.transactions.filter(
      (transaction) =>
        (transaction.postedDate ?? transaction.authorizedAt ?? "").startsWith(
          monthKey,
        ) &&
        transaction.accountType !== "credit" &&
        !transaction.excludedFromCalculation,
    ),
  );
  const monthlyIncome = $derived(
    monthlyBank.reduce(
      (sum, transaction) =>
        sum + Math.max(transactionValueTwd(transaction, rateValues), 0),
      0,
    ),
  );
  const monthlyExpense = $derived(
    Math.abs(
      monthlyBank.reduce(
        (sum, transaction) =>
          sum + Math.min(transactionValueTwd(transaction, rateValues), 0),
        0,
      ),
    ),
  );
  const unhealthy = $derived(
    ($jobs.data ?? []).filter(
      (job) =>
        job.lastStatus === "failed" || job.lastStatus === "needs_user_action",
    ),
  );
  const sourceCount = $derived(Math.max(($jobs.data ?? []).length, 4));
  const healthyCount = $derived(Math.max(sourceCount - unhealthy.length, 0));
  const accountRows = $derived(bankData.accounts.slice(0, 4));
  const recent = $derived.by(() =>
    [
      ...bankData.transactions.map((transaction) => ({
        id: `bank-${transaction.id}`,
        date: transaction.postedDate ?? transaction.authorizedAt ?? "",
        title:
          transaction.description ?? transaction.counterparty ?? "銀行交易",
        detail: transaction.institutionName ?? "銀行",
        amount: transaction.amount,
        currency: transaction.currency,
      })),
      ...($trades.data ?? []).map((trade) => ({
        id: `trade-${trade.id}`,
        date: normalizeFinancialDate(trade.tradeDate ?? trade.postedDate),
        title: trade.name ?? trade.transactionName ?? "投資交易",
        detail: trade.brokerName ?? "投資",
        amount: trade.price === 1 ? undefined : trade.amount,
        currency: trade.currency,
      })),
      ...($invoices.data ?? []).map((invoice) => ({
        id: `invoice-${invoice.id}`,
        date: invoice.invoiceDate,
        title: invoice.sellerName ?? "電子發票",
        detail: "電子發票",
        amount: -Math.abs(invoice.amount),
        currency: "TWD",
      })),
    ]
      .sort((a, b) => b.date.localeCompare(a.date))
      .slice(0, 4),
  );
  const missingRates = $derived([
    ...new Set(
      bankData.accounts
        .map((account) => account.currency)
        .filter((currency) => currency !== "TWD" && !rateValues[currency]),
    ),
  ]);
  const loading = $derived(
    $bank.isPending || $investments.isPending || $manualAssets.isPending,
  );
  const failed = $derived(
    $bank.isError || $investments.isError || $manualAssets.isError,
  );
</script>

{#if loading}
  <PageSkeleton />
{:else if failed}
  <EmptyState
    title="無法載入總覽"
    body="請稍後再試，或確認 Worker API 是否可用。"
  />
{:else}
  <div class="c-page-grid">
    {#if missingRates.length}
      <InlineAlert
        tone="warning"
        title={`尚缺 ${missingRates.join("、")} 匯率`}
        body="外幣帳戶暫未納入完整換算，總額可能低於實際資產。"
      >
        <Button size="sm" variant="ghost" onclick={() => navigate("settings")}
          >設定匯率</Button
        >
      </InlineAlert>
    {/if}

    <Surface tone="strong" class="overflow-hidden p-5 md:p-6">
      <div class="grid gap-6 lg:grid-cols-[minmax(0,1fr)_auto] lg:items-end">
        <div>
          <p class="text-xs font-semibold text-white/60">淨資產</p>
          <p
            class="mt-2 text-4xl font-bold tracking-[-0.04em] tabular-nums md:text-[42px]"
          >
            {formatCurrency(netWorth)}
          </p>
          <p class="mt-2 text-sm text-white/65">
            資產 {formatCurrency(gross)}，已扣除信用卡負債 {formatCurrency(
              cardDebt,
            )}
          </p>
        </div>
        <div
          class="min-w-56 rounded-lg border border-white/10 bg-white/[0.045] px-4 py-3"
        >
          <div class="flex items-center justify-between gap-4">
            <span class="text-xs text-white/55">資料來源</span>
            <span
              class={unhealthy.length
                ? "text-xs font-semibold text-amber-300"
                : "text-xs font-semibold text-emerald-300"}
            >
              {unhealthy.length ? `${unhealthy.length} 個待處理` : "全部正常"}
            </span>
          </div>
          <p class="mt-1 text-lg font-bold tabular-nums">
            {healthyCount} / {sourceCount} 正常
          </p>
        </div>
      </div>
      <div
        class="mt-6 flex h-1.5 overflow-hidden rounded-full bg-white/10"
        aria-label="資產配置"
      >
        {#each allocation as item (item.label)}
          <span
            class={`h-full ${item.bar}`}
            style={`width:${pct(item.value)}%`}
            title={`${item.label} ${pct(item.value)}%`}
          ></span>
        {/each}
      </div>
    </Surface>

    <SummaryStrip>
      {#each allocation as item (item.label)}
        <Metric
          label={item.label === "其他" ? "其他資產" : item.label}
          value={formatCurrency(item.value)}
          detail={`${item.detail}，佔 ${pct(item.value)}%`}
          tone={item.label === "存款"
            ? "positive"
            : item.label === "投資"
              ? "brand"
              : "neutral"}
        />
      {/each}
      <Metric
        label="本月淨流入"
        value={formatCurrency(monthlyIncome - monthlyExpense)}
        detail="收入減支出"
        tone={monthlyIncome >= monthlyExpense ? "positive" : "negative"}
      />
    </SummaryStrip>

    {#if unhealthy.length > 0}
      <InlineAlert
        tone="warning"
        title={`${unhealthy.length} 個資料來源需要處理`}
        body="同步失敗或需要重新驗證，處理前部分資料可能不是最新狀態。"
      >
        <Button
          size="sm"
          variant="ghost"
          onclick={() => navigate("data-sources")}>立即處理</Button
        >
      </InlineAlert>
    {/if}

    <div
      class="grid min-w-0 gap-5 xl:grid-cols-[minmax(0,1.75fr)_minmax(280px,0.75fr)]"
    >
      <div class="min-w-0">
        <NetWorthHistoryChart
          data={$history.data ?? []}
          loading={$history.isPending}
        />
      </div>
      <Surface class="p-5">
        <div class="flex items-center justify-between gap-3">
          <div>
            <h2 class="c-section-title">資產配置</h2>
            <p class="mt-1 text-xs text-muted-foreground">以新台幣換算</p>
          </div>
          <Button variant="ghost" size="sm" onclick={() => navigate("assets")}
            >查看全部</Button
          >
        </div>
        <div class="mt-5 grid gap-5">
          {#each allocation as item (item.label)}
            <div>
              <div class="flex items-end justify-between gap-3">
                <div>
                  <p class="text-sm font-semibold">{item.label}</p>
                  <p class="mt-0.5 text-xs text-muted-foreground">
                    {item.detail}
                  </p>
                </div>
                <div class="text-right">
                  <p class={`text-sm font-bold tabular-nums ${item.text}`}>
                    {formatCurrency(item.value)}
                  </p>
                  <p class="mt-0.5 text-xs text-muted-foreground">
                    {pct(item.value)}%
                  </p>
                </div>
              </div>
              <div class="mt-2 h-1.5 overflow-hidden rounded-full bg-secondary">
                <span
                  class={`block h-full rounded-full ${item.bar}`}
                  style={`width:${pct(item.value)}%`}
                ></span>
              </div>
            </div>
          {/each}
        </div>
      </Surface>
    </div>

    <div class="grid gap-5 xl:grid-cols-[minmax(0,1.4fr)_minmax(320px,0.8fr)]">
      <Surface class="overflow-hidden">
        <div
          class="flex items-center justify-between border-b border-border px-5 py-4"
        >
          <div>
            <h2 class="c-section-title">銀行與信用卡</h2>
            <p class="mt-0.5 text-xs text-muted-foreground">最近帳戶餘額</p>
          </div>
          <Button variant="ghost" size="sm" onclick={() => navigate("assets")}
            >查看資產</Button
          >
        </div>
        <div class="divide-y divide-border">
          {#each accountRows as account (account.id)}
            <div
              class="c-data-row grid grid-cols-[minmax(0,1fr)_auto] gap-3 px-5 py-3.5 md:grid-cols-[130px_minmax(0,1fr)_auto_88px] md:items-center"
            >
              <span class="truncate text-sm font-semibold"
                >{account.institutionName ?? account.connectorId}</span
              >
              <span
                class="hidden truncate text-sm text-muted-foreground md:block"
                >{account.accountName ?? formatBankAccountName(account)}</span
              >
              <span
                class={`text-right text-sm font-bold tabular-nums ${account.accountType === "credit" ? "text-coral" : ""}`}
                >{formatCurrency(account.balance ?? 0, account.currency)}</span
              >
              <span
                class="hidden text-right text-xs text-muted-foreground md:block"
                >{account.asOfAt
                  ? formatDate(account.asOfAt)
                  : "尚未同步"}</span
              >
            </div>
          {:else}
            <p class="px-5 py-8 text-center text-sm text-muted-foreground">
              尚無銀行或信用卡資料。
            </p>
          {/each}
        </div>
      </Surface>

      <Surface class="overflow-hidden">
        <div
          class="flex items-center justify-between border-b border-border px-5 py-4"
        >
          <div>
            <h2 class="c-section-title">近期活動</h2>
            <p class="mt-0.5 text-xs text-muted-foreground">跨來源財務紀錄</p>
          </div>
          <Button variant="ghost" size="sm" onclick={() => navigate("activity")}
            >全部活動</Button
          >
        </div>
        <div class="divide-y divide-border">
          {#each recent as item (item.id)}
            <div
              class="c-data-row flex min-w-0 items-center justify-between gap-3 px-5 py-3.5 text-sm"
            >
              <div class="min-w-0 flex-1">
                <p class="truncate font-semibold">{item.title}</p>
                <p class="mt-0.5 truncate text-xs text-muted-foreground">
                  {item.detail}
                </p>
              </div>
              <span
                class={`max-w-[46%] shrink-0 truncate font-bold tabular-nums ${item.amount != null && item.amount < 0 ? "text-coral" : item.amount != null ? "text-moss" : "text-muted-foreground"}`}
                >{item.amount == null
                  ? "—"
                  : `${item.amount >= 0 ? "+" : ""}${formatCurrency(item.amount, item.currency)}`}</span
              >
            </div>
          {:else}
            <p class="px-5 py-8 text-center text-sm text-muted-foreground">
              尚無近期活動。
            </p>
          {/each}
        </div>
      </Surface>
    </div>
  </div>
{/if}
