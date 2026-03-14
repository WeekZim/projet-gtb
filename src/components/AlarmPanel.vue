<script setup>
import { computed } from 'vue'

const props = defineProps({
  alarms: Array,
  sensors: Array
})

const emit = defineEmits(['acknowledge', 'go-to-sensor'])

function getSensorName(sensorId) {
  const s = props.sensors.find(x => x.id === sensorId)
  return s ? s.name : sensorId
}

const unacknowledgedCount = computed(() => props.alarms.filter(a => !a.acknowledged).length)
</script>

<template>
  <div class="status-sidebar__section" style="margin-top: auto; border-top: 1px solid var(--border-primary); border-bottom: none; max-height: 45vh; overflow-y: auto;">
    <div class="status-sidebar__title" style="display: flex; align-items: center; gap: 8px;">
      Alarmes Actives
      <span
        v-if="unacknowledgedCount > 0"
        style="
          font-family: var(--font-mono);
          font-size: 10px;
          font-weight: 700;
          padding: 1px 7px;
          border-radius: 8px;
          background: var(--status-fault-bg);
          color: var(--status-fault);
          border: 1px solid var(--status-fault-border);
          animation: blink 1.2s infinite;
        "
      >{{ unacknowledgedCount }}</span>
    </div>

    <div v-if="alarms.length === 0" style="font-size: 12px; color: var(--text-muted); padding: 12px 0; text-align: center;">
      Aucune alarme active
    </div>

    <div
      v-for="alarm in alarms"
      :key="alarm.id"
      class="alarm-item"
      :class="`alarm-item--${alarm.severity}`"
      @click="emit('go-to-sensor', alarm)"
    >
      <div class="alarm-item__header">
        <span class="alarm-item__severity">{{ alarm.severity === 'fault' ? '✕ DÉFAUT' : alarm.severity === 'alert' ? '⚠ ALERTE' : '○ HORS LIGNE' }}</span>
        <span v-if="alarm.acknowledged" class="alarm-item__ack">ACQ</span>
        <button
          v-else
          class="alarm-item__ack"
          style="cursor: pointer; border: none; font-family: var(--font-mono);"
          @click.stop="emit('acknowledge', alarm.id)"
          title="Acquitter l'alarme"
        >ACQ ?</button>
      </div>
      <div class="alarm-item__message">{{ alarm.message }}</div>
      <div class="alarm-item__time">{{ getSensorName(alarm.sensorId) }} — {{ alarm.timestamp }}</div>
    </div>
  </div>
</template>
