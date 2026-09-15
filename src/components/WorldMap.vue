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
const RADIUS = 230;

const ready = ref(false);
const failed = ref(false);
const landPaths = ref<string[]>([]);

/** 3D 球体旋转角度 [经度偏移, 纬度偏移] */
const rotation = ref<[number, number]>([-105, -32]); // 默认正面对准东亚/中国区
const isDragging = ref(false);
const startX = ref(0);
const startY = ref(0);
const startRot = ref<[number, number]>([-105, -32]);
let autoRotateTimer: number | null = null;
const isHovering = ref(false);

/** 常用拼音/英文城市转中文对照 */
const CITY_NAME_MAP: Record<string, string> = {
  Beijing: "北京",
  Shanghai: "上海",
  Guangzhou: "广州",
  Shenzhen: "深圳",
  Hangzhou: "杭州",
  Jinan: "济南",
  Chongqing: "重庆",
  Chengdu: "成都",
  Kunming: "昆明",
  Wuhan: "武汉",
  Nanjing: "南京",
  Xian: "西安",
  Urumqi: "乌鲁木齐",
  HongKong: "香港",
  Taipei: "台北",
  Tokyo: "东京",
  Osaka: "大阪",
  Seoul: "首尔",
  Singapore: "新加坡",
  London: "伦敦",
  Frankfurt: "法兰克福",
  "San Jose": "圣何塞",
  "Los Angeles": "洛杉矶",
  Auckland: "奥克兰",
  Sydney: "悉尼",
};

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

// eslint-disable-next-line @typescript-eslint/no-explicit-any
let projectionInstance: any = null;
// eslint-disable-next-line @typescript-eslint/no-explicit-any
let pathGenInstance: any = null;
// eslint-disable-next-line @typescript-eslint/no-explicit-any
let d3GeoModule: any = null;

interface GlobeNode {
  id: number;
  code: string;
  name: string;
  displayName: string;
  lng: number;
  lat: number;
  x: number;
  y: number;
  visible: boolean; // 是否在地球正面
  online: boolean;
  raw: PreparedServer;
}

interface ArcLine {
  id: string;
  pathD: string;
  color: string;
}

/** 智能推导城市显示名 */
function resolveNodeName(server: PreparedServer): string {
  const rawName = server.server.name || "";
  for (const [en, zh] of Object.entries(CITY_NAME_MAP)) {
    if (rawName.toLowerCase().includes(en.toLowerCase()) || rawName.includes(zh)) {
      return zh;
    }
  }
  const code = (server.server.country_code || "").trim().toUpperCase();
  return getCountryName(code);
}

/** 计算当前地球正面的所有节点与飞线 */
const globeData = computed(() => {
  if (!ready.value || !projectionInstance || !d3GeoModule) {
    return { nodes: [], lines: [], hub: null };
  }

  projectionInstance.rotate(rotation.value);

  const nodes: GlobeNode[] = [];
  const rot = rotation.value;

  for (const item of props.items) {
    const code = (item.server.country_code || "").trim().toUpperCase();
    if (!code) continue;
    const centroid = centroidOf(code);
    if (!centroid) continue;
    const [lat, lng] = centroid;

    const coords: [number, number] = [lng, lat];
    const point = projectionInstance(coords);
    if (!point) continue;

    // 正背面判定：点积角距小于 90 度为正面
    const distance = d3GeoModule.geoDistance(coords, [-rot[0], -rot[1]]);
    const visible = distance < Math.PI / 2;

    nodes.push({
      id: item.server.id,
      code,
      name: item.server.name,
      displayName: resolveNodeName(item),
      lng,
      lat,
      x: point[0],
      y: point[1],
      visible,
      online: item.online,
      raw: item,
    });
  }

  // 选定中心主控 Hub（优先寻找国内或香港节点，没有则默认取第一个在线节点）
  let hub = nodes.find((n) => ["CN", "HK", "TW"].includes(n.code) && n.online);
  if (!hub && nodes.length > 0) hub = nodes[0];

  // 生成通往 Hub 的贝塞尔立体弧线
  const lines: ArcLine[] = [];
  if (hub && hub.visible) {
    for (const node of nodes) {
      if (node.id === hub.id || !node.visible) continue;

      const sx = hub.x;
      const sy = hub.y;
      const ex = node.x;
      const ey = node.y;

      // 弧度高度：根据两点间距计算中点隆起弧度
      const dx = ex - sx;
      const dy = ey - sy;
      const dr = Math.hypot(dx, dy);
      const mx = (sx + ex) / 2 - dy * 0.28;
      const my = (sy + ey) / 2 + dx * 0.28;

      lines.push({
        id: `${hub.id}-${node.id}`,
        pathD: `M ${sx} ${sy} Q ${mx} ${my} ${ex} ${ey}`,
        color: node.online ? "rgba(56, 189, 248, 0.75)" : "rgba(248, 113, 113, 0.5)",
      });
    }
  }

  return { nodes, lines, hub };
});

