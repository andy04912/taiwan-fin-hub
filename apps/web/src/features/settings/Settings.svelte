<script lang="ts">
  import { createQuery } from "@tanstack/svelte-query";
  import { AlertTriangle, CircleCheck, Coins, Database, ListFilter, RefreshCw } from "@lucide/svelte";
  import type { ApiClient } from "../../lib/api";
  import { bankQuery, classificationRulesQuery, syncJobsQuery } from "../../lib/queries";
  import type { ConnectorField, ConnectorId, MobileSettingsView, View } from "../../lib/types";
  import Surface from "../../components/ui/Surface.svelte";
  import SourceCard from "./SourceCard.svelte";
  import ExchangeRatesPanel from "./ExchangeRatesPanel.svelte";
  import ClassificationRulesPanel from "./ClassificationRulesPanel.svelte";
  import ConnectorPanel from "./ConnectorPanel.svelte";
  import MobileMore from "./MobileMore.svelte";
  import DefaultSchedulePanel from "./DefaultSchedulePanel.svelte";

  let { api, demoMode, mobileView, navigate }: {
    api: ApiClient;
    demoMode: boolean;
    mobileView?: MobileSettingsView | "more";
    navigate: (view: View) => void;
  } = $props();

  const sources: { id: ConnectorId; title: string; description: string }[] = [
    { id: "einvoice", title: "電子發票", description: "財政部載具與品項明細" },
    { id: "tdcc", title: "集保 e 存摺", description: "持倉、投資交易與銀行帳戶" },
    { id: "esun", title: "玉山銀行", description: "帳戶、信用卡與交易" },
    { id: "cathaybk", title: "國泰世華銀行", description: "帳戶與交易" },
    { id: "sinopac", title: "永豐行動銀行", description: "信用卡帳務、近期帳單與消費" },
  ];

  const connectorFields: Record<ConnectorId, ConnectorField[]> = {
    einvoice: [
      { key: "mobile", label: "手機號碼（電子發票帳號）", type: "text" },
      { key: "password", label: "電子發票 App 登入密碼", type: "password" },
      { key: "periodsBack", label: "往回期數", type: "number", placeholder: "6" },
      { key: "fetchDetails", label: "同步品項明細", type: "checkbox" },
    ],
    tdcc: [
      { key: "userId", label: "身分證字號", type: "text" },
      { key: "password", label: "集保 App 密碼", type: "password" },
    ],
    esun: [
      { key: "username", label: "使用者名稱", type: "text" },
      { key: "password", label: "密碼", type: "password" },
      { key: "lookbackMonths", label: "往回月份", type: "number", placeholder: "3" },
    ],
    cathaybk: [
      { key: "username", label: "使用者名稱", type: "text" },
      { key: "password", label: "密碼", type: "password" },
      { key: "lookbackMonths", label: "往回月份", type: "number", placeholder: "3" },
    ],
    sinopac: [
      { key: "userId", label: "身分證字號／統編", type: "text" },
      { key: "account", label: "行動／網路銀行使用者代碼", type: "text" },
      { key: "password", label: "網路密碼", type: "password" },
      { key: "lookbackMonths", label: "帳單往回月份", type: "number", placeholder: "3" },
    ],
  };

  const jobs = createQuery(syncJobsQuery(() => api));
  const rules = createQuery(classificationRulesQuery(() => api));
  const bank = createQuery(bankQuery(() => api));
  let selectedConnector = $state<ConnectorId>("einvoice");
  const selectedSource = $derived(sources.find((source) => source.id === selectedConnector)!);
  const enabledJobs = $derived(($jobs.data ?? []).filter((job) => job.enabled).length);
  const needsAction = $derived(
    ($jobs.data ?? []).filter((job) => job.lastStatus === "failed" || job.lastStatus === "needs_user_action").length,
  );
  const navItems = $derived([
    { href: "#settings-schedule", label: "同步排程", icon: RefreshCw },
    { href: "#settings-sources", label: "資料來源", detail: String(sources.length), icon: Database },
    { href: "#settings-rates", label: "匯率", icon: Coins },
    { href: "#settings-rules", label: "分類規則", detail: String($rules.data?.length ?? 0), icon: ListFilter },
  ]);
</script>

