<script lang="ts">
  import { onMount, onDestroy } from "svelte";
  import { listen, type UnlistenFn } from "@tauri-apps/api/event";
  import { invoke } from "@tauri-apps/api/core";

  interface TagStatus {
    is_selected: boolean;
    is_urg: boolean;
    is_filled: boolean;
    is_occ: boolean;
  }
  interface MonitorInfoSnapshot {
    monitor_num: number;
    monitor_width: number;
    monitor_height: number;
    monitor_x: number;
    monitor_y: number;
    tag_status_vec: TagStatus[];
    client_name: string;
    ltsymbol: string;
  }
  interface SystemSnapshot {
    cpu_average: number;
    memory_used: number;
    memory_total: number;
    memory_usage_percent: number;
    battery_percent: number;
    is_charging: boolean;
  }

  const BUTTONS = ["🐖", "🐄", "🐂", "🐃", "🦥", "🦣", "🐏", "🦆", "🐢"];

  let monitor = $state<MonitorInfoSnapshot | null>(null);
  let system = $state<SystemSnapshot | null>(null);
  let pressed = $state<number | null>(null);
  let layoutOpen = $state(false);
  let showSeconds = $state(true);
  let now = $state(new Date());
  let isTaking = $state(false);

  let unlistenMonitor: UnlistenFn | undefined;
  let unlistenSystem: UnlistenFn | undefined;
  let timer: ReturnType<typeof setInterval> | undefined;

  function getButtonClass(t: TagStatus) {
    if (t.is_filled) return "emoji-button state-filtered";
    if (t.is_selected) return "emoji-button state-selected";
    if (t.is_urg) return "emoji-button state-urgent";
    if (t.is_occ) return "emoji-button state-occupied";
    return "emoji-button state-default";
  }
  function formatBytes(bytes: number) {
    if (bytes === 0) return "0B";
    const U = ["B", "KB", "MB", "GB", "TB"];
    const i = Math.floor(Math.log(bytes) / Math.log(1024));
    const s = parseFloat((bytes / Math.pow(1024, i)).toFixed(i === 0 ? 0 : 1));
    return `${s}${U[i]}`;
  }
  function parseLtSymbol(lts: string | undefined) {
    if (!lts) return { symbol: "[]=", scale: undefined as number | undefined };
    const sym = lts.match(/^(\S+)/);
    const sc = lts.match(/s:\s*([0-9.]+)/i);
    return {
      symbol: sym ? sym[1] : "[]=",
      scale: sc ? parseFloat(sc[1]) : undefined,
    };
  }
  function monitorIcon(num: number) {
    if (num === 0) return "󰎡";
    if (num === 1) return "󰎤";
    return `M${num}`;
  }
  function sev(p: number) {
    return p <= 30
      ? "usage-good"
      : p <= 60
        ? "usage-warn"
        : p <= 80
          ? "usage-caution"
          : "usage-danger";
  }

  function handlePress(i: number) {
    pressed = i;
  }
  function handleRelease(i: number, monitorNum: number) {
    pressed = null;
    invoke("send_tag_command", {
      tagIndex: i,
      isView: true,
      monitorId: monitorNum,
    }).catch((e) => console.error(e));
  }
  function selectLayout(idx: number, monitorNum: number) {
    layoutOpen = false;
    invoke("send_layout_command", {
      layoutIndex: idx,
      monitorId: monitorNum,
    }).catch((e) => console.error(e));
  }
  async function takeScreenshot() {
    if (isTaking) return;
    isTaking = true;
    try {
      await invoke("take_screenshot");
    } catch (e) {
      console.error(e);
    } finally {
      setTimeout(() => (isTaking = false), 500);
    }
  }

  onMount(async () => {
    console.log("Tauri Svelte frontend has loaded.");
    unlistenMonitor = await listen<MonitorInfoSnapshot>(
      "monitor-update",
      (e) => (monitor = e.payload),
    );
    unlistenSystem = await listen<SystemSnapshot>(
      "system-update",
      (e) => (system = e.payload),
    );
  });

  $effect(() => {
    if (timer) clearInterval(timer);
    timer = setInterval(
      () => (now = new Date()),
      showSeconds ? 1000 : 60000,
    );
  });

  onDestroy(() => {
    unlistenMonitor?.();
    unlistenSystem?.();
    if (timer) clearInterval(timer);
  });

  const pad = (n: number) => n.toString().padStart(2, "0");
  let formattedTime = $derived.by(() => {
    const d = now;
    const ts = `${pad(d.getHours())}:${pad(d.getMinutes())}${
      showSeconds ? `:${pad(d.getSeconds())}` : ""
    }`;
    return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())} ${ts}`;
  });
  let lt = $derived(parseLtSymbol(monitor?.ltsymbol));
</script>

{#if !monitor}
  <div class="button-row">Loading...</div>
{:else}
  <div class="button-row">
    <div class="buttons-container">
      {#each BUTTONS as emoji, i}
        {@const tag = monitor.tag_status_vec[i] ?? {
          is_selected: false,
          is_urg: false,
          is_filled: false,
          is_occ: false,
        }}
        <button
          class={`${getButtonClass(tag)}${pressed === i ? " pressed" : ""}`}
          onmousedown={() => handlePress(i)}
          onmouseup={() => handleRelease(i, monitor.monitor_num)}
          onmouseleave={() => (pressed = null)}
        >
          {emoji}
        </button>
      {/each}

      <div class="layout-controls">
        <div
          class={`pill layout-toggle ${layoutOpen ? "open" : "closed"}`}
          onclick={() => (layoutOpen = !layoutOpen)}
          role="button"
          tabindex="0"
          onkeydown={(e) => e.key === "Enter" && (layoutOpen = !layoutOpen)}
          title="切换布局"
        >
          {lt.symbol}
        </div>
        {#if layoutOpen}
          <div class="layout-selector">
            <div
              class={`pill layout-option ${lt.symbol === "[]=" ? "current" : ""}`}
              onclick={() => selectLayout(0, monitor.monitor_num)}
              role="button"
              tabindex="0"
              onkeydown={(e) =>
                e.key === "Enter" && selectLayout(0, monitor.monitor_num)}
            >
              []=
            </div>
            <div
              class={`pill layout-option ${lt.symbol === "><>" ? "current" : ""}`}
              onclick={() => selectLayout(1, monitor.monitor_num)}
              role="button"
              tabindex="0"
              onkeydown={(e) =>
                e.key === "Enter" && selectLayout(1, monitor.monitor_num)}
            >
              {"><>"}
            </div>
            <div
              class={`pill layout-option ${lt.symbol === "[M]" ? "current" : ""}`}
              onclick={() => selectLayout(2, monitor.monitor_num)}
              role="button"
              tabindex="0"
              onkeydown={(e) =>
                e.key === "Enter" && selectLayout(2, monitor.monitor_num)}
            >
              [M]
            </div>
          </div>
        {/if}
      </div>
    </div>

    <div class="spacer"></div>

    <div class="right-info-container">
      <div class="system-info-container">
        {#if system}
          <div
            class={`pill usage-pill ${sev(system.cpu_average)}`}
            title="CPU 平均使用率"
          >
            {`CPU ${system.cpu_average.toFixed(0)}%`}
          </div>
          <div
            class={`pill usage-pill ${sev(system.memory_usage_percent)}`}
            title={`内存使用: ${formatBytes(system.memory_used)} / ${formatBytes(system.memory_total)}`}
          >
            {`MEM ${system.memory_usage_percent.toFixed(0)}%`}
          </div>
          <div
            class={`pill usage-pill ${
              system.battery_percent > 50
                ? "usage-good"
                : system.battery_percent > 20
                  ? "usage-warn"
                  : "usage-danger"
            }`}
            title={system.is_charging
              ? `电池充电中: ${system.battery_percent.toFixed(1)}%`
              : `电池电量: ${system.battery_percent.toFixed(1)}%`}
          >
            {`${system.is_charging ? "🔌" : "🔋"} ${system.battery_percent.toFixed(0)}%`}
          </div>
        {:else}
          <div class="pill usage-pill usage-warn">CPU --%</div>
          <div class="pill usage-pill usage-warn">MEM --%</div>
          <div class="pill usage-pill usage-warn">🔋 --%</div>
        {/if}
      </div>

      <div
        class={`pill screenshot-pill ${isTaking ? "taking" : ""}`}
        onclick={takeScreenshot}
        role="button"
        tabindex="0"
        onkeydown={(e) => e.key === "Enter" && takeScreenshot()}
        title="截图 (Flameshot)"
      >
        {isTaking ? "⏳" : "📸"}
      </div>

      <div
        class="pill time-pill"
        onclick={() => (showSeconds = !showSeconds)}
        role="button"
        tabindex="0"
        onkeydown={(e) => e.key === "Enter" && (showSeconds = !showSeconds)}
        title="点击切换秒显示"
      >
        {formattedTime}
      </div>

      <div class="pill monitor-pill" title="显示器">
        {"🖥️ " + monitorIcon(monitor.monitor_num)}
      </div>

      <div class="pill scale-pill" title="Scale Factor">
        {lt.scale !== undefined ? `s: ${lt.scale.toFixed(2)}` : "s: --"}
      </div>
    </div>
  </div>
{/if}
