<template>
  <div class="fleet-page">

    <!-- ── Topbar ─────────────────────────────────────────── -->
    <header class="topbar">
      <div class="topbar-brand">
        <div class="brand-icon">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8">
            <rect x="2" y="2" width="20" height="8" rx="2"/>
            <rect x="2" y="14" width="20" height="8" rx="2"/>
            <line x1="6" y1="6" x2="6.01" y2="6" stroke-width="3" stroke-linecap="round"/>
            <line x1="6" y1="18" x2="6.01" y2="18" stroke-width="3" stroke-linecap="round"/>
          </svg>
        </div>
        <div>
          <div class="brand-title">Control Center</div>
          <div class="brand-sub">HPE iLO 5 · <span class="brand-count">{{ servers.length }} nodos</span></div>
        </div>
      </div>

      <div class="topbar-right">
        <div class="live-pill">
          <span class="live-dot"></span>
          LIVE
        </div>
        <div class="countdown-pill" v-if="countdownSec > 0">
          <div class="cd-ring-wrap">
            <svg viewBox="0 0 28 28" class="cd-ring">
              <circle cx="14" cy="14" r="11" class="cd-bg"/>
              <circle cx="14" cy="14" r="11" class="cd-fill" :stroke-dasharray="`${countdownPct * 69.1 / 100}, 69.1`" transform="rotate(-90 14 14)"/>
            </svg>
          </div>
          <span>Refresco en {{ countdownMin }}</span>
        </div>
        <span class="last-refresh" v-if="lastRefresh">sync {{ lastRefresh }}</span>

        <button class="btn" @click="$emit('open-reports')">
          <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/>
            <line x1="16" y1="13" x2="8" y2="13"/><line x1="16" y1="17" x2="8" y2="17"/>
          </svg>
          Reportes
        </button>
        <button class="btn" @click="reloadAll" :disabled="loading">
          <span :class="{ spin: loading }">↻</span> Actualizar
        </button>
        <button class="btn btn--primary" @click="showAdd = true">
          <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
            <line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/>
          </svg>
          Servidor
        </button>
      </div>
    </header>

    <!-- ── Hero Section ───────────────────────────────────── -->
    <section class="hero-section" v-if="servers.length > 0">

      <!-- Status banner if anything is wrong -->
      <div class="alert-banner" v-if="stats.crit > 0">
        <span class="ab-dot"></span>
        <span>{{ stats.crit }} servidor(es) en estado <strong>Crítico</strong> — Requiere atención inmediata</span>
      </div>
      <div class="warn-banner" v-else-if="stats.warn > 0">
        <span class="wb-dot"></span>
        <span>{{ stats.warn }} servidor(es) con <strong>Advertencias</strong> activas</span>
      </div>

      <!-- Executive Status Ticker -->
      <div class="kpi-ticker">
        <!-- Compact Status Ring -->
        <div class="kpi-ring-container">
          <Doughnut :data="chartData" :options="chartOptions" />
          <div class="kpi-ring-inner">
             <span class="kr-dot" :class="overallCls"></span>
          </div>
        </div>

        <div class="ticker-items">
          <div class="ticker-item ks--blue">
            <div class="ti-icon"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M12 2v20M2 12h20"/><rect x="4" y="4" width="16" height="16" rx="2"/></svg></div>
            <div class="ti-data">
              <span class="ti-lbl">Flota Total</span>
              <span class="ti-val">{{ servers.length }}</span>
            </div>
          </div>
          
          <div class="ticker-divider"></div>

          <div class="ticker-item ks--green">
            <div class="ti-icon"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"/><polyline points="22 4 12 14.01 9 11.01"/></svg></div>
            <div class="ti-data">
              <span class="ti-lbl">Óptimos</span>
              <span class="ti-val tc-ok">{{ stats.ok }}</span>
            </div>
          </div>

          <div class="ticker-divider"></div>

          <div class="ticker-item ks--amber">
            <div class="ti-icon"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z"/><line x1="12" y1="9" x2="12" y2="13"/><line x1="12" y1="17" x2="12.01" y2="17"/></svg></div>
            <div class="ti-data">
              <span class="ti-lbl">Advertencia</span>
              <span class="ti-val" :class="stats.warn > 0 ? 'tc-warn' : ''">{{ stats.warn }}</span>
            </div>
          </div>

          <div class="ticker-divider"></div>

          <div class="ticker-item ks--red" :class="{ 'ti-alert': stats.crit > 0 }">
            <div class="ti-icon"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="12" r="10"/><line x1="15" y1="9" x2="9" y2="15"/><line x1="9" y1="9" x2="15" y2="15"/></svg></div>
            <div class="ti-data">
              <span class="ti-lbl">Críticos</span>
              <span class="ti-val" :class="stats.crit > 0 ? 'tc-crit' : ''">{{ stats.crit }}</span>
            </div>
          </div>

          <div class="ticker-divider"></div>

          <div class="ticker-item ks--gray">
            <div class="ti-icon"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M18.36 6.64a9 9 0 1 1-12.73 0"/><line x1="12" y1="2" x2="12" y2="12"/></svg></div>
            <div class="ti-data">
              <span class="ti-lbl">Desconectados</span>
              <span class="ti-val">{{ stats.off }}</span>
            </div>
          </div>
        </div>
      </div>
    </section>

    <!-- ── Filter / Search bar ────────────────────────────── -->
    <div class="filter-bar" v-if="servers.length > 0">
      <div class="search-wrap">
        <svg class="search-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <circle cx="11" cy="11" r="8"/><line x1="21" y1="21" x2="16.65" y2="16.65"/>
        </svg>
        <input v-model="searchQ" class="search-input" placeholder="Buscar servidor…" />
      </div>
      <div class="filter-chips">
        <button
          v-for="f in filters" :key="f.key"
          class="fchip"
          :class="{ active: activeFilter === f.key, [`fchip--${f.cls}`]: activeFilter === f.key }"
          @click="activeFilter = activeFilter === f.key ? 'all' : f.key"
        >
          <span class="fchip-dot" :class="`fd--${f.cls}`"></span>
          {{ f.label }}
          <span class="fchip-count">{{ f.count }}</span>
        </button>
      </div>
      <div class="grid-toggle">
        <button class="gt-btn" :class="{ active: gridMode === 'grid' }" @click="gridMode = 'grid'" title="Vista cuadrícula">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" width="14" height="14">
            <rect x="3" y="3" width="7" height="7"/><rect x="14" y="3" width="7" height="7"/>
            <rect x="3" y="14" width="7" height="7"/><rect x="14" y="14" width="7" height="7"/>
          </svg>
        </button>
        <button class="gt-btn" :class="{ active: gridMode === 'list' }" @click="gridMode = 'list'" title="Vista lista">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" width="14" height="14">
            <line x1="8" y1="6" x2="21" y2="6"/><line x1="8" y1="12" x2="21" y2="12"/>
            <line x1="8" y1="18" x2="21" y2="18"/>
            <circle cx="3" cy="6" r="1" fill="currentColor"/><circle cx="3" cy="12" r="1" fill="currentColor"/><circle cx="3" cy="18" r="1" fill="currentColor"/>
          </svg>
        </button>
      </div>
    </div>

    <!-- ── Empty State ────────────────────────────────────── -->
    <div class="empty-state" v-if="!loading && servers.length === 0">
      <div class="empty-glow"></div>
      <div class="empty-icon">
        <svg viewBox="0 0 24 24" fill="none" stroke-width="1.5">
          <rect x="2" y="2" width="20" height="8" rx="2"/><rect x="2" y="14" width="20" height="8" rx="2"/>
          <line x1="6" y1="6" x2="6.01" y2="6"/><line x1="6" y1="18" x2="6.01" y2="18"/>
        </svg>
      </div>
      <div class="empty-title">Sin infraestructura monitoreada</div>
      <div class="empty-sub">Conecta tus primeros servidores HPE iLO 5 para comenzar.</div>
      <button class="btn btn--primary btn--lg" @click="showAdd = true">+ Añadir Servidor</button>
    </div>

    <!-- ── Server Grid ────────────────────────────────────── -->
    <main class="fleet-grid" :class="{ 'fleet-list': gridMode === 'list' }" v-else>
      <div class="no-results" v-if="filteredServers.length === 0">
        Sin resultados para "{{ searchQ }}"
      </div>
      <ServerCard
        v-for="srv in filteredServers"
        :key="srv.id"
        :server="srv"
        :list-mode="gridMode === 'list'"
        :ref="el => { if (el) cardRefs[srv.id] = el }"
        @select="$emit('open-detail', $event)"
        @status="updateStatus(srv.id, $event)"
        @deleted="onServerDeleted"
      />
    </main>

    <!-- ── Add Modal ──────────────────────────────────────── -->
    <AddServerModal v-if="showAdd" @close="showAdd = false" @added="onServerAdded" />
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'
import ServerCard     from '../components/ServerCard.vue'
import AddServerModal from '../components/AddServerModal.vue'
import { useIlo }     from '../composables/useIlo.js'
import { REFRESH_INTERVAL_SEC } from '../config/servers.js'
import { Doughnut } from 'vue-chartjs'
import { Chart as ChartJS, ArcElement, Tooltip, Legend } from 'chart.js'

