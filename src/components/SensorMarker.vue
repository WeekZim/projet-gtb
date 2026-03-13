<script setup>
import { computed } from 'vue'

const props = defineProps({
  sensor: { type: Object, required: true },
  isSelected: { type: Boolean, default: false },
  size: { type: Number, default: 1 }
})

const typeIcons = {
  temperature: 'bx bxs-thermometer',
  humidity: 'bx bxs-droplet-half',
  presence: 'bx bxs-show',
  door: 'bx bxs-door-open',
  smoke: 'bx bxs-flame',
  hvac: 'bx bx-wind',
  light: 'bx bxs-bulb',
  energy: 'bx bxs-bolt',
  co2: 'bx bxs-cloud',
  camera: 'bx bxs-camera'
}

const icon = computed(() => typeIcons[props.sensor.type] || 'bx bxs-map-pin')
const statusClass = computed(() => `sensor-marker--${props.sensor.status}`)

const s = computed(() => {
  const b = props.size
  return {
    ring: 16 * b,
    bg: 13 * b,
    inner: 9 * b,
    icon: 16 * b,
    iconOffset: -8 * b,
    fontSize: 12 * b,
    hit: 18 * b,
    select: 20 * b,
    pulseMax: 16 * b * 1.4,
    pulseAlertMax: 16 * b * 1.25
  }
})
</script>

<template>
  <g :transform="`translate(${sensor.x}, ${sensor.y})`">
    <g
      class="sensor-marker"
      :class="[statusClass, { 'sensor-marker--selected': isSelected }]"
      :style="{
        '--pulse-r': s.ring,
        '--pulse-r-max': s.pulseMax,
        '--pulse-alert-max': s.pulseAlertMax
      }"
    >
      <!-- Pulse ring -->
      <circle
        class="sensor-marker__ring"
        cx="0" cy="0" :r="s.ring"
        stroke-width="2"
      />

      <!-- Cercle de fond -->
      <circle
        class="sensor-marker__bg"
        cx="0" cy="0" :r="s.bg"
        stroke-width="1"
      />

      <!-- Cercle intérieur plein -->
      <circle
        class="sensor-marker__inner"
        cx="0" cy="0" :r="s.inner"
      />

      <!-- Icône Boxicons -->
      <foreignObject
        :x="s.iconOffset" :y="s.iconOffset"
        :width="s.icon" :height="s.icon"
        style="pointer-events: none;"
      >
        <i
          xmlns="http://www.w3.org/1999/xhtml"
          :class="icon"
          class="sensor-marker__icon"
          :style="{
            fontSize: s.fontSize + 'px',
            width: s.icon + 'px',
            height: s.icon + 'px',
            lineHeight: s.icon + 'px'
          }"
        ></i>
      </foreignObject>

      <!-- Zone de clic élargie -->
      <circle cx="0" cy="0" :r="s.hit" fill="transparent" />

      <!-- Anneau de sélection -->
      <circle
        v-if="isSelected"
        class="sensor-marker__select"
        cx="0" cy="0" :r="s.select"
        stroke-width="1.5"
        stroke-dasharray="4 3"
      >
        <animateTransform
          attributeName="transform" type="rotate"
          from="0" to="360" dur="8s" repeatCount="indefinite"
        />
      </circle>

      <title>{{ sensor.name }} — {{ sensor.value }}</title>
    </g>
  </g>
</template>

<style scoped>
/* ══════════════════════════════════════
   Variables de couleurs — à modifier ici
   ══════════════════════════════════════ */
.sensor-marker {
  --color-icon: #fff;

  /* Couleur active (écrasée par les classes --status) */
  --color: var(--status-offline);

  cursor: pointer;
  transition: transform 150ms ease;
}

/* Assignation de --color selon le statut */
.sensor-marker--ok      { --color: var(--status-ok); }
.sensor-marker--fault   { --color: var(--status-fault); }
.sensor-marker--alert   { --color: var(--status-alert); }
.sensor-marker--offline { --color: var(--status-offline); opacity: 0.5; }

.sensor-marker:hover {
  filter: brightness(1.2);
}

/* ── Éléments SVG pilotés par --color ── */
.sensor-marker__ring {
  fill: none;
  stroke: var(--color);
  opacity: 0;
  transition: opacity 200ms ease;
}

.sensor-marker__bg {
  fill: color-mix(in srgb, var(--color) 13%, transparent);
  stroke: var(--color);
}

.sensor-marker__inner {
  fill: color-mix(in srgb, var(--color) 80%, transparent);
}

.sensor-marker__select {
  fill: none;
  stroke: var(--color);
  opacity: 0.7;
}

.sensor-marker__icon {
  color: var(--color-icon);
  display: flex;
  align-items: center;
  justify-content: center;
}

/* ── Pulse fault ── */
.sensor-marker--fault .sensor-marker__ring {
  opacity: 0.6;
  animation: pulse-fault 1.2s ease-in-out infinite;
}

/* ── Pulse alert ── */
.sensor-marker--alert .sensor-marker__ring {
  opacity: 0.5;
  animation: pulse-alert 2s ease-in-out infinite;
}

/* ── Animations ── */
@keyframes pulse-fault {
  0%, 100% { r: var(--pulse-r); opacity: 0.6; }
  50%      { r: var(--pulse-r-max); opacity: 0; }
}

@keyframes pulse-alert {
  0%, 100% { r: var(--pulse-r); opacity: 0.5; }
  50%      { r: var(--pulse-alert-max); opacity: 0; }
}
</style>