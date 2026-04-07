<template>
  <div class="fleet-page">

    <!-- ── Topbar ── -->
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
          <span class="live-dot"></span>LIVE
        </div>

        <div class="countdown-pill" v-if="countdownSec > 0">
          <svg viewBox="0 0 28 28" width="18" height="18">
            <circle cx="14" cy="14" r="11" fill="none" stroke="#ddd8d0" stroke-width="2.5"/>
            <circle cx="14" cy="14" r="11" fill="none" stroke="#1a8a7a" stroke-width="2.5"
              stroke-linecap="round"
              :stroke-dasharray="`${countdownPct * 69.1 / 100} 69.1`"
              transform="rotate(-90 14 14)"/>
          </svg>
          <span>{{ countdownSec }}s</span>
        </div>

        <span class="last-refresh" v-if="lastRefresh">↺ {{ lastRefresh }}</span>

        <button class="btn-icon" @click="$emit('open-reports')" title="Reportes">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/>
            <polyline points="14 2 14 8 20 8"/>
            <line x1="16" y1="13" x2="8" y2="13"/><line x1="16" y1="17" x2="8" y2="17"/>
          </svg>
          Reportes
        </button>

        <button class="btn-icon" @click="reloadAll" :disabled="loading">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" :class="{ spin: loading }">
            <polyline points="23 4 23 10 17 10"/>
            <path d="M20.49 15a9 9 0 1 1-.09-3.7"/>
          </svg>
          Actualizar
        </button>

        <button class="btn-primary" @click="showAdd = true">
          <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
            <line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/>
          </svg>
          Servidor
        </button>
      </div>
    </header>

    <!-- ── Alert banners ── -->
    <div class="alert-crit" v-if="stats.crit > 0">
      <span class="alert-led"></span>
      <span>{{ stats.crit }} servidor{{ stats.crit > 1 ? 'es' : '' }} en estado <strong>Crítico</strong> — Requiere atención inmediata</span>
    </div>
    <div class="alert-warn" v-else-if="stats.warn > 0">
      <span class="alert-led alert-led--warn"></span>
      <span>{{ stats.warn }} servidor{{ stats.warn > 1 ? 'es' : '' }} con <strong>Advertencias</strong> activas</span>
    </div>

    <!-- ── KPI Bar ── -->
    <section class="kpi-bar" v-if="servers.length > 0">
      <!-- Donut ring -->
      <div class="kpi-donut-wrap">
        <Doughnut :data="chartData" :options="chartOptions" />
        <div class="kpi-donut-center">
          <span class="kdc-pct">{{ healthPct }}<small>%</small></span>
          <span class="kdc-lbl">OK</span>
        </div>
      </div>

      <div class="kpi-divider"></div>

      <!-- Stats -->
      <div class="kpi-stats">
        <div class="kpi-stat">
          <span class="ks-val ks--total">{{ servers.length }}</span>
          <span class="ks-lbl">Flota total</span>
        </div>
        <div class="kpi-stat">
          <span class="ks-val ks--ok">{{ stats.ok }}</span>
          <span class="ks-lbl">Óptimos</span>
        </div>
        <div class="kpi-stat">
          <span class="ks-val ks--warn" :class="{ 'ks-blink': stats.warn > 0 }">{{ stats.warn }}</span>
          <span class="ks-lbl">Advertencia</span>
        </div>
        <div class="kpi-stat">
          <span class="ks-val ks--crit" :class="{ 'ks-blink': stats.crit > 0 }">{{ stats.crit }}</span>
          <span class="ks-lbl">Críticos</span>
        </div>
        <div class="kpi-stat">
          <span class="ks-val ks--off">{{ stats.off }}</span>
          <span class="ks-lbl">Offline</span>
        </div>
      </div>

      <div class="kpi-divider"></div>

      <!-- Fleet health bar -->
      <div class="kpi-health-col">
        <div class="khc-label">Estado de la flota</div>
        <div class="health-bar-track">
          <div class="health-bar-seg hbs--ok"   :style="{ width: pct(stats.ok)   + '%' }" :title="`${stats.ok} óptimos`"></div>
          <div class="health-bar-seg hbs--warn"  :style="{ width: pct(stats.warn) + '%' }" :title="`${stats.warn} advertencias`"></div>
          <div class="health-bar-seg hbs--crit"  :style="{ width: pct(stats.crit) + '%' }" :title="`${stats.crit} críticos`"></div>
          <div class="health-bar-seg hbs--off"   :style="{ width: pct(stats.off)  + '%' }" :title="`${stats.off} offline`"></div>
        </div>
        <div class="health-bar-legend">
          <span class="hbl-item"><span class="hbl-dot hbl--ok"></span>OK</span>
          <span class="hbl-item"><span class="hbl-dot hbl--warn"></span>Warn</span>
          <span class="hbl-item"><span class="hbl-dot hbl--crit"></span>Crit</span>
          <span class="hbl-item"><span class="hbl-dot hbl--off"></span>Off</span>
        </div>
      </div>


    </section>

    <!-- ── Filter / Search bar ── -->
    <div class="filter-bar" v-if="servers.length > 0">
      <div class="search-wrap">
        <svg class="search-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" width="14" height="14">
          <circle cx="11" cy="11" r="8"/><line x1="21" y1="21" x2="16.65" y2="16.65"/>
        </svg>
        <input v-model="searchQ" class="search-input" placeholder="Buscar por nombre o IP…" />
        <button class="search-clear" v-if="searchQ" @click="searchQ = ''">×</button>
      </div>

      <div class="filter-chips">
        <button
          v-for="f in filters" :key="f.key"
          class="fchip" :class="{ active: activeFilter === f.key, [`fchip--${f.cls}`]: activeFilter === f.key }"
          @click="activeFilter = activeFilter === f.key ? 'all' : f.key"
        >
          <span class="fchip-dot" :class="`fd--${f.cls}`"></span>
          {{ f.label }}
          <span class="fchip-count">{{ f.count }}</span>
        </button>
      </div>

      <!-- Sort -->
      <div class="sort-wrap">
        <select v-model="sortBy" class="sort-select">
          <option value="name">Nombre A–Z</option>
          <option value="status">Estado</option>
          <option value="temp">Temperatura</option>
          <option value="watts">Consumo</option>
        </select>
      </div>

      <!-- Grid/List toggle -->
      <div class="grid-toggle">
        <button class="gt-btn" :class="{ active: gridMode === 'grid' }" @click="gridMode = 'grid'" title="Cuadrícula">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" width="14" height="14">
            <rect x="3" y="3" width="7" height="7"/><rect x="14" y="3" width="7" height="7"/>
            <rect x="3" y="14" width="7" height="7"/><rect x="14" y="14" width="7" height="7"/>
          </svg>
        </button>
        <button class="gt-btn" :class="{ active: gridMode === 'list' }" @click="gridMode = 'list'" title="Lista">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" width="14" height="14">
            <line x1="8" y1="6" x2="21" y2="6"/><line x1="8" y1="12" x2="21" y2="12"/>
            <line x1="8" y1="18" x2="21" y2="18"/>
            <circle cx="3" cy="6" r="1" fill="currentColor"/>
            <circle cx="3" cy="12" r="1" fill="currentColor"/>
            <circle cx="3" cy="18" r="1" fill="currentColor"/>
          </svg>
        </button>
      </div>

      <span class="result-count" v-if="searchQ || activeFilter !== 'all'">
        {{ filteredServers.length }} de {{ servers.length }}
      </span>
    </div>

    <!-- ── Empty State ── -->
    <div class="empty-state" v-if="!loading && servers.length === 0">
      <div class="empty-icon">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.4" width="36" height="36">
          <rect x="2" y="2" width="20" height="8" rx="2"/>
          <rect x="2" y="14" width="20" height="8" rx="2"/>
          <line x1="6" y1="6" x2="6.01" y2="6" stroke-width="3"/>
          <line x1="6" y1="18" x2="6.01" y2="18" stroke-width="3"/>
        </svg>
      </div>
      <div class="empty-title">Sin infraestructura monitoreada</div>
      <div class="empty-sub">Conecta tus primeros servidores HPE iLO 5 para comenzar a monitorear en tiempo real.</div>
      <button class="btn-primary" @click="showAdd = true">
        <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/></svg>
        Añadir primer servidor
      </button>
    </div>

    <!-- ── Server Grid ── -->
    <main class="fleet-grid" :class="{ 'fleet-list': gridMode === 'list' }" v-else>
      <div class="no-results" v-if="filteredServers.length === 0">
        <svg width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5"><circle cx="11" cy="11" r="8"/><line x1="21" y1="21" x2="16.65" y2="16.65"/></svg>
        Sin resultados para <strong>"{{ searchQ }}"</strong>
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

    <!-- ── Add Modal ── -->
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