ChartJS.register(ArcElement, Tooltip, Legend)

const props = defineProps({
  refreshCount: Number
})
defineEmits(['open-detail', 'open-reports'])

const { getServers } = useIlo()
const servers    = ref([])
const cardRefs   = ref({})
const showAdd    = ref(false)
const lastRefresh= ref('')
const loading    = ref(true)
const statusMap  = ref({})
const searchQ    = ref('')
const gridMode   = ref('grid')
const activeFilter = ref('all')

let cdTimer = null
const countdownSec = ref(REFRESH_INTERVAL_SEC)

const countdownPct = computed(() => {
  const t = REFRESH_INTERVAL_SEC
  return ((t - countdownSec.value) / t) * 100
})
const countdownMin = computed(() => {
  return `${countdownSec.value}s`
})

const stats = computed(() => {
  const s = { ok: 0, warn: 0, crit: 0, off: 0, unknown: 0 }
  servers.value.forEach(srv => {
    const st = statusMap.value[srv.id]
    if      (st === 'ok')      s.ok++
    else if (st === 'warn')    s.warn++
    else if (st === 'crit')    s.crit++
    else if (st === 'off')     s.off++
    else                       s.unknown++
  })
  return s
})

const overallCls = computed(() => {
  if (stats.value.crit > 0) return 'nc--crit'
  if (stats.value.warn > 0) return 'nc--warn'
  return 'nc--ok'
})

