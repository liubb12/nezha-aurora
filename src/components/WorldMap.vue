<script setup lang="ts">
import { computed, onMounted, ref } from "vue";
import { useRouter } from "vue-router";
import type { PreparedServer } from "@/store/nezha";
import { formatSpeed, percent } from "@/utils/format";
import { centroidOf } from "@/utils/country-centroids";

const props = defineProps<{
  items: PreparedServer[];
}>();

const router = useRouter();

const WIDTH = 960;
const HEIGHT = 500;

const ready = ref(false);
const failed = ref(false);
const landPaths = ref<string[]>([]);

/** 缩放与平移状态 */
const scale = ref(1);
const translateX = ref(0);
const translateY = ref(0);
const isDragging = ref(false);
const startX = ref(0);
const startY = ref(0);

const mapTransform = computed(() => {
  return `translate(${translateX.value}, ${translateY.value}) scale(${scale.value})`;
});

function onWheel(e: WheelEvent) {
  e.preventDefault();
  const zoomFactor = e.deltaY < 0 ? 1.2 : 0.8;
  const newScale = Math.min(Math.max(scale.value * zoomFactor, 1), 8);
  scale.value = Number(newScale.toFixed(2));
  if (scale.value === 1) {
    translateX.value = 0;
    translateY.value = 0;
  }
}

function onMouseDown(e: MouseEvent) {
  if (e.button !== 0) return;
  isDragging.value = true;
  startX.value = e.clientX - translateX.value;
  startY.value = e.clientY - translateY.value;
}

function onMouseMove(e: MouseEvent) {
  if (!isDragging.value) return;
  translateX.value = e.clientX - startX.value;
  translateY.value = e.clientY - startY.value;
}

function onMouseUp() {
  isDragging.value = false;
}

function resetZoom() {
  scale.value = 1;
  translateX.value = 0;
  translateY.value = 0;
}

function getFlagUrl(code: string): string {
  if (!code) return "";
  return `https://flagcdn.com/24x18/${code.toLowerCase()}.png`;
}

const regionNames = new Intl.DisplayNames(["zh-CN"], { type: "region" });
function getCountryName(code: string): string {
  try {
    return regionNames.of(code.toUpperCase()) || code;
  } catch {
    return code;
  }
}

let project: ((coords: [number, number]) => [number, number] | null) | null = null;

interface ClusterNode {
  server: PreparedServer;
  x: number;
  y: number;
}

interface Cluster {
  code: string;
  x: number;
  y: number;
  online: number;
  offline: number;
  entries: PreparedServer[];
  scatteredNodes: ClusterNode[];
}

/** 计算聚合与展开散点 */
const clusters = computed<Cluster[]>(() => {
  if (!ready.value || !project) return [];

  const grouped = new Map<string, PreparedServer[]>();
  for (const item of props.items) {
    const code = (item.server.country_code || "").trim().toUpperCase();
    if (!code) continue;
    const list = grouped.get(code);
    if (list) list.push(item);
    else grouped.set(code, [item]);
  }

  const result: Cluster[] = [];
  const currentScale = scale.value;
  const shouldSpread = currentScale > 1.8; // 当放大超过 1.8 倍时展开散点

  for (const [code, entries] of grouped) {
    const centroid = centroidOf(code);
    if (!centroid) continue;
    const [lat, lng] = centroid;
    const point = project([lng, lat]);
    if (!point) continue;

    const centerX = Number(point[0].toFixed(2));
    const centerY = Number(point[1].toFixed(2));
    const sortedEntries = entries.sort((a, b) => Number(b.online) - Number(a.online));

    // 计算散开节点分布（星环算法）
    const scatteredNodes: ClusterNode[] = [];
    if (shouldSpread && sortedEntries.length > 1) {
      const count = sortedEntries.length;
      const baseRadius = Math.min(22, 9 + count * 1.5); // 散开半径
      for (let i = 0; i < count; i++) {
        const angle = (2 * Math.PI * i) / count;
        scatteredNodes.push({
          server: sortedEntries[i],
          x: centerX + Math.cos(angle) * (baseRadius / Math.sqrt(currentScale)),
          y: centerY + Math.sin(angle) * (baseRadius / Math.sqrt(currentScale)),
        });
      }
    }

    result.push({
      code,
      x: centerX,
      y: centerY,
      online: sortedEntries.filter((item) => item.online).length,
      offline: sortedEntries.filter((item) => !item.online).length,
      entries: sortedEntries,
      scatteredNodes,
    });
  }
  return result;
});

