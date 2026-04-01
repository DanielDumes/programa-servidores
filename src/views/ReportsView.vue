<template>
  <div class="reports-page">

    <!-- ── Topbar ─────────────────────────────────────────────── -->
    <header class="topbar">
      <button class="btn-back" @click="$emit('back')">
        <svg viewBox="0 0 24 24" fill="none" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <polyline points="15 18 9 12 15 6"/>
        </svg>
        Volver
      </button>
      <div class="topbar-left">
        <span class="topbar-title">Auditoría y Reportes</span>
        <span class="topbar-sub">Monitoreo Automático · MongoDB</span>
      </div>

      <!-- Tabs -->
      <div class="tab-bar">
        <button
          v-for="tab in tabs" :key="tab.key"
          class="tab-btn"
          :class="{ active: activeTab === tab.key }"
          @click="switchTab(tab.key)"
        >
          <span class="tab-icon">{{ tab.icon }}</span>
          {{ tab.label }}
        </button>
      </div>

      <div class="topbar-right">
        <button class="btn btn--primary" @click="downloadCSV">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" style="margin-right:6px;vertical-align:middle;">
            <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" y1="15" x2="12" y2="3"/>
          </svg>
          Exportar CSV
        </button>
        <button class="btn btn--ghost" @click="triggerMonitor" :disabled="triggering" title="Fuerza un ciclo de monitoreo ahora">
          <span :class="{ spin: triggering }">⟳</span>
          {{ triggering ? 'Sincronizando…' : 'Forzar Ciclo' }}
        </button>
      </div>
    </header>

    <!-- ══════════════════════════════════════════════════════════ -->
    <!-- PESTAÑA 1 — RESUMEN SEMANAL                               -->
    <!-- ══════════════════════════════════════════════════════════ -->
    <div v-if="activeTab === 'weekly'">

      <div class="content" v-if="loadingWeekly">
        <div class="center-loader"><div class="spinner"></div> Recuperando eventos de los últimos 7 días…</div>
      </div>
      <div class="content" v-else-if="weeklyError">
        <div class="alert-bar">{{ weeklyError }}</div>
      </div>

      <div class="content" v-else>
        <!-- KPI Cards -->
        <div class="stat-row">
          <div class="stat-card">
            <div class="stat-icon icon--blue">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M3 3h18v18H3z"/><path d="M3 9h18M9 21V9"/></svg>
            </div>
            <div class="stat-body">
              <div class="stat-lbl">Total Incidencias</div>
              <div class="stat-val">{{ weekly.metrics?.total_events ?? 0 }}</div>
              <div class="stat-sub">Registros de los últimos 7 días</div>
            </div>
          </div>
          <div class="stat-card">
            <div class="stat-icon icon--amber">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M18.36 6.64a9 9 0 1 1-12.73 0"/><line x1="12" y1="2" x2="12" y2="12"/></svg>
            </div>
            <div class="stat-body">
              <div class="stat-lbl">Cambios de Energía</div>
              <div class="stat-val warn">{{ weekly.metrics?.power_events ?? 0 }}</div>
              <div class="stat-sub">Encendidos y apagados detectados</div>
            </div>
          </div>
          <div class="stat-card">
            <div class="stat-icon icon--red">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="12" r="10"/><line x1="12" y1="8" x2="12" y2="12"/><line x1="12" y1="16" x2="12.01" y2="16"/></svg>
            </div>
            <div class="stat-body">
              <div class="stat-lbl">Alertas Críticas</div>
              <div class="stat-val crit">{{ weekly.metrics?.critical_alarms ?? 0 }}</div>
              <div class="stat-sub">Sensores o componentes fallando</div>
            </div>
          </div>
          <div class="stat-card">
            <div class="stat-icon icon--green">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"/><polyline points="22 4 12 14.01 9 11.01"/></svg>
            </div>
            <div class="stat-body">
              <div class="stat-lbl">Recuperaciones</div>
              <div class="stat-val ok">{{ weekly.metrics?.recoveries ?? 0 }}</div>
              <div class="stat-sub">Sistemas que volvieron a estado Óptimo</div>
            </div>
          </div>
        </div>

        <!-- Event Log -->
        <div class="panel list-panel">
          <div class="panel-header">
            <div class="panel-title">Registro de Eventos — Últimos 7 días</div>
            <div class="panel-badge-count">{{ weekly.logs?.length ?? 0 }} registros</div>
          </div>
          <div v-if="!weekly.logs?.length" class="empty-log">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" width="32" height="32"><path d="M9 14 4 9l5-5"/><path d="M20 20v-7a4 4 0 0 0-4-4H4"/></svg>
            <span>Sin incidencias esta semana. Todo en orden.</span>
          </div>
          <div class="table-wrap" v-else>
            <table class="report-table">
              <thead>
                <tr>
                  <th>Fecha y Hora</th>
                  <th>Servidor</th>
                  <th>IP</th>
                  <th>Tipo Evento</th>
                  <th>Cambio de Estado</th>
                  <th>Detalles</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="log in filteredWeeklyLogs" :key="log._id">
                  <td class="td-time">{{ formatTime(log.timestamp) }}</td>
                  <td><span class="label-badge">{{ log.server_label || '—' }}</span></td>
                  <td><span class="host-badge">{{ log.server }}</span></td>
                  <td><span class="chip" :class="typeCls(log.type)">{{ typeLbl(log.type) }}</span></td>
                  <td>
                    <span class="state-pill" :class="stateCls(log.old_status)">{{ log.old_status || '—' }}</span>
                    <span class="arrow">→</span>
                    <span class="state-pill" :class="stateCls(log.new_status)">{{ log.new_status || '—' }}</span>
                  </td>
                  <td class="td-details">{{ log.details }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>

    <!-- ══════════════════════════════════════════════════════════ -->
    <!-- PESTAÑA 2 — HISTORIAL POR DÍA                            -->
    <!-- ══════════════════════════════════════════════════════════ -->
    <div v-if="activeTab === 'history'" class="history-layout">

      <!-- Sidebar: lista de días -->
      <aside class="day-sidebar">
        <div class="sidebar-title">Días con Datos</div>
        <div v-if="loadingHistory" class="center-loader" style="padding:24px 0">
          <div class="spinner"></div>
        </div>
        <div v-else-if="historyDays.length === 0" class="empty-log" style="padding:20px 16px; font-size:12px">
          Sin datos históricos aún.
        </div>
        <div
          v-else
          v-for="day in historyDays" :key="day.date"
          class="day-item"
          :class="{ active: selectedDate === day.date }"
          @click="loadDay(day.date)"
        >
          <div class="day-date">{{ formatDate(day.date) }}</div>
          <div class="day-meta">
            <span class="day-chip">{{ day.server_count }} srvs</span>
            <span class="day-chip day-chip--ev" v-if="day.events > 0">{{ day.events }} eventos</span>
          </div>
        </div>
      </aside>

      <!-- Main: grid del día seleccionado -->
      <main class="day-main">
        <div v-if="!selectedDate" class="center-placeholder">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" width="48" height="48"><rect x="3" y="4" width="18" height="18" rx="2"/><line x1="16" y1="2" x2="16" y2="6"/><line x1="8" y1="2" x2="8" y2="6"/><line x1="3" y1="10" x2="21" y2="10"/></svg>
          <p>Selecciona un día del panel izquierdo</p>
        </div>

        <div v-else-if="loadingDay" class="center-loader">
          <div class="spinner"></div> Cargando datos del {{ selectedDate }}…
        </div>

        <template v-else-if="dayData">
          <div class="day-main-header">
            <h2 class="day-main-title">{{ formatDateLong(dayData.date) }}</h2>
            <div class="fleet-status-summary" v-if="dayData.fleet">
              <span class="fss-item">
                <strong>{{ dayData.fleet.length }}</strong> Servidores en total
              </span>
              <span class="fss-item">
                <strong>{{ uniqueSnapshotServers.length }}</strong> Con actividad hoy
              </span>
              <span class="fss-item crit" v-if="dayData.fleet.length > uniqueSnapshotServers.length">
                <strong>{{ dayData.fleet.length - uniqueSnapshotServers.length }}</strong> Sin datos registrados
              </span>
            </div>
          </div>

          <!-- ── Hourly Status Grid (Heatmap) ── -->
          <div class="panel status-grid-panel" style="margin-bottom: 24px;">
            <div class="panel-header">
              <div class="panel-title">Estado de Salud por Hora — Disponibilidad</div>
              <div class="panel-badge-count">00:00 - 23:59</div>
            </div>
            <div class="hourly-grid-wrap" v-if="hourlyData">
              <table class="hourly-table">
                <thead>
                  <tr>
                    <th>Servidor</th>
                    <th v-for="h in 24" :key="h" class="th-h">{{ (h-1).toString().padStart(2,'0') }}</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="(info, sid) in hourlyData" :key="sid">
                    <td class="td-srv-name">{{ info.label }}</td>
                    <td v-for="h in 24" :key="h" class="td-h">
                      <div 
                        class="h-cell" 
                        :class="cellCls(info.hours[h-1])"
                        :title="`Hora ${(h-1)}:00 - ${info.hours[h-1] || 'Sin datos'}`"
                      ></div>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
            <div v-else-if="loadingHourly" class="center-loader" style="padding:40px">
              <div class="spinner"></div> Calculando disponibilidad…
            </div>
          </div>

          <!-- Timeline Filter -->
          <div class="timeline-controls" v-if="dayData.snapshots?.length">
            <div class="search-box">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="11" cy="11" r="8"/><line x1="21" y1="21" x2="16.65" y2="16.65"/></svg>
              <input type="text" v-model="historySearch" placeholder="Filtrar por nombre o IP de servidor...">
            </div>
            <div class="timeline-stats">
              {{ filteredSnapshots.length }} reporte(s) encontrados
            </div>
          </div>

          <!-- Chronological Table -->
          <div class="panel list-panel" v-if="filteredSnapshots.length">
            <div class="table-wrap">
              <table class="report-table">
                <thead>
                  <tr>
                    <th>Hora</th>
                    <th>Servidor</th>
                    <th>Estado</th>
                    <th>Energía</th>
                    <th>Temp</th>
                    <th>Acción</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="snap in filteredSnapshots" :key="snap._id">
                    <td class="td-time">{{ formatTimeOnly(snap.timestamp) }}</td>
                    <td>
                       <div class="td-srv-info">
                         <span class="label-badge">{{ snap.server_label }}</span>
                         <span class="host-badge-small">{{ snap.server_host }}</span>
                       </div>
                    </td>
                    <td>
                      <div v-if="snap.reachable" class="snap-health-row">
                        <span class="health-dot" :class="healthCls(snap.health)"></span>
                        {{ healthLabel(snap.health) }}
                      </div>
                      <span v-else class="tc-off">Offline</span>
                    </td>
                    <td>
                      <span v-if="snap.reachable" :class="snap.power_state === 'On' ? 'tc-ok' : 'tc-off'">
                        {{ snap.power_state }}
                      </span>
                      <span v-else>—</span>
                    </td>
                    <td>
                      <span v-if="snap.reachable" :class="tempCls(snap.max_temp_c)">
                        {{ snap.max_temp_c ?? '—' }}°
                      </span>
                      <span v-else>—</span>
                    </td>
                    <td>
                      <button class="btn-inspect" @click="viewSnapshot(snap)">
                        Inspeccionar Reporte →
                      </button>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
          <div v-else class="empty-log" style="margin-top:32px">
            No se encontraron reportes que coincidan con la búsqueda.
          </div>

          <!-- Registros Detallados (Audit Logs) -->
          <div class="audit-section" v-if="dayData.events?.length">
             <div class="section-divider">
               <span>REGISTROS DETALLADOS DE EVENTOS (AUDITORIA)</span>
             </div>
             <div class="audit-grid">
                <div v-for="ev in dayData.events" :key="ev._id" class="audit-card" :class="`audit--${(ev.severity||'info').toLowerCase()}`">
                   <div class="audit-header">
                      <span class="audit-time">{{ formatTime(ev.timestamp) }}</span>
                      <span class="audit-type">{{ typeLbl(ev.type) }}</span>
                   </div>
                   <div class="audit-body">
                      <div class="audit-srv">{{ ev.server_label }} <small>({{ ev.server }})</small></div>
                      <div class="audit-msg">{{ ev.details }}</div>
                   </div>
                   <div class="audit-footer">
                      <span class="audit-pill" :class="stateCls(ev.old_status)">{{ ev.old_status || '—' }}</span>
                      <span class="audit-arrow">→</span>
                      <span class="audit-pill" :class="stateCls(ev.new_status)">{{ ev.new_status || '—' }}</span>
                   </div>
                </div>
             </div>
          </div>
        </template>
      </main>
    </div>

    <!-- ══════════════════════════════════════════════════════════ -->
    <!-- PESTAÑA 3 — TENDENCIAS Y MÉTRICAS                      -->
    <!-- ══════════════════════════════════════════════════════════ -->
    <div v-if="activeTab === 'trends'" class="content">
      
      <div class="metric-controls">
        <div class="control-group">
          <label>Seleccionar Servidor:</label>
          <select v-model="selectedMetricSrv" @change="fetchMetrics">
            <option :value="null">Toda la Flota (Promedio)</option>
            <option v-for="s in srvList" :key="s.id" :value="s.id">{{ s.label }} ({{ s.host }})</option>
          </select>
        </div>
        <div class="control-group">
          <label>Periodo:</label>
          <select v-model="metricDays" @change="fetchMetrics">
            <option :value="1">Últimas 24h</option>
            <option :value="7">Últimos 7 días</option>
            <option :value="30">Últimos 30 días</option>
          </select>
        </div>
      </div>

      <div v-if="loadingMetrics" class="center-loader" style="padding:100px">
         <div class="spinner"></div> Generando informes de rendimiento…
      </div>

      <div class="metrics-grid" v-else-if="metricPoints.length">
        <div class="chart-card">
          <div class="chart-header">
            <div class="chart-title">🌡 Histórico de Temperatura (Max Sensor)</div>
            <div class="chart-sub">Promedio de temperatura máxima detectada por hora</div>
          </div>
          <div class="chart-container">
            <Line :data="tempChartData" :options="chartOptions" />
          </div>
        </div>

        <div class="chart-card">
          <div class="chart-header">
            <div class="chart-title">⚡ Histórico de Consumo Eléctrico</div>
            <div class="chart-sub">Consumo promedio en Watts (W) extraído del iLO Power Control</div>
          </div>
          <div class="chart-container">
            <Line :data="powerChartData" :options="chartOptions" />
          </div>
        </div>
      </div>

      <div v-else class="empty-log" style="padding:100px">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" width="48" height="48"><path d="M12 20V10"/><path d="M18 20V4"/><path d="M6 20v-4"/></svg>
        <p>No hay datos suficientes para generar tendencias en este periodo.</p>
      </div>

    </div>

    <!-- ── Snapshot Inspector Modal ── -->
    <div v-if="inspectSnap" class="modal-overlay" @click.self="inspectSnap = null">
      <div class="modal-content">
        <header class="modal-header">
           <div class="mh-top">
              <span class="mh-title">Detalle Histórico: {{ inspectSnap.server_label }}</span>
              <button class="btn-close" @click="inspectSnap = null">×</button>
           </div>
           <div class="mh-sub">Reporte del {{ formatTime(inspectSnap.timestamp) }} (iLO Snapshot)</div>
        </header>
        <div class="modal-body">
           <!-- Reutilizar diseño de DetailView -->
           <div class="snap-grid">
              <div class="panel panel--glow">
                 <div class="panel-title">Estado en ese momento</div>
                 <div class="info-grid">
                    <div class="info-row"><span class="info-lbl">Salud</span> <span :class="healthCls(inspectSnap.health)">{{ healthLabel(inspectSnap.health) }}</span></div>
                    <div class="info-row"><span class="info-lbl">Energía</span> <span :class="inspectSnap.power_state === 'On' ? 'tc-ok' : 'tc-off'">{{ inspectSnap.power_state }}</span></div>
                    <div class="info-row"><span class="info-lbl">Max Temp</span> <span :class="tempCls(inspectSnap.max_temp_c)">{{ inspectSnap.max_temp_c }}°C</span></div>
                    <div class="info-row"><span class="info-lbl">Consumo</span> <span>{{ inspectSnap.consumed_watts }}W</span></div>
                 </div>
              </div>
              <!-- Resumen de Hardware -->
              <div class="panel panel--glow">
                 <div class="panel-title">Inventario detectado</div>
                 <div class="info-grid">
                    <div class="info-row"><span class="info-lbl">CPUs</span> <span>{{ inspectSnap.systems_raw?.ProcessorSummary?.Count ?? '—' }} núcleos</span></div>
                    <div class="info-row"><span class="info-lbl">RAM</span> <span>{{ inspectSnap.memory_data?.length ?? 0 }} DIMMs</span></div>
                    <div class="info-row"><span class="info-lbl">Discos</span> <span>{{ inspectSnap.storage_data?.length ?? 0 }} Controladoras</span></div>
                 </div>
              </div>
           </div>

           <!-- Lista de Componentes -->
           <div class="panel panel--glow" style="margin-top:20px" v-if="inspectSnap.storage_data?.length">
              <div class="panel-title">Almacenamiento Histórico</div>
              <div class="table-wrap">
                 <table class="report-table">
                   <thead><tr><th>Disco</th><th>Salud</th><th>Tipo</th><th>Capacidad</th></tr></thead>
                   <tbody>
                     <template v-for="ctrl in inspectSnap.storage_data" :key="ctrl.name">
                        <tr v-for="d in ctrl.drives" :key="d.name">
                          <td>{{ d.name }}</td>
                          <td><span :class="healthCls(d.health)">{{ d.health }}</span></td>
                          <td>{{ d.type }}</td>
                          <td>{{ d.capacity_gb }} GB</td>
                        </tr>
                     </template>
                   </tbody>
                 </table>
              </div>
           </div>

           <div class="panel panel--glow" style="margin-top:20px" v-if="inspectSnap.memory_data?.length">
              <div class="panel-title">Mapeo de RAM Histórico</div>
              <div class="table-wrap">
                 <table class="report-table">
                   <thead><tr><th>DIMM</th><th>Estado</th><th>Velocidad</th><th>Tipo</th></tr></thead>
                   <tbody>
                      <tr v-for="m in inspectSnap.memory_data" :key="m.name">
                        <td>{{ m.name }}</td>
                        <td><span :class="healthCls(m.health)">{{ m.health }}</span></td>
                        <td>{{ m.speed_mhz }} MHz</td>
                        <td>{{ m.type }}</td>
                      </tr>
                   </tbody>
                 </table>
              </div>
           </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { BACKEND_URL } from '../config/servers.js'
import { useIlo } from '../composables/useIlo.js'
import { Line } from 'vue-chartjs'
import {
  Chart as ChartJS, CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip, Legend, Filler
} from 'chart.js'

ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip, Legend, Filler)