const filters = computed(() => [
  { key: 'ok',   label: 'Óptimos',      cls: 'green', count: stats.value.ok },
  { key: 'warn', label: 'Advertencia',  cls: 'amber', count: stats.value.warn },
  { key: 'crit', label: 'Críticos',     cls: 'red',   count: stats.value.crit },
  { key: 'off',  label: 'Offline',      cls: 'gray',  count: stats.value.off },
])

const filteredServers = computed(() => {
  let list = servers.value
  if (activeFilter.value !== 'all') {
    list = list.filter(srv => statusMap.value[srv.id] === activeFilter.value)
  }
  if (searchQ.value.trim()) {
    const q = searchQ.value.toLowerCase()
    list = list.filter(srv =>
      srv.label.toLowerCase().includes(q) || srv.host.toLowerCase().includes(q)
    )
  }
  return list
})

function pct(n) {
  return servers.value.length ? Math.round((n / servers.value.length) * 100) : 0
}

const chartData = computed(() => {
  const { ok, warn, crit, off } = stats.value
  if (ok === 0 && warn === 0 && crit === 0 && off === 0) {
    return { labels: ['Escaneando…'], datasets: [{ data: [1], backgroundColor: ['rgba(255,255,255,0.04)'], borderWidth: 0 }] }
  }
  return {
    labels: ['Óptimos', 'Advertencia', 'Críticos', 'Offline'],
    datasets: [{
      data: [ok, warn, crit, off],
      backgroundColor: ['#2EB253', '#F5A623', '#E03535', '#475569'],
      hoverBackgroundColor: ['#38d467', '#ffc13d', '#f54545', '#64748b'],
      borderWidth: 0, hoverOffset: 8,
    }]
  }
})

const chartOptions = {
  responsive: true, maintainAspectRatio: false, cutout: '78%',
  plugins: {
    legend: { display: false },
    tooltip: { enabled: true }
  }
}

