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
  interface AudioSnapshot {
    volume: number;
    is_muted: boolean;
    device_name: string;
    has_device: boolean;
  }
  interface BrightnessSnapshot {
    percent: number | null;
  }

  const TAG_ICONS = [
    "\u{F0A1E}",
    "\u{F0239}",
    "\u{F0A1B}",
    "\u{F0B79}",
    "\u{F024B}",
    "\u{F0388}",
    "\u{F0567}",
    "\u{F01F0}",
    "\u{F0297}",
  ];
  const ICON_CPU = "\u{F4BC}";
  const ICON_MEM = "\u{F035B}";
  const ICON_BAT_FULL = "\u{F0079}";
  const ICON_BAT_CHG = "\u{F0084}";
  const ICON_VOL_HIGH = "\u{F057E}";
  const ICON_VOL_MID = "\u{F0580}";
  const ICON_VOL_LOW = "\u{F057F}";
  const ICON_VOL_MUTE = "\u{F075F}";
  const ICON_BRIGHT = "\u{F00DE}";
  const ICON_SHOT = "\u{F0104}";
  const ICON_TIME = "\u{F0954}";
  const ICON_MON = "\u{F0379}";

  let monitor = $state<MonitorInfoSnapshot | null>(null);
  let system = $state<SystemSnapshot | null>(null);
  let audio = $state<AudioSnapshot | null>(null);
  let brightness = $state<BrightnessSnapshot | null>(null);
  let pressed = $state<number | null>(null);
  let layoutOpen = $state(false);
  let showSeconds = $state(true);
  let now = $state(new Date());
  let isTaking = $state(false);

  let unlistenMonitor: UnlistenFn | undefined;
  let unlistenSystem: UnlistenFn | undefined;
  let unlistenAudio: UnlistenFn | undefined;
  let unlistenBrightness: UnlistenFn | undefined;
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
    if (num === 0) return "\u{F02DA}";
    if (num === 1) return "\u{F02DB}";
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
  function volumeIconChar(a: AudioSnapshot | null) {
    if (!a || !a.has_device) return ICON_VOL_MUTE;
    if (a.is_muted) return ICON_VOL_MUTE;
    if (a.volume <= 0) return ICON_VOL_MUTE;
    if (a.volume < 34) return ICON_VOL_LOW;
    if (a.volume < 67) return ICON_VOL_MID;
    return ICON_VOL_HIGH;
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
  function onToggleMute() {
    invoke("toggle_mute").catch((e) => console.error(e));
  }
  function onVolumeWheel(e: WheelEvent) {
    e.preventDefault();
    const delta = e.deltaY < 0 ? 5 : -5;
    invoke("adjust_volume", { delta }).catch((err) => console.error(err));
  }
  function onBrightnessClick() {
    invoke("adjust_brightness", { delta: 5 }).catch((e) => console.error(e));
  }
  function onBrightnessRight(e: MouseEvent) {
    e.preventDefault();
    invoke("adjust_brightness", { delta: -5 }).catch((err) => console.error(err));
  }
  function onBrightnessWheel(e: WheelEvent) {
    e.preventDefault();
    const delta = e.deltaY < 0 ? 5 : -5;
    invoke("adjust_brightness", { delta }).catch((err) => console.error(err));
  }

  onMount(async () => {
    console.log("Tauri Svelte frontend has loaded.");
    try {
      unlistenMonitor = await listen<MonitorInfoSnapshot | null>(
        "monitor-update",
        (e) => (monitor = e.payload),
      );
      unlistenSystem = await listen<SystemSnapshot>(
        "system-update",
        (e) => (system = e.payload),
      );
      unlistenAudio = await listen<AudioSnapshot>(
        "audio-update",
        (e) => (audio = e.payload),
      );
      unlistenBrightness = await listen<BrightnessSnapshot>(
        "brightness-update",
        (e) => (brightness = e.payload),
      );
      await invoke<void>("frontend_ready");
    } catch (error) {
      console.error("Failed to initialize Tauri event bridge:", error);
    }
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
    unlistenAudio?.();
    unlistenBrightness?.();
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
      {#each TAG_ICONS as icon, i}
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
          title={`Tag ${i + 1}`}
        >
          <span class="nf-icon">{icon}</span>
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
            <span class="nf-icon">{ICON_CPU}</span> {`${system.cpu_average.toFixed(0)}%`}
          </div>
          <div
            class={`pill usage-pill ${sev(system.memory_usage_percent)}`}
            title={`内存使用: ${formatBytes(system.memory_used)} / ${formatBytes(system.memory_total)}`}
          >
            <span class="nf-icon">{ICON_MEM}</span> {`${system.memory_usage_percent.toFixed(0)}%`}
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
            <span class="nf-icon">{system.is_charging ? ICON_BAT_CHG : ICON_BAT_FULL}</span>
            {`${system.battery_percent.toFixed(0)}%`}
          </div>
        {:else}
          <div class="pill usage-pill usage-warn">
            <span class="nf-icon">{ICON_CPU}</span> --%
          </div>
          <div class="pill usage-pill usage-warn">
            <span class="nf-icon">{ICON_MEM}</span> --%
          </div>
          <div class="pill usage-pill usage-warn">
            <span class="nf-icon">{ICON_BAT_FULL}</span> --%
          </div>
        {/if}
      </div>

      <div
        class="pill brightness-pill"
        onclick={onBrightnessClick}
        onwheel={onBrightnessWheel}
        oncontextmenu={onBrightnessRight}
        role="button"
        tabindex="0"
        onkeydown={(e) => e.key === "Enter" && onBrightnessClick()}
        title="左键加亮 / 右键减暗 / 滚轮调节"
      >
        <span class="nf-icon">{ICON_BRIGHT}</span>
        {brightness?.percent != null ? `${brightness.percent}%` : "--"}
      </div>

      <div
        class={`pill volume-pill ${!audio || audio.is_muted || !audio.has_device ? "muted" : ""}`}
        onclick={onToggleMute}
        onwheel={onVolumeWheel}
        role="button"
        tabindex="0"
        onkeydown={(e) => e.key === "Enter" && onToggleMute()}
        title="左键静音 / 滚轮调节"
      >
        <span class="nf-icon">{volumeIconChar(audio)}</span>
        {audio?.has_device ? `${audio.volume}%` : "--"}
      </div>

      <div
        class={`pill screenshot-pill ${isTaking ? "taking" : ""}`}
        onclick={takeScreenshot}
        role="button"
        tabindex="0"
        onkeydown={(e) => e.key === "Enter" && takeScreenshot()}
        title="截图 (Flameshot)"
      >
        <span class="nf-icon">{ICON_SHOT}</span>
      </div>

      <div
        class="pill time-pill"
        onclick={() => (showSeconds = !showSeconds)}
        role="button"
        tabindex="0"
        onkeydown={(e) => e.key === "Enter" && (showSeconds = !showSeconds)}
        title="点击切换秒显示"
      >
        <span class="nf-icon">{ICON_TIME}</span> {formattedTime}
      </div>

      <div class="pill monitor-pill" title="显示器">
        <span class="nf-icon">{ICON_MON}</span> {monitorIcon(monitor.monitor_num)}
      </div>

      <div class="pill scale-pill" title="Scale Factor">
        {lt.scale !== undefined ? `s: ${lt.scale.toFixed(2)}` : "s: --"}
      </div>
    </div>
  </div>
{/if}