defineEmits(['back'])

// ── State ────────────────────────────────────────────────────────
const activeTab     = ref('weekly')
const tabs = [
  { key: 'weekly',  label: 'Resumen Semanal', icon: '📊' },
  { key: 'history', label: 'Historial por Día', icon: '📅' },
  { key: 'trends',  label: 'Tendencias y Métricas', icon: '📈' },
]

const { getServers } = useIlo()
const srvList = ref([])

// Weekly
const loadingWeekly = ref(true)
const weeklyError   = ref(null)
const weekly        = ref({})

// History
const loadingHistory = ref(false)
const historyDays    = ref([])
const selectedDate   = ref(null)
const loadingDay     = ref(false)
const dayData        = ref(null)
const loadingHourly  = ref(false)
const hourlyData     = ref(null)

const historySearch  = ref('')
const inspectSnap    = ref(null)

const filteredSnapshots = computed(() => {
  if (!dayData.value?.snapshots) return []
  const q = historySearch.value.toLowerCase().trim()
  if (!q) return dayData.value.snapshots
  return dayData.value.snapshots.filter(s => {
    const label = s.server_label || ""
    const host  = s.server_host  || ""
    return label.toLowerCase().includes(q) || host.toLowerCase().includes(q)
  })
})

const uniqueSnapshotServers = computed(() => {
  if (!dayData.value?.snapshots) return []
  const sids = new Set(dayData.value.snapshots.map(s => s.server_id))
  return Array.from(sids)
})

