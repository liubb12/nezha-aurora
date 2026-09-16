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

const stageRef = ref<HTMLElement | null>(null);
const width = ref(1200);
const height = ref(720);

const ready = ref(false);
const failed = ref(false);

const isMobile = computed(() => width.value < 640);

/** 球体半径：基于计算后的精准高度自适应最大化展现 */
const baseRadius = computed(() => {
  if (isMobile.value) {
    return Math.min(width.value * 0.44, height.value * 0.44);
  }
  // 电脑大屏：自适应高度填满，留出少量晕环边距
  return Math.min(width.value * 0.45, height.value * 0.46);
});

/** 缩放与旋转参数 */
const zoom = ref(1);
const rotation = ref<[number, number]>([-105, -30]);
const isDragging = ref(false);
const startX = ref(0);
const startY = ref(0);
const startRot = ref<[number, number]>([-105, -30]);
const isHovering = ref(false);
let autoRotateTimer: number | null = null;
let resizeObserver: ResizeObserver | null = null;

// eslint-disable-next-line @typescript-eslint/no-explicit-any
let d3Geo: any = null;
// eslint-disable-next-line @typescript-eslint/no-explicit-any
let rawGeoFeatures: any = null;
// eslint-disable-next-line @typescript-eslint/no-explicit-any
let graticuleGenerator: any = null;

/** 3D 正射投影器（动态基于全屏尺寸精准居中） */
const projection = computed(() => {
  if (!d3Geo) return null;
  return d3Geo
    .geoOrthographic()
    .scale(baseRadius.value * zoom.value)
    .translate([width.value / 2, height.value / 2])
    .clipAngle(90)
    .rotate(rotation.value);
});

const currentLandPath = computed(() => {
  if (!projection.value || !rawGeoFeatures || !d3Geo) return "";
  const pathGenerator = d3Geo.geoPath(projection.value);
  return pathGenerator(rawGeoFeatures) || "";
});

const currentGraticulePath = computed(() => {
  if (!projection.value || !graticuleGenerator || !d3Geo) return "";
  const pathGenerator = d3Geo.geoPath(projection.value);
  return pathGenerator(graticuleGenerator()) || "";
});

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

const COLOR_PALETTE = [
  "#38bdf8",
  "#f97316",
  "#a855f7",
  "#10b981",
  "#fbbf24",
  "#f43f5e",
  "#06b6d4",
  "#e879f9",
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
      offsetY: isMobile.value ? -12 : -16,
      visible,
      isEdge,
      online: isAnyOnline,
      color: assignedColor,
      raw: rep,
    });
  }

  const visibleNodes = nodes.filter((n) => n.visible);
  visibleNodes.sort((a, b) => a.x - b.x);

  const collideDist = isMobile.value ? 30 : 44;
  for (let i = 0; i < visibleNodes.length; i++) {
    const a = visibleNodes[i];
    let collisionCount = 0;
    for (let j = 0; j < i; j++) {
      const b = visibleNodes[j];
      const dist = Math.hypot(a.x - b.x, a.y - b.y);
      if (dist < collideDist) {
        collisionCount++;
      }
    }
    if (collisionCount > 0) {
      const pattern = collisionCount % 3;
      if (pattern === 1) {
        a.offsetY = isMobile.value ? 14 : 18;
        a.offsetX = -6;
      } else if (pattern === 2) {
        a.offsetY = isMobile.value ? -22 : -28;
        a.offsetX = 6;
      } else {
        a.offsetY = -10;
        a.offsetX = isMobile.value ? 22 : 30;
      }
    }
  }

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

      const maxArch = isMobile.value ? 55 : 85;
      const archHeight = Math.min(maxArch, Math.max(20, d * 0.22));
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

/* ---------------- 交互事件 ---------------- */
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
  const sens = (isMobile.value ? 0.45 : 0.35) / zoom.value;
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
  zoom.value = Math.min(Math.max(zoom.value * factor, 0.75), 3.5);
}

function onTouchStart(e: TouchEvent) {
  if (e.touches.length === 1) {
    isDragging.value = true;
    startX.value = e.touches[0].clientX;
    startY.value = e.touches[0].clientY;
    startRot.value = [...rotation.value];
  }
}

function onTouchMove(e: TouchEvent) {
  if (!isDragging.value || e.touches.length !== 1) return;
  const dx = e.touches[0].clientX - startX.value;
  const dy = e.touches[0].clientY - startY.value;
  if (Math.abs(dx) > Math.abs(dy)) {
    e.preventDefault();
  }
  const sens = 0.45 / zoom.value;
  rotation.value = [
    startRot.value[0] + dx * sens,
    Math.max(-80, Math.min(80, startRot.value[1] - dy * sens)),
  ];
}

