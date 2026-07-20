<script lang="ts">
  import { createQuery } from "@tanstack/svelte-query";
  import type { ApiClient } from "../../lib/api";
  import {
    bankQuery,
    classificationRulesQuery,
    syncJobsQuery,
  } from "../../lib/queries";
  import type {
    ConnectorField,
    ConnectorId,
    MobileSettingsView,
    View,
  } from "../../lib/types";
  import InlineAlert from "../../components/ui/InlineAlert.svelte";
  import Metric from "../../components/ui/Metric.svelte";
  import SummaryStrip from "../../components/ui/SummaryStrip.svelte";
  import Surface from "../../components/ui/Surface.svelte";
  import SourceCard from "./SourceCard.svelte";
  import ExchangeRatesPanel from "./ExchangeRatesPanel.svelte";
  import ClassificationRulesPanel from "./ClassificationRulesPanel.svelte";
  import ConnectorPanel from "./ConnectorPanel.svelte";
  import MobileMore from "./MobileMore.svelte";
  import DefaultSchedulePanel from "./DefaultSchedulePanel.svelte";
  let {
    api,
    demoMode,
    mobileView,
    navigate,
  }: {
    api: ApiClient;
    demoMode: boolean;
    mobileView?: MobileSettingsView | "more";
    navigate: (view: View) => void;
  } = $props();
  const sources: { id: ConnectorId; title: string; description: string }[] = [
    { id: "einvoice", title: "電子發票", description: "財政部載具與品項明細" },
    {
      id: "tdcc",
      title: "集保 e 存摺",
      description: "持倉、投資交易與銀行帳戶",
    },
    { id: "esun", title: "玉山銀行", description: "帳戶、信用卡與交易" },
    { id: "cathaybk", title: "國泰世華銀行", description: "帳戶與交易" },
    {
      id: "sinopac",
      title: "永豐行動銀行",
      description: "信用卡帳務、近期帳單與消費",
    },
  ];
  const connectorFields: Record<ConnectorId, ConnectorField[]> = {
    einvoice: [
      {
        key: "mobile",
        label: "手機號碼（電子發票帳號）",
        type: "text",
      },
      {
        key: "password",
        label: "電子發票 App 登入密碼",
        type: "password",
      },
      {
        key: "periodsBack",
        label: "往回期數",
        type: "number",
        placeholder: "6",
      },
      { key: "fetchDetails", label: "同步品項明細", type: "checkbox" },
    ],
    tdcc: [
      { key: "userId", label: "身分證字號", type: "text" },
      { key: "password", label: "集保 App 密碼", type: "password" },
    ],
    esun: [
      { key: "username", label: "使用者名稱", type: "text" },
      { key: "password", label: "密碼", type: "password" },
      {
        key: "lookbackMonths",
        label: "往回月份",
        type: "number",
        placeholder: "3",
      },
    ],
    cathaybk: [
      { key: "username", label: "使用者名稱", type: "text" },
      { key: "password", label: "密碼", type: "password" },
      {
        key: "lookbackMonths",
        label: "往回月份",
        type: "number",
        placeholder: "3",
      },
    ],
    sinopac: [
      { key: "userId", label: "身分證字號／統編", type: "text" },
      { key: "account", label: "行動／網路銀行使用者代碼", type: "text" },
      { key: "password", label: "網路密碼", type: "password" },
      {
        key: "lookbackMonths",
        label: "帳單往回月份",
        type: "number",
        placeholder: "3",
      },
    ],
  };
  const jobs = createQuery(syncJobsQuery(() => api));
  const rules = createQuery(classificationRulesQuery(() => api));
  const bank = createQuery(bankQuery(() => api));
  let selectedConnector = $state<ConnectorId>("einvoice");
  const selectedSource = $derived(
    sources.find((source) => source.id === selectedConnector)!,
  );
  const enabledJobs = $derived(
    ($jobs.data ?? []).filter((j) => j.enabled).length,
  );
  const needsAction = $derived(
    ($jobs.data ?? []).filter(
      (j) => j.lastStatus === "failed" || j.lastStatus === "needs_user_action",
    ).length,
  );
  function selectConnector(id: ConnectorId) {
    selectedConnector = id;
  }
</script>