const filteredWeeklyLogs = computed(() => {
  if (!weekly.value.logs) return []
  return weekly.value.logs.filter(l => l.type !== 'ConnectionLoss')
})

// Metrics / Trends
const loadingMetrics    = ref(false)
const selectedMetricSrv = ref(null)
const metricDays        = ref(7)
const metricPoints      = ref([])

// Trigger
const triggering = ref(false)

// ── Fetch ────────────────────────────────────────────────────────
async function fetchWeekly() {
  loadingWeekly.value = true
  weeklyError.value   = null
  try {
    const res  = await fetch(`${BACKEND_URL}/api/reports/weekly`)
    const data = await res.json()
    if (!res.ok) throw new Error(data.error || `HTTP ${res.status}`)
    weekly.value = data
  } catch(e) {
    weeklyError.value = e.message
  } finally {
    loadingWeekly.value = false
  }
}

async function fetchHistory() {
  loadingHistory.value = true
  try {
    const res  = await fetch(`${BACKEND_URL}/api/reports/history`)
    const data = await res.json()
    if (!res.ok) throw new Error(data.error)
    historyDays.value = data.days
  } catch(e) {
    console.error('Error cargando historial:', e.message)
  } finally {
    loadingHistory.value = false
  }
}

async function loadDay(date) {
  selectedDate.value = date
  loadingDay.value   = true
  dayData.value      = null
  hourlyData.value   = null
  try {
    const res  = await fetch(`${BACKEND_URL}/api/reports/daily?date=${date}`)
    const data = await res.json()
    if (!res.ok) throw new Error(data.error)
    dayData.value = data
    
    // Cargar también la cuadrícula de horas
    fetchHourly(date)
  } catch(e) {
    console.error('Error cargando día:', e.message)
  } finally {
    loadingDay.value = false
  }
}

