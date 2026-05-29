<script lang="ts">
  import WarehouseScene from './lib/WarehouseScene.svelte';
  import { SEGMENTS_WAREHOUSE5 } from './data/segmentsWarehouse5';
  import type { SegmentType } from './types';

  const ALL_TYPES: SegmentType[] = ['AISLE', 'BAY', 'LEVEL', 'SPACE'];

  const TYPE_COLORS: Record<SegmentType, string> = {
    AISLE: '#3b82f6',
    BAY:   '#f97316',
    LEVEL: '#22c55e',
    SPACE: '#ef4444',
  };

  let visibleTypes = new Set<SegmentType>(ALL_TYPES);

  function toggle(type: SegmentType) {
    const next = new Set(visibleTypes);
    if (next.has(type)) next.delete(type);
    else next.add(type);
    visibleTypes = next;
  }

  $: counts = (() => {
    const out: Record<SegmentType, number> = { AISLE: 0, BAY: 0, LEVEL: 0, SPACE: 0 };
    for (const s of SEGMENTS_WAREHOUSE5) out[s.type]++;
    return out;
  })();
</script>

<main>
  <header>
    <div class="title">
      <h1>Warehouse 3D View</h1>
      <span class="subtitle">
        {counts.AISLE} aisles · {counts.BAY} bays · {counts.LEVEL} levels · {counts.SPACE.toLocaleString()} spaces
      </span>
    </div>
    <div class="filters">
      {#each ALL_TYPES as type}
        <button
          class="chip"
          class:active={visibleTypes.has(type)}
          style="--chip-color: {TYPE_COLORS[type]}"
          on:click={() => toggle(type)}
        >
          <span class="dot"></span>{type}
          <span class="count">{counts[type].toLocaleString()}</span>
        </button>
      {/each}
    </div>
  </header>
  <WarehouseScene segments={SEGMENTS_WAREHOUSE5} {visibleTypes} />
</main>

<style>
  main {
    display: flex;
    flex-direction: column;
    height: 100%;
    background: #0f172a;
  }
  header {
    background: #0b1220;
    border-bottom: 1px solid #1e293b;
    padding: 10px 18px;
    display: flex;
    align-items: center;
    gap: 24px;
    color: #e2e8f0;
  }
  .title { display: flex; flex-direction: column; }
  h1 { font-size: 16px; margin: 0; font-weight: 600; letter-spacing: 0.2px; }
  .subtitle { font-size: 11px; opacity: 0.65; font-family: monospace; margin-top: 2px; }
  .filters { display: flex; gap: 8px; flex-wrap: wrap; }
  .chip {
    background: transparent;
    border: 1.5px solid #334155;
    color: #94a3b8;
    padding: 4px 10px;
    border-radius: 999px;
    cursor: pointer;
    font-size: 11px;
    font-family: monospace;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    transition: all 0.15s;
  }
  .chip:hover { border-color: var(--chip-color); color: #e2e8f0; }
  .chip.active { border-color: var(--chip-color); color: var(--chip-color); background: rgba(255,255,255,0.03); }
  .dot {
    display: inline-block;
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: var(--chip-color);
  }
  .count { opacity: 0.55; }
</style>
