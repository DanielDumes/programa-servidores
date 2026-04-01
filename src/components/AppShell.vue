<template>
  <div class="shell">
    <aside class="sidebar">
      <div class="sb-logo">iL5</div>

      <nav class="sb-nav">
        <button
          class="sb-item"
          :class="{ 'sb-item--active': active === 'fleet' }"
          @click="$emit('navigate', 'fleet')"
          title="Flota"
        >
          <svg viewBox="0 0 24 24" fill="none" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
            <rect x="3" y="3" width="7" height="7" rx="1"/>
            <rect x="14" y="3" width="7" height="7" rx="1"/>
            <rect x="3" y="14" width="7" height="7" rx="1"/>
            <rect x="14" y="14" width="7" height="7" rx="1"/>
          </svg>
        </button>

        <button
          class="sb-item"
          :class="{ 'sb-item--active': active === 'detail' }"
          title="Servidor activo"
          v-if="active === 'detail'"
        >
          <svg viewBox="0 0 24 24" fill="none" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
            <rect x="2" y="2" width="20" height="8" rx="2"/>
            <rect x="2" y="14" width="20" height="8" rx="2"/>
            <circle cx="6" cy="6" r="1" fill="currentColor"/>
            <circle cx="6" cy="18" r="1" fill="currentColor"/>
          </svg>
        </button>
      </nav>

      <div class="sb-bottom">
        <div class="sb-avatar" title="Administrador">A</div>
      </div>
    </aside>

    <div class="shell-main">
      <slot />
    </div>
  </div>
</template>

<script setup>
defineProps({ active: { type: String, default: 'fleet' } })
defineEmits(['navigate'])
</script>

<style scoped>
.shell { display: flex; min-height: 100vh; }

.sidebar {
  width: 56px; flex-shrink: 0;
  background: #0C447C;
  display: flex; flex-direction: column;
  align-items: center; padding: 14px 0;
  position: sticky; top: 0; height: 100vh;
}

.sb-logo {
  width: 32px; height: 32px; border-radius: 8px;
  background: #185FA5; display: flex; align-items: center;
  justify-content: center; font-size: 10px; font-weight: 700;
  color: #E6F1FB; letter-spacing: -.3px; margin-bottom: 16px;
  flex-shrink: 0;
}

.sb-nav { display: flex; flex-direction: column; gap: 4px; flex: 1; }

.sb-item {
  width: 36px; height: 36px; border-radius: 8px;
  display: flex; align-items: center; justify-content: center;
  background: none; border: none; cursor: pointer;
  transition: background .15s;
}
.sb-item svg { width: 16px; height: 16px; stroke: #85B7EB; }
.sb-item:hover { background: rgba(255,255,255,.08); }
.sb-item--active { background: #185FA5; }
.sb-item--active svg { stroke: #E6F1FB; }

.sb-bottom { margin-top: auto; }
.sb-avatar {
  width: 30px; height: 30px; border-radius: 50%;
  background: #185FA5; display: flex; align-items: center;
  justify-content: center; font-size: 12px; font-weight: 600;
  color: #B5D4F4; cursor: pointer;
}

.shell-main { flex: 1; min-width: 0; display: flex; flex-direction: column; }
</style>