async function fetchHourly(date) {
  loadingHourly.value = true
  try {
    const res = await fetch(`${BACKEND_URL}/api/reports/hourly?date=${date}`)
    const data = await res.json()
    hourlyData.value = data.data
  } catch(e) {
    console.error('Error cargando grid horario:', e)
  } finally {
    loadingHourly.value = false
  }
}

async function fetchMetrics() {
  loadingMetrics.value = true
  try {
    let url = `${BACKEND_URL}/api/reports/metrics?days=${metricDays.value}`
    if (selectedMetricSrv.value) url += `&server_id=${selectedMetricSrv.value}`
    
    const res = await fetch(url)
    const data = await res.json()
    metricPoints.value = data.points
  } catch(e) {
    console.error('Error fetching metrics:', e)
  } finally {
    loadingMetrics.value = false
  }
}

async function triggerMonitor() {
  triggering.value = true
  try {
    await fetch(`${BACKEND_URL}/api/monitor/run-now`, { method: 'POST' })
    setTimeout(() => {
      fetchWeekly()
      if (activeTab.value === 'history') fetchHistory()
    }, 3000)
  } finally {
    setTimeout(() => { triggering.value = false }, 3000)
  }
}

function downloadCSV() {
  window.open(`${BACKEND_URL}/api/reports/download`, '_blank')
}

function switchTab(tab) {
  activeTab.value = tab
  if (tab === 'history' && historyDays.value.length === 0) {
    fetchHistory()
  }
  if (tab === 'trends' && metricPoints.value.length === 0) {
    fetchMetrics()
  }
}

// ── Chart Config ──────────────────────────────────────────────────
const tempChartData = computed(() => ({
  labels: metricPoints.value.map(p => new Date(p.ts).toLocaleString('es-EC', {
    timeZone: 'America/Guayaquil',
    day: 'numeric', month: 'short', hour: '2-digit', minute: '2-digit', hour12: false
  })),
  datasets: [{
    label: 'Temp. Máxima (°C)',
    data: metricPoints.value.map(p => p.avg_temp),
    borderColor: '#E03535',
    backgroundColor: 'rgba(224, 53, 53, 0.1)',
    tension: 0.4,
    fill: true,
    pointRadius: 2
  }]
}))