async function loadServers() {
  loading.value = true
  try { servers.value = await getServers() }
  catch (e) { console.error(e) }
  finally { loading.value = false }
}

function onServerAdded(srv) { showAdd.value = false; servers.value.push(srv) }
function onServerDeleted(id) {
  servers.value = servers.value.filter(s => s.id !== id)
  delete statusMap.value[id]; delete cardRefs.value[id]
}
function updateStatus(id, status) { statusMap.value[id] = status }
function reloadAll() {
  Object.values(cardRefs.value).forEach(c => c?.reload?.())
  lastRefresh.value = new Date().toLocaleTimeString('es-EC', { hour: '2-digit', minute: '2-digit' })
  countdownSec.value = REFRESH_INTERVAL_SEC
}

onMounted(() => {
  loadServers()
  lastRefresh.value = new Date().toLocaleTimeString('es-EC', { hour: '2-digit', minute: '2-digit' })
  cdTimer = setInterval(() => { countdownSec.value = Math.max(0, countdownSec.value - 1) }, 1000)
})

// Auto-reload cuando llega una señal de App.vue (via SocketIO)
watch(() => props.refreshCount, () => {
  console.log('[FleetView] Auto-refresh trigger recibido')
  reloadAll()
})

onUnmounted(() => clearInterval(cdTimer))
</script>

<style scoped>
/* ── Base ── */
.fleet-page { display: flex; flex-direction: column; min-height: 100vh; background: var(--page); }

/* ── Topbar ── */
.topbar {
  background: rgba(255, 255, 255, 0.85); border-bottom: 1px solid var(--border);
  padding: 0 28px; height: 64px;
  display: flex; align-items: center; justify-content: space-between;
  position: sticky; top: 0; z-index: 100; backdrop-filter: blur(20px);
  gap: 16px;
}
.topbar-brand { display: flex; align-items: center; gap: 14px; }
.brand-icon {
  width: 40px; height: 40px; border-radius: 10px; flex-shrink: 0;
  background: rgba(62,146,239,0.12); border: 1px solid rgba(62,146,239,0.25);
  display: flex; align-items: center; justify-content: center;
  box-shadow: 0 0 20px rgba(62,146,239,0.1);
}
.brand-icon svg { width: 20px; height: 20px; stroke: var(--blue-400); }
.brand-title { font-size: 17px; font-weight: 700; color: var(--text); letter-spacing: -0.01em; }
.brand-sub   { font-size: 11px; color: var(--text-4); margin-top: 1px; }
.brand-count { color: var(--blue-400); font-weight: 600; }

.topbar-right { display: flex; align-items: center; gap: 10px; }
.countdown-pill {
  display: flex; align-items: center; gap: 6px;
  font-size: 11px; font-weight: 600; color: var(--text-3);
  background: rgba(0,0,0,0.05); border: 1px solid var(--border);
  padding: 4px 10px 4px 6px; border-radius: 999px;
}
.cd-ring-wrap { width: 20px; height: 20px; }
.cd-ring { width: 100%; height: 100%; }
.cd-bg   { fill: none; stroke: rgba(0,0,0,0.08); stroke-width: 2.5; }
.cd-fill { fill: none; stroke: var(--blue-400); stroke-width: 2.5; stroke-linecap: round; }
.last-refresh { font-size: 10px; color: var(--text-4); font-family: 'JetBrains Mono'; }

/* LIVE dot */
.live-pill {
  display: flex; align-items: center; gap: 6px;
  background: rgba(46,178,83,0.1); border: 1px solid rgba(46,178,83,0.2);
  padding: 4px 10px; border-radius: 999px;
  font-size: 9px; font-weight: 800; color: var(--green-600);
  letter-spacing: 0.05em;
}
.live-dot {
  width: 6px; height: 6px; border-radius: 50%;
  background: var(--green-600);
  box-shadow: 0 0 8px var(--green-600);
  animation: heart-beat 1.5s infinite;
}
@keyframes heart-beat { 0%, 100% { transform: scale(1); opacity: 1; } 50% { transform: scale(1.3); opacity: 0.5; } }

/* ── Hero ── */
.hero-section { padding: 16px 28px 0px; }