const locatedCount = computed(() =>
  clusters.value.reduce((sum, cluster) => sum + cluster.entries.length, 0),
);

const tooltip = ref<{
  x: number;
  y: number;
  cluster: Cluster;
  singleNode?: PreparedServer;
} | null>(null);

function showTooltip(cluster: Cluster, event: MouseEvent, singleNode?: PreparedServer) {
  const container = (event.currentTarget as HTMLElement).closest(".world-map");
  if (!container) return;
  const rect = container.getBoundingClientRect();
  let x = event.clientX - rect.left;
  let y = event.clientY - rect.top;

  const count = singleNode ? 1 : cluster.entries.length;
  const popoverWidth = count > 5 ? 560 : 360;
  if (x + popoverWidth > rect.width) {
    x = Math.max(10, rect.width - popoverWidth - 16);
  }
  tooltip.value = { x, y, cluster, singleNode };
}

function hideTooltip() {
  tooltip.value = null;
}

function openServer(id: number) {
  router.push(`/server/${id}`);
}

onMounted(async () => {
  try {
    const [d3, topojson, atlasModule] = await Promise.all([
      import("d3-geo"),
      import("topojson-client"),
      import("world-atlas/countries-110m.json"),
    ]);

    const atlas = (atlasModule.default ?? atlasModule) as unknown as {
      objects: { countries: unknown };
    };
    const features = topojson.feature(
      atlas as never,
      atlas.objects.countries as never,
    ) as unknown as { features: unknown[] };

    const projection = d3.geoNaturalEarth1();
    projection.fitExtent(
      [
        [12, 12],
        [WIDTH - 12, HEIGHT - 12],
      ],
      features as never,
    );

    const pathGenerator = d3.geoPath(projection);
    landPaths.value = features.features
      .map((feature) => pathGenerator(feature as never))
      .filter((path): path is string => Boolean(path));

    project = (coords) => {
      const point = projection(coords);
      return point ? [point[0], point[1]] : null;
    };
    ready.value = true;
  } catch (error) {
    failed.value = true;
    console.error("[Aurora] 世界地图加载失败", error);
  }
});
</script>

