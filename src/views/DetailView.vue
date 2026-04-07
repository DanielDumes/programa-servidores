<template>
  <div class="dp">

    <!-- Topbar -->
    <header class="topbar">
      <button class="btn-back" @click="$emit('back')">
        <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"><polyline points="15 18 9 12 15 6"/></svg>
        Volver
      </button>
      <div class="topbar-center" v-if="data">
        <span class="srv-name">{{ server.label }}</span>
        <span class="divider-slash">/</span>
        <span class="srv-ip">{{ server.host }}</span>
        <span class="health-tag" :class="'ht--' + healthCls(data.summary?.health)">
          <span class="ht-dot"></span>{{ healthLabel(data.summary?.health) }}
        </span>
      </div>
      <div class="topbar-right">
        <span class="last-sync" v-if="lastUpdate">↺ {{ lastUpdate }}</span>
        <button class="btn-sync" @click="load" :disabled="loading">
          <span :class="{ spin: loading }">⟳</span>
          {{ loading ? 'Cargando…' : 'Sincronizar' }}
        </button>
      </div>
    </header>

    <!-- Error -->
    <div class="err-bar" v-if="error">
      <span>⚠ {{ error }}</span>
      <button @click="load" class="err-retry">Reintentar</button>
    </div>

    <!-- Loader -->
    <div class="page-loader" v-if="loading && !data">
      <div class="loader-track"><div class="loader-car"></div></div>
      <div class="loader-msg">Conectando con <strong>{{ server.host }}</strong>…</div>
    </div>

    <main class="main" v-if="data">

      <!-- ── HERO STRIP ── -->
      <section class="hero-strip">
        <div class="hero-tile" :class="'tile--' + healthCls(data.summary?.health)">
          <div class="tile-icon">
            <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M22 12h-4l-3 9L9 3l-3 9H2"/></svg>
          </div>
          <div class="tile-body">
            <div class="tile-label">Salud del Sistema</div>
            <div class="tile-value">{{ healthLabel(data.summary?.health) }}</div>
            <div class="tile-sub">{{ data.summary?.model }}</div>
          </div>
        </div>

        <div class="hero-tile" :class="data.summary?.power_state === 'On' ? 'tile--ok' : 'tile--crit'">
          <div class="tile-icon">
            <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M18.36 6.64a9 9 0 1 1-12.73 0M12 2v10"/></svg>
          </div>
          <div class="tile-body">
            <div class="tile-label">Estado de Energía</div>
            <div class="tile-value">{{ data.summary?.power_state ?? '—' }}</div>
            <div class="tile-sub">{{ data.summary?.power_state === 'On' ? 'Sistema activo' : 'Apagado' }}</div>
          </div>
        </div>

        <div class="hero-tile tile--info">
          <div class="tile-icon">
            <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"/></svg>
          </div>
          <div class="tile-body">
            <div class="tile-label">Consumo Activo</div>
            <div class="tile-value">{{ data.power?.consumed_watts ?? '—' }}<small> W</small></div>
            <div class="tile-sub">Cap. {{ data.power?.capacity_watts ?? '?' }} W</div>
          </div>
        </div>

        <div class="hero-tile" :class="'tile--' + ambientTempCls">
          <div class="tile-icon">
            <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M14 14.76V3.5a2.5 2.5 0 0 0-5 0v11.26a4.5 4.5 0 1 0 5 0z"/></svg>
          </div>
          <div class="tile-body">
            <div class="tile-label">Temperatura Ambiente</div>
            <div class="tile-value">{{ ambientTemp !== null ? ambientTemp + ' °C' : '—' }}</div>
            <div class="tile-sub">{{ ambientTempName }}</div>
          </div>
        </div>
      </section>

      <!-- ── ROW 2: CPU + RAM + POWER ── -->
      <section class="trio-grid">

        <!-- CPU -->
        <div class="panel">
          <header class="panel-head">
            <div class="ph-mark ph-mark--teal"></div>
            <div class="ph-text">
              <span class="ph-title">Procesador</span>
              <span class="ph-meta">{{ data.summary?.cpu_count ?? '?' }} núcleos físicos</span>
            </div>
            <div class="ph-chip" :class="maxCpuReading > 75 ? 'chip--red' : maxCpuReading > 60 ? 'chip--amber' : 'chip--teal'">{{ maxCpuReading }}°C</div>
          </header>

          <div class="panel-body">
            <div class="gauge-wrap">
              <svg viewBox="0 0 180 100" class="gauge-svg">
                <path d="M 18 90 A 72 72 0 0 1 162 90" fill="none" stroke="#e8e4df" stroke-width="10" stroke-linecap="round"/>
                <path d="M 18 90 A 72 72 0 0 1 162 90" fill="none"
                  :stroke="cpuArcColor" stroke-width="10" stroke-linecap="round"
                  :stroke-dasharray="`${(maxCpuReading / 100) * 226} 226`"/>
                <text x="90" y="83" text-anchor="middle" font-size="30" font-weight="700" :fill="cpuArcColor" font-family="'IBM Plex Mono', monospace" letter-spacing="-1">{{ maxCpuReading }}</text>
                <text x="90" y="97" text-anchor="middle" font-size="9" fill="#9a9490" font-family="'Sora', system-ui" letter-spacing="1.5">°C NÚCLEO MÁX</text>
              </svg>
            </div>

            <div class="spec-row">
              <div class="spec-tile">
                <div class="st-val">{{ data.summary?.cpu_count ?? '—' }}</div>
                <div class="st-lbl">Sockets</div>
              </div>
              <div class="spec-tile">
                <div class="st-val">{{ data.summary?.logical_cpu_count || '—' }}</div>
                <div class="st-lbl">Hilos lógicos</div>
              </div>
              <div class="spec-tile spec-wide">
                <div class="st-val st-mono" style="font-size:10px">{{ data.summary?.cpu_model || 'Intel Xeon Scalable' }}</div>
                <div class="st-lbl">Modelo CPU</div>
              </div>
            </div>

            <div class="sensor-list" v-if="cpuTempSensors.length">
              <div class="sensor-row" v-for="s in cpuTempSensors" :key="s.name">
                <span class="s-led" :class="'led--' + tempClsStr(s)"></span>
                <span class="s-name">{{ s.name.replace('CPU ', '') }}</span>
                <span class="s-val" :class="'sval--' + tempClsStr(s)">{{ s.reading_c }}°</span>
              </div>
            </div>
          </div>

          <footer class="panel-foot">
            <span class="pf-label">S/N Sistema</span>
            <span class="pf-val">{{ data.summary?.serial || '—' }}</span>
          </footer>
        </div>

        <!-- RAM -->
        <div class="panel">
          <header class="panel-head">
            <div class="ph-mark ph-mark--coral"></div>
            <div class="ph-text">
              <span class="ph-title">Memoria RAM</span>
              <span class="ph-meta">{{ data.summary?.memory_gib ?? '?' }} GB · {{ data.memory?.dimms?.length ?? 0 }} módulos</span>
            </div>
          </header>

          <div class="panel-body">
            <div class="dimm-grid">
              <div class="dimm-card" v-for="d in data.memory?.dimms" :key="d.name" :class="'dc--' + dimmCls(d.health)">
                <div class="dc-accent"></div>
                <div class="dc-inner">
                  <div class="dc-slot">{{ d.name }}</div>
                  <div class="dc-size">{{ dimmGb(d) }}<span class="dc-unit"> GB</span></div>
                  <div class="dc-info">{{ d.speed_mhz }} MHz · {{ d.type }}</div>
                  <div class="dc-badge" :class="'dcb--' + dimmCls(d.health)">{{ d.health }}</div>
                </div>
              </div>
            </div>
          </div>

          <footer class="panel-foot">
            <span class="pf-label">Velocidad</span>
            <span class="pf-val">{{ data.memory?.dimms?.[0]?.speed_mhz ?? '—' }} MHz</span>
            <span class="pf-sep">·</span>
            <span class="pf-label">Tipo</span>
            <span class="pf-val">{{ data.memory?.dimms?.[0]?.type ?? 'DDR4' }}</span>
          </footer>
        </div>

        <!-- Power -->
        <div class="panel">
          <header class="panel-head">
            <div class="ph-mark ph-mark--amber"></div>
            <div class="ph-text">
              <span class="ph-title">Fuente de Poder</span>
              <span class="ph-meta">{{ powerPct.toFixed(0) }}% carga actual</span>
            </div>
          </header>

          <div class="panel-body">
            <div class="gauge-wrap">
              <svg viewBox="0 0 180 100" class="gauge-svg">
                <path d="M 18 90 A 72 72 0 0 1 162 90" fill="none" stroke="#e8e4df" stroke-width="10" stroke-linecap="round"/>
                <path d="M 18 90 A 72 72 0 0 1 162 90" fill="none"
                  :stroke="powerPct > 85 ? '#c0392b' : powerPct > 65 ? '#e67e22' : '#1a8a7a'" stroke-width="10" stroke-linecap="round"
                  :stroke-dasharray="`${(powerPct / 100) * 226} 226`"/>
                <text x="90" y="83" text-anchor="middle" font-size="30" font-weight="700" :fill="powerPct > 85 ? '#c0392b' : '#1a3a34'" font-family="'IBM Plex Mono', monospace" letter-spacing="-1">{{ data.power?.consumed_watts ?? '—' }}</text>
                <text x="90" y="97" text-anchor="middle" font-size="9" fill="#9a9490" font-family="'Sora', system-ui" letter-spacing="1.5">WATTS ACTIVOS</text>
              </svg>
            </div>

            <div class="psu-list">
              <div class="psu-row" v-for="p in data.power?.power_supplies" :key="p.name">
                <span class="psu-led" :class="p.health === 'OK' ? 'led--ok' : 'led--warn'"></span>
                <span class="psu-name">{{ p.name }}</span>
                <span class="psu-w">{{ p.power_watts ?? '—' }} W</span>
                <span class="psu-status" :class="p.health === 'OK' ? 'st--ok' : 'st--warn'">{{ p.health }}</span>
              </div>
            </div>
          </div>

          <footer class="panel-foot">
            <span class="pf-label">Capacidad total</span>
            <span class="pf-val">{{ data.power?.capacity_watts ?? '—' }} W</span>
          </footer>
        </div>
      </section>

      <!-- ── ROW 3: TEMPS + FANS ── -->
      <section class="duo-grid">
        <div class="panel">
          <header class="panel-head">
            <div class="ph-mark ph-mark--red"></div>
            <div class="ph-text">
              <span class="ph-title">Mapa térmico</span>
              <span class="ph-meta">{{ groupedTemperatures.length }} zonas · {{ (data.temperatures ?? []).length }} sensores</span>
            </div>
          </header>
          <div class="panel-body">
            <div class="temp-grid">
              <div class="temp-card" v-for="g in groupedTemperatures" :key="g.name + g.count" :class="'tc--' + g.status">
                <div class="tc-val">{{ g.reading !== null ? Math.round(g.reading) + '°' : 'N/A' }}</div>
                <div class="tc-name">{{ g.name }}</div>
                <div class="tc-badge" v-if="g.isGroup">⌀ {{ g.count }}</div>
                <div class="tc-thresh" v-if="!g.isGroup && (g.caution || g.critical)">
                  <span v-if="g.caution">C:{{ g.caution }}°</span>
                  <span v-if="g.critical">K:{{ g.critical }}°</span>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div class="panel">
          <header class="panel-head">
            <div class="ph-mark ph-mark--teal"></div>
            <div class="ph-text">
              <span class="ph-title">Ventilación</span>
              <span class="ph-meta">{{ data.fans?.length ?? 0 }} ventiladores</span>
            </div>
          </header>
          <div class="panel-body">
            <div class="fan-grid">
              <div class="fan-card" v-for="f in data.fans" :key="f.name">
                <div class="fan-rotor" :class="{ 'fan-spin': (f.rpm ?? 0) > 0 }" :style="{ animationDuration: fanDuration(f.rpm) }">
                  <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor"><path d="M12 12c1.1 0 2-.9 2-2s-.9-2-2-2-2 .9-2 2 .9 2 2 2zm0-8C6.5 4 4 7.6 4 12s2.5 8 8 8 8-3.6 8-8-3.6-8-8-8z"/></svg>
                </div>
                <div class="fan-info">
                  <div class="fan-name">{{ f.name }}</div>
                  <div class="fan-rpm" :class="f.health !== 'OK' ? 'rpm--warn' : ''">
                    {{ f.rpm ?? 0 }}<span class="rpm-unit"> RPM</span>
                  </div>
                </div>
                <span class="fan-led" :class="f.health === 'OK' ? 'led--ok' : 'led--warn'"></span>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- ── ROW 4: STORAGE + FICHA ── -->
      <section class="duo-grid">
        <div class="panel" v-if="data.storage?.controllers?.some(c => c.drives?.length)">
          <header class="panel-head">
            <div class="ph-mark ph-mark--slate"></div>
            <div class="ph-text">
              <span class="ph-title">Almacenamiento</span>
              <span class="ph-meta">Inventario de discos físicos</span>
            </div>
          </header>
          <div class="panel-body" style="gap:20px;">
            <div v-for="(ctrl, idx) in data.storage.controllers.filter(c => c.drives?.length)" :key="idx" class="ctrl-block">
              <div class="ctrl-label">
                <svg width="10" height="10" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3"><polyline points="9 18 15 12 9 6"/></svg>
                {{ ctrl.name }}
              </div>
              <div class="drive-list">
                <div class="drive-row" v-for="d in ctrl.drives" :key="d.name">
                  <span class="drive-type" :class="d.type?.includes('SSD') ? 'dtype--ssd' : 'dtype--hdd'">{{ d.type || 'HDD' }}</span>
                  <div class="drive-meta">
                    <span class="drive-name">{{ d.name }}</span>
                    <span class="drive-model">{{ d.model }}</span>
                  </div>
                  <span class="drive-cap">{{ d.capacity_gb }} GB</span>
                  <span class="drive-health" :class="d.health === 'OK' ? 'st--ok' : 'st--warn'">{{ d.health }}</span>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div class="panel">
          <header class="panel-head">
            <div class="ph-mark ph-mark--slate"></div>
            <div class="ph-text">
              <span class="ph-title">Ficha del sistema</span>
              <span class="ph-meta">HPE ProLiant · iLO 5</span>
            </div>
          </header>
          <div class="panel-body">
            <div class="info-table">
              <div class="it-row"><span class="it-lbl">Modelo</span><span class="it-val">{{ data.summary?.model ?? 'N/A' }}</span></div>
              <div class="it-row"><span class="it-lbl">N° de serie</span><span class="it-val mono">{{ data.summary?.serial ?? 'N/A' }}</span></div>
              <div class="it-row"><span class="it-lbl">Versión BIOS</span><span class="it-val mono">{{ data.summary?.bios_version ?? 'N/A' }}</span></div>
              <div class="it-row"><span class="it-lbl">IP iLO</span><span class="it-val mono">{{ server.host }}</span></div>
              <div class="it-row"><span class="it-lbl">Procesadores</span><span class="it-val">{{ data.summary?.cpu_count ?? '—' }} instalados</span></div>
              <div class="it-row"><span class="it-lbl">Memoria total</span><span class="it-val">{{ data.summary?.memory_gib ?? '—' }} GB DDR4</span></div>
            </div>
          </div>
        </div>
      </section>
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