{#if mobileView === "more"}
  <MobileMore {demoMode} jobs={$jobs.data ?? []} rules={$rules.data ?? []} bank={$bank.data ?? { accounts: [], transactions: [] }} {navigate} {api} />
{:else if mobileView === "data-sources"}
  <div class="c-page-grid">
    <DefaultSchedulePanel {api} {demoMode} jobs={$jobs.data ?? []} />
    <div class="grid items-start gap-4 lg:grid-cols-[minmax(280px,0.85fr)_minmax(0,1.4fr)]">
      <Surface class="overflow-hidden">
        {#each sources as source (source.id)}
          <SourceCard {api} {...source} id={source.id} jobs={$jobs.data ?? []} selected={selectedConnector === source.id} onConfigure={() => (selectedConnector = source.id)} />
        {/each}
      </Surface>
      <Surface class="min-w-0 p-5">
        {#key selectedConnector}<ConnectorPanel {api} connectorId={selectedConnector} {demoMode} title={selectedSource.title} fields={connectorFields[selectedConnector]} embedded />{/key}
      </Surface>
    </div>
  </div>
{:else if mobileView === "exchange-rates"}<ExchangeRatesPanel {api} />
{:else if mobileView === "classification-rules"}<ClassificationRulesPanel {api} />
{:else}
  <div class="grid min-w-0 gap-5">
    <section class={`flex flex-col gap-3 rounded-xl border px-4 py-3.5 sm:flex-row sm:items-center sm:justify-between ${needsAction ? "border-coral/25 bg-coral/[0.045]" : "border-moss/20 bg-moss/[0.045]"}`} aria-live="polite">
      <div class="flex min-w-0 items-start gap-3">
        <span class={`mt-0.5 grid size-8 shrink-0 place-items-center rounded-full ${needsAction ? "bg-coral/10 text-coral" : "bg-moss/10 text-moss"}`}>
          {#if needsAction}<AlertTriangle class="size-4.5" />{:else}<CircleCheck class="size-4.5" />{/if}
        </span>
        <div class="min-w-0">
          <p class="font-semibold">{needsAction ? `${needsAction} 個資料來源需要處理` : "所有資料來源運作正常"}</p>
          <p class="mt-0.5 text-xs text-muted-foreground">{sources.length} 個來源 · {enabledJobs} 個已啟用 · {$rules.data?.length ?? 0} 條分類規則</p>
        </div>
      </div>
      {#if needsAction}<a class="self-start text-sm font-semibold text-coral hover:underline sm:self-center" href="#settings-sources">立即查看</a>{/if}
    </section>

    <div class="grid items-start gap-6 lg:grid-cols-[208px_minmax(0,1fr)]">
      <nav aria-label="設定區段" class="sticky top-5 hidden rounded-xl border border-border bg-card p-2 shadow-sm lg:grid">
        <p class="px-3 pb-2 pt-1 text-xs font-semibold text-muted-foreground">設定選單</p>
        {#each navItems as item (item.href)}
          {@const NavIcon = item.icon}
          <a class="flex min-h-10 items-center gap-2.5 rounded-lg px-3 text-sm font-medium text-muted-foreground transition hover:bg-secondary hover:text-ink" href={item.href}>
            <NavIcon class="size-4.5" /><span class="flex-1">{item.label}</span>{#if item.detail}<span class="text-xs tabular-nums text-muted-foreground">{item.detail}</span>{/if}
          </a>
        {/each}
      </nav>

      <div class="grid min-w-0 gap-8">
        <section id="settings-schedule" class="scroll-mt-24"><DefaultSchedulePanel {api} {demoMode} jobs={$jobs.data ?? []} /></section>

        <section id="settings-sources" aria-labelledby="settings-sources-title" class="scroll-mt-24">
          <div class="mb-3">
            <h2 id="settings-sources-title" class="text-lg font-semibold tracking-tight">資料來源</h2>
            <p class="mt-1 text-sm text-muted-foreground">選擇來源後，可在右側管理連線、同步與排程。</p>
          </div>
          <div class="grid min-w-0 items-start overflow-hidden rounded-xl border border-border bg-card shadow-sm xl:grid-cols-[minmax(300px,0.82fr)_minmax(0,1.35fr)]">
            <div class="border-b border-border xl:border-b-0 xl:border-r">
              {#each sources as source (source.id)}
                <SourceCard {api} {...source} id={source.id} jobs={$jobs.data ?? []} selected={selectedConnector === source.id} onConfigure={() => (selectedConnector = source.id)} />
              {/each}
            </div>
            <div class="min-w-0 p-5 xl:p-6">
              {#key selectedConnector}<ConnectorPanel {api} connectorId={selectedConnector} {demoMode} title={selectedSource.title} fields={connectorFields[selectedConnector]} embedded />{/key}
            </div>
          </div>
          <p class="mt-3 text-xs leading-relaxed text-muted-foreground">連接器憑證只用於個人資料同步；機密欄位會加密保存，儲存後不會重新顯示。</p>
        </section>

        <section id="settings-rates" class="scroll-mt-24"><ExchangeRatesPanel {api} /></section>
        <section id="settings-rules" class="scroll-mt-24"><ClassificationRulesPanel {api} /></section>
      </div>
    </div>
  </div>
{/if}
