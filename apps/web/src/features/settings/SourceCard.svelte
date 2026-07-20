<script lang="ts">
  import {
    ChartCandlestick,
    ChevronRight,
    Landmark,
    ReceiptText,
  } from "@lucide/svelte";
  import { toStore } from "svelte/store";
  import { createQuery } from "@tanstack/svelte-query";
  import Badge from "../../components/ui/Badge.svelte";
  import type { ApiClient } from "../../lib/api";
  import { connectorSettingsQuery } from "../../lib/queries";
  import type { ConnectorId, SyncJobRow } from "../../lib/types";
  import { formatDateTime } from "../../lib/format.svelte";
  let {
    api,
    id,
    title,
    description,
    selected,
    onConfigure,
    jobs,
  }: {
    api: ApiClient;
    id: ConnectorId;
    title: string;
    description: string;
    selected: boolean;
    onConfigure: () => void;
    jobs?: SyncJobRow[];
  } = $props();
  const settings = createQuery(
    toStore(() => connectorSettingsQuery(() => api, id)),
  );
  const job = $derived(
    (jobs ?? []).find(
      (item) => item.connectorId === id && item.scope === "all",
    ),
  );
  const needsAction = $derived(
    job?.lastStatus === "failed" || job?.lastStatus === "needs_user_action",
  );
  const weekdays = ["週日", "週一", "週二", "週三", "週四", "週五", "週六"];
  const scheduleLabel = $derived(
    !job?.enabled
      ? "關閉"
      : job.scheduleMode === "inherit"
        ? "跟隨預設"
        : job.intervalMinutes === 1440
          ? `每天 ${job.preferredTime}`
          : job.intervalMinutes === 10080
            ? `每${weekdays[job.preferredWeekday] ?? "週一"} ${job.preferredTime}`
            : `每 ${job.intervalMinutes / 60} 小時`,
  );
  const SourceIcon = $derived(
    id === "einvoice"
      ? ReceiptText
      : id === "tdcc"
        ? ChartCandlestick
        : Landmark,
  );
</script>

<button
  class={`c-data-row w-full border-b border-border px-4 py-3.5 text-left last:border-b-0 ${selected ? "bg-accent/60 shadow-[inset_3px_0_0_#287080]" : "bg-card"}`}
  aria-current={selected ? "true" : undefined}
  onclick={onConfigure}
>
  <div class="flex items-start justify-between gap-4">
    <div class="flex min-w-0 items-start gap-3">
      <span
        class={`flex size-9 shrink-0 items-center justify-center rounded-lg ${selected ? "bg-steel text-white" : "bg-steel/10 text-steel"}`}
      >
        <SourceIcon class="size-5" />
      </span>
      <div class="min-w-0">
        <h2 class="font-semibold">{title}</h2>
        <p class="mt-0.5 truncate text-xs text-muted-foreground">
          {description}
        </p>
      </div>
    </div>
    <Badge
      class="shrink-0 whitespace-nowrap"
      variant={needsAction
        ? "destructive"
        : $settings.data?.configured
          ? "success"
          : "secondary"}
      >{needsAction
        ? "需要處理"
        : $settings.data?.configured
          ? "已設定"
          : "未設定"}</Badge
    >
  </div>
  <div class="mt-2 flex items-center justify-between gap-3 pl-12">
    <p class="truncate text-xs text-muted-foreground">
      {job?.lastSuccessAt
        ? `上次成功 ${formatDateTime(job.lastSuccessAt)}`
        : "尚無同步紀錄"}
      <span class="mx-1.5 text-border">|</span>{scheduleLabel}
    </p>
    <ChevronRight
      class={`size-4 shrink-0 text-muted-foreground transition-transform ${selected ? "translate-x-0.5 text-steel" : ""}`}
    />
  </div>
</button>