/* ── Professional Monitoring Ticker (Improved) ── */
.kpi-ticker {
  display: flex; align-items: center; gap: 24px;
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 14px; padding: 14px 32px;
  box-shadow: 0 10px 40px rgba(0,0,0,0.05);
  backdrop-filter: blur(20px);
}

.kpi-ring-container { position: relative; width: 42px; height: 42px; flex-shrink: 0; }
.kpi-ring-inner {
  position: absolute; top: 0; left: 0; width: 100%; height: 100%;
  display: flex; align-items: center; justify-content: center;
}
.kr-dot { width: 8px; height: 8px; border-radius: 50%; }
.kr-dot.nc--ok { background: var(--green-600); box-shadow: 0 0 15px var(--green-600); }
.kr-dot.nc--warn { background: var(--amber-600); box-shadow: 0 0 15px var(--amber-600); }
.kr-dot.nc--crit { background: var(--red-600); box-shadow: 0 0 15px var(--red-600); }

.ticker-items { display: flex; align-items: center; flex: 1; justify-content: space-between; }
.ticker-divider { width: 1px; height: 32px; background: var(--border); margin: 0 15px; }

.ticker-item { display: flex; align-items: center; gap: 14px; }
.ti-icon { width: 22px; height: 22px; transition: transform 0.3s; }
.ti-icon svg { width: 100%; height: 100%; }