const ambientTempSensor = computed(() => {
  const sensors = data.value?.temperatures ?? []
  if (!sensors.length) return null
  return sensors.find(s => { const n = (s.name || '').toLowerCase(); return n.includes('inlet') || n.includes('ambient') }) || null
})
const ambientTemp = computed(() => {
  if (ambientTempSensor.value?.reading_c != null) return ambientTempSensor.value.reading_c
  const vals = (data.value?.temperatures ?? []).map(t => t.reading_c).filter(v => v != null)
  return vals.length ? Math.max(...vals) : null
})
const ambientTempName = computed(() => {
  if (ambientTempSensor.value) return ambientTempSensor.value.name
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
  function getBaseName(name, location) {
    if (!name) return ''
    let n = name.replace(/^\d+[\s\-\.]+/, '')
    if (location === 'Memory' || location === 'CPU') n = n.replace(/^P\d+[\s\-]+/, '')
    n = n.replace(/[\s\-]+\d+([\-\s]\d+)?$/, '').replace(/\s+[A-Z]{1,4}$/, '')
    return n.trim()
  }
  function zoneKey(s) {
    const n = (s.name || '').toLowerCase()
    if (n.includes('ambient') || n.includes('inlet')) return `unique-${s.name}`
    return `${getBaseName(s.name, s.location).toLowerCase()}:${s.location || 'System'}:${s.upper_caution || '-'}-${s.upper_critical || '-'}`
  }
  const zoneMap = new Map()
  sensors.forEach(s => { const k = zoneKey(s); if (!zoneMap.has(k)) zoneMap.set(k, { sensors: [] }); zoneMap.get(k).sensors.push(s) })
  const result = []
  zoneMap.forEach((zone) => {
    const valid = zone.sensors.filter(s => s.reading_c != null)
    if (!valid.length) return
    const isGroup = valid.length > 1
    const avg = valid.reduce((a, b) => a + b.reading_c, 0) / valid.length
    const ref = valid[0]
    let name = ref.name
    if (isGroup) {
      if (ref.location === 'Memory') name = 'Memoria RAM'
      else if (ref.location === 'CPU') name = 'CPU / Procesadores'
      else if (ref.name.includes('VR')) name = 'Reguladores (VRM)'
      else name = getBaseName(ref.name, ref.location)
    }
    const worstStatus = valid.map(s => tempClsStr(s)).reduce((a, b) => { if (a==='crit'||b==='crit') return 'crit'; if (a==='warn'||b==='warn') return 'warn'; return 'ok' }, 'ok')
    result.push({ name, reading: isGroup ? avg : ref.reading_c, status: worstStatus, isGroup, count: valid.length, caution: ref.upper_caution, critical: ref.upper_critical, location: ref.location })
  })
  result.sort((a, b) => {
    const score = i => { const n=i.name.toLowerCase(),l=(i.location||'').toLowerCase(); if(n.includes('ambient')||n.includes('inlet')) return 1; if(l==='cpu'||n.includes('cpu')) return 2; if(l==='memory'||n.includes('ram')) return 3; return 10 }
    const d = score(a) - score(b); return d !== 0 ? d : a.name.localeCompare(b.name)
  })
  return result
})

