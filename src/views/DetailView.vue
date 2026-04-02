<template>
  <div class="dp">

    <!-- Topbar -->
    <header class="topbar">
      <button class="btn-back" @click="$emit('back')">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"><polyline points="15 18 9 12 15 6"/></svg>
        Dashboard
      </button>
      <div class="topbar-mid" v-if="data">
        <span class="srv-name">{{ server.label }}</span>
        <span class="srv-ip mono">{{ server.host }}</span>
        <span class="health-badge" :class="'hb--' + healthCls(data.summary?.health)">
          <span class="hb-dot"></span>{{ healthLabel(data.summary?.health) }}
        </span>
      </div>
      <div class="topbar-right">
        <span class="last-update" v-if="lastUpdate">↺ {{ lastUpdate }}</span>
        <button class="btn-sync" @click="load" :disabled="loading">
          <span :class="{ spin: loading }">↻</span> {{ loading ? 'Cargando…' : 'Sincronizar' }}
        </button>
      </div>
    </header>

    <!-- Error Banner -->
    <div class="err-bar" v-if="error">
      ⚠ {{ error }}
      <button @click="load" class="err-btn">Reintentar</button>
    </div>

    <!-- Loader -->
    <div class="page-loader" v-if="loading && !data">
      <div class="loader-ring"></div>
      <div>Conectando con {{ server.host }}…</div>
    </div>

    <main class="main-grid" v-if="data">

      <!-- ─── ROW 1: HERO STATS (KEEP AS REQUESTED) ─── -->
      <div class="hero-row">
        <div class="hero-card" :class="'hc--' + healthCls(data.summary?.health)">
          <div class="hc-icon">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M22 12h-4l-3 9L9 3l-3 9H2"/></svg>
          </div>
          <div class="hc-body">
            <div class="hc-lbl">Salud del Sistema</div>
            <div class="hc-val">{{ healthLabel(data.summary?.health) }}</div>
            <div class="hc-sub">{{ data.summary?.model }}</div>
          </div>
        </div>

        <div class="hero-card" :class="data.summary?.power_state === 'On' ? 'hc--ok' : 'hc--crit'">
          <div class="hc-icon">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M18.36 6.64a9 9 0 1 1-12.73 0M12 2v10"/></svg>
          </div>
          <div class="hc-body">
            <div class="hc-lbl">Estado de Energía</div>
            <div class="hc-val">{{ data.summary?.power_state ?? '—' }}</div>
            <div class="hc-sub">{{ data.summary?.power_state === 'On' ? 'Sistema Activo' : 'Apagado' }}</div>
          </div>
        </div>

        <div class="hero-card hc--blue">
          <div class="hc-icon">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"/></svg>
          </div>
          <div class="hc-body">
            <div class="hc-lbl">Consumo Activo</div>
            <div class="hc-val">{{ data.power?.consumed_watts ?? '—' }}W</div>
            <div class="hc-sub">Cap. {{ data.power?.capacity_watts ?? '?' }} W</div>
          </div>
        </div>

        <div class="hero-card" :class="'hc--' + ambientTempCls">
          <div class="hc-icon">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M14 14.76V3.5a2.5 2.5 0 0 0-5 0v11.26a4.5 4.5 0 1 0 5 0z"/></svg>
          </div>
          <div class="hc-body">
            <div class="hc-lbl">Temperatura Ambiente</div>
            <div class="hc-val">{{ ambientTemp !== null ? ambientTemp + ' °C' : '—' }}</div>
            <div class="hc-sub">{{ ambientTempName }}</div>
          </div>
        </div>
      </div>

      <!-- ─── ROW 2: CPU + RAM + POWER ─── -->
      <div class="triple-grid">

        <!-- CPU PANEL -->
        <div class="panel">
          <div class="panel-head">
            <div class="ph-icon ph-blue">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="4" y="4" width="16" height="16" rx="2"/><rect x="9" y="9" width="6" height="6"/><path d="M15 2v2M9 2v2M15 20v2M9 20v2M20 15h2M20 9h2M2 15h2M2 9h2"/></svg>
            </div>
            <div>
              <div class="ph-title">PROCESADOR</div>
              <div class="ph-sub">{{ data.summary?.cpu_count ?? '?' }} NÚCLEOS FÍSICOS</div>
            </div>
            <div class="ph-badge" :class="cpuStatusCls === 'tc--ok' ? 'b--blue' : cpuStatusCls === 'tc--warn' ? 'b--amber' : 'b--rose'">
              {{ maxCpuReading }}°C
            </div>
          </div>

          <div class="panel-body">
            <!-- CPU Thermal Arc -->
            <div class="cpu-arc-wrap">
              <svg viewBox="0 0 200 110" class="cpu-arc-svg">
                <!-- Background arc -->
                <path d="M 20 100 A 80 80 0 0 1 180 100" fill="none" stroke="#f1f5f9" stroke-width="12" stroke-linecap="round"/>
                <!-- Filled arc -->
                <path d="M 20 100 A 80 80 0 0 1 180 100" fill="none"
                  :stroke="cpuArcColor" stroke-width="12" stroke-linecap="round"
                  :stroke-dasharray="`${(maxCpuReading / 100) * 251.3} 251.3`"/>
                <!-- Temp label -->
                <text x="100" y="92" text-anchor="middle" font-size="28" font-weight="900" :fill="cpuArcColor" font-family="system-ui">{{ maxCpuReading }}</text>
                <text x="100" y="108" text-anchor="middle" font-size="11" fill="#94a3b8" font-family="system-ui">°C NÚCLEO</text>
              </svg>
            </div>

            <!-- CPU Spec tiles -->
            <div class="spec-grid">
              <div class="spec-tile">
                <div class="st-val">{{ data.summary?.cpu_count ?? '—' }}</div>
                <div class="st-lbl">CPU Sockets</div>
              </div>
              <div class="spec-tile">
                <div class="st-val">{{ (data.summary?.cpu_count ?? 0) * 10 }}</div>
                <div class="st-lbl">Hilos Totales</div>
              </div>
              <div class="spec-tile st-wide">
                <div class="st-val st-mono" style="font-size: 11px">{{ data.summary?.cpu_model || 'Intel Xeon Scalable' }}</div>
                <div class="st-lbl">Modelo CPU</div>
              </div>
            </div>

            <!-- Thermal sensors for CPU -->
            <div class="thermal-chips" v-if="cpuTempSensors.length">
              <div class="thermal-chip" v-for="s in cpuTempSensors" :key="s.name">
                <span class="tc-dot" :class="'tcd--' + tempClsStr(s)"></span>
                <span class="tc-name">{{ s.name.replace('CPU ', '') }}</span>
                <span class="tc-val" :class="'tcv--' + tempClsStr(s)">{{ s.reading_c }}°</span>
              </div>
            </div>
          </div>

          <div class="panel-foot">
            <span class="foot-label">S/N SISTEMA</span>
            <span class="foot-val mono">{{ data.summary?.serial || '—' }}</span>
          </div>
        </div>

        <!-- RAM PANEL -->
        <div class="panel">
          <div class="panel-head">
            <div class="ph-icon ph-emerald">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M6 12h12M6 8h12M6 16h12M3 3v18h18V3H3z"/></svg>
            </div>
            <div>
              <div class="ph-title">MEMORIA RAM</div>
              <div class="ph-sub">{{ data.summary?.memory_gib ?? '?' }} GB · {{ data.memory?.dimms?.length ?? 0 }} MÓDULOS</div>
            </div>
          </div>

          <div class="panel-body">
            <!-- DIMM Topology -->
            <div class="dimm-grid">
              <div class="dimm-card" v-for="d in data.memory?.dimms" :key="d.name" :class="'dc--' + dimmCls(d.health)">
                <div class="dc-bar"></div>
                <div class="dc-content">
                  <div class="dc-name">{{ d.name }}</div>
                  <div class="dc-cap">{{ dimmGb(d) }} <span class="dc-unit">GB</span></div>
                  <div class="dc-speed">{{ d.speed_mhz }} MHz</div>
                  <div class="dc-type">{{ d.type }}</div>
                </div>
                <div class="dc-status" :class="'dcs--' + dimmCls(d.health)">{{ d.health }}</div>
              </div>
            </div>
          </div>

          <div class="panel-foot">
            <span class="foot-label">VELOCIDAD</span>
            <span class="foot-val">{{ data.memory?.dimms?.[0]?.speed_mhz ?? '—' }} MHz</span>
            <span class="foot-label">TIPO</span>
            <span class="foot-val">{{ data.memory?.dimms?.[0]?.type ?? 'DDR4' }}</span>
          </div>
        </div>

        <!-- POWER PANEL -->
        <div class="panel">
          <div class="panel-head">
            <div class="ph-icon ph-amber">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"/></svg>
            </div>
            <div>
              <div class="ph-title">FUENTE DE PODER</div>
              <div class="ph-sub">{{ powerPct.toFixed(0) }}% CARGA ACTUAL</div>
            </div>
          </div>

          <div class="panel-body">
            <!-- Power arc gauge -->
            <div class="cpu-arc-wrap">
              <svg viewBox="0 0 200 110" class="cpu-arc-svg">
                <path d="M 20 100 A 80 80 0 0 1 180 100" fill="none" stroke="#f1f5f9" stroke-width="12" stroke-linecap="round"/>
                <path d="M 20 100 A 80 80 0 0 1 180 100" fill="none"
                  :stroke="powerPct > 85 ? '#f43f5e' : powerPct > 65 ? '#f59e0b' : '#3b82f6'" stroke-width="12" stroke-linecap="round"
                  :stroke-dasharray="`${(powerPct / 100) * 251.3} 251.3`"/>
                <text x="100" y="92" text-anchor="middle" font-size="28" font-weight="900" :fill="powerPct > 85 ? '#f43f5e' : '#1e293b'" font-family="system-ui">{{ data.power?.consumed_watts ?? '—' }}</text>
                <text x="100" y="108" text-anchor="middle" font-size="11" fill="#94a3b8" font-family="system-ui">WATTS ACTIVOS</text>
              </svg>
            </div>

            <!-- PSU list -->
            <div class="psu-list">
              <div class="psu-row" v-for="p in data.power?.power_supplies" :key="p.name">
                <div class="psu-dot" :class="p.health === 'OK' ? 'pd--ok' : 'pd--warn'"></div>
                <span class="psu-name">{{ p.name }}</span>
                <span class="psu-watts mono">{{ p.power_watts ?? '—' }}W</span>
                <span class="psu-health" :class="p.health === 'OK' ? 's--ok' : 's--warn'">{{ p.health }}</span>
              </div>
            </div>
          </div>

          <div class="panel-foot">
            <span class="foot-label">CAPACIDAD</span>
            <span class="foot-val">{{ data.power?.capacity_watts ?? '—' }} W</span>
          </div>
        </div>
      </div>

      <!-- ─── ROW 3: TEMPS + FANS ─── -->
      <div class="double-grid">
        <!-- Temps -->
        <div class="panel">
          <div class="panel-head">
            <div class="ph-icon ph-rose">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M14 14.76V3.5a2.5 2.5 0 0 0-5 0v11.26a4.5 4.5 0 1 0 5 0z"/></svg>
            </div>
            <div>
              <div class="ph-title">MAPA TÉRMICO</div>
              <div class="ph-sub">{{ groupedTemperatures.length }} RESÚMENES DE SENSORES</div>
            </div>
          </div>
          <div class="panel-body">
            <div class="temp-grid">
              <div class="temp-card" v-for="g in groupedTemperatures" :key="g.name" :class="'tc--' + g.status">
                <div class="temp-val">{{ g.reading !== null ? Math.round(g.reading) + '°' : 'N/A' }}</div>
                <div class="temp-name">{{ g.name }}</div>
                <div class="temp-tag" v-if="g.isGroup">Promedio</div>
                <div class="temp-thresholds" v-if="!g.isGroup && (g.caution || g.critical)">
                  <span v-if="g.caution">C:{{ g.caution }}°</span>
                  <span v-if="g.critical">K:{{ g.critical }}°</span>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- Fans -->
        <div class="panel">
          <div class="panel-head">
            <div class="ph-icon ph-blue">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="12" r="3"/><path d="M12 9V3m0 0a3 3 0 0 1 3 3v3h-6V6a3 3 0 0 1 3-3zM12 15v6m0 0a3 3 0 0 1-3-3v-3h6v3a3 3 0 0 1-3 3zM9 12H3m0 0a3 3 0 0 0 3 3h3v-6H6a3 3 0 0 0-3 3zM15 12h6m0 0a3 3 0 0 0-3-3h-3v6h3a3 3 0 0 0 3-3z"/></svg>
            </div>
            <div>
              <div class="ph-title">VENTILACIÓN</div>
              <div class="ph-sub">{{ data.fans?.length ?? 0 }} VENTILADORES</div>
            </div>
          </div>
          <div class="panel-body">
            <div class="fan-grid">
              <div class="fan-card" v-for="f in data.fans" :key="f.name">
                <div class="fan-rotor" :class="{ 'fan-spin': (f.rpm ?? 0) > 0 }" :style="{ animationDuration: fanDuration(f.rpm) }">
                  <svg viewBox="0 0 24 24" fill="currentColor"><path d="M12 12c0 0 0 0 0 0c1.1 0 2-0.9 2-2c0-1.1-0.9-2-2-2s-2 0.9-2 2C10 11.1 10.9 12 12 12z M12 2C6.5 2 2 6.5 2 12s4.5 10 10 10s10-4.5 10-10S17.5 2 12 2z M12 4c4.4 0 8 3.6 8 8s-3.6 8-8 8s-8-3.6-8-8S7.6 4 12 4z"/></svg>
                </div>
                <div class="fan-info">
                  <div class="fan-name">{{ f.name }}</div>
                  <div class="fan-rpm mono" :class="f.health !== 'OK' ? 'rpm--warn' : ''">
                    {{ f.rpm ?? 0 }} <span class="rpm-unit">RPM</span>
                  </div>
                </div>
                <div class="fan-dot" :class="f.health === 'OK' ? 'fd--ok' : 'fd--warn'"></div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- ─── ROW 4: STORAGE + SYSTEM INFO ─── -->
      <div class="double-grid">
        <!-- Storage -->
        <div class="panel" v-if="data.storage?.controllers?.length">
          <div class="panel-head">
            <div class="ph-icon ph-slate">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="2" y="2" width="20" height="8" rx="2"/><rect x="2" y="14" width="20" height="8" rx="2"/><line x1="6" y1="6" x2="6.01" y2="6"/><line x1="6" y1="18" x2="6.01" y2="18"/></svg>
            </div>
            <div>
              <div class="ph-title">ALMACENAMIENTO</div>
              <div class="ph-sub">{{ data.storage?.controllers?.[0]?.name }}</div>
            </div>
          </div>
          <div class="panel-body">
            <div class="drive-list">
              <div class="drive-row" v-for="d in data.storage?.controllers?.[0]?.drives" :key="d.name">
                <div class="drive-type-badge" :class="d.type === 'SSD' ? 'dtb--ssd' : 'dtb--hdd'">{{ d.type || 'HDD' }}</div>
                <div class="drive-info">
                  <div class="drive-name mono">{{ d.name }}</div>
                  <div class="drive-model">{{ d.model }}</div>
                </div>
                <div class="drive-cap">{{ d.capacity_gb }} GB</div>
                <div class="drive-health" :class="d.health === 'OK' ? 's--ok' : 's--warn'">{{ d.health }}</div>
              </div>
            </div>
          </div>
        </div>

        <!-- System Footprint -->
        <div class="panel">
          <div class="panel-head">
            <div class="ph-icon ph-slate">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M12 22s8-4 8-10V5l-8-3-8 3v7c0 6 8 10 8 10z"/></svg>
            </div>
            <div>
              <div class="ph-title">FICHA DEL SISTEMA</div>
              <div class="ph-sub">HPE PROLIANT · iLO 5</div>
            </div>
          </div>
          <div class="panel-body">
            <div class="info-table">
              <div class="it-row">
                <span class="it-lbl">Modelo</span>
                <span class="it-val">{{ data.summary?.model ?? 'N/A' }}</span>
              </div>
              <div class="it-row">
                <span class="it-lbl">Número de Serie</span>
                <span class="it-val mono">{{ data.summary?.serial ?? 'N/A' }}</span>
              </div>
              <div class="it-row">
                <span class="it-lbl">Versión BIOS</span>
                <span class="it-val mono">{{ data.summary?.bios_version ?? 'N/A' }}</span>
              </div>
              <div class="it-row">
                <span class="it-lbl">IP iLO</span>
                <span class="it-val mono">{{ server.host }}</span>
              </div>
              <div class="it-row">
                <span class="it-lbl">CPUs</span>
                <span class="it-val">{{ data.summary?.cpu_count ?? '—' }} Procesadores Instalados</span>
              </div>
              <div class="it-row">
                <span class="it-lbl">Memoria Total</span>
                <span class="it-val">{{ data.summary?.memory_gib ?? '—' }} GB DDR4</span>
              </div>
            </div>
          </div>
        </div>
      </div>

    </main>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { useIlo } from '../composables/useIlo.js'
