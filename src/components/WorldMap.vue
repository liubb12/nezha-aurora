<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref } from "vue";
import { useRouter } from "vue-router";
import type { PreparedServer } from "@/store/nezha";
import { formatSpeed, percent } from "@/utils/format";
import { centroidOf } from "@/utils/country-centroids";

const props = defineProps<{
  items: PreparedServer[];
}>();

const router = useRouter();

const WIDTH = 960;
const HEIGHT = 580;
const BASE_RADIUS = 220;

const ready = ref(false);
const failed = ref(false);

/** 缩放与旋转参数 */
const zoom = ref(1);
const rotation = ref<[number, number]>([-105, -30]); // 默认正面对准亚太/欧亚
const isDragging = ref(false);
const startX = ref(0);
const startY = ref(0);
const startRot = ref<[number, number]>([-105, -30]);
const isHovering = ref(false);
let autoRotateTimer: number | null = null;

// eslint-disable-next-line @typescript-eslint/no-explicit-any
let d3Geo: any = null;
// eslint-disable-next-line @typescript-eslint/no-explicit-any
let rawGeoFeatures: any = null;

/** 动态计算当前投影器（随旋转和缩放实时变化） */
const projection = computed(() => {
  if (!d3Geo) return null;
  return d3Geo
    .geoOrthographic()
    .scale(BASE_RADIUS * zoom.value)
    .translate([WIDTH / 2, HEIGHT / 2])
    .clipAngle(90)
    .rotate(rotation.value);
});

/** 实时计算陆地轮廓 Path 字符串 */
const currentLandPath = computed(() => {
  if (!projection.value || !rawGeoFeatures || !d3Geo) return "";
  const pathGenerator = d3Geo.geoPath(projection.value);
  return pathGenerator(rawGeoFeatures) || "";
});

/** 国家代码转中文名称 */
const regionNames = new Intl.DisplayNames(["zh-CN"], { type: "region" });
function getCountryName(code: string): string {
  try {
    return regionNames.of(code.toUpperCase()) || code;
  } catch {
    return code;
  }
}

function getFlagUrl(code: string): string {
  if (!code) return "";
  return `https://flagcdn.com/24x18/${code.toLowerCase()}.png`;
}

interface GlobeNode {
  id: number;
  code: string;
  name: string;
  displayName: string;
  x: number;
  y: number;
  visible: boolean;
  online: boolean;
  raw: PreparedServer;
}

interface ArcLine {
  id: string;
  pathD: string;
  color: string;
}

/** 实时解析球体表面节点与飞线 */
const globeData = computed(() => {
  const proj = projection.value;
  if (!ready.value || !proj || !d3Geo) {
    return { nodes: [], lines: [], hub: null };
  }

  const nodes: GlobeNode[] = [];
  const rot = rotation.value;

  // 聚合去重：按国家归类
  const grouped = new Map<string, PreparedServer[]>();
  for (const item of props.items) {
    const code = (item.server.country_code || "").trim().toUpperCase();
    if (!code) continue;
    const list = grouped.get(code);
    if (list) list.push(item);
    else grouped.set(code, [item]);
  }

  for (const [code, entries] of grouped) {
    const centroid = centroidOf(code);
    if (!centroid) continue;
    const [lat, lng] = centroid;
    const coords: [number, number] = [lng, lat];
    const point = proj(coords);
    if (!point) continue;

    // 正背面剔除判断
    const distance = d3Geo.geoDistance(coords, [-rot[0], -rot[1]]);
    const visible = distance < Math.PI / 2;

    const rep = entries[0];
    nodes.push({
      id: rep.server.id,
      code,
      name: entries.length > 1 ? `${getCountryName(code)} (${entries.length}台)` : rep.server.name,
      displayName: getCountryName(code),
      x: point[0],
      y: point[1],
      visible,
      online: entries.some((e) => e.online),
      raw: rep,
    });
  }

  // 设定 Hub 中心（优先中国/香港，没有则按在线节点首位）
  let hub = nodes.find((n) => ["CN", "HK", "TW", "US"].includes(n.code) && n.online && n.visible);
  if (!hub && nodes.length > 0) hub = nodes.find((n) => n.visible) || nodes[0];

  const lines: ArcLine[] = [];
  if (hub && hub.visible) {
    for (const node of nodes) {
      if (node.id === hub.id || !node.visible) continue;

      const sx = hub.x;
      const sy = hub.y;
      const ex = node.x;
      const ey = node.y;

      const dx = ex - sx;
      const dy = ey - sy;
      const mx = (sx + ex) / 2 - dy * 0.22;
      const my = (sy + ey) / 2 + dx * 0.22;

      lines.push({
        id: `${hub.id}-${node.id}`,
        pathD: `M ${sx} ${sy} Q ${mx} ${my} ${ex} ${ey}`,
        color: node.online ? "rgba(56, 189, 248, 0.85)" : "rgba(248, 113, 113, 0.6)",
      });
    }
  }

  return { nodes, lines, hub };
});