<template>
  <div class="world-map panel" @mouseleave="hideTooltip">
    <div v-if="failed" class="world-map__state">地图资源加载失败，请检查网络或改用本地依赖。</div>
    <div v-else-if="!ready" class="world-map__state">
      <span class="spinner" />
      <span>正在加载地图数据…</span>
    </div>

    <template v-else>
      <div class="world-map__legend">
        <span class="chip">
          <i class="dot dot--online" /> 在线 {{ items.filter((i) => i.online).length }}
        </span>
        <span class="chip">
          <i class="dot dot--offline" /> 离线 {{ items.filter((i) => !i.online).length }}
        </span>
        <span class="chip num">{{ locatedCount }}/{{ items.length }} 个节点可定位</span>
        <span class="chip hint">滚轮缩放地图，放大后自动散开节点</span>
        <button v-if="scale > 1" type="button" class="chip reset-btn" @click="resetZoom">
          重置视角
        </button>
      </div>

      <div
        class="world-map__viewport"
        @wheel="onWheel"
        @mousedown="onMouseDown"
        @mousemove="onMouseMove"
        @mouseup="onMouseUp"
        @mouseleave="onMouseUp"
      >
        <svg
          :viewBox="`0 0 ${WIDTH} ${HEIGHT}`"
          preserveAspectRatio="xMidYMid meet"
          role="img"
          aria-label="节点世界分布"
        >
          <g :transform="mapTransform">
            <g class="world-map__land">
              <path v-for="(path, index) in landPaths" :key="index" :d="path" />
            </g>

            <g class="world-map__nodes">
              <g v-for="cluster in clusters" :key="cluster.code" class="world-map__cluster">
                <!-- 状态 A：放大后散开显示的独立节点 -->
                <template v-if="cluster.scatteredNodes.length > 0">
                  <g
                    v-for="node in cluster.scatteredNodes"
                    :key="node.server.server.id"
                    class="world-map__node"
                    @mouseenter="showTooltip(cluster, $event, node.server)"
                    @mousemove="showTooltip(cluster, $event, node.server)"
                    @click.stop="openServer(node.server.server.id)"
                  >
                    <circle
                      class="node-glow"
                      :class="node.server.online ? 'is-online' : 'is-offline'"
                      :cx="node.x"
                      :cy="node.y"
                      :r="5 / Math.sqrt(scale)"
                    />
                    <circle
                      class="node-core"
                      :class="node.server.online ? 'is-online' : 'is-offline'"
                      :cx="node.x"
                      :cy="node.y"
                      :r="2.8 / Math.sqrt(scale)"
                      :style="{ strokeWidth: `${1 / Math.sqrt(scale)}px` }"
                    />
                  </g>
                </template>

                <!-- 状态 B：全局概览时的精致微光聚合点 -->
                <template v-else>
                  <g
                    class="world-map__node"
                    @mouseenter="showTooltip(cluster, $event)"
                    @mousemove="showTooltip(cluster, $event)"
                    @click="showTooltip(cluster, $event)"
                  >
                    <!-- 外呼吸光晕 -->
                    <circle
                      class="world-map__pulse"
                      :class="cluster.offline ? 'has-offline' : 'all-online'"
                      :cx="cluster.x"
                      :cy="cluster.y"
                      :r="6.5 / Math.sqrt(scale)"
                    />
                    <!-- 中心实体发光点 -->
                    <circle
                      class="world-map__dot"
                      :class="cluster.offline ? 'has-offline' : 'all-online'"
                      :cx="cluster.x"
                      :cy="cluster.y"
                      :r="3.8 / Math.sqrt(scale)"
                      :style="{ strokeWidth: `${1.2 / Math.sqrt(scale)}px` }"
                    />
                    <!-- 精致右上微型角标（不再压在正中） -->
                    <g
                      v-if="cluster.entries.length > 1"
                      :transform="`translate(${cluster.x + 5 / Math.sqrt(scale)}, ${cluster.y - 5 / Math.sqrt(scale)})`"
                    >
                      <circle
                        class="badge-bg"
                        :r="4.2 / Math.sqrt(scale)"
                      />
                      <text
                        class="badge-text"
                        :style="{
                          fontSize: `${Math.max(5.5, 6.8 / Math.sqrt(scale))}px`
                        }"
                        text-anchor="middle"
                        dominant-baseline="central"
                      >
                        {{ cluster.entries.length }}
                      </text>
                    </g>
                  </g>
                </template>
              </g>
            </g>
          </g>
        </svg>
      </div>

      <!-- 精致浮动面板 -->
      <div
        v-if="tooltip"
        class="world-map__tooltip"
        :class="{ 'is-multi-col': !tooltip.singleNode && tooltip.cluster.entries.length > 5 }"
        :style="{ left: `${tooltip.x}px`, top: `${tooltip.y}px` }"
      >
        <div class="world-map__tooltip-head">
          <div class="head-title">
            <img
              class="country-flag-img"
              :src="getFlagUrl(tooltip.cluster.code)"
              :alt="tooltip.cluster.code"
              @error="($event.target as HTMLElement).style.display = 'none'"
            />
            <span class="country-name">{{ getCountryName(tooltip.cluster.code) }}</span>
            <span class="country-code">({{ tooltip.cluster.code }})</span>
          </div>
          <span class="world-map__tooltip-stat">
            <template v-if="tooltip.singleNode">
              节点详情
            </template>
            <template v-else>
              在线 {{ tooltip.cluster.online }} / 离线 {{ tooltip.cluster.offline }}
            </template>
          </span>
        </div>

        <div class="world-map__tooltip-body">
          <button
            v-for="entry in (tooltip.singleNode ? [tooltip.singleNode] : tooltip.cluster.entries)"
            :key="entry.server.id"
            type="button"
            class="world-map__tooltip-item"
            @click="openServer(entry.server.id)"
          >
            <div class="item-left">
              <i class="dot" :class="entry.online ? 'dot--online' : 'dot--offline'" />
              <span class="world-map__tooltip-name" :title="entry.server.name">{{ entry.server.name }}</span>
            </div>
            <div class="world-map__tooltip-meta num">
              <span>CPU {{ (entry.server.state?.cpu || 0).toFixed(0) }}%</span>
              <span>MEM {{ percent(entry.server.state?.mem_used, entry.server.host?.mem_total).toFixed(0) }}%</span>
              <span v-if="entry.online" class="speed">↓{{ formatSpeed(entry.server.state?.net_in_speed, 1) }}</span>
            </div>
          </button>
        </div>
      </div>
    </template>
  </div>
</template>

<style scoped>
.world-map {
  position: relative;
  padding: 14px;
  --map-land: rgba(148, 178, 255, 0.08);
  --map-stroke: rgba(148, 178, 255, 0.18);
}

:global([data-theme="light"]) .world-map {
  --map-land: #e4ecf8;
  --map-stroke: #c6d4e8;
}

.world-map__viewport {
  overflow: hidden;
  cursor: grab;
  user-select: none;
}