const props = defineProps({ refreshCount: Number })
defineEmits(['open-detail', 'open-reports'])

const { getServers } = useIlo()
const servers      = ref([])
const cardRefs     = ref({})
const showAdd      = ref(false)
const lastRefresh  = ref('')
const loading      = ref(true)
const statusMap    = ref({})
const searchQ      = ref('')
const gridMode     = ref('grid')
const activeFilter = ref('all')
const sortBy       = ref('name')

let cdTimer = null
const countdownSec = ref(REFRESH_INTERVAL_SEC)

const countdownPct = computed(() => {
  const t = REFRESH_INTERVAL_SEC
  return ((t - countdownSec.value) / t) * 100
})

const stats = computed(() => {
  const s = { ok: 0, warn: 0, crit: 0, off: 0 }
  servers.value.forEach(srv => {
    const st = statusMap.value[srv.id]
    if      (st === 'ok')   s.ok++
    else if (st === 'warn') s.warn++
    else if (st === 'crit') s.crit++
    else                    s.off++
  })
  return s
})

const healthPct = computed(() => {
  if (!servers.value.length) return 0
  return Math.round((stats.value.ok / servers.value.length) * 100)
})


const filters = computed(() => [
  { key: 'ok',   label: 'Óptimos',     cls: 'green', count: stats.value.ok },
  { key: 'warn', label: 'Advertencia', cls: 'amber', count: stats.value.warn },
  { key: 'crit', label: 'Críticos',    cls: 'red',   count: stats.value.crit },
  { key: 'off',  label: 'Offline',     cls: 'gray',  count: stats.value.off },
])

