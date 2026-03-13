<script setup>
import { computed } from 'vue'

const props = defineProps({
  floors: { type: Array, required: true },
  sensors: { type: Array, required: true },
  selectedFloor: { type: Number, required: true }
})

const emit = defineEmits(['select-floor'])

function floorStats(floorId) {
  const floorSensors = props.sensors.filter(s => s.floor === floorId)
  return {
    total: floorSensors.length,
    ok: floorSensors.filter(s => s.status === 'ok').length,
    fault: floorSensors.filter(s => s.status === 'fault').length,
    alert: floorSensors.filter(s => s.status === 'alert').length
  }
}

const globalStats = computed(() => {
  return {
    total: props.sensors.length,
    ok: props.sensors.filter(s => s.status === 'ok').length,
    fault: props.sensors.filter(s => s.status === 'fault').length,
    alert: props.sensors.filter(s => s.status === 'alert').length,
    offline: props.sensors.filter(s => s.status === 'offline').length
  }
})
</script>

<template>
  <aside class="floor-selector">
    <div class="floor-selector__header">
      <span class="floor-selector__label">Niveaux</span>
    </div>

    <div class="floor-selector__list">
      <button
        v-for="floor in floors"
        :key="floor.id"
        class="floor-btn"
        :class="{ 'floor-btn--active': selectedFloor === floor.id }"
        @click="emit('select-floor', floor.id)"
      >
        <span class="floor-btn__code">{{ floor.label }}</span>
        <span class="floor-btn__name">
          {{ floor.name.split(' — ')[1] || floor.name }}
          <span
            v-if="floorStats(floor.id).fault > 0"
            style="color: var(--status-fault); margin-left: 4px; font-size: 10px;"
          >
            ● {{ floorStats(floor.id).fault }}
          </span>
          <span
            v-if="floorStats(floor.id).alert > 0"
            style="color: var(--status-alert); margin-left: 4px; font-size: 10px;"
          >
            ● {{ floorStats(floor.id).alert }}
          </span>
        </span>
      </button>
    </div>

    <div class="floor-selector__stats">
      <div class="floor-selector__stat-row">
        <span class="floor-selector__stat-label">Total capteurs</span>
        <span class="floor-selector__stat-value" style="color: var(--text-primary);">
          {{ globalStats.total }}
        </span>
      </div>
      <div class="floor-selector__stat-row">
        <span class="floor-selector__stat-label">En service</span>
        <span class="floor-selector__stat-value" style="color: var(--status-ok);">
          {{ globalStats.ok }}
        </span>
      </div>
      <div class="floor-selector__stat-row">
        <span class="floor-selector__stat-label">Défauts</span>
        <span class="floor-selector__stat-value" style="color: var(--status-fault);">
          {{ globalStats.fault }}
        </span>
      </div>
      <div class="floor-selector__stat-row">
        <span class="floor-selector__stat-label">Alertes</span>
        <span class="floor-selector__stat-value" style="color: var(--status-alert);">
          {{ globalStats.alert }}
        </span>
      </div>
    </div>
  </aside>
</template>
