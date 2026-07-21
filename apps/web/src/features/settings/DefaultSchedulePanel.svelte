<script lang="ts">
  import { onMount } from "svelte";
  import { createMutation, createQuery, useQueryClient } from "@tanstack/svelte-query";
  import { Check, Clock3, Save } from "@lucide/svelte";
  import Card from "../../components/ui/Card.svelte";
  import Button from "../../components/ui/Button.svelte";
  import Select from "../../components/ui/Select.svelte";
  import TimePicker from "../../components/ui/TimePicker.svelte";
  import type { ApiClient } from "../../lib/api";
  import { queryKeys } from "../../lib/api";
  import { syncScheduleQuery } from "../../lib/queries";
  import type { SyncJobRow, SyncScheduleSettings } from "../../lib/types";

  let { api, demoMode, jobs }: { api: ApiClient; demoMode: boolean; jobs: SyncJobRow[] } = $props();
  const queryClient = useQueryClient();
  const schedule = createQuery(syncScheduleQuery(() => api));
  const intervalOptions = [
    { label: "每小時", minutes: 60 },
    { label: "每 6 小時", minutes: 360 },
    { label: "每 12 小時", minutes: 720 },
    { label: "每天", minutes: 1440 },
    { label: "每週", minutes: 10080 },
  ];
  const weekdayOptions = ["週日", "週一", "週二", "週三", "週四", "週五", "週六"];
  let intervalMinutes = $state(1440);
  let preferredTime = $state("06:00");
  let preferredWeekday = $state(1);
  const inheritedJobs = $derived(jobs.filter((job) => job.scheduleMode === "inherit").length);

  onMount(() =>
    schedule.subscribe((result) => {
      if (!result.data) return;
      intervalMinutes = result.data.intervalMinutes;
      preferredTime = result.data.preferredTime;
      preferredWeekday = result.data.preferredWeekday;
    }),
  );

  const save = createMutation({
    mutationFn: () => api.put<SyncScheduleSettings>("/api/sync-schedule", { intervalMinutes, preferredTime, preferredWeekday }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: queryKeys.syncSchedule });
      queryClient.invalidateQueries({ queryKey: queryKeys.syncJobs });
    },
  });
</script>

<Card as="section" class="overflow-hidden shadow-sm">
  <div class="flex flex-col gap-3 border-b border-border px-5 py-4 sm:flex-row sm:items-start sm:justify-between">
    <div class="flex items-start gap-3">
      <span class="grid size-9 shrink-0 place-items-center rounded-lg bg-steel/8 text-steel"><Clock3 class="size-4.5" /></span>
      <div>
        <h2 class="font-semibold">預設同步排程</h2>
        <p class="mt-1 text-xs text-muted-foreground">管理所有「跟隨預設」資料來源的執行時間。</p>
      </div>
    </div>
    <div class="text-xs text-muted-foreground sm:text-right">
      <p class="font-semibold text-ink/70">Asia/Taipei</p>
      <p class="mt-1">{inheritedJobs} 個來源跟隨此排程</p>
    </div>
  </div>

  <div class="grid gap-4 p-5 md:grid-cols-[minmax(150px,0.8fr)_minmax(140px,0.7fr)_minmax(0,1fr)_auto] md:items-end">
    <label class="grid gap-1.5 text-sm font-medium">
      同步頻率
      <Select bind:value={intervalMinutes}>
        {#each intervalOptions as option (option.minutes)}<option value={option.minutes}>{option.label}</option>{/each}
      </Select>
    </label>

    {#if intervalMinutes === 10080}
      <label class="grid gap-1.5 text-sm font-medium">
        執行日
        <Select bind:value={preferredWeekday}>
          {#each weekdayOptions as weekday, index (weekday)}<option value={index}>{weekday}</option>{/each}
        </Select>
      </label>
    {:else if intervalMinutes >= 1440}
      <label class="grid gap-1.5 text-sm font-medium">
        開始時間
        <TimePicker bind:value={preferredTime} />
      </label>
    {:else}
      <div class="grid gap-1.5 text-sm font-medium">
        計時方式
        <div class="flex h-10 items-center rounded-md border border-border bg-muted/45 px-3 text-sm text-muted-foreground">從上次同步完成後計算</div>
      </div>
    {/if}

    {#if intervalMinutes === 10080}
      <label class="grid gap-1.5 text-sm font-medium">
        開始時間
        <TimePicker bind:value={preferredTime} />
      </label>
    {:else}
      <p class="text-xs leading-relaxed text-muted-foreground md:pb-2">修改後只影響跟隨預設的來源，自訂排程不會改變。</p>
    {/if}

    <Button class="w-full md:w-auto" disabled={demoMode || $save.isPending} onclick={() => $save.mutate()}>
      {#if $save.isSuccess}<Check class="size-4" />{:else}<Save class="size-4" />{/if}
      {$save.isPending ? "儲存中…" : $save.isSuccess ? "已儲存" : "儲存變更"}
    </Button>
  </div>

  {#if $save.isError}<p class="border-t border-border bg-coral/5 px-5 py-2.5 text-xs font-medium text-coral">預設排程儲存失敗，請稍後再試。</p>{/if}
</Card>