/** 鼠标交互：拖拽与滚轮 */
function onMouseDown(e: MouseEvent) {
  if (e.button !== 0) return;
  isDragging.value = true;
  startX.value = e.clientX;
  startY.value = e.clientY;
  startRot.value = [...rotation.value];
}

function onMouseMove(e: MouseEvent) {
  if (!isDragging.value) return;
  const dx = e.clientX - startX.value;
  const dy = e.clientY - startY.value;
  const sens = 0.35 / zoom.value;
  rotation.value = [
    startRot.value[0] + dx * sens,
    Math.max(-80, Math.min(80, startRot.value[1] - dy * sens)),
  ];
}

function onMouseUp() {
  isDragging.value = false;
}

function onWheel(e: WheelEvent) {
  e.preventDefault();
  const factor = e.deltaY < 0 ? 1.15 : 0.85;
  zoom.value = Math.min(Math.max(zoom.value * factor, 0.8), 3.5);
}

function resetView() {
  zoom.value = 1;
  rotation.value = [-105, -30];
}

/** 弹窗提示 */
const tooltip = ref<{ x: number; y: number; node: GlobeNode } | null>(null);
function showTooltip(node: GlobeNode, event: MouseEvent) {
  const container = (event.currentTarget as HTMLElement).closest(".globe-panel");
  if (!container) return;
  const rect = container.getBoundingClientRect();
  tooltip.value = {
    x: Math.min(rect.width - 320, event.clientX - rect.left + 12),
    y: event.clientY - rect.top + 12,
    node,
  };
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

    d3Geo = d3;
    const atlas = (atlasModule.default ?? atlasModule) as unknown as {
      objects: { countries: unknown };
    };
    rawGeoFeatures = topojson.feature(
      atlas as never,
      atlas.objects.countries as never,
    );

    ready.value = true;

    // 平滑自转动画
    autoRotateTimer = window.setInterval(() => {
      if (!isDragging.value && !isHovering.value) {
        rotation.value = [rotation.value[0] + 0.18, rotation.value[1]];
      }
    }, 35);
  } catch (err) {
    failed.value = true;
    console.error("[Aurora] 3D地球仪加载失败", err);
  }
});

onUnmounted(() => {
  if (autoRotateTimer) clearInterval(autoRotateTimer);
});
</script>