const powerChartData = computed(() => ({
  labels: metricPoints.value.map(p => new Date(p.ts).toLocaleString('es-EC', {
    timeZone: 'America/Guayaquil',
    day: 'numeric', month: 'short', hour: '2-digit', minute: '2-digit', hour12: false
  })),
  datasets: [{
    label: 'Consumo Promedio (W)',
    data: metricPoints.value.map(p => p.avg_power),
    borderColor: '#3E92EF',
    backgroundColor: 'rgba(62, 146, 239, 0.1)',
    tension: 0.4,
    fill: true,
    pointRadius: 2
  }]
}))

const chartOptions = {
  responsive: true, maintainAspectRatio: false,
  scales: {
    y: { grid: { color: 'rgba(255,255,255,0.05)' }, ticks: { color: '#64748B', font: { size: 10 } } },
    x: { display: false }
  },
  plugins: {
    legend: { display: false },
    tooltip: { backgroundColor: 'rgba(10,14,24,0.9)', titleFont: { size: 11 }, bodyFont: { size: 12 }, padding: 10, cornerRadius: 6 }
  }
}

// ── Format helpers ───────────────────────────────────────────────
const EC_LOCALE_OPTS = { timeZone: 'America/Guayaquil' }

function formatTime(iso) {
  if (!iso) return '—'
  try {
    return new Date(iso).toLocaleString('es-EC', {
      timeZone: 'America/Guayaquil',
      day: '2-digit', month: 'short',
      hour: '2-digit', minute: '2-digit',
      hour12: false
    })
  } catch (e) { return '—' }
}

function formatTimeOnly(iso) {
  if (!iso) return '—'
  try {
    return new Date(iso).toLocaleTimeString('es-EC', {
      timeZone: 'America/Guayaquil',
      hour: '2-digit', minute: '2-digit',
      hour12: false
    })
  } catch (e) { return '—' }
}

function viewSnapshot(snap) {
  inspectSnap.value = snap
}

function formatDate(dateStr) {
  const [y, m, d] = dateStr.split('-')
  const months = ['Ene','Feb','Mar','Abr','May','Jun','Jul','Ago','Sep','Oct','Nov','Dic']
  return `${d} ${months[parseInt(m)-1]} ${y}`
}

function formatDateLong(dateStr) {
  const [y, m, d] = dateStr.split('-')
  const months = ['Enero','Febrero','Marzo','Abril','Mayo','Junio','Julio','Agosto','Septiembre','Octubre','Noviembre','Diciembre']
  return `${parseInt(d)} de ${months[parseInt(m)-1]} de ${y}`
}

// ── Class helpers ────────────────────────────────────────────────
function typeLbl(t) {
  return {
    ConnectionLoss:    'Desconexión',
    HealthDegradation: 'Degradación',
    HealthRecovery:    'Recuperación',
    PowerOff:          'Apagado',
    PowerOn:           'Encendido',
    HealthChange:      'Cambio Health',
  }[t] || t
}
function typeCls(t) {
  return {
    ConnectionLoss:    'chip--warn',
    HealthDegradation: 'chip--crit',
    HealthRecovery:    'chip--ok',
    PowerOff:          'chip--off',
    PowerOn:           'chip--ok',
    HealthChange:      'chip--warn',
  }[t] || ''
}
function stateCls(s) {
  return { OK: 'tc-ok', Warning: 'tc-warn', Critical: 'tc-crit', Offline: 'tc-off', Off: 'tc-off' }[s] || ''
}
function healthLabel(h) {
  return { OK: 'Óptimo', Warning: 'Advertencia', Critical: 'Crítico' }[h] || h || '—'
}
function healthCls(h) {
  return { OK: 'tc-ok', Warning: 'tc-warn', Critical: 'tc-crit' }[h] || ''
}
function tempCls(t) {
  if (t == null) return ''
  if (t > 80)   return 'tc-crit'
  if (t > 65)   return 'tc-warn'
  return 'tc-ok'
}
function cellCls(status) {
  if (!status) return 'cell--empty'
  if (status === 'OK') return 'cell--ok'
  if (status === 'Warning') return 'cell--warn'
  if (status === 'Critical') return 'cell--crit'
  if (status === 'Offline') return 'cell--off'
  return ''
}

// ── Init ─────────────────────────────────────────────────────────
onMounted(async () => { 
  fetchWeekly() 
  srvList.value = await getServers()
})
</script>

<style scoped>
/* ── Base ── */
.reports-page { min-height: 100vh; display: flex; flex-direction: column; background: var(--page); }

/* ── Topbar ── */
.topbar {
  background: rgba(255, 255, 255, 0.85); border-bottom: 1px solid var(--border);
  padding: 0 24px; height: 64px;
  display: flex; align-items: center; gap: 16px; flex-wrap: nowrap;
  position: sticky; top: 0; z-index: 40; backdrop-filter: blur(20px);
}
.btn-back {
  display: flex; align-items: center; gap: 8px; font-size: 13px; font-weight: 500;
  padding: 6px 14px; border-radius: 8px; border: 1px solid transparent;
  background: rgba(0,0,0,0.03); color: var(--text-2); cursor: pointer; transition: all .2s; flex-shrink:0;
}
.btn-back svg { width: 16px; height: 16px; stroke: var(--text-2); }
.btn-back:hover { background: rgba(0,0,0,0.06); color: var(--text); border-color: rgba(0,0,0,0.1); }
.topbar-left { flex-shrink:0; }
.topbar-title { font-size: 16px; font-weight: 600; color: var(--text); display:block; }
.topbar-sub   { font-size: 10px; color: var(--text-3); text-transform: uppercase; letter-spacing: 0.1em; }
.topbar-right { margin-left: auto; display:flex; align-items:center; gap:8px; flex-shrink:0; }

/* ── Tabs ── */
.tab-bar { display:flex; gap:4px; background: rgba(0,0,0,0.03); padding:4px; border-radius:10px; border:1px solid var(--border); }
.tab-btn {
  display:flex; align-items:center; gap:6px; padding:6px 14px; border-radius:7px;
  font-size: 13px; font-weight: 500; color: var(--text-3);
  cursor:pointer; border:none; background:transparent; transition: all .2s;
}
.tab-btn .tab-icon { font-size:14px; }
.tab-btn:hover { color: var(--text); background: rgba(0,0,0,0.05); }
.tab-btn.active { background: rgba(62,146,239,0.18); color: var(--blue-400); border: 1px solid rgba(62,146,239,0.3); }