const statusOrder = { crit: 0, warn: 1, ok: 2, off: 3 }

const filteredServers = computed(() => {
  let list = [...servers.value]

  // Filter by status
  if (activeFilter.value !== 'all') {
    list = list.filter(srv => statusMap.value[srv.id] === activeFilter.value)
  }

  // Filter by search
  if (searchQ.value.trim()) {
    const q = searchQ.value.toLowerCase()
    list = list.filter(srv =>
      srv.label.toLowerCase().includes(q) || srv.host.toLowerCase().includes(q)
    )
  }

  // Sort
  list.sort((a, b) => {
    if (sortBy.value === 'status') {
      return (statusOrder[statusMap.value[a.id]] ?? 4) - (statusOrder[statusMap.value[b.id]] ?? 4)
    }
    if (sortBy.value === 'temp') {
      const ta = metricsMap.value[a.id]?.temp ?? -1
      const tb = metricsMap.value[b.id]?.temp ?? -1
      return tb - ta
    }
    if (sortBy.value === 'watts') {
      const wa = metricsMap.value[a.id]?.watts ?? -1
      const wb = metricsMap.value[b.id]?.watts ?? -1
      return wb - wa
    }
    return a.label.localeCompare(b.label) // name
  })

  return list
})

function pct(n) {
  return servers.value.length ? Math.round((n / servers.value.length) * 100) : 0
}

const chartData = computed(() => {
  const { ok, warn, crit, off } = stats.value
  if (!ok && !warn && !crit && !off) {
    return { labels: ['—'], datasets: [{ data: [1], backgroundColor: ['#e8e4df'], borderWidth: 0 }] }
  }
  return {
    labels: ['Óptimos', 'Advertencia', 'Críticos', 'Offline'],
    datasets: [{
      data: [ok, warn, crit, off],
      backgroundColor: ['#1a8a7a', '#e67e22', '#c0392b', '#b0aba3'],
      hoverBackgroundColor: ['#1da898', '#f39c12', '#e74c3c', '#9a9490'],
      borderWidth: 0, hoverOffset: 6,
    }]
  }
})

