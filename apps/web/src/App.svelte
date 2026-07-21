<script lang="ts">
  import { onMount, type Component } from "svelte";
  import { fade, fly } from "svelte/transition";
  import {
    BarChart3,
    Wallet,
    History,
    FileText,
    Settings,
    Ellipsis,
    Eye,
    EyeOff,
    RefreshCw,
    CircleCheck,
    Menu,
    X,
    LockKeyhole,
    ChevronRight,
  } from "@lucide/svelte";
  import { QueryClient, QueryClientProvider } from "@tanstack/svelte-query";
  import { parseViewHash, viewHash } from "./app/navigation";
  import Button from "./components/ui/Button.svelte";
  import Icon from "./components/ui/Icon.svelte";
  import Overview from "./features/overview/Overview.svelte";
  import Assets from "./features/assets/Assets.svelte";
  import Activity from "./features/activity/Activity.svelte";
  import Invoices from "./features/invoices/Invoices.svelte";
  import Investments from "./features/assets/Investments.svelte";
  import Cards from "./features/assets/Cards.svelte";
  import Bank from "./features/assets/Bank.svelte";
  import ManualAssets from "./features/assets/ManualAssets.svelte";
  import SettingsView from "./features/settings/Settings.svelte";
  import { createApiClient } from "./lib/api";
  import { moneyState } from "./lib/format.svelte";
  import { swipeBack } from "./lib/swipe-back";
  import type {
    DetailView,
    MobileSettingsView,
    PrimaryView,
    RuntimeInfo,
    View,
  } from "./lib/types";
  import "./styles.css";

  const api = createApiClient();
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false, staleTime: 30_000 } },
  });
  let view = $state<View>("overview");
  let runtime = $state<RuntimeInfo>({ demoMode: false });
  let mobileNavOpen = $state(false);

  const navItems: {
    view: PrimaryView;
    label: string;
    shortLabel: string;
    description: string;
    icon: Component;
  }[] = [
    {
      view: "overview",
      label: "總覽",
      shortLabel: "總覽",
      description: "淨資產、同步健康度與近期財務活動。",
      icon: BarChart3,
    },
    {
      view: "assets",
      label: "資產",
      shortLabel: "資產",
      description: "銀行、信用卡、投資與其他資產集中管理。",
      icon: Wallet,
    },
    {
      view: "activity",
      label: "活動",
      shortLabel: "活動",
      description: "銀行、刷卡、投資與發票的統一時間軸。",
      icon: History,
    },
    {
      view: "invoices",
      label: "發票",
      shortLabel: "發票",
      description: "搜尋電子發票、商家與品項明細。",
      icon: FileText,
    },
    {
      view: "settings",
      label: "設定",
      shortLabel: "設定",
      description: "管理資料來源、同步排程、匯率與交易分類。",
      icon: Settings,
    },
  ];
  const mobilePrimaryViews: PrimaryView[] = ["overview", "assets", "activity"];
  const detailLabels: Record<
    DetailView,
    { label: string; description: string }
  > = {
    bank: { label: "銀行帳戶", description: "帳戶餘額、現金流與交易分類。" },
    cards: { label: "信用卡", description: "信用卡帳戶、帳單與刷卡紀錄。" },
    investments: { label: "投資", description: "投資持倉與交易紀錄。" },
    "manual-assets": {
      label: "其他資產",
      description: "保險、不動產、交通工具與估值紀錄。",
    },
  };
  const mobileSettingsLabels: Record<
    MobileSettingsView,
    { label: string; description: string }
  > = {
    "data-sources": {
      label: "資料來源與連接器",
      description: "管理來源狀態、憑證、自動同步與重新驗證。",
    },
    "exchange-rates": {
      label: "匯率",
      description: "管理資產換算使用的參考匯率。",
    },
    "classification-rules": {
      label: "分類規則",
      description: "讓銀行交易依條件自動分類。",
    },
  };
  const isDetail = (value: View): value is DetailView =>
    Object.hasOwn(detailLabels, value);
  const isMobileSetting = (value: View): value is MobileSettingsView =>
    Object.hasOwn(mobileSettingsLabels, value);
  const primaryView = $derived(
    view === "more" || isMobileSetting(view)
      ? "settings"
      : isDetail(view)
        ? "assets"
        : (view as PrimaryView),
  );
  const currentView = $derived(
    navItems.find((item) => item.view === primaryView) ?? navItems[0]!,
  );
  const detail = $derived(isDetail(view) ? detailLabels[view] : undefined);
  const mobileSetting = $derived(
    isMobileSetting(view) ? mobileSettingsLabels[view] : undefined,
  );
  const pageTitle = $derived(
    detail?.label ??
      mobileSetting?.label ??
      (view === "more" ? "更多" : currentView.label),
  );
  const pageDescription = $derived(
    detail?.description ??
      mobileSetting?.description ??
      currentView.description,
  );

  const isStandalone = () =>
    document.documentElement.classList.contains("is-standalone");

  function scrollToTop() {
    const options: ScrollToOptions = { top: 0, behavior: "smooth" };
    requestAnimationFrame(() => {
      for (const id of ["app-scroll-area", "global-scroll-area"]) {
        const scrollArea = document.getElementById(id);
        if (scrollArea) {
          scrollArea.scrollTo(options);
          return;
        }
      }
      if (isStandalone()) document.getElementById("root")?.scrollTo(options);
      else window.scrollTo(options);
    });
  }

  onMount(() => {
    const routeView = parseViewHash(window.location.hash);
    if (routeView) view = routeView;
    else
      window.history.replaceState(
        null,
        "",
        `${window.location.pathname}${window.location.search}${viewHash(view)}`,
      );

    const handleHashChange = () => {
      const next = parseViewHash(window.location.hash);
      if (next) {
        view = next;
        mobileNavOpen = false;
        scrollToTop();
      }
    };
    window.addEventListener("hashchange", handleHashChange);

    void api
      .get<RuntimeInfo>("/api/runtime")
      .then((info) => (runtime = info))
      .catch(() => (runtime = { demoMode: false }));
    moneyState.hidden =
      localStorage.getItem("taiwan-fin-hub-money-hidden") === "true";

    return () => window.removeEventListener("hashchange", handleHashChange);
  });

  function navigate(next: View) {
    view = next;
    mobileNavOpen = false;
    const nextHash = viewHash(next);
    if (window.location.hash !== nextHash) {
      if (isStandalone()) {
        window.history.replaceState(
          null,
          "",
          `${window.location.pathname}${window.location.search}${nextHash}`,
        );
      } else {
        window.location.hash = nextHash;
      }
    }
    scrollToTop();
  }

  function navigateBack() {
    if (isDetail(view)) navigate("assets");
    else if (isMobileSetting(view)) navigate("more");
  }

  function toggleMoneyVisibility() {
    moneyState.hidden = !moneyState.hidden;
    localStorage.setItem(
      "taiwan-fin-hub-money-hidden",
      String(moneyState.hidden),
    );
  }

  function handleKeydown(event: KeyboardEvent) {
    if (event.key === "Escape") mobileNavOpen = false;
  }
