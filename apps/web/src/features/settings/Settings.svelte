<script lang="ts">
  import { createQuery } from "@tanstack/svelte-query";
  import {
    AlertTriangle,
    ChevronRight,
    CircleCheck,
    Coins,
    Database,
    LayoutDashboard,
    ListFilter,
    Menu,
    RefreshCw,
    X,
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
    {
    id: "taishin",
    title: "台新銀行",
    description: "信用卡額度、帳單與即時消費",
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
    taishin: [
    { key: "userId", label: "身分證字號／統編", type: "text" },
    { key: "account", label: "使用者代號", type: "text" },
    { key: "password", label: "使用者密碼", type: "password" },
    {
      key: "lookbackMonths",
      label: "帳單往回月份（最多 6 期）",
      type: "number",
      placeholder: "6",
    },
  ],
  };

  const jobs = createQuery(syncJobsQuery(() => api));
  const rules = createQuery(classificationRulesQuery(() => api));
  const bank = createQuery(bankQuery(() => api));
  let selectedConnector = $state<ConnectorId>("einvoice");
  let activeSection = $state("settings-schedule");
  let mobileMenuOpen = $state(false);

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
  const sectionNavItems = $derived([
    {
      sectionId: "settings-schedule",
      label: "同步排程",
      icon: RefreshCw,
    },
    {
      sectionId: "settings-sources",
      label: "資料來源",
      detail: String(sources.length),
      icon: Database,
    },
    { sectionId: "settings-rates", label: "匯率", icon: Coins },
    {
      sectionId: "settings-rules",
      label: "分類規則",
      detail: String($rules.data?.length ?? 0),
      icon: ListFilter,
    },
  ]);
  const mobileRouteItems = $derived([
    {
      view: "more" as View,
      label: "設定首頁",
      description: "狀態摘要與設定入口",
      icon: LayoutDashboard,
    },
    {
      view: "data-sources" as View,
      label: "同步與資料來源",
      description: `${sources.length} 個連接器`,
      icon: Database,
    },
    {
      view: "exchange-rates" as View,
      label: "匯率",
      description: "外幣換算設定",
      icon: Coins,
    },
    {
      view: "classification-rules" as View,
      label: "分類規則",
      description: `${$rules.data?.length ?? 0} 條規則`,
      icon: ListFilter,
    },
  ]);
  const currentMobileLabel = $derived(
    mobileView === "data-sources"
      ? "同步與資料來源"
      : mobileView === "exchange-rates"
        ? "匯率"
        : mobileView === "classification-rules"
          ? "分類規則"
          : "設定選單",
  );
  const currentSectionLabel = $derived(
    sectionNavItems.find((item) => item.sectionId === activeSection)?.label ??
      "設定選單",
  );

  function scrollToSection(sectionId: string) {
    const scrollArea = document.getElementById("app-scroll-area");
    const section = document.getElementById(sectionId);
    if (!scrollArea || !section) return;

    const scrollAreaTop = scrollArea.getBoundingClientRect().top;
    const sectionTop = section.getBoundingClientRect().top;
    scrollArea.scrollTo({
      top: scrollArea.scrollTop + sectionTop - scrollAreaTop - 24,
      behavior: "smooth",
    });
    activeSection = sectionId;
    mobileMenuOpen = false;
  }

  function handleSettingsScroll(event: Event) {
    if (mobileView) return;
    const scrollArea = event.currentTarget as HTMLElement;
    const threshold = scrollArea.getBoundingClientRect().top + 120;
    let nextSection = sectionNavItems[0]?.sectionId ?? "settings-schedule";

    for (const item of sectionNavItems) {
      const section = document.getElementById(item.sectionId);
      if (section && section.getBoundingClientRect().top <= threshold) {
        nextSection = item.sectionId;
      }
    }
    activeSection = nextSection;
  }

  function navigateFromMenu(next: View) {
    mobileMenuOpen = false;
    navigate(next);
  }

  function handleKeydown(event: KeyboardEvent) {
    if (event.key === "Escape") mobileMenuOpen = false;
  }
</script>

<svelte:window onkeydown={handleKeydown} />

<div class="flex h-full min-h-0 flex-col bg-paper">
  {#if mobileView === "more"}
    <div
      id="app-scroll-area"
      class="min-h-0 flex-1 overflow-y-auto overscroll-contain px-4 pb-[calc(env(safe-area-inset-bottom)+6rem)] pt-1 sm:px-6 md:py-5"
    >
      <MobileMore
        {demoMode}
        jobs={$jobs.data ?? []}
        rules={$rules.data ?? []}
        bank={$bank.data ?? { accounts: [], transactions: [] }}
        {navigate}
        {api}
      />
    </div>
  {:else if mobileView}
    <div
      class="flex shrink-0 items-center gap-3 border-b border-border bg-card px-4 py-2.5 lg:hidden"
    >
      <button
        type="button"
        class="grid size-10 shrink-0 place-items-center rounded-lg border border-border bg-white text-ink transition hover:bg-secondary"
        aria-label="開啟設定選單"
        aria-expanded={mobileMenuOpen}
        onclick={() => (mobileMenuOpen = true)}
      >
        <Menu class="size-5" />
      </button>
      <div class="min-w-0">
        <p class="truncate text-sm font-semibold">{currentMobileLabel}</p>
        <p class="truncate text-xs text-muted-foreground">
          切換設定區段
        </p>
      </div>
    </div>

    <div
      id="app-scroll-area"
      class="min-h-0 flex-1 overflow-y-auto overscroll-contain px-4 pb-[calc(env(safe-area-inset-bottom)+2rem)] pt-4 sm:px-6 md:py-5"
    >
      {#if mobileView === "data-sources"}
        <div class="grid min-w-0 gap-5">
          <DefaultSchedulePanel {api} {demoMode} jobs={$jobs.data ?? []} />
          <section aria-labelledby="mobile-settings-sources-title">
            <div class="mb-3">
              <h2
                id="mobile-settings-sources-title"
                class="text-lg font-semibold tracking-tight"
              >
                資料來源
              </h2>
              <p class="mt-1 text-sm text-muted-foreground">
                選擇來源後管理連線、同步與排程。
              </p>
            </div>
            <div
              class="grid min-w-0 items-start gap-4 xl:grid-cols-[minmax(300px,0.82fr)_minmax(0,1.35fr)]"
            >
              <Surface class="overflow-hidden">
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
        </div>
      {:else if mobileView === "exchange-rates"}
        <ExchangeRatesPanel {api} />
      {:else if mobileView === "classification-rules"}
        <ClassificationRulesPanel {api} />
      {/if}
    </div>
  {:else}
    <div
      class="flex shrink-0 items-center gap-3 border-b border-border bg-card px-4 py-2.5 lg:hidden"
    >
      <button
        type="button"
        class="grid size-10 shrink-0 place-items-center rounded-lg border border-border bg-white text-ink transition hover:bg-secondary"
        aria-label="開啟設定選單"
        aria-expanded={mobileMenuOpen}
        onclick={() => (mobileMenuOpen = true)}
      >
        <Menu class="size-5" />
      </button>
      <div class="min-w-0">
        <p class="truncate text-sm font-semibold">{currentSectionLabel}</p>
        <p class="truncate text-xs text-muted-foreground">
          設定頁內容獨立捲動
        </p>
      </div>
    </div>

    <div class="flex min-h-0 flex-1">
      <aside
        class="hidden w-[224px] shrink-0 flex-col border-r border-border bg-card lg:flex"
        aria-label="設定區段"
      >
        <div class="border-b border-border px-5 py-4">
          <p class="text-sm font-semibold">設定選單</p>
          <p class="mt-1 text-xs leading-relaxed text-muted-foreground">
            外框保持固定，只有右側內容捲動。
          </p>
        </div>
        <nav class="grid gap-1 p-3">
          {#each sectionNavItems as item (item.sectionId)}
            {@const NavIcon = item.icon}
            <button
              type="button"
              class={`flex min-h-11 items-center gap-3 rounded-lg px-3 text-left text-sm transition ${activeSection === item.sectionId ? "bg-accent font-semibold text-accent-foreground" : "font-medium text-muted-foreground hover:bg-secondary hover:text-ink"}`}
              aria-current={activeSection === item.sectionId ? "page" : undefined}
              onclick={() => scrollToSection(item.sectionId)}
            >
              <NavIcon class="size-[18px] shrink-0" />
              <span class="min-w-0 flex-1 truncate">{item.label}</span>
              {#if item.detail}
                <span class="text-xs tabular-nums text-muted-foreground">
                  {item.detail}
                </span>
              {/if}
            </button>
          {/each}
        </nav>
        <div class="mt-auto border-t border-border p-4">
          <p class="text-xs font-semibold text-ink/65">安全與隱私</p>
          <p class="mt-1 text-xs leading-relaxed text-muted-foreground">
            連接器憑證會加密保存，儲存後不重新顯示。
          </p>
        </div>
      </aside>

      <div class="flex min-w-0 flex-1 flex-col">
        <section
          class={`flex shrink-0 items-center justify-between gap-4 border-b px-5 py-3 ${needsAction ? "border-coral/20 bg-coral/[0.045]" : "border-moss/15 bg-moss/[0.045]"}`}
          aria-live="polite"
        >
          <div class="flex min-w-0 items-center gap-3">
            <span
              class={`grid size-8 shrink-0 place-items-center rounded-full ${needsAction ? "bg-coral/10 text-coral" : "bg-moss/10 text-moss"}`}
            >
              {#if needsAction}
                <AlertTriangle class="size-4" />
              {:else}
                <CircleCheck class="size-4" />
              {/if}
            </span>
            <div class="min-w-0">
              <p class="truncate text-sm font-semibold">
                {needsAction
                  ? `${needsAction} 個資料來源需要處理`
                  : "所有資料來源運作正常"}
              </p>
              <p class="truncate text-xs text-muted-foreground">
                {sources.length} 個來源 · {enabledJobs} 個已啟用 · {$rules
                  .data?.length ?? 0} 條分類規則
              </p>
            </div>
          </div>
          {#if needsAction}
            <button
              type="button"
              class="shrink-0 text-sm font-semibold text-coral hover:underline"
              onclick={() => scrollToSection("settings-sources")}
            >
              立即查看
            </button>
          {/if}
        </section>

        <div
          id="app-scroll-area"
          class="min-h-0 flex-1 scroll-smooth overflow-y-auto overscroll-contain"
          onscroll={handleSettingsScroll}
        >
          <div class="grid min-w-0 gap-8 p-4 pb-10 sm:p-6 xl:p-8">
            <section id="settings-schedule" class="scroll-mt-6">
              <DefaultSchedulePanel
                {api}
                {demoMode}
                jobs={$jobs.data ?? []}
              />
            </section>

            <section
              id="settings-sources"
              aria-labelledby="settings-sources-title"
              class="scroll-mt-6"
            >
              <div class="mb-3">
                <h2
                  id="settings-sources-title"
                  class="text-lg font-semibold tracking-tight"
                >
                  資料來源
                </h2>
                <p class="mt-1 text-sm text-muted-foreground">
                  選擇來源後，可在右側管理連線、同步與排程。
                </p>
              </div>
              <div
                class="grid min-w-0 items-start overflow-hidden rounded-xl border border-border bg-card shadow-sm xl:grid-cols-[minmax(320px,0.78fr)_minmax(0,1.45fr)]"
              >
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
                  {#key selectedConnector}<ConnectorPanel
                      {api}
                      connectorId={selectedConnector}
                      {demoMode}
                      title={selectedSource.title}
                      fields={connectorFields[selectedConnector]}
                      embedded
                    />{/key}
                </div>
              </div>
              <p class="mt-3 text-xs leading-relaxed text-muted-foreground">
                連接器憑證只用於個人資料同步；機密欄位會加密保存，儲存後不會重新顯示。
              </p>
            </section>

            <section id="settings-rates" class="scroll-mt-6">
              <ExchangeRatesPanel {api} />
            </section>

            <section id="settings-rules" class="scroll-mt-6">
              <ClassificationRulesPanel {api} />
            </section>

            <section class="border-t border-border pt-5">
              <p class="text-xs leading-relaxed text-ink/35">
                <strong class="font-medium text-ink/60">免責聲明：</strong
                >本程式僅供個人研究與自用，未與臺灣集中保管結算所、財政部、金融監督管理委員會、各銀行或任何金融機構合作，亦未獲前述機構授權或背書。本程式所呈現的資料以您自行提供之憑證取得，作者不保證資料的即時性、正確性與完整性，亦不對因使用本程式所產生的任何直接或間接損失負責。請勿將本程式用於任何商業用途。
              </p>
            </section>
          </div>
        </div>
      </div>
    </div>
  {/if}
</div>

{#if mobileMenuOpen}
  <button
    type="button"
    class="fixed inset-0 z-[70] bg-ink/45 backdrop-blur-[2px] lg:hidden"
    aria-label="關閉設定選單"
    onclick={() => (mobileMenuOpen = false)}
  ></button>
  <aside
    class="fixed inset-y-0 left-0 z-[80] flex w-[min(86vw,320px)] flex-col border-r border-border bg-card shadow-2xl lg:hidden"
    role="dialog"
    aria-modal="true"
    aria-label="設定選單"
  >
    <div
      class="flex items-center justify-between border-b border-border px-4 pb-3 pt-[max(env(safe-area-inset-top),1rem)]"
    >
      <div>
        <p class="font-semibold">設定選單</p>
        <p class="mt-0.5 text-xs text-muted-foreground">
          選擇要管理的項目
        </p>
      </div>
      <button
        type="button"
        class="grid size-10 place-items-center rounded-lg text-muted-foreground transition hover:bg-secondary hover:text-ink"
        aria-label="關閉設定選單"
        onclick={() => (mobileMenuOpen = false)}
      >
        <X class="size-5" />
      </button>
    </div>

    <nav class="min-h-0 flex-1 overflow-y-auto p-3">
      {#if mobileView}
        {#each mobileRouteItems as item (item.view)}
          {@const NavIcon = item.icon}
          <button
            type="button"
            class={`flex w-full items-center gap-3 rounded-xl px-3 py-3 text-left transition ${mobileView === item.view ? "bg-accent text-accent-foreground" : "text-ink hover:bg-secondary"}`}
            aria-current={mobileView === item.view ? "page" : undefined}
            onclick={() => navigateFromMenu(item.view)}
          >
            <span
              class={`grid size-10 shrink-0 place-items-center rounded-lg ${mobileView === item.view ? "bg-steel text-white" : "bg-steel/10 text-steel"}`}
            >
              <NavIcon class="size-5" />
            </span>
            <span class="min-w-0 flex-1">
              <span class="block truncate text-sm font-semibold">
                {item.label}
              </span>
              <span class="mt-0.5 block truncate text-xs text-muted-foreground">
                {item.description}
              </span>
            </span>
            <ChevronRight class="size-4 shrink-0 text-muted-foreground" />
          </button>
        {/each}
      {:else}
        {#each sectionNavItems as item (item.sectionId)}
          {@const NavIcon = item.icon}
          <button
            type="button"
            class={`flex w-full items-center gap-3 rounded-xl px-3 py-3 text-left transition ${activeSection === item.sectionId ? "bg-accent text-accent-foreground" : "text-ink hover:bg-secondary"}`}
            aria-current={activeSection === item.sectionId ? "page" : undefined}
            onclick={() => scrollToSection(item.sectionId)}
          >
            <span
              class={`grid size-10 shrink-0 place-items-center rounded-lg ${activeSection === item.sectionId ? "bg-steel text-white" : "bg-steel/10 text-steel"}`}
            >
              <NavIcon class="size-5" />
            </span>
            <span class="min-w-0 flex-1 truncate text-sm font-semibold">
              {item.label}
            </span>
            {#if item.detail}
              <span class="text-xs tabular-nums text-muted-foreground">
                {item.detail}
              </span>
            {/if}
            <ChevronRight class="size-4 shrink-0 text-muted-foreground" />
          </button>
        {/each}
      {/if}
    </nav>

    <div
      class="border-t border-border px-4 pb-[max(env(safe-area-inset-bottom),1rem)] pt-3"
    >
      <p class="text-xs leading-relaxed text-muted-foreground">
        Taiwan Fin Hub 會將金融資料保留在你的私人環境中。
      </p>
    </div>
  </aside>
{/if}
