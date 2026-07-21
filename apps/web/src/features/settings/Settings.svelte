<script lang="ts">
  import { createQuery } from "@tanstack/svelte-query";
  import { fade, fly } from "svelte/transition";
  import {
    AlertTriangle,
    ChevronRight,
    CircleCheck,
    Coins,
    Database,
    KeyRound,
    LayoutDashboard,
    ListFilter,
    RefreshCw,
    ShieldCheck,
    WalletCards,
  } from "@lucide/svelte";
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
  import Button from "../../components/ui/Button.svelte";
  import Metric from "../../components/ui/Metric.svelte";
  import Surface from "../../components/ui/Surface.svelte";
  import SourceCard from "./SourceCard.svelte";
  import ExchangeRatesPanel from "./ExchangeRatesPanel.svelte";
  import ClassificationRulesPanel from "./ClassificationRulesPanel.svelte";
  import ConnectorPanel from "./ConnectorPanel.svelte";
  import DefaultSchedulePanel from "./DefaultSchedulePanel.svelte";

  type SettingsPanel =
    | "overview"
    | "data-sources"
    | "exchange-rates"
    | "classification-rules";

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
      {
        key: "account",
        label: "行動／網路銀行使用者代碼",
        type: "text",
      },
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
  let desktopPanel = $state<SettingsPanel>("overview");

  const selectedSource = $derived(
    sources.find((source) => source.id === selectedConnector)!,
  );
  const enabledJobs = $derived(
    ($jobs.data ?? []).filter((job) => job.enabled).length,
  );
  const needsAction = $derived(
    ($jobs.data ?? []).filter(
      (job) =>
        job.lastStatus === "failed" || job.lastStatus === "needs_user_action",
    ).length,
  );
  const healthySources = $derived(Math.max(sources.length - needsAction, 0));
  const routePanel = $derived<SettingsPanel>(
    mobileView === "data-sources"
      ? "data-sources"
      : mobileView === "exchange-rates"
        ? "exchange-rates"
        : mobileView === "classification-rules"
          ? "classification-rules"
          : "overview",
  );
  const activePanel = $derived(mobileView ? routePanel : desktopPanel);

  const tabs: {
    id: SettingsPanel;
    label: string;
    shortLabel: string;
    description: string;
    icon: typeof LayoutDashboard;
  }[] = [
    {
      id: "overview",
      label: "設定總覽",
      shortLabel: "總覽",
      description: "健康狀態與快速入口",
      icon: LayoutDashboard,
    },
    {
      id: "data-sources",
      label: "同步與資料來源",
      shortLabel: "資料來源",
      description: "連接器、憑證與排程",
      icon: Database,
    },
    {
      id: "exchange-rates",
      label: "匯率",
      shortLabel: "匯率",
      description: "外幣資產換算",
      icon: Coins,
    },
    {
      id: "classification-rules",
      label: "分類規則",
      shortLabel: "分類",
      description: "交易自動歸類",
      icon: ListFilter,
    },
  ];

  function selectPanel(panel: SettingsPanel) {
    if (mobileView) {
      navigate(
        panel === "overview"
          ? "more"
          : panel === "data-sources"
            ? "data-sources"
            : panel === "exchange-rates"
              ? "exchange-rates"
              : "classification-rules",
      );
      return;
    }
    desktopPanel = panel;
    requestAnimationFrame(() => {
      document.getElementById("app-scroll-area")?.scrollTo({
        top: 0,
        behavior: "smooth",
      });
    });
  }

  function sourceStatus(connectorId: ConnectorId) {
    const job = ($jobs.data ?? []).find((item) => item.connectorId === connectorId);
    if (
      job?.lastStatus === "failed" ||
      job?.lastStatus === "needs_user_action"
    ) {
      return { label: "需要處理", className: "text-coral" };
    }
    if (job?.lastSuccessAt) {
      return { label: "正常", className: "text-moss" };
    }
    return { label: "尚未同步", className: "text-muted-foreground" };
  }
</script>