{#if mobileView === "more"}
  <MobileMore
    {demoMode}
    jobs={$jobs.data ?? []}
    rules={$rules.data ?? []}
    bank={$bank.data ?? { accounts: [], transactions: [] }}
    {navigate}
    {api}
  />
{:else if mobileView === "data-sources"}
  <div class="c-page-grid">
    <DefaultSchedulePanel {api} {demoMode} jobs={$jobs.data ?? []} />
    <div
      class="grid items-start gap-4 lg:grid-cols-[minmax(280px,0.85fr)_minmax(0,1.4fr)]"
    >
      <Surface class="overflow-hidden">
        {#each sources as source (source.id)}
          <SourceCard
            {api}
            {...source}
            id={source.id}
            jobs={$jobs.data ?? []}
            selected={selectedConnector === source.id}
            onConfigure={() => selectConnector(source.id)}
          />
        {/each}
      </Surface>
      <Surface class="min-w-0 p-5">
        {#key selectedConnector}<ConnectorPanel
            {api}
            connectorId={selectedConnector}
            {demoMode}
            title={selectedSource.title}
            fields={connectorFields[selectedConnector]}
            embedded
          />{/key}
      </Surface>
    </div>
  </div>
{:else if mobileView === "exchange-rates"}<ExchangeRatesPanel {api} />
{:else if mobileView === "classification-rules"}<ClassificationRulesPanel
    {api}
  />
{:else}
  <div class="c-page-grid">
    <SummaryStrip>
      <Metric
        label="資料來源"
        value={String(sources.length)}
        detail="支援的連接器"
      /><Metric
        label="同步排程"
        value={String(enabledJobs)}
        detail="已啟用"
        tone={enabledJobs ? "positive" : "neutral"}
      /><Metric
        label="分類規則"
        value={String($rules.data?.length ?? 0)}
        detail="銀行交易"
      /><Metric
        label="需要處理"
        value={String(needsAction)}
        detail="同步或驗證狀態"
        tone={needsAction ? "negative" : "positive"}
      />
    </SummaryStrip>

    <div class="grid items-start gap-5 lg:grid-cols-[176px_minmax(0,1fr)]">
      <nav
        aria-label="設定區段"
        class="sticky top-5 hidden rounded-xl border border-border bg-card p-2 lg:grid"
      >
        <a
          class="rounded-lg bg-accent px-3 py-2 text-sm font-semibold text-accent-foreground"
          href="#settings-schedule">同步排程</a
        >
        <a
          class="rounded-lg px-3 py-2 text-sm font-medium text-muted-foreground hover:bg-secondary hover:text-ink"
          href="#settings-sources">資料來源</a
        >
        <a
          class="rounded-lg px-3 py-2 text-sm font-medium text-muted-foreground hover:bg-secondary hover:text-ink"
          href="#settings-rates">匯率</a
        >
        <a
          class="rounded-lg px-3 py-2 text-sm font-medium text-muted-foreground hover:bg-secondary hover:text-ink"
          href="#settings-rules">分類規則</a
        >
      </nav>

      <div class="grid min-w-0 gap-5">
        <div id="settings-schedule">
          <DefaultSchedulePanel {api} {demoMode} jobs={$jobs.data ?? []} />
        </div>

        <section
          id="settings-sources"
          aria-labelledby="settings-sources-title"
          class="grid gap-3"
        >
          <div>
            <h2 id="settings-sources-title" class="c-section-title">
              資料來源
            </h2>
            <p class="mt-1 text-sm text-muted-foreground">
              比較同步狀態，並在固定區域管理憑證與排程。
            </p>
          </div>
          <div
            class="grid min-w-0 items-start gap-4 xl:grid-cols-[minmax(300px,0.85fr)_minmax(0,1.35fr)]"
          >
            <Surface class="overflow-hidden">
              {#each sources as source (source.id)}
                <SourceCard
                  {api}
                  {...source}
                  id={source.id}
                  jobs={$jobs.data ?? []}
                  selected={selectedConnector === source.id}
                  onConfigure={() => selectConnector(source.id)}
                />
              {/each}
            </Surface>
            <Surface class="min-w-0 p-5">
              {#key selectedConnector}<ConnectorPanel
                  {api}
                  connectorId={selectedConnector}
                  {demoMode}
                  title={selectedSource.title}
                  fields={connectorFields[selectedConnector]}
                  embedded
                />{/key}
            </Surface>
          </div>
        </section>

        <InlineAlert
          title="憑證安全"
          tone="success"
          body="連接器憑證只用於個人資料同步；機密欄位會加密保存，儲存後不會重新顯示。"
        />

        <div class="grid items-start gap-5 xl:grid-cols-2">
          <section id="settings-rates" class="min-w-0">
            <ExchangeRatesPanel {api} />
          </section>
          <section id="settings-rules" class="min-w-0">
            <ClassificationRulesPanel {api} />
          </section>
        </div>
      </div>
    </div>
  </div>
{/if}
