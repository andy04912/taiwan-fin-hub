<script lang="ts">
  import { ChartCandlestick, Landmark, ReceiptText } from "@lucide/svelte";
  import { toStore } from "svelte/store";
  import { createQuery } from "@tanstack/svelte-query";
  import Badge from "../../components/ui/Badge.svelte";
  import type { ApiClient } from "../../lib/api";
  import { connectorSettingsQuery } from "../../lib/queries";
  import type { ConnectorId, SyncJobRow } from "../../lib/types";
  import { formatDateTime } from "../../lib/format.svelte";

  let { api, id, title, description, selected, onConfigure, jobs }: {
    api: ApiClient;
    id: ConnectorId;
    title: string;
    description: string;
    selected: boolean;
    onConfigure: () => void;
    jobs?: SyncJobRow[];
  } = $props();

  const settings = createQuery(toStore(() => connectorSettingsQuery(() => api, id)));
  const job = $derived((jobs ?? []).find((item) => item.connectorId === id && item.scope === "all"));
  const needsAction = $derived(job?.lastStatus === "failed" || job?.lastStatus === "needs_user_action");
  const weekdays = ["週日", "週一", "週二", "週三", "週四", "週五", "週六"];
  const scheduleLabel = $derived(
    !job?.enabled
      ? "未啟用排程"
      : job.scheduleMode === "inherit"
        ? "跟隨預設排程"
        : job.intervalMinutes === 1440
          ? `每天 ${job.preferredTime}`
          : job.intervalMinutes === 10080
            ? `每${weekdays[job.preferredWeekday] ?? "週一"} ${job.preferredTime}`
            : `每 ${job.intervalMinutes / 60} 小時`,
  );
  const SourceIcon = $derived(id === "einvoice" ? ReceiptText : id === "tdcc" ? ChartCandlestick : Landmark);
</script>

<button
  class={`group relative w-full border-b border-border px-4 py-4 text-left transition last:border-b-0 ${selected ? "bg-steel/[0.055]" : "bg-card hover:bg-secondary/55"}`}
  aria-current={selected ? "true" : undefined}
  onclick={onConfigure}
>
  {#if selected}<span class="absolute inset-y-3 left-0 w-0.5 rounded-r-full bg-steel"></span>{/if}
  <div class="flex items-start gap-3">
    <span class={`grid size-9 shrink-0 place-items-center rounded-lg ${selected ? "bg-steel text-white" : "bg-steel/8 text-steel"}`}>
      <SourceIcon class="size-5" />
    </span>
    <div class="min-w-0 flex-1">
      <div class="flex items-start justify-between gap-3">
        <div class="min-w-0">
          <h3 class="truncate text-sm font-semibold text-ink">{title}</h3>
          <p class="mt-0.5 truncate text-xs text-muted-foreground">{description}</p>
        </div>
        <Badge
          class="shrink-0 whitespace-nowrap"
          variant={needsAction ? "destructive" : $settings.data?.configured ? "success" : "secondary"}
        >{needsAction ? "需要處理" : $settings.data?.configured ? "已連線" : "未設定"}</Badge>
      </div>
      <div class="mt-3 grid gap-1 text-xs text-muted-foreground sm:grid-cols-2 xl:grid-cols-1 2xl:grid-cols-2">
        <span class="truncate">{job?.lastSuccessAt ? `上次同步 ${formatDateTime(job.lastSuccessAt)}` : "尚無同步紀錄"}</span>
        <span class="truncate sm:text-right xl:text-left 2xl:text-right">{scheduleLabel}</span>
      </div>
    </div>
  </div>
</button>