import { REFRESH_INTERVAL_SEC } from '../config/servers.js'

const props = defineProps({ server: Object })
defineEmits(['back'])
const { fetchAll } = useIlo()

const data       = ref(null)
const loading    = ref(true)
const error      = ref(null)
const lastUpdate = ref('')
let autoTimer    = null

// ── Computed ───────────────────────────────────────────────
const ambientTempSensor = computed(() => {
  const sensors = data.value?.temperatures ?? []
  if (!sensors.length) return null
  
  // Buscar sensor Inlet Ambient
  const found = sensors.find(s => {
    const n = (s.name || "").toLowerCase()
    return n.includes("inlet") || n.includes("ambient")
  })
  return found || null
})

const ambientTemp = computed(() => {
  if (ambientTempSensor.value && ambientTempSensor.value.reading_c != null) {
    return ambientTempSensor.value.reading_c
  }
  // Fallback: Max temp
  const vals = (data.value?.temperatures ?? []).map(t => t.reading_c).filter(v => v != null)
  return vals.length ? Math.max(...vals) : null
})

const ambientTempName = computed(() => {
  if (ambientTempSensor.value) return ambientTempSensor.value.name
  
  // Fallback: Nombre del sensor con temp máxima
  if (!data.value?.temperatures?.length) return 'Sin sensores'
  const s = data.value.temperatures.reduce((a, b) => (b.reading_c ?? -999) > (a.reading_c ?? -999) ? b : a)
  return s?.name ?? 'Sensor de reserva'
})