/* ── Shared Content ── */
.content { padding: 28px 32px; display: flex; flex-direction: column; gap: 24px; }
.center-loader { text-align:center; color: var(--text-3); padding:60px; display:flex; align-items:center; justify-content:center; gap:10px; font-size:14px; }
.center-placeholder { display:flex; flex-direction:column; align-items:center; justify-content:center; gap:16px; color: var(--text-4); padding:80px 0; font-size:14px; }
.center-placeholder svg { opacity:.4; }
.alert-bar { background: var(--red-50); color: var(--red-600); padding: 12px 16px; border-radius: 8px; border: 1px solid rgba(224,53,53,0.3); font-size:13px; }

/* ── KPI Stat Row ── */
.stat-row { display: grid; grid-template-columns: repeat(4, 1fr); gap: 16px; }
.stat-card {
  background: var(--grad-surface); border-radius: 12px; border: 1px solid var(--border);
  padding: 20px 18px; box-shadow: 0 4px 20px rgba(0,0,0,0.15);
  display:flex; align-items:center; gap:14px;
}
.stat-icon { width:42px; height:42px; border-radius:10px; display:flex; align-items:center; justify-content:center; flex-shrink:0; }
.stat-icon svg { width:20px; height:20px; }
.icon--blue  { background: rgba(62,146,239,0.15);  color: var(--blue-400);  border: 1px solid rgba(62,146,239,0.3);  box-shadow: 0 0 14px rgba(62,146,239,0.15); }
.icon--amber { background: rgba(245,166,35,0.15);  color: var(--amber-600); border: 1px solid rgba(245,166,35,0.3);  box-shadow: 0 0 14px rgba(245,166,35,0.15); }
.icon--red   { background: rgba(224,53,53,0.15);   color: var(--red-600);   border: 1px solid rgba(224,53,53,0.3);   box-shadow: 0 0 14px rgba(224,53,53,0.15); }
.icon--green { background: rgba(46,178,83,0.15);   color: var(--green-600); border: 1px solid rgba(46,178,83,0.3);  box-shadow: 0 0 14px rgba(46,178,83,0.15); }
.stat-body { display:flex; flex-direction:column; }
.stat-lbl { font-size: 10px; color: var(--text-3); text-transform: uppercase; letter-spacing:.1em; font-weight: 600; }
.stat-val { font-size: 30px; font-weight: 700; color: var(--text); margin-top: 4px; line-height:1; }
.stat-val.ok   { color: var(--green-600); }
.stat-val.warn { color: var(--amber-600); }
.stat-val.crit { color: var(--red-600); }
.stat-sub { font-size: 11px; color: var(--text-4); margin-top: 5px; }

/* ── Panel / Table ── */
.list-panel { background: var(--grad-surface); border: 1px solid var(--border); border-radius: 12px; overflow: hidden; }
.panel-header { display:flex; align-items:center; justify-content:space-between; padding:16px 20px; border-bottom:1px solid var(--border); }
.panel-title { font-size:14px; font-weight:600; color:var(--text); }
.panel-badge-count { font-size:11px; font-weight:600; color: var(--text-3); background: rgba(0,0,0,0.05); padding:3px 10px; border-radius:999px; border:1px solid var(--border-light); }

.table-wrap { overflow-x: auto; }
.report-table { width:100%; border-collapse:collapse; font-size:13px; }
.report-table th { text-align:left; padding:11px 16px; border-bottom:1px solid var(--border-light); font-size:10px; color: var(--text-3); text-transform:uppercase; letter-spacing:.06em; white-space:nowrap; }
.report-table td { padding:11px 16px; border-bottom:1px solid rgba(0,0,0,0.05); color: var(--text-2); }
.report-table tbody tr:hover td { background: rgba(0,0,0,0.02); }
.report-table tbody tr:last-child td { border-bottom:none; }

.td-time    { font-family:'JetBrains Mono', monospace; font-size:11px; color: var(--text-3); white-space:nowrap; }
.td-details { font-size:11px; color: var(--text-3); max-width:280px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }

.host-badge  { font-family:'JetBrains Mono'; background:rgba(62,146,239,0.1); color: var(--blue-400); padding:3px 8px; border-radius:4px; border:1px solid rgba(62,146,239,0.2); font-size:11px; }
.label-badge { background:rgba(0,0,0,0.05); color: var(--text-2); padding:3px 8px; border-radius:4px; font-size:11px; font-weight:600; border:1px solid var(--border-light); }

.chip       { font-size:10px; font-weight:600; padding:3px 10px; border-radius:999px; display:inline-block; border:1px solid; }
.chip--ok   { background: rgba(46,178,83,0.1);  color: var(--green-600); border-color:rgba(46,178,83,0.3);  }
.chip--warn { background: rgba(245,166,35,0.1); color: var(--amber-600); border-color:rgba(245,166,35,0.3); }
.chip--crit { background: rgba(224,53,53,0.1);  color: var(--red-600);   border-color:rgba(224,53,53,0.3);  }
.chip--off  { background: rgba(148,163,184,0.1); color: var(--gray-400); border-color:rgba(148,163,184,0.2); }

.state-pill { font-size:11px; font-weight:600; }
.arrow      { color: var(--text-4); margin: 0 6px; }
.tc-ok   { color: var(--green-600); }
.tc-warn { color: var(--amber-600); }
.tc-crit { color: var(--red-600); }
.tc-off  { color: var(--gray-400); }

.empty-log { padding:40px; text-align:center; color: var(--text-4); font-size:13px; display:flex; flex-direction:column; align-items:center; gap:12px; }

/* ── History Layout ── */
.history-layout { display:flex; flex:1; min-height:0; height:calc(100vh - 64px); }

