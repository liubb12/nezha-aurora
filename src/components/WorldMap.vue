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
const rotation = ref<[number, number]>([-105, -30]);
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
// eslint-disable-next-line @typescript-eslint/no-explicit-any
let graticuleGenerator: any = null;

/** 3D 正射投影器 */
const projection = computed(() => {
  if (!d3Geo) return null;
  return d3Geo
    .geoOrthographic()
    .scale(BASE_RADIUS * zoom.value)
    .translate([WIDTH / 2, HEIGHT / 2])
    .clipAngle(90)
    .rotate(rotation.value);
});

/** 动态计算大陆轮廓 Path */
const currentLandPath = computed(() => {
  if (!projection.value || !rawGeoFeatures || !d3Geo) return "";
  const pathGenerator = d3Geo.geoPath(projection.value);
  return pathGenerator(rawGeoFeatures) || "";
});

/** 动态计算经纬度网格线 Path */
const currentGraticulePath = computed(() => {
  if (!projection.value || !graticuleGenerator || !d3Geo) return "";
  const pathGenerator = d3Geo.geoPath(projection.value);
  return pathGenerator(graticuleGenerator()) || "";
});

/** 常驻地理背景底图文字 */
const BASE_GEO_LABELS = [
  { name: "中国", lng: 104.1954, lat: 35.8617, code: "CN" },
  { name: "俄罗斯", lng: 105.3188, lat: 61.524, code: "RU" },
  { name: "蒙古", lng: 103.8467, lat: 46.8625, code: "MN" },
  { name: "哈萨克斯坦", lng: 66.9237, lat: 48.0196, code: "KZ" },
  { name: "印度", lng: 78.9629, lat: 20.5937, code: "IN" },
  { name: "澳大利亚", lng: 133.7751, lat: -25.2744, code: "AU" },
  { name: "加拿大", lng: -106.3468, lat: 56.1304, code: "CA" },
  { name: "美国", lng: -95.7129, lat: 37.0902, code: "US" },
  { name: "巴西", lng: -51.9253, lat: -14.235, code: "BR" },
  { name: "阿根廷", lng: -63.6167, lat: -38.4161, code: "AR" },
  { name: "南非", lng: 22.9375, lat: -30.5595, code: "ZA" },
  { name: "沙特阿拉伯", lng: 45.0792, lat: 23.8859, code: "SA" },
  { name: "印度尼西亚", lng: 113.9213, lat: -0.7893, code: "ID" },
];

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
  offsetX: number;
  offsetY: number;
  visible: boolean;
  online: boolean;
  isEdge: boolean;
  color: string;
  raw: PreparedServer;
}

interface GeoLabel {
  name: string;
  x: number;
  y: number;
  visible: boolean;
}

interface ArcLine {
  id: string;
  pathD: string;
  color: string;
}

/** 炫彩流光配色表 */
const COLOR_PALETTE = [
  "#38bdf8", // 荧光青蓝
  "#f97316", // 珊瑚亮橙
  "#a855f7", // 霓虹亮紫
  "#10b981", // 薄荷翠绿
  "#fbbf24", // 琥珀金黄
  "#f43f5e", // 鲜艳玫红
  "#06b6d4", // 极光碧青
  "#e879f9", // 炫动亮粉
];

const activeCountryCodes = computed(() => {
  const codes = new Set<string>();
  for (const item of props.items) {
    const code = (item.server.country_code || "").trim().toUpperCase();
    if (code) codes.add(code);
  }
  return codes;
});

const geoLabels = computed<GeoLabel[]>(() => {
  const proj = projection.value;
  if (!ready.value || !proj || !d3Geo) return [];

  const rot = rotation.value;
  const result: GeoLabel[] = [];

  for (const item of BASE_GEO_LABELS) {
    if (activeCountryCodes.value.has(item.code)) continue;

    const coords: [number, number] = [item.lng, item.lat];
    const point = proj(coords);
    if (!point) continue;

    const distance = d3Geo.geoDistance(coords, [-rot[0], -rot[1]]);
    if (distance < (Math.PI / 2) * 0.85) {
      result.push({
        name: item.name,
        x: point[0],
        y: point[1],
        visible: true,
      });
    }
  }
  return result;
});