</script>

<svelte:window onkeydown={handleKeydown} />

<QueryClientProvider client={queryClient}>
  <div
    class="c-app-shell"
    use:swipeBack={{
      enabled: isStandalone() && (isDetail(view) || isMobileSetting(view)),
      onBack: navigateBack,
    }}
  >
    <aside class="c-desktop-sidebar">
      <div class="c-sidebar-glow" aria-hidden="true"></div>
      <div class="relative z-10 flex h-full min-h-0 flex-col">
        <div class="border-b border-white/8 px-3 pb-5 pt-1">
          <div class="flex items-center gap-3">
            <span class="c-brand-mark"><BarChart3 class="size-5" /></span>
            <div>
              <h1 class="text-[17px] font-semibold tracking-[-0.025em]">
                Taiwan Fin Hub
              </h1>
              <p class="mt-1 text-[11px] font-medium text-white/45">
                Personal finance console
              </p>
            </div>
          </div>
          <p class="mt-4 flex items-center gap-2 text-xs text-white/60">
            <span class="c-live-dot" aria-hidden="true"></span>
            資料服務正常
          </p>
        </div>

        <nav class="mt-4 grid gap-1.5" aria-label="主要導覽">
          {#each navItems as item (item.view)}
            {@const NavIcon = item.icon}
            <button
              class={`c-primary-nav-item ${primaryView === item.view ? "is-active" : ""}`}
              aria-current={primaryView === item.view ? "page" : undefined}
              onclick={() => navigate(item.view)}
            >
              <span class="c-primary-nav-icon">
                <NavIcon class="size-[21px] stroke-[1.8]" />
              </span>
              <span class="min-w-0 flex-1">
                <span class="block truncate">{item.label}</span>
                <span class="mt-0.5 block truncate text-[11px] font-normal text-white/38">
                  {item.description}
                </span>
              </span>
              <ChevronRight
                class={`size-4 shrink-0 transition ${primaryView === item.view ? "translate-x-0 opacity-100" : "-translate-x-1 opacity-0"}`}
              />
            </button>
          {/each}
        </nav>

        <div class="mt-auto grid gap-3">
          <div class="c-sidebar-context">
            <p class="text-[11px] font-semibold uppercase tracking-[0.14em] text-white/38">
              Current workspace
            </p>
            <p class="mt-2 text-sm font-semibold text-white">{pageTitle}</p>
            <p class="mt-1 text-xs leading-relaxed text-white/52">
              {pageDescription}
            </p>
          </div>
          <div class="flex items-start gap-2.5 px-2 text-xs leading-relaxed text-white/48">
            <LockKeyhole class="mt-0.5 size-4 shrink-0 text-cyan-200/70" />
            <span>金融資料保留於你的私人環境。</span>
          </div>
        </div>
      </div>
    </aside>

    <div class="c-app-frame">
      <header class="c-app-header">
        <div class="flex min-w-0 flex-1 items-center gap-3">
          <button
            type="button"
            class="c-header-menu-button"
            aria-label="開啟主要選單"
            aria-expanded={mobileNavOpen}
            onclick={() => (mobileNavOpen = true)}
          >
            <Menu class="size-5" />
          </button>

          <div class="min-w-0">
            {#if detail}
              <button
                class="mb-1 inline-flex items-center gap-1 text-xs font-semibold text-steel transition hover:-translate-x-0.5"
                onclick={() => navigate("assets")}
              >
                ← 返回資產
              </button>
            {:else if mobileSetting}
              <button
                class="mb-1 inline-flex items-center gap-1 text-xs font-semibold text-steel transition hover:-translate-x-0.5 md:hidden"
                onclick={() => navigate("more")}
              >
                ← 返回更多
              </button>
            {/if}
            <div class="flex items-center gap-3">
              <h1 class="truncate text-xl font-semibold tracking-[-0.025em] md:text-2xl xl:text-[28px]">
                {pageTitle}
              </h1>
              {#if runtime.demoMode}
                <span class="hidden rounded-full border border-steel/20 bg-steel/8 px-2.5 py-1 text-[11px] font-semibold text-steel sm:inline-flex">
                  Demo
                </span>
              {/if}
            </div>
            <p class="mt-1 hidden truncate text-sm text-muted-foreground md:block">
              {pageDescription}
            </p>
          </div>
        </div>

        <div class="flex shrink-0 items-center gap-2">
          <div class="c-header-status hidden lg:flex">
            <span class="c-live-dot" aria-hidden="true"></span>
            <span>
              <span class="block text-xs font-semibold text-ink/70">資料已載入</span>
              <span class="block text-[11px] text-muted-foreground">依來源顯示最新狀態</span>
            </span>
          </div>
          <Button
            class={mobileSetting
              ? "hidden"
              : "rounded-full border border-border bg-white shadow-sm"}
            aria-label={moneyState.hidden ? "顯示金額" : "隱藏金額"}
            onclick={toggleMoneyVisibility}
            size="icon"
            variant="secondary"
          >
            <Icon icon={moneyState.hidden ? Eye : EyeOff} size="lg" />
          </Button>
          {#if primaryView === "settings"}
            <Button
              class="hidden md:inline-flex"
              onclick={() => queryClient.invalidateQueries()}
              variant="secondary"
            >
              <Icon icon={RefreshCw} size="sm" />重新整理
            </Button>
          {/if}
        </div>
      </header>

      <main class="min-h-0 flex-1 overflow-hidden">
        {#if primaryView === "settings"}
          {#key view}
            <div class="h-full min-h-0" in:fade={{ duration: 180 }}>
              <SettingsView
                {api}
                demoMode={runtime.demoMode}
                mobileView={view === "more"
                  ? "more"
                  : isMobileSetting(view)
                    ? view
                    : undefined}
                {navigate}
              />
            </div>
          {/key}
        {:else}
          <div id="global-scroll-area" class="c-global-scroll-area">
            <div class="c-global-content">
              {#key view}
                <div
                  class="c-view-stage"
                  in:fly={{ y: 14, duration: 280 }}
                  out:fade={{ duration: 110 }}
                >
                  {#if view === "overview"}
                    <Overview {api} {navigate} />
                  {:else if view === "assets"}
                    <Assets {api} {navigate} />
                  {:else if view === "activity"}
                    <Activity {api} {navigate} />
                  {:else if view === "invoices"}
                    <Invoices {api} />
                  {:else if view === "investments"}
                    <Investments {api} />
                  {:else if view === "cards"}
                    <Cards {api} />
                  {:else if view === "bank"}
                    <Bank {api} {navigate} />
                  {:else if view === "manual-assets"}
                    <ManualAssets {api} />
                  {/if}
                </div>
              {/key}

              <footer class="c-app-footer">
                <p class="text-xs leading-relaxed text-ink/35">
                  <strong class="font-medium text-ink/60">免責聲明：</strong
                  >本程式僅供個人研究與自用，未與臺灣集中保管結算所、財政部、金融監督管理委員會、各銀行或任何金融機構合作，亦未獲前述機構授權或背書。本程式所呈現的資料以您自行提供之憑證取得，作者不保證資料的即時性、正確性與完整性，亦不對因使用本程式所產生的任何直接或間接損失負責。請勿將本程式用於任何商業用途。
                </p>
              </footer>
            </div>
          </div>
        {/if}
      </main>

      {#if !mobileSetting}
        <nav aria-label="主要導覽" class="c-mobile-bottom-nav">
          {#each mobilePrimaryViews as mobileView (mobileView)}
            {@const item = navItems.find(
              (candidate) => candidate.view === mobileView,
            )!}
            {@const NavIcon = item.icon}
            <button
              class={`c-mobile-bottom-item ${primaryView === item.view ? "is-active" : ""}`}
              onclick={() => navigate(item.view)}
            >
              <span class="c-mobile-bottom-icon">
                <NavIcon class="size-5" />
              </span>
              <span>{item.shortLabel}</span>
            </button>
          {/each}
          <button
            class={`c-mobile-bottom-item ${view === "more" || !mobilePrimaryViews.includes(primaryView) ? "is-active" : ""}`}
            onclick={() => navigate("more")}
          >
            <span class="c-mobile-bottom-icon"><Ellipsis class="size-5" /></span>
            <span>更多</span>
          </button>
        </nav>
      {/if}
    </div>

    {#if mobileNavOpen}
      <button
        type="button"
        class="fixed inset-0 z-[90] bg-ink/45 backdrop-blur-[3px] xl:hidden"
        aria-label="關閉主要選單"
        onclick={() => (mobileNavOpen = false)}
        transition:fade={{ duration: 170 }}
      ></button>
      <aside
        class="c-mobile-drawer"
        role="dialog"
        aria-modal="true"
        aria-label="主要選單"
        transition:fly={{ x: -28, duration: 240 }}
      >
        <div class="flex items-center justify-between border-b border-border px-5 pb-4 pt-[max(env(safe-area-inset-top),1.25rem)]">
          <div class="flex items-center gap-3">
            <span class="c-brand-mark c-brand-mark--light">
              <BarChart3 class="size-5" />
            </span>
            <div>
              <p class="font-semibold tracking-tight">Taiwan Fin Hub</p>
              <p class="mt-0.5 text-xs text-muted-foreground">Personal finance console</p>
            </div>
          </div>
          <button
            type="button"
            class="grid size-11 place-items-center rounded-xl text-muted-foreground transition hover:bg-secondary hover:text-ink active:scale-95"
            aria-label="關閉主要選單"
            onclick={() => (mobileNavOpen = false)}
          >
            <X class="size-5" />
          </button>
        </div>

        <nav class="grid gap-2 overflow-y-auto p-4" aria-label="行動版主要導覽">
          {#each navItems as item (item.view)}
            {@const NavIcon = item.icon}
            <button
              class={`c-drawer-nav-item ${primaryView === item.view ? "is-active" : ""}`}
              aria-current={primaryView === item.view ? "page" : undefined}
              onclick={() => navigate(item.view)}
            >
              <span class="c-drawer-nav-icon"><NavIcon class="size-5" /></span>
              <span class="min-w-0 flex-1 text-left">
                <span class="block font-semibold">{item.label}</span>
                <span class="mt-0.5 block truncate text-xs text-muted-foreground">
                  {item.description}
                </span>
              </span>
              <ChevronRight class="size-4 shrink-0 text-muted-foreground" />
            </button>
          {/each}
        </nav>

        <div class="mt-auto border-t border-border p-5">
          <div class="rounded-xl bg-secondary/65 p-4">
            <p class="text-xs font-semibold text-ink/65">目前位置</p>
            <p class="mt-1 font-semibold">{pageTitle}</p>
            <p class="mt-1 text-xs leading-relaxed text-muted-foreground">
              {pageDescription}
            </p>
          </div>
        </div>
      </aside>
    {/if}
  </div>
</QueryClientProvider>