.ti-data { display: flex; flex-direction: column; gap: 2px; }
.ti-lbl { font-size: 10px; font-weight: 700; color: #94a3b8; text-transform: uppercase; letter-spacing: 0.1em; line-height: 1; }
.ti-val { font-size: 24px; font-weight: 800; font-family: 'JetBrains Mono', monospace; color: var(--text); line-height: 1; }

/* Status specific colors for Icons and Values */
.ks--blue  .ti-icon { color: var(--blue-400); }
.ks--blue  .ti-val  { color: var(--blue-600); }

.ks--green .ti-icon { color: var(--green-600); }
.ks--green .ti-val  { color: var(--green-600); }

.ks--amber .ti-icon { color: var(--amber-600); }
.ks--amber .ti-val  { color: var(--amber-600); text-shadow: 0 0 20px rgba(245,166,35,0.1); }

.ks--red   .ti-icon { color: var(--red-600); }
.ks--red   .ti-val  { color: var(--red-600); text-shadow: 0 0 20px rgba(224,53,53,0.2); }

.ks--gray  .ti-icon { color: #64748b; }
.ks--gray  .ti-val  { color: #94a3b8; }

.ti-alert .ti-icon { animation: ti-pulse 1.5s infinite; }
@keyframes ti-pulse { 0%, 100% { transform: scale(1); opacity: 1; } 50% { transform: scale(1.1); opacity: 0.6; } }

@media (max-width: 1024px) {
  .ticker-items { flex-wrap: wrap; gap: 20px; }
  .ticker-divider { display: none; }
  .kpi-ticker { padding: 16px 20px; }
}

@media (max-width: 640px) {
  .ti-lbl { display: none; }
}

/* ── Filter Bar ── */
.filter-bar {
  display: flex; align-items: center; gap: 12px; flex-wrap: wrap;
  padding: 14px 28px; border-bottom: 1px solid var(--border-light);
}
.search-wrap { position: relative; }
.search-icon { position: absolute; left: 10px; top: 50%; transform: translateY(-50%); width: 14px; height: 14px; stroke: var(--text-4); pointer-events: none; }
.search-input {
  background: rgba(0,0,0,0.03); border: 1px solid var(--border); border-radius: 8px;
  padding: 7px 12px 7px 32px; font-size: 13px; color: var(--text-2); outline: none;
  width: 220px; transition: border-color .2s, width .3s; font-family: inherit;
}
.search-input::placeholder { color: var(--text-4); }
.search-input:focus { border-color: rgba(62,146,239,0.4); background: rgba(62,146,239,0.04); width: 280px; }

.filter-chips { display: flex; gap: 6px; }
.fchip {
  display: flex; align-items: center; gap: 6px;
  padding: 5px 12px; border-radius: 999px; font-size: 11px; font-weight: 600;
  background: rgba(0,0,0,0.03); border: 1px solid var(--border); color: var(--text-3);
  cursor: pointer; transition: all .2s;
}
.fchip:hover { background: rgba(0,0,0,0.06); color: var(--text-2); }
.fchip.active { color: var(--text); }
.fchip.fchip--green { background: rgba(46,178,83,0.12);  border-color: rgba(46,178,83,0.3);  color: var(--green-600); }
.fchip.fchip--amber { background: rgba(245,166,35,0.12); border-color: rgba(245,166,35,0.3); color: var(--amber-600); }
.fchip.fchip--red   { background: rgba(224,53,53,0.12);  border-color: rgba(224,53,53,0.3);  color: var(--red-600); }
.fchip.fchip--gray  { background: rgba(148,163,184,0.1); border-color: rgba(148,163,184,0.2);color: var(--gray-500); }
.fchip-dot  { width: 6px; height: 6px; border-radius: 50%; }
.fd--green  { background: var(--green-600); box-shadow: 0 0 4px var(--green-600); }
.fd--amber  { background: var(--amber-600); box-shadow: 0 0 4px var(--amber-600); }
.fd--red    { background: var(--red-600);   box-shadow: 0 0 4px var(--red-600); }
.fd--gray   { background: var(--gray-400); }
.fchip-count { background: rgba(0,0,0,0.06); padding: 1px 6px; border-radius: 999px; font-size: 10px; }

.grid-toggle { display: flex; gap: 2px; margin-left: auto; background: rgba(0,0,0,0.03); border: 1px solid var(--border); border-radius: 8px; padding: 3px; }
.gt-btn { width: 28px; height: 28px; display: flex; align-items: center; justify-content: center; border-radius: 5px; border: none; background: transparent; color: var(--text-4); cursor: pointer; transition: all .2s; }
.gt-btn.active { background: rgba(62,146,239,0.15); color: var(--blue-400); }
.gt-btn:hover:not(.active) { background: rgba(0,0,0,0.05); color: var(--text-2); }

/* ── Server Grid ── */
.fleet-grid {
  flex: 1; padding: 24px 28px;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 18px; align-items: start;
}
.fleet-list {
  grid-template-columns: 1fr;
  gap: 10px;
}
.no-results { grid-column: 1/-1; text-align: center; color: var(--text-4); padding: 60px; font-size: 14px; }

/* ── Empty State ── */
.empty-state {
  flex: 1; display: flex; flex-direction: column; align-items: center; justify-content: center;
  gap: 16px; padding: 80px 24px; text-align: center; position: relative;
}
.empty-glow { position: absolute; width: 400px; height: 400px; border-radius: 50%; background: radial-gradient(circle, rgba(62,146,239,0.05) 0%, transparent 70%); pointer-events: none; }
.empty-icon {
  width: 80px; height: 80px; border-radius: 22px;
  background: rgba(62,146,239,0.08); border: 1px solid rgba(62,146,239,0.2);
  display: flex; align-items: center; justify-content: center;
  box-shadow: 0 0 40px rgba(62,146,239,0.12);
}
.empty-icon svg { width: 36px; height: 36px; stroke: var(--blue-400); }
.empty-title { font-size: 22px; font-weight: 700; color: var(--text); }
.empty-sub   { font-size: 14px; color: var(--text-3); max-width: 340px; line-height: 1.5; }
.btn--lg { padding: 10px 24px; font-size: 14px; }

/* ── Responsive ── */
@media (max-width: 1200px) { .kpi-stats { grid-template-columns: repeat(3, 1fr); } }
@media (max-width: 1024px) {
  .kpi-row { grid-template-columns: 1fr; }
  .kpi-chart-wrap { height: 160px; }
}
@media (max-width: 768px)  {
  .kpi-stats { grid-template-columns: repeat(2, 1fr); }
  .fleet-grid { padding: 16px; gap: 12px; }
  .hero-section { padding: 16px 16px 0; }
  .filter-bar { padding: 12px 16px; }
}
@media (max-width: 640px)  {
  .topbar { padding: 0 16px; height: auto; padding-top: 12px; padding-bottom: 12px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .topbar-right { align-self: stretch; justify-content: space-between; }
  .fleet-grid { grid-template-columns: 1fr; }
  .kpi-stats { grid-template-columns: 1fr 1fr; }
  .filter-chips { display: none; }
}
</style>