const globeData = computed(() => {
  const proj = projection.value;
  if (!ready.value || !proj || !d3Geo) {
    return { nodes: [], lines: [], hub: null };
  }

  const nodes: GlobeNode[] = [];
  const rot = rotation.value;

  const grouped = new Map<string, PreparedServer[]>();
  for (const item of props.items) {
    const code = (item.server.country_code || "").trim().toUpperCase();
    if (!code) continue;
    const list = grouped.get(code);
    if (list) list.push(item);
    else grouped.set(code, [item]);
  }

  let colorCounter = 0;
  for (const [code, entries] of grouped) {
    const centroid = centroidOf(code);
    if (!centroid) continue;
    const [lat, lng] = centroid;
    const coords: [number, number] = [lng, lat];
    const point = proj(coords);
    if (!point) continue;

    const distance = d3Geo.geoDistance(coords, [-rot[0], -rot[1]]);
    const visible = distance < Math.PI / 2;
    const isEdge = distance > (Math.PI / 2) * 0.92;

    const isAnyOnline = entries.some((e) => e.online);
    const assignedColor = isAnyOnline
      ? COLOR_PALETTE[colorCounter % COLOR_PALETTE.length]
      : "#ef4444";
    colorCounter++;

    const rep = entries[0];
    nodes.push({
      id: rep.server.id,
      code,
      name: entries.length > 1 ? `${getCountryName(code)} (${entries.length}台)` : rep.server.name,
      displayName: getCountryName(code),
      x: point[0],
      y: point[1],
      offsetX: 0,
      offsetY: -16,
      visible,
      isEdge,
      online: isAnyOnline,
      color: assignedColor,
      raw: rep,
    });
  }

  // 阶梯式防重叠错开
  const visibleNodes = nodes.filter((n) => n.visible);
  visibleNodes.sort((a, b) => a.x - b.x);

  for (let i = 0; i < visibleNodes.length; i++) {
    const a = visibleNodes[i];
    let collisionCount = 0;
    for (let j = 0; j < i; j++) {
      const b = visibleNodes[j];
      const dist = Math.hypot(a.x - b.x, a.y - b.y);
      if (dist < 42) {
        collisionCount++;
      }
    }
    if (collisionCount > 0) {
      const pattern = collisionCount % 3;
      if (pattern === 1) {
        a.offsetY = 16;
        a.offsetX = -6;
      } else if (pattern === 2) {
        a.offsetY = -28;
        a.offsetX = 8;
      } else {
        a.offsetY = -12;
        a.offsetX = 28;
      }
    }
  }

  // 中心 Hub 优先中国/香港/美西
  let hub = nodes.find((n) => ["US", "CN", "HK", "TW"].includes(n.code) && n.online && n.visible);
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
      const d = Math.hypot(dx, dy);

      const midX = (sx + ex) / 2;
      const midY = (sy + ey) / 2;
      const nx = -dy / d;
      const ny = dx / d;

      const archHeight = Math.min(75, Math.max(25, d * 0.22));
      const mx = midX + nx * archHeight;
      const my = midY + ny * archHeight;

      lines.push({
        id: `${hub.id}-${node.id}`,
        pathD: `M ${sx} ${sy} Q ${mx} ${my} ${ex} ${ey}`,
        color: node.color,
      });
    }
  }

  return { nodes, lines, hub };
});

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
    graticuleGenerator = d3.geoGraticule();

    const atlas = (atlasModule.default ?? atlasModule) as unknown as {
      objects: { countries: unknown };
    };
    rawGeoFeatures = topojson.feature(
      atlas as never,
      atlas.objects.countries as never,
    );

    ready.value = true;

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
      <span>正在构建 3D 蔚蓝科技地球…</span>
    </div>

    <template v-else>
      <div class="globe-legend">
        <span class="chip">
          <i class="dot dot--online" /> 在线 {{ items.filter((i) => i.online).length }}
        </span>
        <span class="chip">
          <i class="dot dot--offline" /> 离线 {{ items.filter((i) => !i.online).length }}
        </span>
        <span class="chip hint">滚轮缩放 · 拖拽自转</span>
        <button type="button" class="chip reset-btn" @click="resetView">复位中心</button>
      </div>

      <div
        class="globe-viewport"
        @wheel="onWheel"
        @mousedown="onMouseDown"
        @mousemove="onMouseMove"
        @mouseup="onMouseUp"
      >
        <div class="globe-stage" :style="{ width: `${WIDTH}px`, height: `${HEIGHT}px` }">
          <svg
            :viewBox="`0 0 ${WIDTH} ${HEIGHT}`"
            preserveAspectRatio="xMidYMid meet"
            role="img"
            aria-label="3D全球监控拓扑"
          >
            <defs>
              <!-- 真实深海钴蓝球体渐变与大气发光 -->
              <radialGradient id="oceanGrad" cx="45%" cy="40%" r="65%">
                <stop offset="0%" stop-color="#1d4ed8" stop-opacity="0.85" />
                <stop offset="55%" stop-color="#0f2b5c" stop-opacity="0.95" />
                <stop offset="85%" stop-color="#07132b" />
                <stop offset="100%" stop-color="#38bdf8" stop-opacity="0.7" />
              </radialGradient>

              <!-- 外太空环晕 -->
              <radialGradient id="haloGrad" cx="50%" cy="50%" r="50%">
                <stop offset="90%" stop-color="transparent" />
                <stop offset="97%" stop-color="#38bdf8" stop-opacity="0.2" />
                <stop offset="100%" stop-color="transparent" />
              </radialGradient>
            </defs>

            <!-- 1. 外层柔和大气漫反射 -->
            <circle
              :cx="WIDTH / 2"
              :cy="HEIGHT / 2"
              :r="BASE_RADIUS * zoom * 1.04"
              fill="url(#haloGrad)"
              pointer-events="none"
            />

            <!-- 2. 蔚蓝海洋底球 -->
            <circle
              :cx="WIDTH / 2"
              :cy="HEIGHT / 2"
              :r="BASE_RADIUS * zoom"
              fill="url(#oceanGrad)"
              stroke="rgba(56, 189, 248, 0.55)"
              :stroke-width="1.8"
            />

            <!-- 3. 科技经纬网格线 -->
            <g class="globe-graticule">
              <path :d="currentGraticulePath" />
            </g>

            <!-- 4. 晶石微青大陆板块 -->
            <g class="globe-land">
              <path :d="currentLandPath" />
            </g>

            <!-- 5. 常驻地理底图文字 -->
            <g class="globe-base-labels">
              <text
                v-for="label in geoLabels"
                :key="label.name"
                :x="label.x"
                :y="label.y"
                text-anchor="middle"
                dominant-baseline="central"
                class="base-geo-text"
              >
                {{ label.name }}
              </text>
            </g>

            <!-- 6. 炫彩流动飞线 -->
            <g class="globe-lines">
              <path
                v-for="line in globeData.lines"
                :key="line.id"
                :d="line.pathD"
                fill="none"
                :stroke="line.color"
                :stroke-width="1.8"
                stroke-linecap="round"
                stroke-dasharray="8 4"
                class="flowing-arc"
                :style="{ filter: `drop-shadow(0 0 5px ${line.color})` }"
              />
            </g>

            <!-- 7. 核心节点发光点 -->
            <g class="globe-dots">
              <g
                v-for="node in globeData.nodes"
                :key="node.id"
                v-show="node.visible"
                class="dot-group"
                @mouseenter="showTooltip(node, $event)"
                @mousemove="showTooltip(node, $event)"
                @click.stop="openServer(node.id)"
              >
                <circle
                  class="node-halo"
                  :fill="node.color"
                  :cx="node.x"
                  :cy="node.y"
                  :r="5.5"
                />
                <circle
                  class="node-dot"
                  :fill="node.color"
                  stroke="#ffffff"
                  stroke-width="1.2px"
                  :cx="node.x"
                  :cy="node.y"
                  :r="3.2"
                />
              </g>
            </g>
          </svg>

          <!-- 8. 彩色服务器高清胶囊标签 -->
          <div class="html-labels-layer">
            <div
              v-for="node in globeData.nodes"
              :key="`lbl-${node.id}`"
              v-show="node.visible"
              class="clear-city-tag"
              :class="{ 'is-edge': node.isEdge }"
              :style="{
                left: `${(node.x / WIDTH) * 100}%`,
                top: `${(node.y / HEIGHT) * 100}%`,
                transform: `translate(calc(-50% + ${node.offsetX}px), calc(-50% + ${node.offsetY}px))`,
                color: node.color,
                borderColor: node.color,
                boxShadow: `0 0 12px color-mix(in srgb, ${node.color} 35%, transparent), 0 3px 8px rgba(0,0,0,0.6)`,
              }"
              @mouseenter="showTooltip(node, $event)"
              @mousemove="showTooltip(node, $event)"
              @click.stop="openServer(node.id)"
            >
              {{ node.displayName }}
            </div>
          </div>
        </div>
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
            <span class="country-name" :style="{ color: tooltip.node.color }">
              {{ tooltip.node.displayName }}
            </span>
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
  background: radial-gradient(circle at 50% 50%, rgba(13, 22, 44, 0.7) 0%, rgba(5, 10, 24, 0.96) 100%);
  border-radius: 12px;
  overflow: hidden;
  user-select: none;
}