function onTouchEnd() {
  isDragging.value = false;
}

function resetView() {
  zoom.value = 1;
  rotation.value = [-105, -30];
}

/* ---------------- 悬浮面板 ---------------- */
const tooltip = ref<{ x: number; y: number; node: GlobeNode } | null>(null);
function showTooltip(node: GlobeNode, event: MouseEvent | TouchEvent) {
  if (!stageRef.value) return;
  const rect = stageRef.value.getBoundingClientRect();
  const clientX = "touches" in event ? event.touches[0].clientX : event.clientX;
  const clientY = "touches" in event ? event.touches[0].clientY : event.clientY;

  tooltip.value = {
    x: Math.min(rect.width - (isMobile.value ? 230 : 320), clientX - rect.left + 12),
    y: clientY - rect.top + 12,
    node,
  };
}

function hideTooltip() {
  tooltip.value = null;
}

function openServer(id: number) {
  router.push(`/server/${id}`);
}

/* ---------------- 精准满屏高度计算（扣除顶部导航和底部版权） ---------------- */
function refreshDimensions() {
  if (!stageRef.value) return;
  const clientWidth = stageRef.value.clientWidth || window.innerWidth;
  width.value = clientWidth;

  if (clientWidth < 640) {
    height.value = Math.min(480, Math.max(380, Math.floor(window.innerHeight * 0.55)));
  } else {
    // 窗口高度减去顶栏(约60px)、工具栏(约45px)、底部版权(约50px)以及内边距
    const fitHeight = window.innerHeight - 175;
    height.value = Math.max(560, fitHeight);
  }
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

    refreshDimensions();

    if (stageRef.value) {
      resizeObserver = new ResizeObserver(() => {
        refreshDimensions();
      });
      resizeObserver.observe(stageRef.value);
    }
    window.addEventListener("resize", refreshDimensions);

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
  if (resizeObserver) resizeObserver.disconnect();
  window.removeEventListener("resize", refreshDimensions);
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
        <span class="chip hint">
          {{ isMobile ? '单指滑动 360° 旋转' : '滚轮缩放 · 拖拽自转' }}
        </span>
        <button type="button" class="chip reset-btn" @click="resetView">复位中心</button>
      </div>

      <!-- 纯自适应宽度的全景舞台 -->
      <div
        ref="stageRef"
        class="globe-viewport"
        :style="{ height: `${height}px` }"
        @wheel="onWheel"
        @mousedown="onMouseDown"
        @mousemove="onMouseMove"
        @mouseup="onMouseUp"
        @touchstart="onTouchStart"
        @touchmove="onTouchMove"
        @touchend="onTouchEnd"
      >
        <svg
          class="globe-svg"
          :viewBox="`0 0 ${width} ${height}`"
          preserveAspectRatio="xMidYMid meet"
          role="img"
          aria-label="3D全球监控拓扑"
        >
          <defs>
            <radialGradient id="oceanGrad" cx="45%" cy="40%" r="65%">
              <stop offset="0%" stop-color="#1d4ed8" stop-opacity="0.85" />
              <stop offset="55%" stop-color="#0f2b5c" stop-opacity="0.95" />
              <stop offset="85%" stop-color="#07132b" />
              <stop offset="100%" stop-color="#38bdf8" stop-opacity="0.7" />
            </radialGradient>

            <radialGradient id="haloGrad" cx="50%" cy="50%" r="50%">
              <stop offset="90%" stop-color="transparent" />
              <stop offset="97%" stop-color="#38bdf8" stop-opacity="0.2" />
              <stop offset="100%" stop-color="transparent" />
            </radialGradient>
          </defs>

          <!-- 1. 外层大气晕环 -->
          <circle
            :cx="width / 2"
            :cy="height / 2"
            :r="baseRadius * zoom * 1.04"
            fill="url(#haloGrad)"
            pointer-events="none"
          />

          <!-- 2. 蔚蓝海洋底球 -->
          <circle
            :cx="width / 2"
            :cy="height / 2"
            :r="baseRadius * zoom"
            fill="url(#oceanGrad)"
            stroke="rgba(56, 189, 248, 0.55)"
            :stroke-width="isMobile ? 1.2 : 1.8"
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
              :style="{ fontSize: isMobile ? '8.5px' : '11px' }"
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
              :stroke-width="isMobile ? 1.3 : 1.8"
              stroke-linecap="round"
              stroke-dasharray="8 4"
              class="flowing-arc"
              :style="{ filter: `drop-shadow(0 0 4px ${line.color})` }"
            />
          </g>

          <!-- 7. 核心节点光点 -->
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
                :r="isMobile ? 4.2 : 5.5"
              />
              <circle
                class="node-dot"
                :fill="node.color"
                stroke="#ffffff"
                :stroke-width="isMobile ? '0.8px' : '1.2px'"
                :cx="node.x"
                :cy="node.y"
                :r="isMobile ? 2.5 : 3.2"
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
            :class="{ 'is-edge': node.isEdge, 'is-mobile': isMobile }"
            :style="{
              left: `${(node.x / width) * 100}%`,
              top: `${(node.y / height) * 100}%`,
              transform: `translate(calc(-50% + ${node.offsetX}px), calc(-50% + ${node.offsetY}px))`,
              color: node.color,
              borderColor: node.color,
              boxShadow: `0 0 10px color-mix(in srgb, ${node.color} 30%, transparent), 0 2px 6px rgba(0,0,0,0.6)`,
            }"
            @mouseenter="showTooltip(node, $event)"
            @mousemove="showTooltip(node, $event)"
            @click.stop="openServer(node.id)"
          >
            {{ node.displayName }}
          </div>
        </div>
      </div>

      <!-- 悬浮弹窗 -->
      <div
        v-if="tooltip"
        class="globe-tooltip"
        :class="{ 'is-mobile-tip': isMobile }"
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
  width: 100vw !important;
  max-width: 100vw !important;
  margin-left: calc(-50vw + 50%) !important;
  margin-right: calc(-50vw + 50%) !important;
  padding: 10px 24px 4px 24px;
  background: radial-gradient(circle at 50% 50%, rgba(13, 22, 44, 0.7) 0%, rgba(5, 10, 24, 0.96) 100%);
  border-radius: 0;
  overflow: hidden;
  user-select: none;
  box-sizing: border-box;
}