const ambientTempCls = computed(() => {
  if (ambientTemp.value === null) return ''
  if (ambientTemp.value > 80) return 'crit'
  if (ambientTemp.value > 65) return 'warn'
  return 'ok'
})

const groupedTemperatures = computed(() => {
  const sensors = data.value?.temperatures ?? []
  if (!sensors.length) return []

  const groups = {
    cpu_vrm: { name: 'Reguladores CPU (VRM)', sensors: [], isGroup: true },
    mem_vrm: { name: 'Reguladores RAM (VRM)', sensors: [], isGroup: true },
    storage: { name: 'Área Almacenamiento', sensors: [], isGroup: true },
    others:  []
  }

  sensors.forEach(s => {
    const name = (s.name || "").toLowerCase()
    
    // Clasificar inteligentemente
    if ((name.includes('vr') || name.includes('voltage')) && (name.includes('p1') || name.includes('p2') || name.includes('cpu')) && !name.includes('mem')) {
      groups.cpu_vrm.sensors.push(s)
    } 
    else if ((name.includes('vr') || name.includes('dimm')) && (name.includes('mem') || name.includes('ram'))) {
      groups.mem_vrm.sensors.push(s)
    }
    else if (name.includes('hpp') || name.includes('drive') || name.includes('disk') || name.includes('storage')) {
      groups.storage.sensors.push(s)
    }
    else {
      groups.others.push({
        name: s.name,
        reading: s.reading_c,
        status: tempClsStr(s),
        isGroup: false,
        caution: s.upper_caution,
        critical: s.upper_critical
      })
    }
  })

  const finalizeGroup = (g) => {
    if (!g.sensors.length) return null
    const valid = g.sensors.filter(s => s.reading_c != null)
    if (!valid.length) return null
    
    const avg = valid.reduce((a, b) => a + b.reading_c, 0) / valid.length
    // El estatus del grupo es el del sensor más alto por seguridad
    const worstStatus = valid.map(s => tempClsStr(s)).reduce((a, b) => {
      if (a === 'crit' || b === 'crit') return 'crit'
      if (a === 'warn' || b === 'warn') return 'warn'
      return 'ok'
    }, 'ok')

    return {
      name: g.name,
      reading: avg,
      status: worstStatus,
      isGroup: true,
      count: valid.length
    }
  }

  const result = [
    finalizeGroup(groups.cpu_vrm),
    finalizeGroup(groups.mem_vrm),
    finalizeGroup(groups.storage),
    ...groups.others
  ].filter(x => x !== null)

  return result
})