<div class="flex h-full min-h-0 flex-col bg-paper">
  <div class="shrink-0 border-b border-border bg-white/92 backdrop-blur-md">
    <div
      class="no-scrollbar flex min-h-[68px] items-stretch gap-1 overflow-x-auto px-3 sm:px-5"
      role="tablist"
      aria-label="設定分類"
    >
      {#each tabs as tab (tab.id)}
        {@const TabIcon = tab.icon}
        <button
          id={`settings-tab-${tab.id}`}
          type="button"
          role="tab"
          aria-selected={activePanel === tab.id}
          aria-controls={`settings-panel-${tab.id}`}
          class={`group relative flex min-w-max items-center gap-2.5 border-b-2 px-3 py-3 text-left transition-all duration-200 sm:min-w-[170px] sm:px-4 ${
            activePanel === tab.id
              ? "border-steel text-ink"
              : "border-transparent text-muted-foreground hover:border-steel/25 hover:bg-secondary/45 hover:text-ink"
          }`}
          onclick={() => selectPanel(tab.id)}
        >
          <span
            class={`grid size-9 shrink-0 place-items-center rounded-lg transition-all duration-200 ${
              activePanel === tab.id
                ? "bg-steel/10 text-steel"
                : "bg-secondary text-muted-foreground group-hover:text-steel"
            }`}
          >
            <TabIcon class="size-[18px]" />
          </span>
          <span class="min-w-0">
            <span class="block text-sm font-semibold sm:hidden">
              {tab.shortLabel}
            </span>
            <span class="hidden text-sm font-semibold sm:block">{tab.label}</span>
            <span class="mt-0.5 hidden truncate text-[11px] font-normal text-muted-foreground sm:block">
              {tab.description}
            </span>
          </span>
        </button>
      {/each}
    </div>
  </div>

  <div
    id="app-scroll-area"
    class="min-h-0 flex-1 overflow-y-auto overscroll-contain scroll-smooth"
  >
    <div class="grid min-w-0 gap-6 p-4 pb-[calc(env(safe-area-inset-bottom)+6rem)] sm:p-6 md:pb-10 xl:p-8">
      {#key activePanel}
        <section
          id={`settings-panel-${activePanel}`}
          role="tabpanel"
          aria-labelledby={`settings-tab-${activePanel}`}
          class="min-w-0"
          in:fly={{ y: 8, duration: 220 }}
          out:fade={{ duration: 100 }}
        >
          {#if activePanel === "overview"}
            <div class="grid min-w-0 gap-6">
              <Surface
                tone="strong"
                class="overflow-hidden p-5 md:p-6"
              >
                <div class="grid gap-5 lg:grid-cols-[minmax(0,1fr)_auto] lg:items-end">
                  <div class="flex min-w-0 items-start gap-4">
                    <span
                      class={`grid size-11 shrink-0 place-items-center rounded-xl ${
                        needsAction
                          ? "bg-coral/15 text-red-200"
                          : "bg-white/10 text-emerald-200"
                      }`}
                    >
                      {#if needsAction}
                        <AlertTriangle class="size-5" />
                      {:else}
                        <CircleCheck class="size-5" />
                      {/if}
                    </span>
                    <div class="min-w-0">
                      <p class="text-xs font-semibold text-white/55">
                        設定與資料健康度
                      </p>
                      <h2 class="mt-1 text-2xl font-semibold tracking-tight md:text-3xl">
                        {needsAction
                          ? `${needsAction} 個來源需要處理`
                          : "所有資料來源運作正常"}
                      </h2>
                      <p class="mt-2 max-w-2xl text-sm leading-relaxed text-white/62">
                        {needsAction
                          ? "部分同步可能不是最新狀態，建議先檢查連線或重新驗證憑證。"
                          : "連接器、排程與分類規則目前沒有需要立即處理的異常。"}
                      </p>
                    </div>
                  </div>
                  <Button
                    variant={needsAction ? "secondary" : "outline"}
                    class="border-white/15 bg-white/8 text-white hover:bg-white/14 hover:text-white"
                    onclick={() => selectPanel("data-sources")}
                  >
                    管理資料來源 <ChevronRight class="size-4" />
                  </Button>
                </div>
              </Surface>

              <Surface class="overflow-hidden px-5 py-4 md:px-6">
                <div class="grid grid-cols-2 gap-x-5 gap-y-4 lg:grid-cols-4 lg:divide-x lg:divide-border">
                  <Metric
                    label="正常來源"
                    value={`${healthySources} / ${sources.length}`}
                    detail="連接器健康度"
                    tone={needsAction ? "warning" : "positive"}
                  />
                  <Metric
                    class="lg:pl-5"
                    label="啟用排程"
                    value={`${enabledJobs}`}
                    detail="自動同步工作"
                    tone="brand"
                  />
                  <Metric
                    class="lg:pl-5"
                    label="金融帳戶"
                    value={`${$bank.data?.accounts.length ?? 0}`}
                    detail="銀行與信用卡帳戶"
                  />
                  <Metric
                    class="lg:pl-5"
                    label="分類規則"
                    value={`${$rules.data?.length ?? 0}`}
                    detail="自動分類條件"
                  />
                </div>
              </Surface>

              <div class="grid min-w-0 gap-5 xl:grid-cols-[minmax(0,1.35fr)_minmax(300px,0.65fr)]">
                <Surface class="overflow-hidden">
                  <div class="flex items-center justify-between border-b border-border px-5 py-4">
                    <div>
                      <h2 class="c-section-title">資料來源狀態</h2>
                      <p class="mt-1 text-xs text-muted-foreground">
                        最近一次同步與目前健康狀態
                      </p>
                    </div>
                    <Button
                      size="sm"
                      variant="ghost"
                      onclick={() => selectPanel("data-sources")}
                    >
                      查看全部
                    </Button>
                  </div>
                  <div class="divide-y divide-border">
                    {#each sources as source (source.id)}
                      {@const status = sourceStatus(source.id)}
                      <button
                        type="button"
                        class="c-data-row flex min-h-16 w-full items-center gap-3 px-5 py-3 text-left"
                        onclick={() => {
                          selectedConnector = source.id;
                          selectPanel("data-sources");
                        }}
                      >
                        <span class="grid size-10 shrink-0 place-items-center rounded-xl bg-steel/8 text-steel">
                          <Database class="size-[18px]" />
                        </span>
                        <span class="min-w-0 flex-1">
                          <span class="block truncate text-sm font-semibold">
                            {source.title}
                          </span>
                          <span class="mt-0.5 block truncate text-xs text-muted-foreground">
                            {source.description}
                          </span>
                        </span>
                        <span class={`shrink-0 text-xs font-semibold ${status.className}`}>
                          {status.label}
                        </span>
                        <ChevronRight class="size-4 shrink-0 text-muted-foreground" />
                      </button>
                    {/each}
                  </div>
                </Surface>

                <div class="grid content-start gap-5">
                  <Surface class="p-5">
                    <div class="flex items-start gap-3">
                      <span class="grid size-10 shrink-0 place-items-center rounded-xl bg-moss/10 text-moss">
                        <ShieldCheck class="size-5" />
                      </span>
                      <div>
                        <h2 class="c-section-title">安全與隱私</h2>
                        <p class="mt-2 text-sm leading-relaxed text-muted-foreground">
                          連接器憑證會加密保存，儲存後不會重新顯示。金融資料保留於你的私人環境。
                        </p>
                      </div>
                    </div>
                  </Surface>

                  <Surface class="overflow-hidden">
                    <button
                      type="button"
                      class="c-data-row flex min-h-16 w-full items-center gap-3 border-b border-border px-5 py-3 text-left"
                      onclick={() => selectPanel("exchange-rates")}
                    >
                      <span class="grid size-10 place-items-center rounded-xl bg-steel/8 text-steel">
                        <WalletCards class="size-5" />
                      </span>
                      <span class="min-w-0 flex-1">
                        <span class="block text-sm font-semibold">管理匯率</span>
                        <span class="mt-0.5 block text-xs text-muted-foreground">
                          外幣資產換算設定
                        </span>
                      </span>
                      <ChevronRight class="size-4 text-muted-foreground" />
                    </button>
                    <button
                      type="button"
                      class="c-data-row flex min-h-16 w-full items-center gap-3 px-5 py-3 text-left"
                      onclick={() => selectPanel("classification-rules")}
                    >
                      <span class="grid size-10 place-items-center rounded-xl bg-steel/8 text-steel">
                        <ListFilter class="size-5" />
                      </span>
                      <span class="min-w-0 flex-1">
                        <span class="block text-sm font-semibold">管理分類規則</span>
                        <span class="mt-0.5 block text-xs text-muted-foreground">
                          {$rules.data?.length ?? 0} 條自動分類條件
                        </span>
                      </span>
                      <ChevronRight class="size-4 text-muted-foreground" />
                    </button>
                  </Surface>

                  {#if demoMode}
                    <Surface tone="subtle" class="p-4">
                      <p class="text-xs font-semibold text-steel">Demo 模式</p>
                      <p class="mt-1 text-sm leading-relaxed text-muted-foreground">
                        目前顯示的是唯讀展示資料，部分設定操作不會寫入。
                      </p>
                    </Surface>
                  {/if}
                </div>
              </div>
            </div>
          {:else if activePanel === "data-sources"}
            <div class="grid min-w-0 gap-6">
              <div>
                <h2 class="text-xl font-semibold tracking-tight">同步與資料來源</h2>
                <p class="mt-1 text-sm text-muted-foreground">
                  管理預設排程、連接器狀態、憑證與重新驗證。
                </p>
              </div>

              <DefaultSchedulePanel
                {api}
                {demoMode}
                jobs={$jobs.data ?? []}
              />

              <section aria-labelledby="settings-sources-title">
                <div class="mb-3 flex flex-wrap items-end justify-between gap-3">
                  <div>
                    <h3 id="settings-sources-title" class="text-lg font-semibold tracking-tight">
                      資料來源
                    </h3>
                    <p class="mt-1 text-sm text-muted-foreground">
                      選擇來源後，在右側管理連線與同步設定。
                    </p>
                  </div>
                  <span
                    class={`rounded-full px-3 py-1 text-xs font-semibold ${
                      needsAction
                        ? "bg-coral/10 text-coral"
                        : "bg-moss/10 text-moss"
                    }`}
                  >
                    {needsAction ? `${needsAction} 個待處理` : "全部正常"}
                  </span>
                </div>

                <div class="grid min-w-0 items-start overflow-hidden rounded-xl border border-border bg-card shadow-sm xl:grid-cols-[minmax(300px,0.72fr)_minmax(0,1.5fr)]">
                  <div class="border-b border-border xl:border-b-0 xl:border-r">
                    {#each sources as source (source.id)}
                      <SourceCard
                        {api}
                        {...source}
                        id={source.id}
                        jobs={$jobs.data ?? []}
                        selected={selectedConnector === source.id}
                        onConfigure={() => (selectedConnector = source.id)}
                      />
                    {/each}
                  </div>
                  <div class="min-w-0 p-5 xl:p-6">
                    {#key selectedConnector}
                      <div in:fade={{ duration: 180 }}>
                        <ConnectorPanel
                          {api}
                          connectorId={selectedConnector}
                          {demoMode}
                          title={selectedSource.title}
                          fields={connectorFields[selectedConnector]}
                          embedded
                        />
                      </div>
                    {/key}
                  </div>
                </div>
                <div class="mt-3 flex items-start gap-2 text-xs leading-relaxed text-muted-foreground">
                  <KeyRound class="mt-0.5 size-4 shrink-0 text-steel" />
                  <p>
                    連接器憑證只用於個人資料同步；機密欄位會加密保存，儲存後不會重新顯示。
                  </p>
                </div>
              </section>
            </div>
          {:else if activePanel === "exchange-rates"}
            <div class="grid min-w-0 gap-5">
              <div>
                <h2 class="text-xl font-semibold tracking-tight">匯率設定</h2>
                <p class="mt-1 text-sm text-muted-foreground">
                  管理外幣資產換算使用的參考匯率。
                </p>
              </div>
              <ExchangeRatesPanel {api} />
            </div>
          {:else if activePanel === "classification-rules"}
            <div class="grid min-w-0 gap-5">
              <div>
                <h2 class="text-xl font-semibold tracking-tight">分類規則</h2>
                <p class="mt-1 text-sm text-muted-foreground">
                  讓符合條件的銀行交易自動歸入指定分類。
                </p>
              </div>
              <ClassificationRulesPanel {api} />
            </div>
          {/if}
        </section>
      {/key}

      <section class="border-t border-border pt-5">
        <p class="text-xs leading-relaxed text-ink/35">
          <strong class="font-medium text-ink/60">免責聲明：</strong
          >本程式僅供個人研究與自用，未與臺灣集中保管結算所、財政部、金融監督管理委員會、各銀行或任何金融機構合作，亦未獲前述機構授權或背書。本程式所呈現的資料以您自行提供之憑證取得，作者不保證資料的即時性、正確性與完整性，亦不對因使用本程式所產生的任何直接或間接損失負責。請勿將本程式用於任何商業用途。
        </p>
      </section>
    </div>
  </div>
</div>
