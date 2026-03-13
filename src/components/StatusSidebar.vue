<script setup>
import { ref, computed } from 'vue'
import SensorDetails from './SensorDetails.vue'
import CommandPanel from './CommandPanel.vue'

const props = defineProps({
  sensors: { type: Array, required: true },
  allSensors: { type: Array, default: () => [] },
  selectedSensor: { type: Object, default: null },
  floorInfo: { type: Object, default: null },
  eventLog: { type: Array, default: () => [] },
  floorRooms: { type: Array, default: () => [] },
  roomCommands: { type: Object, default: () => ({}) }
})

const emit = defineEmits(['select-sensor', 'resolve-sensor', 'update-room-command'])

const activeFilter = ref('all')
const activeTab = ref('status')

const stats = computed(() => ({
  ok: props.sensors.filter(s => s.status === 'ok').length,
  fault: props.sensors.filter(s => s.status === 'fault').length,
  alert: props.sensors.filter(s => s.status === 'alert').length,
  offline: props.sensors.filter(s => s.status === 'offline').length
}))

const statusPriority = { fault: 0, alert: 1, offline: 2, ok: 3 }

const filteredSensors = computed(() => {
  let list = [...props.sensors]
  if (activeFilter.value !== 'all') list = list.filter(s => s.status === activeFilter.value)
  list.sort((a, b) => (statusPriority[a.status] ?? 4) - (statusPriority[b.status] ?? 4))
  return list
})

const filters = [
  { key: 'all', label: 'Tous' },
  { key: 'fault', label: 'Défauts' },
  { key: 'alert', label: 'Alertes' },
  { key: 'ok', label: 'OK' }
]

const tabs = [
  { key: 'status', label: 'État' },
  { key: 'commands', label: 'Commandes' },
  { key: 'events', label: 'Événements' }
]

const recentEvents = computed(() => props.eventLog.slice(0, 20))
function eventTypeLabel(type) {
  return { fault: 'DÉFAUT', alert: 'ALERTE', resolved: 'RÉSOLU' }[type] || type
}
</script>

<template>
  <aside class="status-sidebar">
    <SensorDetails :sensor="selectedSensor" @resolve-sensor="(id) => emit('resolve-sensor', id)" />

    <div class="sidebar-tabs">
      <button v-for="tab in tabs" :key="tab.key"
        class="sidebar-tab" :class="{ 'sidebar-tab--active': activeTab === tab.key }"
        @click="activeTab = tab.key">
        {{ tab.label }}
        <span v-if="tab.key === 'events' && eventLog.length > 0" class="sidebar-tab__badge">{{ eventLog.length }}</span>
      </button>
    </div>

    <!-- TAB: Status -->
    <template v-if="activeTab === 'status'">
      <div class="sidebar-summary">
        <div class="sidebar-summary__header"><span class="sidebar-summary__label">Résumé · {{ floorInfo?.label }}</span></div>
        <div class="sidebar-summary__cards">
          <div class="summary-card summary-card--ok"><div class="summary-card__count">{{ stats.ok }}</div><div class="summary-card__label">Normal</div></div>
          <div class="summary-card summary-card--fault"><div class="summary-card__count">{{ stats.fault }}</div><div class="summary-card__label">Défaut</div></div>
          <div class="summary-card summary-card--alert"><div class="summary-card__count">{{ stats.alert }}</div><div class="summary-card__label">Alerte</div></div>
          <div class="summary-card summary-card--offline"><div class="summary-card__count">{{ stats.offline }}</div><div class="summary-card__label">Hors ligne</div></div>
        </div>
      </div>
      <div class="sensor-list">
        <div class="sensor-list__header">
          <span class="sensor-list__label">Capteurs</span>
          <div class="sensor-list__filter">
            <button v-for="f in filters" :key="f.key" class="filter-btn" :class="{ 'filter-btn--active': activeFilter === f.key }" @click="activeFilter = f.key">{{ f.label }}</button>
          </div>
        </div>
        <div class="sensor-list__items">
          <div v-for="sensor in filteredSensors" :key="sensor.id"
            class="sensor-list-item" :class="{ 'sensor-list-item--active': selectedSensor?.id === sensor.id }"
            @click="emit('select-sensor', sensor)">
            <span class="sensor-list-item__dot" :class="'sensor-list-item__dot--' + sensor.status"></span>
            <div class="sensor-list-item__info">
              <div class="sensor-list-item__name">{{ sensor.name }}</div>
              <div class="sensor-list-item__type">{{ sensor.type }}</div>
            </div>
            <span class="sensor-list-item__value">{{ sensor.value }}</span>
          </div>
          <div v-if="filteredSensors.length === 0" style="text-align: center; color: var(--text-muted); padding: 24px; font-size: 13px;">
            Aucun capteur avec ce filtre
          </div>
        </div>
      </div>
    </template>

    <!-- TAB: Commands -->
    <template v-if="activeTab === 'commands'">
      <CommandPanel
        :rooms="floorRooms"
        :sensors="sensors"
        :roomCommands="roomCommands"
        @update-room-command="(payload) => emit('update-room-command', payload)"
      />
    </template>

    <!-- TAB: Events -->
    <template v-if="activeTab === 'events'">
      <div class="event-log">
        <div class="event-log__header"><span class="sidebar-summary__label">Journal des événements</span></div>
        <div v-if="recentEvents.length === 0" style="text-align: center; color: var(--text-muted); padding: 32px; font-size: 13px;">
          Aucun événement pour le moment
        </div>
        <div v-for="ev in recentEvents" :key="ev.id" class="event-log__item" :class="'event-log__item--' + ev.type">
          <div class="event-log__item-header">
            <span class="event-log__item-badge" :class="'event-log__item-badge--' + ev.type">{{ eventTypeLabel(ev.type) }}</span>
            <span class="event-log__item-time">{{ ev.time }}</span>
          </div>
          <div class="event-log__item-sensor">{{ ev.sensorName }}</div>
          <div class="event-log__item-reason">{{ ev.reason }}</div>
        </div>
      </div>
    </template>
  </aside>
</template>