/* Sidebar */
.day-sidebar {
  width: 220px; flex-shrink:0; background: var(--surface); border-right:1px solid var(--border);
  overflow-y:auto; display:flex; flex-direction:column;
}
.sidebar-title { padding:16px; font-size:11px; font-weight:700; color: var(--text-3); text-transform:uppercase; letter-spacing:.1em; border-bottom:1px solid var(--border); flex-shrink:0; }
.day-item {
  padding:12px 16px; cursor:pointer; border-bottom:1px solid rgba(0,0,0,0.03);
  transition: background .15s;
}
.day-item:hover { background: rgba(0,0,0,0.04); }
.day-item.active { background: rgba(62,146,239,0.1); border-left: 2px solid var(--blue-400); }
.day-date { font-size:13px; font-weight:600; color: var(--text); }
.day-item.active .day-date { color: var(--blue-400); }
.day-meta { display:flex; gap:6px; margin-top:5px; flex-wrap:wrap; }
.day-chip { font-size:10px; background:rgba(0,0,0,0.04); color: var(--text-3); padding:2px 7px; border-radius:999px; border:1px solid var(--border-light); }
.day-chip--ev { background:rgba(245,166,35,0.1); color: var(--amber-600); border-color:rgba(245,166,35,0.3); }

/* Day Main */
.day-main { flex:1; overflow-y:auto; padding:28px 32px; display:flex; flex-direction:column; }
.day-main-header { display:flex; align-items:center; gap:14px; margin-bottom:20px; }
.day-main-title { font-size:20px; font-weight:700; color:var(--text); }
.day-events-pill { margin-left: auto; }

/* Server Snapshot Grid */
.server-grid { display:grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap:14px; }
.srv-snap-card {
  background: var(--grad-surface); border-radius:12px; border:1px solid var(--border);
  padding:16px; display:flex; flex-direction:column; gap:10px;
  box-shadow: 0 4px 16px rgba(0,0,0,0.15);
  transition: transform .15s, box-shadow .15s;
}
.srv-snap-card:hover { transform:translateY(-2px); box-shadow: 0 8px 24px rgba(0,0,0,0.25); }
.snap--ok     { border-top: 3px solid var(--green-600); }
.snap--warn   { border-top: 3px solid var(--amber-600); }
.snap--crit   { border-top: 3px solid var(--red-600); }
.snap--offline{ border-top: 3px solid var(--gray-400); opacity:.7; }

.snap-top { display:flex; flex-direction:column; gap:2px; }
.snap-label { font-size:14px; font-weight:700; color:var(--text); }
.snap-host  { font-size:10px; font-family:'JetBrains Mono'; color: var(--text-3); }

/* Hourly Grid */
.status-grid-panel { background: rgba(0,0,0,0.01); }
.hourly-grid-wrap { overflow-x: auto; padding: 0 4px 12px; }
.hourly-table { width: 100%; border-collapse: separate; border-spacing: 2px; }
.hourly-table th { font-size: 9px; font-weight: 700; color: var(--text-4); text-align: center; padding: 4px; }
.th-h { width: 28px; }
.td-srv-name { font-size: 11px; font-weight: 600; color: var(--text-2); padding: 4px 12px; white-space: nowrap; width: 140px; }
.h-cell { height: 18px; border-radius: 3px; background: rgba(0,0,0,0.04); transition: transform .1s; }
.h-cell:hover { transform: scale(1.1); filter: brightness(1.2); cursor: help; }
.cell--ok   { background: var(--green-600); box-shadow: 0 0 5px rgba(46,178,83,0.3); }
.cell--warn { background: var(--amber-600); box-shadow: 0 0 5px rgba(245,166,35,0.3); }
.cell--crit { background: var(--red-600);   box-shadow: 0 0 8px rgba(224,53,53,0.5); }
.cell--off  { background: #475569; }
.cell--empty{ background: rgba(0,0,0,0.01); border: 1px dashed rgba(0,0,0,0.05); }

/* Metrics Trends */
.metric-controls { display: flex; gap: 20px; align-items: center; margin-bottom: 20px; padding: 16px; background: rgba(0,0,0,0.02); border-radius: 10px; border: 1px solid var(--border); }
.control-group { display: flex; align-items: center; gap: 10px; }
.control-group label { font-size: 11px; font-weight: 700; color: var(--text-4); text-transform: uppercase; }
.control-group select { background: var(--surface); border: 1px solid var(--border); color: var(--text); padding: 6px 10px; border-radius: 6px; font-size: 13px; outline: none; }
.metrics-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; }
.chart-card { background: var(--grad-surface); border: 1px solid var(--border); border-radius: 12px; padding: 20px; box-shadow: 0 8px 32px rgba(0,0,0,0.2); }
.chart-header { margin-bottom: 16px; }
.chart-title { font-size: 14px; font-weight: 600; color: var(--text); }
.chart-sub { font-size: 11px; color: var(--text-4); margin-top: 2px; }
.chart-container { height: 240px; position: relative; }

.snap-status { display:flex; align-items:center; justify-content:space-between; }

/* ── Timeline & Modal Styles ── */
.timeline-controls {
  display: flex; justify-content: space-between; align-items: center;
  background: rgba(0,0,0,0.02); border: 1px solid var(--border);
  padding: 12px 20px; border-radius: 12px; margin-bottom: 16px;
}
.search-box { display: flex; align-items: center; gap: 10px; flex: 1; max-width: 400px; }
.search-box input {
  background: transparent; border: none; color: var(--text); font-size: 13px; width: 100%; outline: none;
}
.search-box svg { color: var(--text-4); }
.timeline-stats { font-size: 11px; color: var(--text-4); font-weight: 500; }

.td-srv-info { display: flex; flex-direction: column; gap: 4px; }
.host-badge-small { font-family: 'JetBrains Mono'; font-size: 10px; color: var(--text-4); }

.snap-health-row { display: flex; align-items: center; gap: 8px; }
.health-dot { width: 8px; height: 8px; border-radius: 50%; }
.health-dot.tc-ok { background: var(--green-600); box-shadow: 0 0 8px var(--green-600); }
.health-dot.tc-warn { background: var(--amber-600); box-shadow: 0 0 8px var(--amber-600); }
.health-dot.tc-crit { background: var(--red-600); box-shadow: 0 0 8px var(--red-600); }