const cpuTempSensors = computed(() => (data.value?.temperatures ?? []).filter(t => t.name?.toLowerCase().includes('cpu')))
const maxCpuReading  = computed(() => { const v = cpuTempSensors.value.map(s => s.reading_c).filter(v => v != null); return v.length ? Math.max(...v) : 40 })
const cpuArcColor    = computed(() => maxCpuReading.value > 75 ? '#c0392b' : maxCpuReading.value > 60 ? '#e67e22' : '#1a8a7a')
const powerPct       = computed(() => { if (!data.value?.power?.consumed_watts || !data.value?.power?.capacity_watts) return 0; return Math.min(100, (data.value.power.consumed_watts / data.value.power.capacity_watts) * 100) })

function healthLabel(h) { return { OK: 'Óptimo', Warning: 'Advertencia', Critical: 'Crítico' }[h] ?? h ?? '—' }
function healthCls(h)   { return { OK: 'ok', Warning: 'warn', Critical: 'crit' }[h] ?? '' }
function dimmCls(h)     { return { OK: 'ok', Warning: 'warn', Critical: 'crit' }[h] ?? 'ok' }
function dimmGb(d)      { const mb = d.size_mb ?? d.capacity_mib ?? d.size ?? 0; return mb ? Math.round(mb / 1024) : '?' }
function tempClsStr(t)  { if (!t.reading_c) return 'ok'; if (t.upper_critical && t.reading_c >= t.upper_critical) return 'crit'; if (t.upper_caution && t.reading_c >= t.upper_caution) return 'warn'; if (t.reading_c > 85) return 'crit'; if (t.reading_c > 70) return 'warn'; return 'ok' }
function fanDuration(rpm) { if (!rpm || rpm === 0) return '0s'; return `${Math.max(0.3, 3 - (rpm / 2500)).toFixed(2)}s` }