@media (max-width: 640px) {
  .globe-panel {
    padding: 10px 8px 4px 8px;
  }
}

.globe-viewport {
  position: relative;
  cursor: grab;
  width: 100% !important;
  display: block;
  touch-action: pan-y;
}

.globe-viewport:active {
  cursor: grabbing;
}

.globe-svg {
  width: 100% !important;
  height: 100% !important;
  display: block;
}

.globe-graticule path {
  fill: none;
  stroke: rgba(56, 189, 248, 0.12);
  stroke-width: 0.5;
}

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

.base-geo-text {
  fill: rgba(224, 242, 254, 0.35);
  font-weight: 600;
  letter-spacing: 0.5px;
  pointer-events: none;
  user-select: none;
}

:global([data-theme="light"]) .base-geo-text {
  fill: rgba(15, 23, 42, 0.45);
}

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
    r: 9;
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
  padding: 3px 8px;
  background: rgba(10, 20, 38, 0.92);
  border-radius: 4px;
  font-size: 11px;
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

.clear-city-tag.is-mobile {
  font-size: 9.5px;
  padding: 1.5px 5px;
  border-radius: 3px;
}

.clear-city-tag.is-edge {
  opacity: 0.55;
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
  gap: 6px;
  margin-bottom: 8px;
}

.globe-legend .hint {
  font-size: 11px;
  opacity: 0.65;
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
  min-width: 240px;
  padding: 10px 12px;
  border-radius: 8px;
  border: 1px solid var(--border-strong);
  background: color-mix(in srgb, var(--panel-solid) 92%, transparent);
  box-shadow: 0 12px 28px rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(14px);
  pointer-events: none;
}

.globe-tooltip.is-mobile-tip {
  min-width: 210px;
  padding: 8px 10px;
}

.tooltip-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding-bottom: 6px;
  margin-bottom: 6px;
  border-bottom: 1px solid var(--border);
}

.head-left {
  display: flex;
  align-items: center;
  gap: 6px;
}

.country-flag-img {
  width: 16px;
  height: 12px;
  border-radius: 2px;
  object-fit: cover;
}

.country-name {
  font-weight: 650;
  font-size: 12px;
}

.country-code {
  font-size: 10.5px;
  color: var(--text-faint);
}

.node-status.is-online {
  color: #10b981;
  font-size: 11px;
  font-weight: 600;
}

.node-status.is-offline {
  color: #ef4444;
  font-size: 11px;
  font-weight: 600;
}

.server-title {
  font-size: 12px;
  font-weight: 550;
  margin-bottom: 4px;
  color: var(--text);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.meta-row {
  display: flex;
  align-items: center;
  gap: 7px;
  font-size: 10.5px;
  color: var(--text-faint);
}

.meta-row .speed {
  color: #10b981;
}
</style>