.btn-inspect {
  background: rgba(62,146,239,0.1); color: var(--blue-400); border: 1px solid rgba(62,146,239,0.2);
  padding: 6px 12px; border-radius: 6px; font-size: 11px; font-weight: 600;
  cursor: pointer; transition: all .2s;
}
.btn-inspect:hover { background: var(--blue-400); color: #fff; }

/* Modal */
.modal-overlay {
  position: fixed; top: 0; left: 0; width: 100%; height: 100%;
  background: rgba(255,255,255,0.85); backdrop-filter: blur(8px);
  z-index: 1000; display: flex; align-items: center; justify-content: center; padding: 40px;
}
.modal-content {
  background: var(--surface); border: 1px solid var(--border); border-radius: 20px;
  width: 100%; max-width: 1000px; max-height: 90vh; overflow-y: auto;
  box-shadow: 0 20px 60px rgba(0,0,0,0.1); display: flex; flex-direction: column;
}
.modal-header { padding: 24px 32px; border-bottom: 1px solid var(--border); background: rgba(0,0,0,0.02); }
.mh-top { display: flex; justify-content: space-between; align-items: center; }
.mh-title { font-size: 18px; font-weight: 700; color: var(--text); }
.mh-sub { font-size: 12px; color: var(--text-4); margin-top: 4px; }
.btn-close { background: transparent; border: none; color: var(--text-3); font-size: 24px; cursor: pointer; line-height: 1; }
.btn-close:hover { color: var(--text); }

.modal-body { padding: 32px; }
.snap-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; }
.panel--glow { border-color: rgba(62,146,239,0.2); box-shadow: 0 0 20px rgba(62,146,239,0.05); }

.info-grid { display: flex; flex-direction: column; gap: 12px; padding-top: 10px; }
.info-row { display: flex; justify-content: space-between; align-items: center; font-size: 13px; }
.info-lbl { color: var(--text-4); }

@media (max-width: 768px) {
  .snap-grid { grid-template-columns: 1fr; }
  .modal-overlay { padding: 10px; }
}

/* Fleet Summary in Reports */
.fleet-status-summary { display: flex; gap: 20px; font-size: 12px; color: var(--text-3); }
.fss-item strong { color: var(--text); margin-right: 4px; }
.fss-item.crit strong { color: var(--red-600); }

/* Audit Logs */
.audit-section { margin-top: 40px; }
.section-divider {
  display: flex; align-items: center; gap: 16px; margin-bottom: 24px;
}
.section-divider::after { content: ''; flex: 1; height: 1px; background: var(--border); }
.section-divider span { font-size: 11px; font-weight: 700; color: var(--text-4); letter-spacing: .15em; }

.audit-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); gap: 16px; }
.audit-card {
  background: rgba(0,0,0,0.02); border: 1px solid var(--border); border-radius: 12px;
  padding: 16px; display: flex; flex-direction: column; gap: 12px;
  transition: transform .2s;
}
.audit-card:hover { transform: scale(1.02); background: rgba(0,0,0,0.04); }
.audit-header { display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid var(--border-light); padding-bottom: 8px; }
.audit-time { font-size: 10px; font-family: 'JetBrains Mono'; color: var(--text-4); }
.audit-type { font-size: 10px; font-weight: 700; text-transform: uppercase; color: var(--blue-400); }

.audit-srv { font-size: 13px; font-weight: 700; color: var(--text); }
.audit-srv small { color: var(--text-4); font-weight: 400; margin-left: 4px; }
.audit-msg { font-size: 11px; color: var(--text-3); line-height: 1.4; margin-top: 4px; }

.audit-footer { display: flex; align-items: center; gap: 8px; margin-top: auto; padding-top: 8px; }
.audit-pill { font-size: 10px; font-weight: 600; padding: 2px 8px; border-radius: 4px; background: rgba(0,0,0,0.04); }
.audit-arrow { color: var(--text-4); font-size: 10px; }

.audit--critical { border-left: 3px solid var(--red-600); }
.audit--warning { border-left: 3px solid var(--amber-600); }
.audit--info { border-left: 3px solid var(--blue-400); }

.snap-health { font-size:15px; font-weight:700; }
.snap-pills { display:flex; gap:5px; flex-wrap:wrap; }
.snap-pill  { font-size:10px; font-weight:600; padding:2px 8px; border-radius:999px; }
.snap-pill.ok  { background:rgba(46,178,83,0.15);  color: var(--green-600); border:1px solid rgba(46,178,83,0.3); }
.snap-pill.off { background:rgba(148,163,184,0.1); color: var(--gray-400); border:1px solid rgba(148,163,184,0.2); }

.snap-status.offline { flex-direction:column; align-items:flex-start; gap:4px; }
.snap-error { font-size:10px; color: var(--text-4); font-style:italic; }

.snap-metrics { display:flex; flex-direction:column; gap:5px; padding-top:6px; border-top:1px solid rgba(0,0,0,0.05);}
.snap-metric  { display:flex; justify-content:space-between; font-size:11px; }
.sm-lbl { color: var(--text-3); }
.sm-val { font-family:'JetBrains Mono'; font-weight:600; color: var(--text); }
.sm-val.warn  { color: var(--amber-600); }
.sm-val.tc-ok  { color: var(--green-600); }
.sm-val.tc-crit{ color: var(--red-600); }
.sm-val.tc-warn{ color: var(--amber-600); }

.snap-time { font-size:10px; color: var(--text-4); text-align:right; font-family:'JetBrains Mono'; }

/* ── Button variants ── */
.btn { display:inline-flex; align-items:center; padding:7px 14px; border-radius:8px; font-size:13px; font-weight:500; cursor:pointer; border:1px solid transparent; transition:all .2s; font-family:inherit; }
.btn--primary { background: var(--blue-400); color: #fff; }
.btn--primary:hover { filter:brightness(1.1); }
.btn--ghost { background: rgba(0,0,0,0.04); color: var(--text-2); border-color: var(--border); }
.btn--ghost:hover { background: rgba(0,0,0,0.08); color:var(--text); }
.btn:disabled { opacity:.5; cursor:not-allowed; }
.spin { display:inline-block; animation: spin 1s linear infinite; }
@keyframes spin { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }

/* ── Responsive ── */
@media (max-width: 1100px) { .stat-row { grid-template-columns: repeat(2,1fr); } }
@media (max-width: 768px) {
  .tab-bar { display:none; }
  .history-layout { flex-direction:column; }
  .day-sidebar { width:100%; height:120px; flex-direction:row; overflow-x:auto; border-right:none; border-bottom:1px solid var(--border); }
  .day-main { padding:20px; }
  .server-grid { grid-template-columns:1fr 1fr; }
}
@media (max-width: 500px) { .stat-row { grid-template-columns:1fr; } .server-grid { grid-template-columns:1fr; } }
</style>