async function load() {
  loading.value = true; error.value = null
  try {
    data.value = await fetchAll(props.server)
    lastUpdate.value = new Date().toLocaleTimeString('es-EC', { hour: '2-digit', minute: '2-digit' })
  } catch (e) { error.value = `Error al conectar: ${e.message}` }
  finally { loading.value = false }
}
onMounted(() => { load(); autoTimer = setInterval(load, REFRESH_INTERVAL_SEC * 1000) })
onUnmounted(() => { clearInterval(autoTimer) })
</script>

<style scoped>
/* ═══════════════════════════════════════
   WARM EDITORIAL — Sora + IBM Plex Mono
   Paleta: crema #f5f2ee · teal #1a8a7a
           coral #c0582b · amber #e67e22
═══════════════════════════════════════ */
@import url('https://fonts.googleapis.com/css2?family=Sora:wght@400;500;600;700;800&family=IBM+Plex+Mono:wght@400;600&display=swap');

.dp { min-height:100vh; background:#f5f2ee; font-family:'Sora',-apple-system,sans-serif; color:#1a1714; }
.mono { font-family:'IBM Plex Mono',monospace; }

/* ── TOPBAR ── */
.topbar { position:sticky; top:0; z-index:100; background:#faf8f5; border-bottom:1.5px solid #ddd8d0; height:60px; padding:0 36px; display:flex; align-items:center; gap:16px; }
.btn-back { display:flex; align-items:center; gap:6px; font-family:'Sora',sans-serif; font-size:12px; font-weight:600; padding:7px 14px; border-radius:8px; border:1.5px solid #d4cfc8; background:white; color:#6b6560; cursor:pointer; transition:all .15s; letter-spacing:.02em; white-space:nowrap; }
.btn-back:hover { background:#f5f2ee; border-color:#b0aba3; color:#1a1714; }
.topbar-center { display:flex; align-items:center; gap:10px; }
.srv-name { font-size:15px; font-weight:800; color:#1a1714; letter-spacing:-.03em; }
.divider-slash { color:#c4bfb8; font-weight:300; font-size:16px; }
.srv-ip { font-family:'IBM Plex Mono',monospace; font-size:11px; color:#1a8a7a; background:#e6f5f2; padding:3px 8px; border-radius:5px; border:1px solid #b8e4dc; }
.health-tag { display:flex; align-items:center; gap:5px; font-size:11px; font-weight:700; padding:3px 10px; border-radius:20px; letter-spacing:.02em; }
.ht-dot { width:6px; height:6px; border-radius:50%; }
.ht--ok   { background:#e6f5ee; color:#0f6e44; } .ht--ok   .ht-dot { background:#1a8a7a; }
.ht--warn { background:#fef3e6; color:#854f0b; } .ht--warn .ht-dot { background:#e67e22; }
.ht--crit { background:#fdecea; color:#a32d2d; } .ht--crit .ht-dot { background:#c0392b; animation:blink 1s infinite; }
.topbar-right { margin-left:auto; display:flex; align-items:center; gap:12px; }
.last-sync { font-size:11px; color:#9a9490; font-weight:500; }
.btn-sync { display:flex; align-items:center; gap:7px; font-family:'Sora',sans-serif; font-size:12px; font-weight:700; padding:8px 18px; border-radius:8px; background:#1a1714; color:#f5f2ee; border:none; cursor:pointer; letter-spacing:.02em; transition:all .15s; }
.btn-sync:hover { background:#2d2925; }
.btn-sync:disabled { opacity:.5; cursor:not-allowed; }

/* ── ERROR + LOADER ── */
.err-bar { background:#fdecea; border-bottom:1.5px solid #f5c0c0; padding:10px 36px; display:flex; align-items:center; gap:12px; font-size:12px; font-weight:600; color:#a32d2d; }
.err-retry { margin-left:auto; background:#a32d2d; color:white; border:none; padding:5px 14px; border-radius:6px; cursor:pointer; font-size:11px; font-weight:700; }
.page-loader { display:flex; flex-direction:column; align-items:center; justify-content:center; gap:20px; height:calc(100vh - 60px); }
.loader-track { width:240px; height:3px; background:#e0dbd4; border-radius:2px; overflow:hidden; }
.loader-car { width:80px; height:3px; background:#1a8a7a; border-radius:2px; animation:slide 1.4s ease-in-out infinite; }
.loader-msg { font-size:13px; color:#6b6560; font-weight:500; }

/* ── LAYOUT ── */
.main { padding:28px 36px; display:flex; flex-direction:column; gap:20px; max-width:1700px; margin:0 auto; }

/* ── HERO STRIP ── */
.hero-strip { display:grid; grid-template-columns:repeat(4,1fr); gap:14px; }
.hero-tile { background:white; border:1.5px solid #ddd8d0; border-radius:16px; padding:20px; display:flex; gap:14px; align-items:flex-start; transition:transform .18s,box-shadow .18s; position:relative; overflow:hidden; }
.hero-tile::before { content:''; position:absolute; top:0; left:0; right:0; height:3px; border-radius:16px 16px 0 0; }
.tile--ok::before   { background:#1a8a7a; }
.tile--warn::before { background:#e67e22; }
.tile--crit::before { background:#c0392b; }
.tile--info::before { background:#2563eb; }
.hero-tile:hover { transform:translateY(-2px); box-shadow:0 8px 24px rgba(26,23,20,.08); }
.tile-icon { width:40px; height:40px; border-radius:10px; background:#f5f2ee; border:1.5px solid #ddd8d0; display:flex; align-items:center; justify-content:center; flex-shrink:0; color:#6b6560; }
.tile--ok   .tile-icon { background:#e6f5f2; border-color:#b8e4dc; color:#1a8a7a; }
.tile--warn .tile-icon { background:#fef3e6; border-color:#f9d5a5; color:#e67e22; }
.tile--crit .tile-icon { background:#fdecea; border-color:#f5c0c0; color:#c0392b; }
.tile--info .tile-icon { background:#e6f0fb; border-color:#b5d0f4; color:#2563eb; }
.tile-label { font-size:10px; font-weight:700; color:#9a9490; text-transform:uppercase; letter-spacing:.09em; }
.tile-value { font-size:22px; font-weight:800; color:#1a1714; margin-top:4px; line-height:1; letter-spacing:-.03em; }
.tile-value small { font-size:13px; font-weight:600; color:#6b6560; }
.tile--ok   .tile-value { color:#0f6e44; }
.tile--warn .tile-value { color:#854f0b; }
.tile--crit .tile-value { color:#a32d2d; }
.tile--info .tile-value { color:#185fa5; }
.tile-sub { font-size:11px; color:#9a9490; font-weight:500; margin-top:5px; }

/* ── GRIDS ── */
.trio-grid { display:grid; grid-template-columns:repeat(3,1fr); gap:20px; }
.duo-grid  { display:grid; grid-template-columns:3fr 2fr; gap:20px; }

/* ── PANEL ── */
.panel { background:white; border:1.5px solid #ddd8d0; border-radius:18px; display:flex; flex-direction:column; overflow:hidden; transition:transform .18s,box-shadow .18s; }
.panel:hover { transform:translateY(-2px); box-shadow:0 12px 28px rgba(26,23,20,.07); }
.panel-head { padding:18px 22px; display:flex; align-items:center; gap:12px; border-bottom:1px solid #ede9e4; background:#faf8f5; }
.ph-mark { width:3px; height:28px; border-radius:2px; flex-shrink:0; }
.ph-mark--teal  { background:#1a8a7a; }
.ph-mark--coral { background:#c0582b; }
.ph-mark--amber { background:#e67e22; }
.ph-mark--red   { background:#c0392b; }
.ph-mark--slate { background:#6b6560; }
.ph-text { flex:1; }
.ph-title { display:block; font-size:12px; font-weight:800; color:#1a1714; text-transform:uppercase; letter-spacing:.07em; }
.ph-meta  { display:block; font-size:10px; font-weight:500; color:#9a9490; margin-top:2px; text-transform:uppercase; letter-spacing:.05em; }
.ph-chip  { font-family:'IBM Plex Mono',monospace; font-size:13px; font-weight:600; padding:5px 11px; border-radius:8px; }
.chip--teal  { background:#e6f5f2; color:#0f6e44; border:1px solid #b8e4dc; }
.chip--amber { background:#fef3e6; color:#854f0b; border:1px solid #f9d5a5; }
.chip--red   { background:#fdecea; color:#a32d2d; border:1px solid #f5c0c0; }
.panel-body { padding:22px; flex:1; display:flex; flex-direction:column; gap:16px; }
.panel-foot { padding:12px 22px; border-top:1px solid #ede9e4; background:#faf8f5; display:flex; align-items:center; gap:10px; font-size:10px; font-weight:600; }
.pf-label { color:#9a9490; text-transform:uppercase; letter-spacing:.08em; }
.pf-val   { color:#1a1714; font-family:'IBM Plex Mono',monospace; font-size:11px; }
.pf-sep   { color:#d4cfc8; }

/* ── GAUGE ── */
.gauge-wrap { display:flex; justify-content:center; }
.gauge-svg  { width:180px; height:105px; }

/* ── SPEC ROW ── */
.spec-row  { display:grid; grid-template-columns:1fr 1fr; gap:8px; }
.spec-tile { background:#f5f2ee; border-radius:12px; padding:12px 14px; border:1px solid #ddd8d0; }
.spec-wide { grid-column:1/-1; }
.st-val  { font-size:20px; font-weight:800; color:#1a1714; letter-spacing:-.03em; }
.st-mono { font-family:'IBM Plex Mono',monospace; font-size:10px !important; letter-spacing:0; font-weight:600; }
.st-lbl  { font-size:9px; font-weight:700; color:#9a9490; text-transform:uppercase; letter-spacing:.09em; margin-top:4px; }

/* ── SENSORS ── */
.sensor-list { display:flex; flex-direction:column; gap:6px; }
.sensor-row  { display:flex; align-items:center; gap:10px; background:#f5f2ee; padding:7px 12px; border-radius:8px; border:1px solid #ddd8d0; }
.s-name { font-size:11px; font-weight:600; color:#4a4540; flex:1; }
.s-val  { font-family:'IBM Plex Mono',monospace; font-size:13px; font-weight:600; }
.sval--ok   { color:#0f6e44; }
.sval--warn { color:#854f0b; }
.sval--crit { color:#a32d2d; }
.s-led      { width:7px; height:7px; border-radius:50%; flex-shrink:0; }
.led--ok    { background:#1a8a7a; }
.led--warn  { background:#e67e22; }
.led--crit  { background:#c0392b; animation:blink 1s infinite; }

/* ── DIMM ── */
.dimm-grid { display:grid; grid-template-columns:1fr 1fr; gap:8px; }
.dimm-card { border-radius:12px; border:1.5px solid #ddd8d0; overflow:hidden; display:flex; transition:all .15s; }
.dimm-card:hover { border-color:#b8e4dc; }
.dc-accent { width:4px; flex-shrink:0; }
.dc--ok   .dc-accent { background:#1a8a7a; }
.dc--warn .dc-accent { background:#e67e22; }
.dc--crit .dc-accent { background:#c0392b; }
.dc-inner { flex:1; padding:10px 12px; }
.dc-slot  { font-size:9px; font-weight:700; color:#9a9490; text-transform:uppercase; letter-spacing:.06em; margin-bottom:4px; }
.dc-size  { font-size:20px; font-weight:800; color:#1a1714; line-height:1; letter-spacing:-.02em; }
.dc-unit  { font-size:11px; font-weight:600; color:#6b6560; }
.dc-info  { font-size:9px; color:#9a9490; font-weight:500; margin-top:4px; }
.dc-badge { display:inline-block; font-size:9px; font-weight:700; margin-top:6px; padding:2px 7px; border-radius:5px; }
.dcb--ok   { background:#e6f5f2; color:#0f6e44; }
.dcb--warn { background:#fef3e6; color:#854f0b; }
.dcb--crit { background:#fdecea; color:#a32d2d; }

/* ── PSU ── */
.psu-list { display:flex; flex-direction:column; gap:8px; }
.psu-row  { display:flex; align-items:center; gap:10px; padding:11px 14px; background:#f5f2ee; border-radius:10px; border:1px solid #ddd8d0; }
.psu-led  { width:7px; height:7px; border-radius:50%; flex-shrink:0; }
.psu-name { flex:1; font-size:11px; font-weight:600; color:#4a4540; }
.psu-w    { font-family:'IBM Plex Mono',monospace; font-size:12px; font-weight:600; color:#1a1714; }
.psu-status { font-size:9px; font-weight:700; padding:3px 8px; border-radius:5px; }
.st--ok   { background:#e6f5f2; color:#0f6e44; border-radius:5px; padding:2px 8px; font-size:9px; font-weight:700; }
.st--warn { background:#fef3e6; color:#854f0b; border-radius:5px; padding:2px 8px; font-size:9px; font-weight:700; }

/* ── TEMPS ── */
.temp-grid { display:grid; grid-template-columns:repeat(auto-fill,minmax(110px,1fr)); gap:8px; }
.temp-card { padding:12px 14px; border-radius:12px; border:1.5px solid #ddd8d0; background:white; transition:all .15s; }
.temp-card:hover { transform:scale(1.02); }
.tc--ok   { background:#f2fbf8; border-color:#b8e4dc; }
.tc--warn { background:#fef9f2; border-color:#f9d5a5; }
.tc--crit { background:#fdf3f3; border-color:#f5c0c0; }
.tc-val  { font-family:'IBM Plex Mono',monospace; font-size:20px; font-weight:600; }
.tc--ok   .tc-val { color:#0f6e44; }
.tc--warn .tc-val { color:#854f0b; }
.tc--crit .tc-val { color:#a32d2d; }
.tc-name  { font-size:9px; font-weight:700; color:#6b6560; text-transform:uppercase; margin-top:4px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; letter-spacing:.04em; }
.tc-badge { display:inline-block; font-size:8px; font-weight:700; margin-top:5px; padding:2px 6px; border-radius:4px; background:#e6f0fb; color:#185fa5; }
.tc-thresh { font-size:8px; color:#9a9490; margin-top:4px; display:flex; gap:6px; font-family:'IBM Plex Mono',monospace; }

/* ── FANS ── */
.fan-grid { display:grid; grid-template-columns:1fr 1fr; gap:8px; }
.fan-card { display:flex; align-items:center; gap:12px; padding:12px 14px; background:#f5f2ee; border-radius:12px; border:1px solid #ddd8d0; }
.fan-rotor { width:36px; height:36px; background:#e6f5f2; color:#1a8a7a; border-radius:50%; display:flex; align-items:center; justify-content:center; flex-shrink:0; }
.fan-spin { animation:spin linear infinite; }
.fan-info { flex:1; min-width:0; }
.fan-name { font-size:9px; font-weight:700; color:#9a9490; text-transform:uppercase; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; letter-spacing:.04em; }
.fan-rpm  { font-family:'IBM Plex Mono',monospace; font-size:15px; font-weight:600; color:#1a1714; margin-top:2px; }
.rpm-unit { font-size:9px; color:#9a9490; }
.rpm--warn { color:#854f0b; }
.fan-led  { width:7px; height:7px; border-radius:50%; flex-shrink:0; }

/* ── STORAGE ── */
.ctrl-block + .ctrl-block { margin-top:16px; }
.ctrl-label { display:flex; align-items:center; gap:5px; font-size:9px; font-weight:800; color:#9a9490; text-transform:uppercase; letter-spacing:.07em; padding-bottom:8px; border-bottom:1px dashed #ddd8d0; margin-bottom:8px; }
.drive-list { display:flex; flex-direction:column; gap:8px; }
.drive-row  { display:flex; align-items:center; gap:12px; padding:12px 14px; background:#f5f2ee; border-radius:10px; border:1px solid #ddd8d0; transition:all .15s; }
.drive-row:hover { border-color:#b8e4dc; background:white; }
.drive-type  { font-size:8px; font-weight:800; padding:3px 7px; border-radius:5px; flex-shrink:0; }
.dtype--ssd  { background:#e6f0fb; color:#185fa5; }
.dtype--hdd  { background:#f5f2ee; color:#6b6560; border:1px solid #ddd8d0; }
.drive-meta  { flex:1; min-width:0; }
.drive-name  { font-family:'IBM Plex Mono',monospace; font-size:11px; font-weight:600; color:#1a1714; display:block; }
.drive-model { font-size:9px; color:#9a9490; margin-top:2px; display:block; }
.drive-cap   { font-family:'IBM Plex Mono',monospace; font-size:12px; font-weight:600; color:#1a1714; white-space:nowrap; }
.drive-health { font-size:9px; font-weight:700; padding:3px 8px; border-radius:5px; white-space:nowrap; }

/* ── INFO TABLE ── */
.info-table { display:flex; flex-direction:column; }
.it-row { display:flex; justify-content:space-between; align-items:center; padding:11px 0; border-bottom:1px solid #ede9e4; gap:12px; }
.it-row:last-child { border-bottom:none; }
.it-lbl { font-size:10px; font-weight:700; color:#9a9490; text-transform:uppercase; letter-spacing:.07em; white-space:nowrap; }
.it-val { font-size:11px; font-weight:600; color:#1a1714; text-align:right; font-family:'IBM Plex Mono',monospace; }

/* ── ANIMATIONS ── */
@keyframes spin  { from { transform:rotate(0deg); } to { transform:rotate(360deg); } }
@keyframes blink { 0%,100% { opacity:1; } 50% { opacity:.25; } }
@keyframes slide { 0% { transform:translateX(-100%); } 50% { transform:translateX(200%); } 100% { transform:translateX(-100%); } }

/* ── RESPONSIVE ── */
@media (max-width:1200px) { .trio-grid { grid-template-columns:1fr 1fr; } .hero-strip { grid-template-columns:1fr 1fr; } }
@media (max-width:800px)  { .trio-grid,.duo-grid,.hero-strip { grid-template-columns:1fr; } .main { padding:18px 20px; } .topbar { padding:0 20px; } }
</style>