.world-map__viewport:active {
  cursor: grabbing;
}

.world-map svg {
  width: 100%;
  height: auto;
  display: block;
}

.world-map__land path {
  fill: var(--map-land);
  stroke: var(--map-stroke);
  stroke-width: 0.5;
  vector-effect: non-scaling-stroke;
}

.world-map__state {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  min-height: 260px;
  color: var(--text-dim);
  font-size: 13px;
}

.world-map__legend {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 8px;
  align-items: center;
}

.world-map__legend .hint {
  font-size: 11.5px;
  opacity: 0.6;
}

.reset-btn {
  cursor: pointer;
  background: var(--accent-soft);
  color: var(--text);
  border: 1px solid var(--border);
}

.world-map__node {
  cursor: pointer;
}

/* 核心圆点样式 */
.world-map__dot.all-online {
  fill: #10b981;
  stroke: rgba(16, 185, 129, 0.4);
}

.world-map__dot.has-offline {
  fill: #ef4444;
  stroke: rgba(239, 68, 68, 0.4);
}

.world-map__pulse {
  opacity: 0.35;
}

.world-map__pulse.all-online {
  fill: #10b981;
  animation: map-pulse 2.2s cubic-bezier(0.24, 0, 0.38, 1) infinite;
}

.world-map__pulse.has-offline {
  fill: #ef4444;
  animation: map-pulse 2.2s cubic-bezier(0.24, 0, 0.38, 1) infinite;
}

/* 散开独立小节点 */
.node-core.is-online {
  fill: #10b981;
  stroke: #059669;
}

.node-core.is-offline {
  fill: #ef4444;
  stroke: #dc2626;
}

.node-glow.is-online {
  fill: rgba(16, 185, 129, 0.25);
}

.node-glow.is-offline {
  fill: rgba(239, 68, 68, 0.25);
}

/* 右上方极小数字角标 */
.badge-bg {
  fill: #1e293b;
  stroke: #334155;
  stroke-width: 0.5;
}

.badge-text {
  fill: #f8fafc;
  font-weight: 700;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
}

@keyframes map-pulse {
  0% {
    transform: scale(0.9);
    opacity: 0.45;
  }
  70% {
    transform: scale(1.6);
    opacity: 0;
  }
  100% {
    transform: scale(1.6);
    opacity: 0;
  }
}

/* 弹窗面板 */
.world-map__tooltip {
  position: absolute;
  z-index: 50;
  min-width: 330px;
  max-width: 420px;
  transform: translate(12px, 12px);
  padding: 12px 14px;
  border-radius: 10px;
  border: 1px solid var(--border-strong);
  background: color-mix(in srgb, var(--panel-solid) 94%, transparent);
  box-shadow: 0 16px 36px rgba(0, 0, 0, 0.4);
  backdrop-filter: blur(14px);
  pointer-events: auto;
}

.world-map__tooltip.is-multi-col {
  min-width: 520px;
  max-width: 620px;
}

.world-map__tooltip-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  font-size: 13px;
  font-weight: 650;
  padding-bottom: 8px;
  margin-bottom: 8px;
  border-bottom: 1px solid var(--border);
}

.head-title {
  display: flex;
  align-items: center;
  gap: 8px;
}

.country-flag-img {
  width: 20px;
  height: 14px;
  border-radius: 2px;
  object-fit: cover;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.35);
  display: inline-block;
  flex-shrink: 0;
}

.country-name {
  color: var(--text);
}

.country-code {
  font-size: 11px;
  color: var(--text-faint);
}

.world-map__tooltip-stat {
  font-size: 11.5px;
  font-weight: 500;
  color: var(--text-faint);
}

.world-map__tooltip-body {
  display: flex;
  flex-direction: column;
  gap: 4px;
  max-height: 380px;
  overflow-y: auto;
}

.world-map__tooltip.is-multi-col .world-map__tooltip-body {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 6px;
}

.world-map__tooltip-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  width: 100%;
  padding: 6px 8px;
  border-radius: 7px;
  text-align: left;
  background: rgba(255, 255, 255, 0.02);
  transition: background 0.15s ease;
}

.world-map__tooltip-item:hover {
  background: var(--accent-soft);
}

.item-left {
  display: flex;
  align-items: center;
  gap: 7px;
  flex: 1;
  min-width: 0;
}

.world-map__tooltip-name {
  font-size: 12.5px;
  font-weight: 550;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.world-map__tooltip-meta {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 11px;
  color: var(--text-faint);
  white-space: nowrap;
}

.world-map__tooltip-meta .speed {
  color: var(--ok);
}
</style>