const cpuTempSensors = computed(() => {
  return (data.value?.temperatures ?? []).filter(t => t.name?.toLowerCase().includes('cpu'))
})

const maxCpuReading = computed(() => {
  const vals = cpuTempSensors.value.map(s => s.reading_c).filter(v => v != null)
  return vals.length ? Math.max(...vals) : 40
})

const cpuStatusCls = computed(() => {
  if (maxCpuReading.value > 75) return 'tc--crit'
  if (maxCpuReading.value > 60) return 'tc--warn'
  return 'tc--ok'
})

const cpuArcColor = computed(() => {
  if (maxCpuReading.value > 75) return '#f43f5e'
  if (maxCpuReading.value > 60) return '#f59e0b'
  return '#3b82f6'
})

const powerPct = computed(() => {
  if (!data.value?.power?.consumed_watts || !data.value?.power?.capacity_watts) return 0
  return Math.min(100, (data.value.power.consumed_watts / data.value.power.capacity_watts) * 100)
})

// ── Helpers ────────────────────────────────────────────────
function healthLabel(h) {
  return { OK: 'Óptimo', Warning: 'Advertencia', Critical: 'Crítico' }[h] ?? h ?? '—'
}
function healthCls(h) {
  return { OK: 'ok', Warning: 'warn', Critical: 'crit' }[h] ?? ''
}
function dimmCls(h) {
  return { OK: 'ok', Warning: 'warn', Critical: 'crit' }[h] ?? 'ok'
}
function dimmGb(d) {
  const mb = d.size_mb ?? d.capacity_mib ?? d.size ?? 0
  return mb ? Math.round(mb / 1024) : '?'
}
function tempClsStr(t) {
  if (!t.reading_c) return 'ok'
  if (t.upper_critical && t.reading_c >= t.upper_critical) return 'crit'
  if (t.upper_caution  && t.reading_c >= t.upper_caution)  return 'warn'
  if (t.reading_c > 85) return 'crit'
  if (t.reading_c > 70) return 'warn'
  return 'ok'
}
function fanDuration(rpm) {
  if (!rpm || rpm === 0) return '0s'
  const dur = Math.max(0.3, 3 - (rpm / 2500))
  return `${dur.toFixed(2)}s`
}

