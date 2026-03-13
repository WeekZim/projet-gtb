<script setup>
import { computed } from 'vue'

const props = defineProps({
  sensor: { type: Object, required: true },
  isSelected: { type: Boolean, default: false }
})

const statusColors = {
  ok: '#00d68f',
  fault: '#ff3d71',
  alert: '#ffaa00',
  offline: '#556677'
}

const typeIcons = {
  temperature: '🌡',
  humidity: '💧',
  presence: '👁',
  door: 'bx bxs-door-open',
  smoke: '🔥',
  hvac: '❄',
  light: '💡',
  energy: '⚡',
  co2: '🌬',
  camera: '📷'
}

const color = computed(() => statusColors[props.sensor.status] || statusColors.offline)
const icon = computed(() => typeIcons[props.sensor.type] || '📍')
const statusClass = computed(() => `sensor-marker--${props.sensor.status}`)
</script>

<template>
  <!-- Outer g: positionnement uniquement -->
  <g :transform="`translate(${sensor.x}, ${sensor.y})`">
    <!-- Inner g: hover scale + interactions -->
    <g
      class="sensor-marker"
      :class="[statusClass, { 'sensor-marker--selected': isSelected }]"
    >
      <!-- Pulse ring (animated for fault/alert) -->
      <circle
        class="sensor-marker__ring"
        cx="0"
        cy="0"
        r="16"
        :stroke="color"
      />

      <!-- Background circle -->
      <circle
        class="sensor-marker__bg"
        cx="0"
        cy="0"
        r="13"
        :fill="color + '22'"
        :stroke="color"
      />

      <!-- Inner solid circle -->
      <circle
        cx="0"
        cy="0"
        r="9"
        :fill="color + 'cc'"
      />

      <!-- Icon -->
      <i
        :class="'sensor-marker__icon' + icon"
        x="0"
        y="0.5"
        font-size="10"
      >
  </i>

      <!-- Selection indicator -->
      <circle
        v-if="isSelected"
        cx="0"
        cy="0"
        r="18"
        fill="none"
        :stroke="color"
        stroke-width="1.5"
        stroke-dasharray="4 3"
        opacity="0.7"
      >
        <animateTransform
          attributeName="transform"
          type="rotate"
          from="0"
          to="360"
          dur="8s"
          repeatCount="indefinite"
        />
      </circle>

      <!-- Tooltip label on hover -->
      <title>{{ sensor.name }} — {{ sensor.value }}</title>
    </g>
  </g>
</template>