<template>
  <div
    class="globe-panel panel"
    @mouseleave="hideTooltip(); isHovering = false"
    @mouseenter="isHovering = true"
  >
    <div v-if="failed" class="globe-state">3D 地图组件加载失败。</div>
    <div v-else-if="!ready" class="globe-state">
      <span class="spinner" />
      <span>正在构建 3D 网络拓扑…</span>
    </div>

    <template v-else>
      <div class="globe-legend">
        <span class="chip">
          <i class="dot dot--online" /> 在线 {{ items.filter((i) => i.online).length }}
        </span>
        <span class="chip">
          <i class="dot dot--offline" /> 离线 {{ items.filter((i) => !i.online).length }}
        </span>
        <span class="chip hint">滚轮放大缩小 · 左键拖拽 360° 自转</span>
        <button type="button" class="chip reset-btn" @click="resetView">复位中心</button>
      </div>

      <div
        class="globe-viewport"
        @wheel="onWheel"
        @mousedown="onMouseDown"
        @mousemove="onMouseMove"
        @mouseup="onMouseUp"
      >
        <svg
          :viewBox="`0 0 ${WIDTH} ${HEIGHT}`"
          preserveAspectRatio="xMidYMid meet"
          role="img"
          aria-label="3D全球监控拓扑"
        >
          <defs>
            <radialGradient id="globeGrad" cx="50%" cy="50%" r="50%">
              <stop offset="60%" stop-color="#141f36" />
              <stop offset="90%" stop-color="#0b1220" />
              <stop offset="100%" stop-color="#38bdf8" stop-opacity="0.3" />
            </radialGradient>
          </defs>

          <!-- 1. 背景球体 -->
          <circle
            :cx="WIDTH / 2"
            :cy="HEIGHT / 2"
            :r="BASE_RADIUS * zoom"
            fill="url(#globeGrad)"
            stroke="rgba(56, 189, 248, 0.4)"
            :stroke-width="1.5"
          />

          <!-- 2. 实时重绘的大陆板块 -->
          <g class="globe-land">
            <path :d="currentLandPath" />
          </g>

          <!-- 3. 动态流动飞线 -->
          <g class="globe-lines">
            <path
              v-for="line in globeData.lines"
              :key="line.id"
              :d="line.pathD"
              fill="none"
              :stroke="line.color"
              :stroke-width="1.4"
              stroke-linecap="round"
              stroke-dasharray="6 3"
              class="flowing-arc"
            />
          </g>

          <!-- 4. 节点与中文标签 -->
          <g class="globe-nodes">
            <g
              v-for="node in globeData.nodes"
              :key="node.id"
              v-show="node.visible"
              class="globe-node"
              @mouseenter="showTooltip(node, $event)"
              @mousemove="showTooltip(node, $event)"
              @click.stop="openServer(node.id)"
            >
              <circle
                class="node-halo"
                :class="node.online ? 'is-online' : 'is-offline'"
                :cx="node.x"
                :cy="node.y"
                :r="5"
              />
              <circle
                class="node-dot"
                :class="node.online ? 'is-online' : 'is-offline'"
                :cx="node.x"
                :cy="node.y"
                :r="3"
              />

              <!-- 悬浮胶囊中文标签 -->
              <g class="node-tag" :transform="`translate(${node.x}, ${node.y - 10})`">
                <rect
                  rx="3"
                  ry="3"
                  x="-20"
                  y="-9"
                  width="40"
                  height="14"
                  class="tag-rect"
                />
                <text class="tag-label" y="1" text-anchor="middle" dominant-baseline="central">
                  {{ node.displayName }}
                </text>
              </g>
            </g>
          </g>
        </svg>
      </div>

      <!-- 悬浮弹窗 -->
      <div
        v-if="tooltip"
        class="globe-tooltip"
        :style="{ left: `${tooltip.x}px`, top: `${tooltip.y}px` }"
      >
        <div class="tooltip-head">
          <div class="head-left">
            <img class="country-flag-img" :src="getFlagUrl(tooltip.node.code)" :alt="tooltip.node.code" />
            <span class="country-name">{{ tooltip.node.displayName }}</span>
            <span class="country-code">({{ tooltip.node.code }})</span>
          </div>
          <span class="node-status" :class="tooltip.node.online ? 'is-online' : 'is-offline'">
            {{ tooltip.node.online ? '在线' : '离线' }}
          </span>
        </div>

        <div class="tooltip-body">
          <div class="server-title">{{ tooltip.node.name }}</div>
          <div class="meta-row num">
            <span>CPU {{ (tooltip.node.raw.server.state?.cpu || 0).toFixed(0) }}%</span>
            <span>MEM {{ percent(tooltip.node.raw.server.state?.mem_used, tooltip.node.raw.server.host?.mem_total).toFixed(0) }}%</span>
            <span v-if="tooltip.node.online" class="speed">
              ↓{{ formatSpeed(tooltip.node.raw.server.state?.net_in_speed, 1) }}
            </span>
          </div>
        </div>
      </div>
    </template>
  </div>
</template>

