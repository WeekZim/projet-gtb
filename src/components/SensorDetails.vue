<script setup>
import { computed } from 'vue'

const props = defineProps({
  sensor: { type: Object, default: null }
})

const emit = defineEmits(['resolve-sensor'])

const statusLabels = { ok: 'Normal', fault: 'Défaut', alert: 'Alerte', offline: 'Hors ligne' }
const typeLabels = {
  temperature: 'Sonde de température',
  humidity: "Sonde d'humidité",
  presence: 'Détecteur de présence',
  door: "Contrôle d'accès",
  smoke: 'Détecteur de fumée',
  hvac: "Centrale de traitement d'air",
  light: 'Contrôle éclairage',
  energy: "Compteur d'énergie",
  co2: 'Sonde CO₂',
  camera: 'Caméra de surveillance',
  meter: 'Centrale de mesure',
  badge: 'Lecteur de badge'
}

const statusLabel = computed(() => statusLabels[props.sensor?.status] || '—')
const typeLabel = computed(() => typeLabels[props.sensor?.type] || props.sensor?.type)
const hasProblem = computed(() => props.sensor && (props.sensor.status === 'fault' || props.sensor.status === 'alert'))

const valueBoxStyle = computed(() => {
  if (!props.sensor) return {}
  const colorMap = {
    ok:      { bg: 'var(--status-ok-bg)',      border: 'var(--status-ok-border)',      color: 'var(--status-ok)' },
    fault:   { bg: 'var(--status-fault-bg)',   border: 'var(--status-fault-border)',   color: 'var(--status-fault)' },
    alert:   { bg: 'var(--status-alert-bg)',   border: 'var(--status-alert-border)',   color: 'var(--status-alert)' },
    offline: { bg: 'var(--status-offline-bg)', border: 'var(--status-offline-border)', color: 'var(--status-offline)' }
  }
  const c = colorMap[props.sensor.status] || colorMap.offline
  return { background: c.bg, border: `1px solid ${c.border}`, '--value-color': c.color }
})
</script>

<template>
  <div class="sensor-detail">
    <div class="sensor-detail__header">
      <span class="sensor-detail__section-label">Détail capteur</span>
    </div>

    <div v-if="!sensor" class="sensor-detail__empty">
      <div class="sensor-detail__empty-icon">◎</div>
      Sélectionnez un capteur sur le plan<br />ou dans la liste ci-dessous
    </div>

    <Transition name="fade">
      <div v-if="sensor" class="sensor-detail__content" :key="sensor.id + sensor.status">
        <div class="sensor-detail__name">{{ sensor.name }}</div>
        <div class="sensor-detail__type">{{ typeLabel }}</div>

        <div class="sensor-detail__value-box" :style="valueBoxStyle">
          <div class="sensor-detail__value-label" style="color: var(--text-muted)">Valeur actuelle</div>
          <div class="sensor-detail__value" :style="{ color: valueBoxStyle['--value-color'] }">{{ sensor.value }}</div>
        </div>

        <!-- Bloc raison — uniquement si une raison est renseignée -->
        <div v-if="hasProblem && sensor.reason" class="sensor-detail__reason" :class="'sensor-detail__reason--' + sensor.status">
          <div class="sensor-detail__reason-header">
            <span class="sensor-detail__reason-icon">{{ sensor.status === 'fault' ? '⚠' : '⚡' }}</span>
            <span class="sensor-detail__reason-title">{{ sensor.status === 'fault' ? 'Cause du défaut' : "Raison de l'alerte" }}</span>
          </div>
          <div class="sensor-detail__reason-text">{{ sensor.reason }}</div>
        </div>

        <!-- Bouton acquitter — visible dès qu'il y a un fault ou alert, raison ou pas -->
        <button
          v-if="hasProblem"
          class="resolve-btn"
          :class="'resolve-btn--' + sensor.status"
          style="margin-top: var(--gap-md)"
          @click="emit('resolve-sensor', sensor.id)"
        >
          <span class="resolve-btn__icon">✓</span>
          {{ sensor.status === 'fault' ? 'Acquitter & Résoudre' : "Acquitter l'alerte" }}
        </button>

        <div class="sensor-detail__meta">
          <div class="sensor-detail__meta-row">
            <span class="sensor-detail__meta-label">État</span>
            <span class="status-badge" :class="'status-badge--' + sensor.status">
              <span class="status-badge__dot"></span>
              {{ statusLabel }}
            </span>
          </div>
          <div class="sensor-detail__meta-row">
            <span class="sensor-detail__meta-label">Identifiant</span>
            <span class="sensor-detail__meta-value">{{ sensor.id }}</span>
          </div>
          <div class="sensor-detail__meta-row">
            <span class="sensor-detail__meta-label">Position</span>
            <span class="sensor-detail__meta-value">x:{{ sensor.x }} y:{{ sensor.y }}</span>
          </div>
          <div class="sensor-detail__meta-row">
            <span class="sensor-detail__meta-label">Source</span>
            <span class="sensor-detail__meta-value">{{ sensor.source || '—' }}</span>
          </div>
          <div class="sensor-detail__meta-row">
            <span class="sensor-detail__meta-label">Dernière MAJ</span>
            <span class="sensor-detail__meta-value">{{ sensor.lastUpdate }}</span>
          </div>
        </div>
      </div>
    </Transition>
  </div>
</template>