const chartOptions = {
  responsive: true, maintainAspectRatio: false, cutout: '74%',
  plugins: { legend: { display: false }, tooltip: { enabled: true } }
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
  delete statusMap.value[id]
  delete metricsMap.value[id]
  delete cardRefs.value[id]
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

watch(() => props.refreshCount, () => { reloadAll() })
onUnmounted(() => clearInterval(cdTimer))
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Sora:wght@400;500;600;700;800&family=IBM+Plex+Mono:wght@400;600&display=swap');

/* ── BASE ── */
.fleet-page { display:flex; flex-direction:column; min-height:100vh; background:#f5f2ee; font-family:'Sora',-apple-system,sans-serif; color:#1a1714; }

/* ── TOPBAR ── */
.topbar { background:#faf8f5; border-bottom:1.5px solid #ddd8d0; padding:0 32px; height:60px; display:flex; align-items:center; justify-content:space-between; position:sticky; top:0; z-index:100; gap:16px; }
.topbar-brand { display:flex; align-items:center; gap:14px; }
.brand-icon { width:38px; height:38px; border-radius:10px; background:#e6f5f2; border:1.5px solid #b8e4dc; display:flex; align-items:center; justify-content:center; flex-shrink:0; }
.brand-icon svg { width:18px; height:18px; stroke:#1a8a7a; }
.brand-title { font-size:15px; font-weight:800; color:#1a1714; letter-spacing:-.03em; }
.brand-sub   { font-size:10px; color:#9a9490; margin-top:1px; font-weight:500; }
.brand-count { color:#1a8a7a; font-weight:700; }

.topbar-right { display:flex; align-items:center; gap:8px; }
.live-pill { display:flex; align-items:center; gap:5px; background:#e6f5ee; border:1px solid #b8e4d8; padding:3px 9px; border-radius:20px; font-size:9px; font-weight:800; color:#0f6e44; letter-spacing:.08em; }
.live-dot { width:5px; height:5px; border-radius:50%; background:#1a8a7a; animation:pulse-dot 1.5s infinite; }
@keyframes pulse-dot { 0%,100% { transform:scale(1); opacity:1; } 50% { transform:scale(1.4); opacity:.5; } }

.countdown-pill { display:flex; align-items:center; gap:5px; font-size:11px; font-weight:600; color:#6b6560; background:#f5f2ee; border:1px solid #ddd8d0; padding:4px 10px; border-radius:20px; }
.last-refresh { font-size:10px; color:#9a9490; font-family:'IBM Plex Mono',monospace; }

.btn-icon { display:flex; align-items:center; gap:6px; font-family:'Sora',sans-serif; font-size:12px; font-weight:600; padding:7px 13px; border-radius:8px; border:1.5px solid #ddd8d0; background:white; color:#6b6560; cursor:pointer; transition:all .15s; letter-spacing:.02em; }
.btn-icon:hover { background:#f5f2ee; border-color:#b0aba3; color:#1a1714; }
.btn-icon:disabled { opacity:.5; cursor:not-allowed; }
.btn-primary { display:flex; align-items:center; gap:6px; font-family:'Sora',sans-serif; font-size:12px; font-weight:700; padding:7px 15px; border-radius:8px; background:#1a1714; color:#f5f2ee; border:none; cursor:pointer; transition:all .15s; letter-spacing:.02em; }
.btn-primary:hover { background:#2d2925; }

/* ── ALERTS ── */
.alert-crit { display:flex; align-items:center; gap:10px; background:#fdecea; border-bottom:1.5px solid #f5c0c0; padding:10px 32px; font-size:12px; font-weight:600; color:#a32d2d; }
.alert-warn { display:flex; align-items:center; gap:10px; background:#fef3e6; border-bottom:1.5px solid #f9d5a5; padding:10px 32px; font-size:12px; font-weight:600; color:#854f0b; }
.alert-led { width:7px; height:7px; border-radius:50%; background:#c0392b; animation:blink 1s infinite; flex-shrink:0; }
.alert-led--warn { background:#e67e22; animation:none; }

/* ── KPI BAR ── */
.kpi-bar {
  display:flex; align-items:center; gap:0;
  background:white; border-bottom:1.5px solid #ddd8d0;
  padding:16px 32px; gap:28px;
}
.kpi-donut-wrap { position:relative; width:72px; height:72px; flex-shrink:0; }
.kpi-donut-center { position:absolute; inset:0; display:flex; flex-direction:column; align-items:center; justify-content:center; pointer-events:none; }
.kdc-pct { font-family:'IBM Plex Mono',monospace; font-size:16px; font-weight:600; color:#1a1714; line-height:1; }
.kdc-pct small { font-size:10px; color:#9a9490; }
.kdc-lbl { font-size:8px; font-weight:700; color:#9a9490; text-transform:uppercase; letter-spacing:.08em; margin-top:2px; }

.kpi-divider { width:1px; height:52px; background:#e8e4df; flex-shrink:0; }

.kpi-stats { display:flex; align-items:center; gap:24px; }
.kpi-stat  { display:flex; flex-direction:column; align-items:center; gap:3px; }
.ks-val    { font-family:'IBM Plex Mono',monospace; font-size:26px; font-weight:600; line-height:1; }
.ks-lbl    { font-size:9px; font-weight:700; color:#9a9490; text-transform:uppercase; letter-spacing:.08em; }
.ks--total { color:#1a1714; }
.ks--ok    { color:#1a8a7a; }
.ks--warn  { color:#e67e22; }
.ks--crit  { color:#c0392b; }
.ks--off   { color:#b0aba3; }
.ks-blink  { animation:blink 1.2s infinite; }

.kpi-health-col { flex:1; min-width:180px; }
.khc-label { font-size:9px; font-weight:700; color:#9a9490; text-transform:uppercase; letter-spacing:.08em; margin-bottom:8px; }
.health-bar-track { height:8px; border-radius:4px; background:#f5f2ee; overflow:hidden; display:flex; border:1px solid #e8e4df; }
.health-bar-seg { height:100%; transition:width .6s ease; }
.hbs--ok   { background:#1a8a7a; }
.hbs--warn { background:#e67e22; }
.hbs--crit { background:#c0392b; }
.hbs--off  { background:#d4d0cb; }
.health-bar-legend { display:flex; gap:12px; margin-top:7px; }
.hbl-item  { display:flex; align-items:center; gap:4px; font-size:9px; font-weight:600; color:#9a9490; }
.hbl-dot   { width:6px; height:6px; border-radius:50%; }
.hbl--ok   { background:#1a8a7a; }
.hbl--warn { background:#e67e22; }
.hbl--crit { background:#c0392b; }
.hbl--off  { background:#d4d0cb; }



/* ── FILTER BAR ── */
.filter-bar { display:flex; align-items:center; gap:10px; flex-wrap:wrap; padding:12px 32px; border-bottom:1px solid #e8e4df; background:#faf8f5; }
.search-wrap { position:relative; }
.search-icon { position:absolute; left:10px; top:50%; transform:translateY(-50%); pointer-events:none; color:#9a9490; }
.search-input { background:white; border:1.5px solid #ddd8d0; border-radius:8px; padding:7px 32px 7px 32px; font-size:12px; font-family:'Sora',sans-serif; color:#1a1714; outline:none; width:210px; transition:border-color .2s,width .25s; }
.search-input::placeholder { color:#b0aba3; }
.search-input:focus { border-color:#1a8a7a; width:260px; }
.search-clear { position:absolute; right:10px; top:50%; transform:translateY(-50%); background:none; border:none; color:#9a9490; cursor:pointer; font-size:16px; line-height:1; padding:0; }
.search-clear:hover { color:#1a1714; }

.filter-chips { display:flex; gap:5px; }
.fchip { display:flex; align-items:center; gap:6px; padding:5px 11px; border-radius:20px; font-size:11px; font-weight:600; background:white; border:1.5px solid #ddd8d0; color:#6b6560; cursor:pointer; transition:all .15s; font-family:'Sora',sans-serif; }
.fchip:hover { border-color:#b0aba3; color:#1a1714; }
.fchip.fchip--green { background:#e6f5f2; border-color:#b8e4dc; color:#0f6e44; }
.fchip.fchip--amber { background:#fef3e6; border-color:#f9d5a5; color:#854f0b; }
.fchip.fchip--red   { background:#fdecea; border-color:#f5c0c0; color:#a32d2d; }
.fchip.fchip--gray  { background:#f5f2ee; border-color:#d4d0cb; color:#6b6560; }
.fchip-dot { width:6px; height:6px; border-radius:50%; flex-shrink:0; }
.fd--green { background:#1a8a7a; }
.fd--amber { background:#e67e22; }
.fd--red   { background:#c0392b; }
.fd--gray  { background:#b0aba3; }
.fchip-count { background:rgba(0,0,0,.07); padding:1px 6px; border-radius:20px; font-size:9px; }

.sort-wrap { }
.sort-select { font-family:'Sora',sans-serif; font-size:11px; font-weight:600; color:#6b6560; background:white; border:1.5px solid #ddd8d0; border-radius:8px; padding:6px 10px; cursor:pointer; outline:none; transition:border-color .15s; }
.sort-select:focus { border-color:#1a8a7a; }

.grid-toggle { display:flex; gap:2px; background:white; border:1.5px solid #ddd8d0; border-radius:8px; padding:3px; margin-left:auto; }
.gt-btn { width:26px; height:26px; display:flex; align-items:center; justify-content:center; border-radius:5px; border:none; background:transparent; color:#9a9490; cursor:pointer; transition:all .15s; }
.gt-btn.active { background:#e6f5f2; color:#1a8a7a; }
.gt-btn:hover:not(.active) { background:#f5f2ee; color:#1a1714; }

.result-count { font-size:11px; font-weight:600; color:#9a9490; font-family:'IBM Plex Mono',monospace; }

/* ── EMPTY STATE ── */
.empty-state { flex:1; display:flex; flex-direction:column; align-items:center; justify-content:center; gap:16px; padding:80px 24px; text-align:center; }
.empty-icon { width:72px; height:72px; border-radius:18px; background:#e6f5f2; border:1.5px solid #b8e4dc; display:flex; align-items:center; justify-content:center; color:#1a8a7a; }
.empty-title { font-size:20px; font-weight:800; color:#1a1714; letter-spacing:-.02em; }
.empty-sub   { font-size:13px; color:#9a9490; max-width:320px; line-height:1.6; }

/* ── FLEET GRID ── */ 
.fleet-grid { flex:1; padding:24px 32px; display:grid; grid-template-columns:repeat(auto-fill,minmax(300px,1fr)); gap:16px; align-items:start; }
.fleet-list { grid-template-columns:1fr; gap:8px; }
.no-results { grid-column:1/-1; display:flex; flex-direction:column; align-items:center; justify-content:center; gap:12px; padding:60px; color:#9a9490; font-size:13px; }
.no-results strong { color:#1a1714; }

/* ── ANIMATIONS ── */
@keyframes blink { 0%,100% { opacity:1; } 50% { opacity:.2; } }
@keyframes spin   { from { transform:rotate(0deg); } to { transform:rotate(360deg); } }
.spin { display:inline-block; animation:spin .8s linear infinite; }

/* ── RESPONSIVE ── */

@media (max-width:900px)  { .kpi-health-col { display:none; } .kpi-bar { gap:20px; } }
@media (max-width:768px)  { .fleet-grid { padding:16px; gap:12px; } .filter-bar { padding:10px 16px; } .kpi-bar { padding:12px 16px; } .topbar { padding:0 16px; } .alert-crit,.alert-warn { padding:10px 16px; } }
@media (max-width:640px)  { .topbar { height:auto; padding:12px 16px; flex-wrap:wrap; } .kpi-stats { gap:16px; } .fleet-grid { grid-template-columns:1fr; } .filter-chips { display:none; } }
</style>