/** 鼠标旋转与拖拽 */
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
  const sens = 0.28; // 旋转灵敏度
  rotation.value = [
    startRot.value[0] + dx * sens,
    Math.max(-75, Math.min(75, startRot.value[1] - dy * sens)),
  ];
}

function onMouseUp() {
  isDragging.value = false;
}

function resetView() {
  rotation.value = [-105, -32];
}

/** 弹窗提示 */
const tooltip = ref<{ x: number; y: number; node: GlobeNode } | null>(null);
function showTooltip(node: GlobeNode, event: MouseEvent) {
  const container = (event.currentTarget as HTMLElement).closest(".globe-panel");
  if (!container) return;
  const rect = container.getBoundingClientRect();
  tooltip.value = {
    x: Math.min(rect.width - 340, event.clientX - rect.left + 14),
    y: event.clientY - rect.top + 14,
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

    d3GeoModule = d3;
    const atlas = (atlasModule.default ?? atlasModule) as unknown as {
      objects: { countries: unknown };
    };
    const features = topojson.feature(
      atlas as never,
      atlas.objects.countries as never,
    ) as unknown as { features: unknown[] };

    // 创建 3D 正射球体投影
    projectionInstance = d3
      .geoOrthographic()
      .scale(RADIUS)
      .translate([WIDTH / 2, HEIGHT / 2])
      .clipAngle(90) // 自动裁切背部区域
      .rotate(rotation.value);

    pathGenInstance = d3.geoPath(projectionInstance);

    landPaths.value = features.features
      .map((f) => pathGenInstance(f as never))
      .filter((p): p is string => Boolean(p));

    ready.value = true;

    // 平滑低速自转
    autoRotateTimer = window.setInterval(() => {
      if (!isDragging.value && !isHovering.value) {
        rotation.value = [rotation.value[0] + 0.15, rotation.value[1]];
      }
    }, 40);
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
    <div v-if="failed" class="globe-state">3D 地图资源加载失败，请检查网络。</div>
    <div v-else-if="!ready" class="globe-state">
      <span class="spinner" />
      <span>正在构建 3D 立体网络地球…</span>
    </div>

    <template v-else>
      <div class="globe-legend">
        <span class="chip">
          <i class="dot dot--online" /> 在线 {{ items.filter((i) => i.online).length }}
        </span>
        <span class="chip">
          <i class="dot dot--offline" /> 离线 {{ items.filter((i) => !i.online).length }}
        </span>
        <span class="chip hint">按住鼠标左键可 360° 旋转地球</span>
        <button type="button" class="chip reset-btn" @click="resetView">复位中心</button>
      </div>

      <div
        class="globe-viewport"
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
            <!-- 球体光晕与渐变底色 -->
            <radialGradient id="globeAtmosphere" cx="50%" cy="50%" r="50%">
              <stop offset="70%" stop-color="#1e293b" stop-opacity="0.8" />
              <stop offset="96%" stop-color="#0f172a" stop-opacity="0.95" />
              <stop offset="100%" stop-color="#38bdf8" stop-opacity="0.25" />
            </radialGradient>
            <!-- 边缘立体外发光阴影 -->
            <filter id="glowEffect" x="-20%" y="-20%" width="140%" height="140%">
              <feGaussianBlur stdDeviation="3" result="blur" />
              <feComposite in="SourceGraphic" in2="blur" operator="over" />
            </filter>
          </defs>

          <!-- 1. 背景球体与大气边缘 -->
          <circle
            :cx="WIDTH / 2"
            :cy="HEIGHT / 2"
            :r="RADIUS"
            fill="url(#globeAtmosphere)"
            stroke="rgba(56, 189, 248, 0.3)"
            stroke-width="1.5"
          />

          <!-- 2. 陆地板快 -->
          <g class="globe-land">
            <path
              v-for="(path, index) in landPaths"
              :key="index"
              :d="pathGenInstance ? pathGenInstance(path as never) : path"
            />
          </g>

          <!-- 3. 动态弧形飞线 -->
          <g class="globe-lines">
            <path
              v-for="line in globeData.lines"
              :key="line.id"
              :d="line.pathD"
              fill="none"
              :stroke="line.color"
              stroke-width="1.2"
              stroke-linecap="round"
              stroke-dasharray="4 2"
              class="fly-line"
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
              <!-- 节点光晕 -->
              <circle
                class="node-pulse"
                :class="node.online ? 'is-online' : 'is-offline'"
                :cx="node.x"
                :cy="node.y"
                :r="6"
              />
              <!-- 节点中心实体 -->
              <circle
                class="node-core"
                :class="node.online ? 'is-online' : 'is-offline'"
                :cx="node.x"
                :cy="node.y"
                :r="3.2"
              />

              <!-- 精致白色中文胶囊标签 -->
              <g class="node-tag" :transform="`translate(${node.x}, ${node.y - 12})`">
                <rect
                  rx="4"
                  ry="4"
                  x="-22"
                  y="-11"
                  width="44"
                  height="16"
                  class="tag-bg"
                />
                <text class="tag-text" y="1" text-anchor="middle" dominant-baseline="central">
                  {{ node.displayName }}
                </text>
              </g>
            </g>
          </g>
        </svg>
      </div>

      <!-- 悬停详情浮窗 -->
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
  border-radius: 14px;
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

/* 陆地 */
.globe-land path {
  fill: #334155;
  stroke: #475569;
  stroke-width: 0.6;
  opacity: 0.85;
}

:global([data-theme="light"]) .globe-land path {
  fill: #cbd5e1;
  stroke: #94a3b8;
}

/* 飞线动效 */
.fly-line {
  animation: lineFlow 1.8s linear infinite;
}

@keyframes lineFlow {
  from {
    stroke-dashoffset: 24;
  }
  to {
    stroke-dashoffset: 0;
  }
}

/* 节点 */
.globe-node {
  cursor: pointer;
}

.node-core.is-online {
  fill: #38bdf8;
  stroke: #ffffff;
  stroke-width: 1px;
}

.node-core.is-offline {
  fill: #f87171;
  stroke: #ffffff;
  stroke-width: 1px;
}

.node-pulse {
  opacity: 0.45;
}

.node-pulse.is-online {
  fill: #38bdf8;
  animation: globePulse 2s infinite ease-out;
}

.node-pulse.is-offline {
  fill: #f87171;
}

@keyframes globePulse {
  0% {
    r: 3.5;
    opacity: 0.6;
  }
  100% {
    r: 10;
    opacity: 0;
  }
}

/* 仿截图的胶囊标签 */
.node-tag {
  pointer-events: none;
  filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.45));
}

.tag-bg {
  fill: rgba(255, 255, 255, 0.95);
  stroke: rgba(203, 213, 225, 0.8);
  stroke-width: 0.5;
}

:global([data-theme="dark"]) .tag-bg {
  fill: rgba(30, 41, 59, 0.92);
  stroke: rgba(71, 85, 105, 0.8);
}

.tag-text {
  fill: #0f172a;
  font-size: 10px;
  font-weight: 600;
  font-family: system-ui, -apple-system, sans-serif;
}

:global([data-theme="dark"]) .tag-text {
  fill: #f8fafc;
}

/* 状态与图例 */
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

/* 悬浮面板 */
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
  color: var(--ok, #10b981);
  font-size: 11.5px;
  font-weight: 600;
}

.node-status.is-offline {
  color: var(--danger, #ef4444);
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
  color: var(--ok, #10b981);
}
</style>