<style scoped>
.globe-panel {
  position: relative;
  padding: 16px;
  background: radial-gradient(circle at 50% 50%, rgba(15, 23, 42, 0.6) 0%, rgba(10, 15, 30, 0.95) 100%);
  border-radius: 12px;
  overflow: hidden;
  user-select: none;
}

.globe-viewport {
  cursor: grab;
  display: flex;
  justify-content: center;
  align-items: center;
}

.globe-viewport:active {
  cursor: grabbing;
}

.globe-viewport svg {
  width: 100%;
  max-height: 580px;
  display: block;
}

/* 陆地板块 */
.globe-land path {
  fill: #273549;
  stroke: #3b4d66;
  stroke-width: 0.6;
  transition: d 0.05s linear;
}

:global([data-theme="light"]) .globe-land path {
  fill: #cbd5e1;
  stroke: #94a3b8;
}

/* 飞线流动动画 */
.flowing-arc {
  animation: arcPulse 1.2s linear infinite;
}

@keyframes arcPulse {
  from {
    stroke-dashoffset: 18;
  }
  to {
    stroke-dashoffset: 0;
  }
}

/* 节点 */
.globe-node {
  cursor: pointer;
}

.node-dot.is-online {
  fill: #38bdf8;
  stroke: #ffffff;
  stroke-width: 1px;
}

.node-dot.is-offline {
  fill: #ef4444;
  stroke: #ffffff;
  stroke-width: 1px;
}

.node-halo {
  opacity: 0.4;
}

.node-halo.is-online {
  fill: #38bdf8;
  animation: haloAnim 2s infinite ease-out;
}

.node-halo.is-offline {
  fill: #ef4444;
}

@keyframes haloAnim {
  0% {
    r: 3;
    opacity: 0.6;
  }
  100% {
    r: 9;
    opacity: 0;
  }
}

/* 中文胶囊标签 */
.node-tag {
  pointer-events: none;
  filter: drop-shadow(0 1px 3px rgba(0, 0, 0, 0.5));
}

.tag-rect {
  fill: rgba(255, 255, 255, 0.95);
  stroke: rgba(203, 213, 225, 0.8);
  stroke-width: 0.5;
}

:global([data-theme="dark"]) .tag-rect {
  fill: rgba(30, 41, 59, 0.92);
  stroke: rgba(71, 85, 105, 0.8);
}

.tag-label {
  fill: #0f172a;
  font-size: 9px;
  font-weight: 600;
}

:global([data-theme="dark"]) .tag-label {
  fill: #f8fafc;
}

.globe-state {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  min-height: 380px;
  color: var(--text-dim);
  font-size: 13px;
}

.globe-legend {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 12px;
}

.globe-legend .hint {
  font-size: 11.5px;
  opacity: 0.6;
}

.reset-btn {
  cursor: pointer;
  background: var(--accent-soft);
  color: var(--text);
  border: 1px solid var(--border);
}

.globe-tooltip {
  position: absolute;
  z-index: 60;
  min-width: 250px;
  padding: 12px 14px;
  border-radius: 10px;
  border: 1px solid var(--border-strong);
  background: color-mix(in srgb, var(--panel-solid) 92%, transparent);
  box-shadow: 0 16px 36px rgba(0, 0, 0, 0.45);
  backdrop-filter: blur(14px);
  pointer-events: none;
}

.tooltip-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding-bottom: 8px;
  margin-bottom: 8px;
  border-bottom: 1px solid var(--border);
}

.head-left {
  display: flex;
  align-items: center;
  gap: 7px;
}

.country-flag-img {
  width: 18px;
  height: 13px;
  border-radius: 2px;
  object-fit: cover;
}

.country-name {
  font-weight: 650;
  font-size: 13px;
}

.country-code {
  font-size: 11px;
  color: var(--text-faint);
}

.node-status.is-online {
  color: #10b981;
  font-size: 11.5px;
  font-weight: 600;
}

.node-status.is-offline {
  color: #ef4444;
  font-size: 11.5px;
  font-weight: 600;
}

.server-title {
  font-size: 12.5px;
  font-weight: 550;
  margin-bottom: 6px;
  color: var(--text);
}

.meta-row {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 11px;
  color: var(--text-faint);
}

.meta-row .speed {
  color: #10b981;
}
</style>