.globe-viewport {
  cursor: grab;
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
}

.globe-viewport:active {
  cursor: grabbing;
}

.globe-stage {
  position: relative;
  max-width: 100%;
  aspect-ratio: 960 / 580;
}

.globe-stage svg {
  width: 100%;
  height: 100%;
  display: block;
}

/* 经纬度网格线 */
.globe-graticule path {
  fill: none;
  stroke: rgba(56, 189, 248, 0.12);
  stroke-width: 0.5;
}

/* 晶石冷青绿陆地 */
.globe-land path {
  fill: #162a3b;
  stroke: #2dd4bf;
  stroke-width: 0.5;
  stroke-opacity: 0.35;
}

:global([data-theme="light"]) .globe-land path {
  fill: #cbd5e1;
  stroke: #0ea5e9;
  stroke-opacity: 0.4;
}

/* 常驻底图文字 */
.base-geo-text {
  fill: rgba(224, 242, 254, 0.35);
  font-size: 10.5px;
  font-weight: 600;
  letter-spacing: 1px;
  pointer-events: none;
  user-select: none;
}

:global([data-theme="light"]) .base-geo-text {
  fill: rgba(15, 23, 42, 0.45);
}

/* 飞线流动动效 */
.flowing-arc {
  animation: arcPulse 1.4s linear infinite;
}

@keyframes arcPulse {
  from {
    stroke-dashoffset: 24;
  }
  to {
    stroke-dashoffset: 0;
  }
}

.dot-group {
  cursor: pointer;
}

.node-halo {
  opacity: 0.5;
  animation: haloAnim 2s infinite ease-out;
}

@keyframes haloAnim {
  0% {
    r: 3;
    opacity: 0.75;
  }
  100% {
    r: 10;
    opacity: 0;
  }
}

.html-labels-layer {
  position: absolute;
  inset: 0;
  pointer-events: none;
}

.clear-city-tag {
  position: absolute;
  padding: 2.5px 7px;
  background: rgba(10, 20, 38, 0.92);
  border-radius: 4px;
  font-size: 10.5px;
  font-weight: 700;
  line-height: 1.2;
  white-space: nowrap;
  border-width: 1px;
  border-style: solid;
  pointer-events: auto;
  cursor: pointer;
  backdrop-filter: blur(8px);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  transition: transform 0.2s ease, background 0.15s ease;
}

.clear-city-tag.is-edge {
  opacity: 0.6;
}

.clear-city-tag:hover {
  background: rgba(15, 32, 64, 0.98);
  z-index: 20;
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