async function load() {
  loading.value = true
  error.value = null
  try {
    const fetched = await fetchAll(props.server)
    data.value = fetched
    lastUpdate.value = new Date().toLocaleTimeString('es-EC', { hour: '2-digit', minute: '2-digit' })
  } catch (e) {
    error.value = `Error al conectar: ${e.message}`
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  load()
  autoTimer = setInterval(load, REFRESH_INTERVAL_SEC * 1000)
})
onUnmounted(() => { clearInterval(autoTimer) })
</script>

<style scoped>
/* ══════════════════════════════════════════
   DESIGN TOKENS — Premium Light System
══════════════════════════════════════════ */
.dp {
  min-height: 100vh;
  background: #f0f2f5;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
  color: #1e293b;
}
.mono { font-family: 'JetBrains Mono', 'Courier New', monospace; }

/* ── TOPBAR ── */
.topbar {
  position: sticky; top: 0; z-index: 100;
  background: rgba(255,255,255,0.92);
  backdrop-filter: blur(24px);
  border-bottom: 1px solid #e2e8f0;
  height: 68px;
  padding: 0 40px;
  display: flex; align-items: center; gap: 20px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.06);
}
.btn-back {
  display: flex; align-items: center; gap: 8px;
  font-size: 13px; font-weight: 600;
  padding: 8px 16px; border-radius: 10px;
  border: 1px solid #e2e8f0; background: white;
  color: #475569; cursor: pointer; transition: all .18s;
  white-space: nowrap;
}
.btn-back:hover { background: #f8fafc; border-color: #cbd5e1; transform: translateX(-2px); }
.topbar-mid { display: flex; align-items: center; gap: 12px; }
.srv-name { font-size: 17px; font-weight: 800; color: #0f172a; letter-spacing: -0.02em; }
.srv-ip { font-size: 11px; color: #2563eb; background: rgba(37,99,235,0.08); padding: 3px 9px; border-radius: 6px; border: 1px solid rgba(37,99,235,0.15); }
.health-badge { display: flex; align-items: center; gap: 6px; font-size: 11px; font-weight: 700; padding: 4px 10px; border-radius: 20px; }
.hb-dot { width: 6px; height: 6px; border-radius: 50%; }
.hb--ok   { background: rgba(16,185,129,0.1); color: #059669; } .hb--ok .hb-dot { background: #10b981; box-shadow: 0 0 6px #10b981; }
.hb--warn { background: rgba(245,158,11,0.1); color: #d97706; } .hb--warn .hb-dot { background: #f59e0b; }
.hb--crit { background: rgba(244,63,94,0.1);  color: #e11d48; } .hb--crit .hb-dot { background: #f43f5e; animation: blink 1s infinite; }
.topbar-right { margin-left: auto; display: flex; align-items: center; gap: 12px; }
.last-update { font-size: 11px; font-weight: 600; color: #94a3b8; }
.btn-sync {
  display: flex; align-items: center; gap: 8px;
  font-size: 13px; font-weight: 600;
  padding: 8px 18px; border-radius: 10px;
  background: #2563eb; color: white; border: none; cursor: pointer; transition: all .18s;
}
.btn-sync:hover { background: #1d4ed8; }
.btn-sync:disabled { opacity: 0.6; cursor: not-allowed; }

/* ── ERROR & LOADER ── */
.err-bar { background: #fef2f2; border-bottom: 1px solid rgba(239,68,68,0.3); padding: 12px 40px; display: flex; align-items: center; gap: 12px; font-size: 13px; font-weight: 600; color: #dc2626; }
.err-btn { margin-left: auto; background: #dc2626; color:white; border: none; padding: 6px 14px; border-radius: 8px; cursor: pointer; font-size: 12px; font-weight: 600; }
.page-loader { display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 16px; height: calc(100vh - 68px); color: #94a3b8; font-size: 14px; font-weight: 500; }
.loader-ring { width: 40px; height: 40px; border: 3px solid #e2e8f0; border-top-color: #3b82f6; border-radius: 50%; animation: spin 1s linear infinite; }

/* ── MAIN LAYOUT ── */
.main-grid { padding: 32px 40px; display: flex; flex-direction: column; gap: 24px; max-width: 1700px; margin: 0 auto; }

/* ── HERO ROW ── */
.hero-row { display: grid; grid-template-columns: repeat(4, 1fr); gap: 20px; }
.hero-card {
  background: white; border-radius: 20px; padding: 24px;
  display: flex; gap: 18px; align-items: flex-start;
  border: 1px solid #e2e8f0;
  box-shadow: 0 2px 8px rgba(0,0,0,0.04);
  transition: transform .2s, box-shadow .2s;
}
.hero-card:hover { transform: translateY(-3px); box-shadow: 0 12px 24px rgba(0,0,0,0.07); }
.hc-icon { width: 48px; height: 48px; border-radius: 14px; display: flex; align-items: center; justify-content: center; flex-shrink: 0; background: #f8fafc; color: #64748b; }
.hc-icon svg { width: 22px; height: 22px; }
.hc--ok   .hc-icon { background: rgba(16,185,129,0.1); color: #10b981; border: 1px solid rgba(16,185,129,0.2); }
.hc--warn .hc-icon { background: rgba(245,158,11,0.1); color: #f59e0b; border: 1px solid rgba(245,158,11,0.2); }
.hc--crit .hc-icon { background: rgba(244,63,94,0.1);  color: #f43f5e; border: 1px solid rgba(244,63,94,0.2); }
.hc--blue .hc-icon { background: rgba(59,130,246,0.1);  color: #3b82f6; border: 1px solid rgba(59,130,246,0.2); }
.hc-lbl { font-size: 10px; font-weight: 700; color: #94a3b8; text-transform: uppercase; letter-spacing: 0.08em; }
.hc-val { font-size: 24px; font-weight: 900; color: #0f172a; margin-top: 4px; line-height: 1; letter-spacing: -0.02em; }
.hc--ok   .hc-val { color: #059669; }
.hc--warn .hc-val { color: #d97706; }
.hc--crit .hc-val { color: #e11d48; }
.hc--blue .hc-val { color: #2563eb; }
.hc-sub { font-size: 11px; color: #94a3b8; font-weight: 500; margin-top: 6px; }

/* ── CONTROL TOWER PANELS ── */
.triple-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 24px; }
.double-grid { display: grid; grid-template-columns: 3fr 2fr; gap: 24px; }

.panel {
  background: white; border-radius: 22px;
  border: 1px solid #e2e8f0;
  box-shadow: 0 4px 12px rgba(0,0,0,0.04);
  overflow: hidden; display: flex; flex-direction: column;
  transition: transform .2s, box-shadow .2s;
}
.panel:hover { transform: translateY(-3px); box-shadow: 0 16px 32px rgba(0,0,0,0.07); }

.panel-head {
  padding: 20px 24px; display: flex; align-items: center; gap: 14px;
  border-bottom: 1px solid #f1f5f9;
  background: linear-gradient(135deg, #fafbfc 0%, #ffffff 100%);
}
.ph-icon { width: 42px; height: 42px; border-radius: 12px; display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
.ph-icon svg { width: 20px; height: 20px; }
.ph-blue  { background: rgba(59,130,246,0.1);  color: #2563eb; border: 1px solid rgba(59,130,246,0.2); }
.ph-emerald { background: rgba(16,185,129,0.1); color: #059669; border: 1px solid rgba(16,185,129,0.2); }
.ph-amber { background: rgba(245,158,11,0.1); color: #d97706; border: 1px solid rgba(245,158,11,0.2); }
.ph-rose  { background: rgba(244,63,94,0.1);  color: #e11d48; border: 1px solid rgba(244,63,94,0.2); }
.ph-slate { background: rgba(100,116,139,0.1); color: #475569; border: 1px solid rgba(100,116,139,0.2); }

.ph-title { font-size: 13px; font-weight: 800; color: #1e293b; text-transform: uppercase; letter-spacing: 0.05em; }
.ph-sub   { font-size: 10px; font-weight: 600; color: #94a3b8; margin-top: 2px; }
.ph-badge { margin-left: auto; font-size: 13px; font-weight: 800; padding: 6px 12px; border-radius: 10px; }
.b--blue  { background: rgba(59,130,246,0.1); color: #2563eb; }
.b--amber { background: rgba(245,158,11,0.1); color: #d97706; }
.b--rose  { background: rgba(244,63,94,0.1); color: #e11d48; }

.panel-body { padding: 24px; flex: 1; display: flex; flex-direction: column; gap: 16px; }

.panel-foot {
  padding: 12px 24px; background: #fafbfc;
  border-top: 1px solid #f1f5f9;
  display: flex; align-items: center; gap: 12px;
  font-size: 10px; font-weight: 600;
}
.foot-label { color: #94a3b8; text-transform: uppercase; letter-spacing: 0.08em; }
.foot-val { color: #334155; font-weight: 700; }

/* ── CPU ARC GAUGE ── */
.cpu-arc-wrap { display: flex; justify-content: center; }
.cpu-arc-svg { width: 200px; height: 115px; }

/* ── CPU SPEC GRID ── */
.spec-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
.spec-tile { background: #f8fafc; border-radius: 14px; padding: 14px; border: 1px solid #f1f5f9; text-align: center; }
.st-wide { grid-column: 1 / -1; text-align: left; }
.st-val { font-size: 22px; font-weight: 900; color: #0f172a; letter-spacing: -0.02em; }
.st-mono { font-family: 'JetBrains Mono', monospace; font-size: 11px !important; letter-spacing: 0; }
.st-lbl { font-size: 9px; font-weight: 700; color: #94a3b8; text-transform: uppercase; margin-top: 4px; letter-spacing: 0.08em; }

/* ── THERMAL CHIPS ── */
.thermal-chips { display: flex; flex-direction: column; gap: 8px; }
.thermal-chip { display: flex; align-items: center; gap: 10px; background: #f8fafc; padding: 8px 12px; border-radius: 10px; border: 1px solid #f1f5f9; }
.tc-dot { width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0; }
.tcd--ok   { background: #10b981; box-shadow: 0 0 6px rgba(16,185,129,0.5); }
.tcd--warn { background: #f59e0b; box-shadow: 0 0 6px rgba(245,158,11,0.5); }
.tcd--crit { background: #f43f5e; box-shadow: 0 0 8px rgba(244,63,94,0.6); animation: blink 1s infinite; }
.tc-name { font-size: 11px; font-weight: 600; color: #475569; flex: 1; }
.tc-val  { font-size: 14px; font-weight: 800; font-family: 'JetBrains Mono', monospace; }
.tcv--ok   { color: #059669; }
.tcv--warn { color: #d97706; }
.tcv--crit { color: #e11d48; }

/* ── DIMM GRID (RAM) ── */
.dimm-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
.dimm-card {
  border-radius: 14px; border: 1px solid #e2e8f0;
  overflow: hidden; display: flex; flex-direction: row;
  transition: all .18s;
}
.dimm-card:hover { border-color: #93c5fd; box-shadow: 0 4px 12px rgba(59,130,246,0.1); }
.dc-bar { width: 5px; flex-shrink: 0; }
.dc--ok   .dc-bar { background: #10b981; }
.dc--warn .dc-bar { background: #f59e0b; }
.dc--crit .dc-bar { background: #f43f5e; }
.dc-content { flex: 1; padding: 12px; }
.dc-name { font-size: 9px; font-weight: 800; color: #94a3b8; text-transform: uppercase; letter-spacing: 0.05em; margin-bottom: 4px; }
.dc-cap  { font-size: 22px; font-weight: 900; color: #0f172a; line-height: 1; letter-spacing: -0.02em; }
.dc-unit { font-size: 12px; font-weight: 600; color: #64748b; }
.dc-speed{ font-size: 10px; font-weight: 600; color: #64748b; margin-top: 4px; }
.dc-type { font-size: 9px; font-weight: 700; color: #94a3b8; text-transform: uppercase; }
.dc-status { font-size: 9px; font-weight: 800; padding: 4px 8px; align-self: start; margin: 8px 8px 0 0; border-radius: 6px; }
.dcs--ok   { background: rgba(16,185,129,0.1); color: #059669; }
.dcs--warn { background: rgba(245,158,11,0.1); color: #d97706; }
.dcs--crit { background: rgba(244,63,94,0.1); color: #e11d48; }

/* ── PSU LIST ── */
.psu-list { display: flex; flex-direction: column; gap: 10px; }
.psu-row { display: flex; align-items: center; gap: 12px; padding: 12px 16px; background: #f8fafc; border-radius: 12px; border: 1px solid #f1f5f9; }
.psu-dot { width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0; }
.pd--ok   { background: #10b981; box-shadow: 0 0 6px rgba(16,185,129,0.5); }
.pd--warn { background: #f59e0b; }
.psu-name { flex: 1; font-size: 12px; font-weight: 600; color: #475569; }
.psu-watts { font-size: 13px; font-weight: 800; color: #0f172a; }
.psu-health { font-size: 10px; font-weight: 700; padding: 3px 8px; border-radius: 6px; }

/* ── STATUS BADGES ── */
.s--ok   { background: rgba(16,185,129,0.1); color: #059669; border-radius: 6px; padding: 3px 8px; font-size: 10px; font-weight: 700; }
.s--warn { background: rgba(245,158,11,0.1); color: #d97706; border-radius: 6px; padding: 3px 8px; font-size: 10px; font-weight: 700; }

/* ── TEMP GRID ── */
.temp-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(120px, 1fr)); gap: 10px; }
.temp-card { padding: 14px; border-radius: 14px; border: 1px solid #e2e8f0; transition: all .18s; }
.temp-card:hover { transform: scale(1.03); }
.tc--ok   .temp-card, .temp-card.tc--ok { background: #f0fdf4; border-color: rgba(16,185,129,0.2); }
.tc--warn .temp-card, .temp-card.tc--warn { background: #fffbeb; border-color: rgba(245,158,11,0.3); }
.tc--crit .temp-card, .temp-card.tc--crit { background: #fff1f2; border-color: rgba(244,63,94,0.3); }
.temp-val { font-size: 22px; font-weight: 900; font-family: 'JetBrains Mono', monospace; color: #0f172a; }
.tc--ok   .temp-val { color: #059669; }
.tc--warn .temp-val { color: #d97706; }
.tc--crit .temp-val { color: #e11d48; }
.temp-name { font-size: 9px; font-weight: 700; color: #64748b; text-transform: uppercase; margin-top: 4px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.temp-tag { display: inline-block; margin-top: 6px; font-size: 8px; font-weight: 800; text-transform: uppercase; padding: 2px 6px; border-radius: 4px; background: rgba(0,0,0,0.05); color: #64748b; letter-spacing: 0.05em; }
.temp-thresholds { font-size: 8px; color: #94a3b8; margin-top: 4px; display: flex; gap: 6px; font-family: 'JetBrains Mono', monospace; }

/* ── FAN GRID ── */
.fan-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
.fan-card { display: flex; align-items: center; gap: 12px; padding: 12px 14px; background: #f8fafc; border-radius: 14px; border: 1px solid #f1f5f9; }
.fan-rotor { width: 38px; height: 38px; background: #e0f0ff; color: #3b82f6; border-radius: 50%; display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
.fan-rotor svg { width: 20px; height: 20px; }
.fan-spin { animation: spin linear infinite; }
.fan-info { flex: 1; min-width: 0; }
.fan-name { font-size: 10px; font-weight: 700; color: #94a3b8; text-transform: uppercase; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.fan-rpm  { font-size: 16px; font-weight: 900; color: #0f172a; margin-top: 2px; }
.rpm-unit { font-size: 9px; font-weight: 600; color: #94a3b8; }
.rpm--warn { color: #d97706; }
.fan-dot  { width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0; }
.fd--ok   { background: #10b981; box-shadow: 0 0 6px rgba(16,185,129,0.5); }
.fd--warn { background: #f59e0b; }

/* ── DRIVE LIST ── */
.drive-list { display: flex; flex-direction: column; gap: 10px; }
.drive-row { display: flex; align-items: center; gap: 14px; padding: 14px 16px; background: #f8fafc; border-radius: 14px; border: 1px solid #f1f5f9; transition: all .18s; }
.drive-row:hover { border-color: #93c5fd; background: white; }
.drive-type-badge { font-size: 9px; font-weight: 800; padding: 4px 8px; border-radius: 6px; }
.dtb--ssd { background: rgba(59,130,246,0.1); color: #2563eb; }
.dtb--hdd { background: rgba(100,116,139,0.1); color: #475569; }
.drive-info { flex: 1; min-width: 0; }
.drive-name { font-size: 12px; font-weight: 700; color: #334155; }
.drive-model { font-size: 10px; color: #94a3b8; margin-top: 2px; }
.drive-cap { font-size: 13px; font-weight: 800; color: #0f172a; white-space: nowrap; }
.drive-health { font-size: 10px; font-weight: 700; padding: 3px 8px; border-radius: 6px; white-space: nowrap; }

/* ── INFO TABLE ── */
.info-table { display: flex; flex-direction: column; }
.it-row { display: flex; justify-content: space-between; align-items: center; padding: 12px 0; border-bottom: 1px solid #f1f5f9; gap: 12px; }
.it-row:last-child { border-bottom: none; }
.it-lbl { font-size: 11px; font-weight: 700; color: #94a3b8; text-transform: uppercase; letter-spacing: 0.05em; white-space: nowrap; }
.it-val { font-size: 12px; font-weight: 700; color: #334155; text-align: right; }

/* ── ANIMATIONS ── */
@keyframes spin  { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }
@keyframes blink { 0%, 100% { opacity: 1; } 50% { opacity: .3; } }

/* ── RESPONSIVE ── */
@media (max-width: 1200px) {
  .triple-grid { grid-template-columns: 1fr 1fr; }
  .hero-row    { grid-template-columns: 1fr 1fr; }
}
@media (max-width: 800px) {
  .triple-grid, .double-grid, .hero-row { grid-template-columns: 1fr; }
  .main-grid { padding: 20px; }
  .topbar { padding: 0 20px; }
}
</style>