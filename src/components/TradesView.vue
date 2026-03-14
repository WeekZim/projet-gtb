<script setup>
import { ref, computed } from 'vue'

const props = defineProps({
  sensors: { type: Array, required: true },
  floors: { type: Array, required: true },
  rooms: { type: Array, required: true },
  eventLog: { type: Array, default: () => [] }
})

const emit = defineEmits(['select-sensor', 'resolve-sensor'])

// ═══════════════════════════════════════════════════════
// TRADES (MÉTIERS) DEFINITION
// ═══════════════════════════════════════════════════════
const trades = [
  { id: 'cvc', label: 'CVC', fullLabel: 'Chauffage, Ventilation, Climatisation', icon: '❄', color: '#00aaff', colorBg: 'rgba(0, 170, 255, 0.08)', colorBorder: 'rgba(0, 170, 255, 0.25)', types: ['temperature', 'humidity', 'hvac', 'co2'],
    subTrades: [{ id: 'temperature', label: 'Température', icon: '🌡' }, { id: 'humidity', label: 'Humidité', icon: '💧' }, { id: 'hvac', label: 'CTA / Climatisation', icon: '❄' }, { id: 'co2', label: 'Qualité d\'air (CO₂)', icon: '🌬' }] },
  { id: 'electricite', label: 'Électricité', fullLabel: 'Éclairage & Énergie', icon: '⚡', color: '#ffaa00', colorBg: 'rgba(255, 170, 0, 0.08)', colorBorder: 'rgba(255, 170, 0, 0.25)', types: ['light', 'energy'],
    subTrades: [{ id: 'light', label: 'Éclairage', icon: '💡' }, { id: 'energy', label: 'Compteurs Énergie', icon: '⚡' }] },
  { id: 'securite_incendie', label: 'Sécurité Incendie', fullLabel: 'Détection Incendie & Désenfumage', icon: '🔥', color: '#ff3d71', colorBg: 'rgba(255, 61, 113, 0.08)', colorBorder: 'rgba(255, 61, 113, 0.25)', types: ['smoke'],
    subTrades: [{ id: 'smoke', label: 'Détecteurs Fumée', icon: '🔥' }] },
  { id: 'surete', label: 'Sûreté', fullLabel: 'Contrôle d\'Accès & Vidéosurveillance', icon: '🔒', color: '#a855f7', colorBg: 'rgba(168, 85, 247, 0.08)', colorBorder: 'rgba(168, 85, 247, 0.25)', types: ['door', 'presence', 'camera'],
    subTrades: [{ id: 'door', label: 'Portes / Accès', icon: '🚪' }, { id: 'presence', label: 'Détection Présence', icon: '👁' }, { id: 'camera', label: 'Vidéosurveillance', icon: '📹' }] }
]

const suggestedActions = {
  temperature: { fault: 'Vérifier câblage sonde · Remplacer si HS', alert: 'Vérifier consigne CVC · Contrôler isolation' },
  humidity: { fault: 'Remplacer la sonde', alert: 'Ajuster ventilation · Vérifier VMC' },
  hvac: { fault: 'Contrôler disjoncteur · Vérifier bus CAN/Modbus', alert: 'Planifier remplacement filtre · Nettoyage CTA' },
  co2: { fault: 'Étalonnage sonde CO₂ · Remplacement si défaillante', alert: 'Augmenter débit ventilation · Ouvrir fenêtres' },
  light: { fault: 'Vérifier driver DALI · Remplacer ballast', alert: 'Planifier remplacement lampe' },
  energy: { fault: 'Vérifier liaison Modbus · Contrôler compteur', alert: 'Analyser profil de consommation' },
  smoke: { fault: 'Nettoyer chambre optique · Remplacer détecteur', alert: 'Vérifier environnement · Test désenfumage' },
  door: { fault: 'Vérifier gâche électrique · Contrôler alimentation', alert: 'Vérifier ferme-porte · Régler temporisation' },
  presence: { fault: 'Nettoyer lentille PIR · Vérifier zone détection', alert: 'Ajuster sensibilité capteur' },
  camera: { fault: 'Vérifier PoE · Contrôler câble réseau', alert: 'Nettoyer lentille caméra' }
}

const severityConfig = {
  fault: { label: 'CRITIQUE', priority: 1, icon: '✕' },
  alert: { label: 'ATTENTION', priority: 2, icon: '⚠' },
  offline: { label: 'HORS LIGNE', priority: 3, icon: '○' }
}

const typeLabels = {
  temperature: 'Sonde de température', humidity: 'Sonde d\'humidité', presence: 'Détecteur de présence',
  door: 'Contrôle d\'accès', smoke: 'Détecteur de fumée', hvac: 'Centrale traitement d\'air',
  light: 'Contrôle éclairage', energy: 'Compteur d\'énergie', co2: 'Sonde CO₂', camera: 'Caméra de surveillance'
}

const sensorTypeIcons = { temperature: '🌡', humidity: '💧', hvac: '❄', co2: '🌬', light: '💡', energy: '⚡', smoke: '🔥', door: '🚪', presence: '👁', camera: '📹' }

// ── State ──
const selectedTradeId = ref('cvc')
const selectedSubTradeId = ref(null)
const selectedSensor = ref(null)
const expandedFloors = ref({})
const problemViewMode = ref('severity')

const activeTrade = computed(() => trades.find(t => t.id === selectedTradeId.value))

const tradeSensors = computed(() => {
  const types = activeTrade.value?.types || []
  let filtered = props.sensors.filter(s => types.includes(s.type))
  if (selectedSubTradeId.value) filtered = filtered.filter(s => s.type === selectedSubTradeId.value)
  return filtered
})

function tradeStats(trade) {
  const s = props.sensors.filter(x => trade.types.includes(x.type))
  return { total: s.length, ok: s.filter(x => x.status === 'ok').length, fault: s.filter(x => x.status === 'fault').length, alert: s.filter(x => x.status === 'alert').length, offline: s.filter(x => x.status === 'offline').length }
}

function tradeHealthPercent(trade) {
  const st = tradeStats(trade)
  return st.total === 0 ? 100 : Math.round((st.ok / st.total) * 100)
}

const activeStats = computed(() => {
  const s = tradeSensors.value
  return { total: s.length, ok: s.filter(x => x.status === 'ok').length, fault: s.filter(x => x.status === 'fault').length, alert: s.filter(x => x.status === 'alert').length, offline: s.filter(x => x.status === 'offline').length }
})

const activeHealthPercent = computed(() => activeStats.value.total === 0 ? 100 : Math.round((activeStats.value.ok / activeStats.value.total) * 100))

const problemSensors = computed(() => tradeSensors.value.filter(s => s.status !== 'ok').sort((a, b) => (severityConfig[a.status]?.priority ?? 9) - (severityConfig[b.status]?.priority ?? 9)))

const problemsBySeverity = computed(() => {
  const groups = {}
  problemSensors.value.forEach(s => { if (!groups[s.status]) groups[s.status] = []; groups[s.status].push(s) })
  return groups
})

const problemsByZone = computed(() => {
  const groups = {}
  problemSensors.value.forEach(s => {
    const key = `${s.floor}-${s.room}`
    if (!groups[key]) groups[key] = { floorId: s.floor, floorLabel: getFloorLabel(s.floor), roomLabel: getRoomLabel(s.room), sensors: [] }
    groups[key].sensors.push(s)
  })
  return Object.values(groups).sort((a, b) => Math.min(...a.sensors.map(s => severityConfig[s.status]?.priority ?? 9)) - Math.min(...b.sensors.map(s => severityConfig[s.status]?.priority ?? 9)))
})

const problemsByType = computed(() => {
  const groups = {}
  problemSensors.value.forEach(s => {
    if (!groups[s.type]) groups[s.type] = { type: s.type, typeLabel: typeLabels[s.type] || s.type, icon: sensorTypeIcons[s.type] || '●', sensors: [] }
    groups[s.type].sensors.push(s)
  })
  return Object.values(groups).sort((a, b) => Math.min(...a.sensors.map(s => severityConfig[s.status]?.priority ?? 9)) - Math.min(...b.sensors.map(s => severityConfig[s.status]?.priority ?? 9)))
})

const affectedZonesCount = computed(() => { const z = new Set(); problemSensors.value.forEach(s => z.add(`${s.floor}-${s.room}`)); return z.size })
const affectedFloors = computed(() => { const f = new Set(); problemSensors.value.forEach(s => f.add(s.floor)); return [...f].map(fid => props.floors.find(fl => fl.id === fid)).filter(Boolean) })

const sensorsByFloor = computed(() => {
  const groups = {}
  props.floors.forEach(f => {
    const fs = tradeSensors.value.filter(s => s.floor === f.id)
    if (fs.length > 0) {
      groups[f.id] = { floor: f, sensors: fs.sort((a, b) => (severityConfig[a.status]?.priority ?? 9) - (severityConfig[b.status]?.priority ?? 9)),
        stats: { total: fs.length, ok: fs.filter(x => x.status === 'ok').length, fault: fs.filter(x => x.status === 'fault').length, alert: fs.filter(x => x.status === 'alert').length, offline: fs.filter(x => x.status === 'offline').length } }
    }
  })
  return groups
})

const tradeEvents = computed(() => {
  const types = activeTrade.value?.types || []
  return props.eventLog.filter(ev => { const s = props.sensors.find(x => x.id === ev.sensorId); return s && types.includes(s.type) }).slice(0, 30)
})

function selectTrade(id) { selectedTradeId.value = id; selectedSubTradeId.value = null; selectedSensor.value = null }
function selectSubTrade(id) { selectedSubTradeId.value = selectedSubTradeId.value === id ? null : id; selectedSensor.value = null }
function selectSensor(s) { selectedSensor.value = selectedSensor.value?.id === s.id ? null : s; emit('select-sensor', s) }
function toggleFloor(id) { expandedFloors.value[id] = !expandedFloors.value[id] }
function isFloorExpanded(id) { return expandedFloors.value[id] !== false }
function resolveSensor(id) { emit('resolve-sensor', id); if (selectedSensor.value?.id === id) selectedSensor.value = null }
function getFloorLabel(fid) { const f = props.floors.find(x => x.id === fid); return f ? f.label : `Ét. ${fid}` }
function getRoomLabel(rid) { const r = props.rooms.find(x => x.id === rid); return r ? r.label : rid }
function getAction(s) { return suggestedActions[s.type]?.[s.status] || 'Vérifier sur site' }
function statusLabel(st) { return severityConfig[st]?.label || 'OK' }
function eventTypeLabel(t) { return { fault: 'DÉFAUT', alert: 'ALERTE', resolved: 'RÉSOLU', offline: 'H/L' }[t] || t }
function healthColor(p) { if (p >= 90) return 'var(--status-ok)'; if (p >= 70) return 'var(--status-alert)'; return 'var(--status-fault)' }
</script>

<template>
  <div class="trades-view">
    <!-- ═══ LEFT NAV ═══ -->
    <aside class="trades-nav">
      <div class="trades-nav__header"><span class="trades-nav__label">Métiers</span></div>
      <div class="trades-nav__list">
        <button v-for="trade in trades" :key="trade.id" class="trade-btn" :class="{ 'trade-btn--active': selectedTradeId === trade.id, 'trade-btn--has-fault': tradeStats(trade).fault > 0, 'trade-btn--has-alert': tradeStats(trade).fault === 0 && tradeStats(trade).alert > 0 }" @click="selectTrade(trade.id)">
          <div class="trade-btn__top">
            <div class="trade-btn__header"><span class="trade-btn__icon">{{ trade.icon }}</span><span class="trade-btn__label">{{ trade.label }}</span></div>
            <div class="trade-btn__badges">
              <span v-if="tradeStats(trade).fault > 0" class="trade-btn__badge trade-btn__badge--fault">{{ tradeStats(trade).fault }}</span>
              <span v-if="tradeStats(trade).alert > 0" class="trade-btn__badge trade-btn__badge--alert">{{ tradeStats(trade).alert }}</span>
            </div>
          </div>
          <div class="trade-btn__health-bar"><div class="trade-btn__health-fill" :style="{ width: tradeHealthPercent(trade) + '%', background: healthColor(tradeHealthPercent(trade)) }"></div></div>
          <div class="trade-btn__health-text"><span>{{ tradeHealthPercent(trade) }}% opérationnel</span><span>{{ tradeStats(trade).total }} pts</span></div>
        </button>
      </div>
      <div class="trades-nav__overview">
        <div class="trades-nav__overview-title">Santé globale</div>
        <div v-for="trade in trades" :key="'ov-' + trade.id" class="trades-nav__overview-row">
          <span class="trades-nav__overview-icon">{{ trade.icon }}</span>
          <span class="trades-nav__overview-label">{{ trade.label }}</span>
          <span class="trades-nav__overview-dot" :class="{ 'trades-nav__overview-dot--ok': tradeStats(trade).fault === 0 && tradeStats(trade).alert === 0, 'trades-nav__overview-dot--fault': tradeStats(trade).fault > 0, 'trades-nav__overview-dot--alert': tradeStats(trade).fault === 0 && tradeStats(trade).alert > 0 }"></span>
        </div>
      </div>
    </aside>

    <!-- ═══ CENTER ═══ -->
    <main class="trades-main">
      <!-- Header with gauge -->
      <div class="trades-main__header" :style="{ borderColor: activeTrade?.colorBorder }">
        <div class="trades-main__title-row">
          <span class="trades-main__icon" :style="{ background: activeTrade?.colorBg, color: activeTrade?.color }">{{ activeTrade?.icon }}</span>
          <div><h2 class="trades-main__title">{{ activeTrade?.label }}</h2><p class="trades-main__subtitle">{{ activeTrade?.fullLabel }}</p></div>
        </div>
        <div class="trades-header-right">
          <div class="health-gauge">
            <svg viewBox="0 0 80 80" class="health-gauge__svg">
              <circle cx="40" cy="40" r="34" fill="none" stroke="var(--border-primary)" stroke-width="6" />
              <circle cx="40" cy="40" r="34" fill="none" :stroke="healthColor(activeHealthPercent)" stroke-width="6" stroke-linecap="round" :stroke-dasharray="(2 * Math.PI * 34)" :stroke-dashoffset="(2 * Math.PI * 34) * (1 - activeHealthPercent / 100)" transform="rotate(-90 40 40)" class="health-gauge__arc" />
            </svg>
            <div class="health-gauge__text"><span class="health-gauge__pct" :style="{ color: healthColor(activeHealthPercent) }">{{ activeHealthPercent }}</span><span class="health-gauge__unit">%</span></div>
          </div>
          <div class="trades-stats">
            <div class="trades-stat trades-stat--ok"><span class="trades-stat__count">{{ activeStats.ok }}</span><span class="trades-stat__label">OK</span></div>
            <div class="trades-stat trades-stat--fault"><span class="trades-stat__count">{{ activeStats.fault }}</span><span class="trades-stat__label">Défaut</span></div>
            <div class="trades-stat trades-stat--alert"><span class="trades-stat__count">{{ activeStats.alert }}</span><span class="trades-stat__label">Alerte</span></div>
            <div class="trades-stat trades-stat--offline"><span class="trades-stat__count">{{ activeStats.offline }}</span><span class="trades-stat__label">H/L</span></div>
          </div>
        </div>
      </div>

      <!-- Subtrades filter -->
      <div class="subtrades-bar">
        <button class="subtrade-btn" :class="{ 'subtrade-btn--active': selectedSubTradeId === null }" @click="selectedSubTradeId = null">Tous</button>
        <button v-for="sub in activeTrade?.subTrades" :key="sub.id" class="subtrade-btn" :class="{ 'subtrade-btn--active': selectedSubTradeId === sub.id }" @click="selectSubTrade(sub.id)"><span class="subtrade-btn__icon">{{ sub.icon }}</span>{{ sub.label }}</button>
      </div>

      <!-- ═══ PROBLEMS SECTION ═══ -->
      <div v-if="problemSensors.length > 0" class="problems-section">
        <div class="problems-section__header">
          <div class="problems-section__title-row">
            <span class="problems-section__icon">⚠</span>
            <div>
              <div class="problems-section__title">{{ problemSensors.length }} problème{{ problemSensors.length > 1 ? 's' : '' }} actif{{ problemSensors.length > 1 ? 's' : '' }}</div>
              <div class="problems-section__impact">{{ affectedZonesCount }} zone{{ affectedZonesCount > 1 ? 's' : '' }} impactée{{ affectedZonesCount > 1 ? 's' : '' }} · {{ affectedFloors.length }} étage{{ affectedFloors.length > 1 ? 's' : '' }} ({{ affectedFloors.map(f => f.label).join(', ') }})</div>
            </div>
          </div>
          <div class="problems-section__view-modes">
            <button class="view-mode-btn" :class="{ 'view-mode-btn--active': problemViewMode === 'severity' }" @click="problemViewMode = 'severity'">Par gravité</button>
            <button class="view-mode-btn" :class="{ 'view-mode-btn--active': problemViewMode === 'zone' }" @click="problemViewMode = 'zone'">Par zone</button>
            <button class="view-mode-btn" :class="{ 'view-mode-btn--active': problemViewMode === 'type' }" @click="problemViewMode = 'type'">Par type</button>
          </div>
        </div>

        <!-- BY SEVERITY -->
        <div v-if="problemViewMode === 'severity'" class="problems-list">
          <div v-for="(sensors, status) in problemsBySeverity" :key="'sev-' + status" class="severity-group">
            <div class="severity-group__header">
              <span class="severity-group__badge" :class="'severity-group__badge--' + status">{{ severityConfig[status]?.icon }} {{ severityConfig[status]?.label }}</span>
              <span class="severity-group__count">{{ sensors.length }} point{{ sensors.length > 1 ? 's' : '' }}</span>
            </div>
            <div class="severity-group__items">
              <div v-for="sensor in sensors" :key="sensor.id"
                class="pcard" :class="['pcard--' + sensor.status, { 'pcard--selected': selectedSensor?.id === sensor.id }]"
                @click="selectSensor(sensor)">
                <div class="pcard__row1">
                  <span class="pcard__icon">{{ sensorTypeIcons[sensor.type] || '●' }}</span>
                  <span class="pcard__name">{{ sensor.name }}</span>
                  <span class="pcard__value" :class="'pcard__value--' + sensor.status">{{ sensor.value }}</span>
                </div>
                <div class="pcard__row2">
                  <span class="pcard__location">{{ getFloorLabel(sensor.floor) }} · {{ getRoomLabel(sensor.room) }}</span>
                  <span class="pcard__time">{{ sensor.lastUpdate }}</span>
                </div>
                <div v-if="sensor.reason" class="pcard__reason" :class="'pcard__reason--' + sensor.status">
                  <span class="pcard__reason-label">Diagnostic :</span> {{ sensor.reason }}
                </div>
                <div class="pcard__bottom">
                  <div class="pcard__action">→ {{ getAction(sensor) }}</div>
                  <button class="pcard__resolve" @click.stop="resolveSensor(sensor.id)">✓ Résoudre</button>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- BY ZONE -->
        <div v-if="problemViewMode === 'zone'" class="problems-list">
          <div v-for="zone in problemsByZone" :key="'z-' + zone.floorId + zone.roomLabel" class="zone-group">
            <div class="zone-group__header">
              <span class="zone-group__floor">{{ zone.floorLabel }}</span>
              <span class="zone-group__room">{{ zone.roomLabel }}</span>
              <span class="zone-group__count">{{ zone.sensors.length }} problème{{ zone.sensors.length > 1 ? 's' : '' }}</span>
            </div>
            <div class="zone-group__items">
              <div v-for="sensor in zone.sensors" :key="sensor.id"
                class="pcard" :class="['pcard--' + sensor.status, { 'pcard--selected': selectedSensor?.id === sensor.id }]"
                @click="selectSensor(sensor)">
                <div class="pcard__row1">
                  <span class="pcard__icon">{{ sensorTypeIcons[sensor.type] || '●' }}</span>
                  <span class="pcard__name">{{ sensor.name }}</span>
                  <span class="pcard__badge-mini" :class="'pcard__badge-mini--' + sensor.status">{{ severityConfig[sensor.status]?.label }}</span>
                  <span class="pcard__value" :class="'pcard__value--' + sensor.status">{{ sensor.value }}</span>
                </div>
                <div v-if="sensor.reason" class="pcard__reason" :class="'pcard__reason--' + sensor.status">
                  <span class="pcard__reason-label">Diagnostic :</span> {{ sensor.reason }}
                </div>
                <div class="pcard__bottom">
                  <div class="pcard__action">→ {{ getAction(sensor) }}</div>
                  <button class="pcard__resolve" @click.stop="resolveSensor(sensor.id)">✓ Résoudre</button>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- BY TYPE -->
        <div v-if="problemViewMode === 'type'" class="problems-list">
          <div v-for="group in problemsByType" :key="'t-' + group.type" class="type-group">
            <div class="type-group__header">
              <span class="type-group__icon">{{ group.icon }}</span>
              <span class="type-group__label">{{ group.typeLabel }}</span>
              <span class="type-group__count">{{ group.sensors.length }}</span>
            </div>
            <div class="type-group__items">
              <div v-for="sensor in group.sensors" :key="sensor.id"
                class="pcard" :class="['pcard--' + sensor.status, { 'pcard--selected': selectedSensor?.id === sensor.id }]"
                @click="selectSensor(sensor)">
                <div class="pcard__row1">
                  <span class="pcard__badge-mini" :class="'pcard__badge-mini--' + sensor.status">{{ severityConfig[sensor.status]?.icon }}</span>
                  <span class="pcard__name">{{ sensor.name }}</span>
                  <span class="pcard__value" :class="'pcard__value--' + sensor.status">{{ sensor.value }}</span>
                </div>
                <div class="pcard__row2">
                  <span class="pcard__location">{{ getFloorLabel(sensor.floor) }} · {{ getRoomLabel(sensor.room) }}</span>
                  <span class="pcard__time">{{ sensor.lastUpdate }}</span>
                </div>
                <div v-if="sensor.reason" class="pcard__reason" :class="'pcard__reason--' + sensor.status">
                  <span class="pcard__reason-label">Diagnostic :</span> {{ sensor.reason }}
                </div>
                <div class="pcard__bottom">
                  <div class="pcard__action">→ {{ getAction(sensor) }}</div>
                  <button class="pcard__resolve" @click.stop="resolveSensor(sensor.id)">✓ Résoudre</button>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- No problems -->
      <div v-if="problemSensors.length === 0 && tradeSensors.length > 0" class="no-problems">
        <div class="no-problems__icon">✓</div>
        <div class="no-problems__title">Tout est opérationnel</div>
        <div class="no-problems__subtitle">{{ tradeSensors.length }} capteurs en fonctionnement normal</div>
      </div>

      <!-- ═══ ALL SENSORS by Floor ═══ -->
      <div class="floor-groups">
        <div class="floor-groups__title">Tous les capteurs</div>
        <div v-for="(group, floorId) in sensorsByFloor" :key="'fl-' + floorId" class="floor-group" :class="{ 'floor-group--has-fault': group.stats.fault > 0, 'floor-group--has-alert': group.stats.fault === 0 && group.stats.alert > 0 }">
          <div class="floor-group__header" @click="toggleFloor(Number(floorId))">
            <div class="floor-group__left"><span class="floor-group__chevron" :class="{ 'floor-group__chevron--open': isFloorExpanded(Number(floorId)) }">▸</span><span class="floor-group__label">{{ group.floor.label }}</span><span class="floor-group__name">{{ group.floor.name }}</span></div>
            <div class="floor-group__right">
              <div class="floor-group__mini-bar"><div class="floor-group__mini-fill" :style="{ width: (group.stats.ok / group.stats.total * 100) + '%' }"></div></div>
              <span class="floor-group__count">{{ group.stats.ok }}/{{ group.stats.total }}</span>
              <span v-if="group.stats.fault > 0" class="floor-group__badge floor-group__badge--fault">{{ group.stats.fault }}</span>
              <span v-if="group.stats.alert > 0" class="floor-group__badge floor-group__badge--alert">{{ group.stats.alert }}</span>
            </div>
          </div>
          <div v-if="isFloorExpanded(Number(floorId))" class="floor-group__body">
            <div class="sensor-grid">
              <div v-for="sensor in group.sensors" :key="sensor.id" class="sensor-card" :class="['sensor-card--' + sensor.status, { 'sensor-card--selected': selectedSensor?.id === sensor.id }]" @click="selectSensor(sensor)">
                <div class="sensor-card__header"><span class="sensor-card__icon">{{ sensorTypeIcons[sensor.type] || '●' }}</span><span class="sensor-card__status-dot" :class="'sensor-card__status-dot--' + sensor.status"></span></div>
                <div class="sensor-card__name">{{ sensor.name }}</div>
                <div class="sensor-card__value" :class="'sensor-card__value--' + sensor.status">{{ sensor.value }}</div>
                <div class="sensor-card__meta"><span>{{ getRoomLabel(sensor.room) }}</span><span>{{ sensor.lastUpdate?.split(' ')[1] || '' }}</span></div>
                <div v-if="sensor.reason" class="sensor-card__reason">{{ sensor.reason }}</div>
              </div>
            </div>
          </div>
        </div>
        <div v-if="Object.keys(sensorsByFloor).length === 0" class="trades-empty">Aucun capteur trouvé pour ce filtre.</div>
      </div>
    </main>

    <!-- ═══ RIGHT DETAIL ═══ -->
    <aside class="trades-detail">
      <template v-if="selectedSensor">
        <div class="detail-panel">
          <div class="detail-panel__header"><span class="detail-panel__label">Détail capteur</span><button class="detail-panel__close" @click="selectedSensor = null">✕</button></div>
          <div class="detail-panel__body">
            <div class="detail-status-banner" :class="'detail-status-banner--' + selectedSensor.status"><span class="detail-status-banner__icon">{{ sensorTypeIcons[selectedSensor.type] || '●' }}</span><span class="detail-status-banner__badge">{{ statusLabel(selectedSensor.status) }}</span></div>
            <h3 class="detail-panel__name">{{ selectedSensor.name }}</h3>
            <div class="detail-panel__type">{{ typeLabels[selectedSensor.type] || selectedSensor.type }}</div>
            <div class="detail-value-box" :class="'detail-value-box--' + selectedSensor.status"><span class="detail-value-box__label">Valeur</span><span class="detail-value-box__value">{{ selectedSensor.value }}</span></div>
            <div class="detail-panel__fields">
              <div class="detail-field"><span class="detail-field__label">Étage</span><span class="detail-field__value">{{ getFloorLabel(selectedSensor.floor) }}</span></div>
              <div class="detail-field"><span class="detail-field__label">Zone</span><span class="detail-field__value">{{ getRoomLabel(selectedSensor.room) }}</span></div>
              <div class="detail-field"><span class="detail-field__label">ID</span><span class="detail-field__value">{{ selectedSensor.id }}</span></div>
              <div class="detail-field"><span class="detail-field__label">MAJ</span><span class="detail-field__value">{{ selectedSensor.lastUpdate }}</span></div>
            </div>
            <div v-if="selectedSensor.status !== 'ok'" class="detail-problem-box" :class="'detail-problem-box--' + selectedSensor.status">
              <div v-if="selectedSensor.reason" class="detail-problem-box__reason"><span class="detail-problem-box__reason-label">Diagnostic</span>{{ selectedSensor.reason }}</div>
              <div class="detail-problem-box__action"><span class="detail-problem-box__action-label">Action recommandée</span>{{ getAction(selectedSensor) }}</div>
              <button class="detail-panel__resolve-btn" @click="resolveSensor(selectedSensor.id)">✓ Résoudre / Acquitter</button>
            </div>
          </div>
        </div>
      </template>
      <div v-if="!selectedSensor" class="detail-empty"><div class="detail-empty__icon">◎</div>Cliquez sur un capteur<br>pour voir ses détails</div>

      <div class="trade-events">
        <div class="trade-events__header"><span class="trade-events__label">Journal {{ activeTrade?.label }}</span><span class="trade-events__count">{{ tradeEvents.length }}</span></div>
        <div class="trade-events__list">
          <div v-if="tradeEvents.length === 0" class="trade-events__empty">Aucun événement récent</div>
          <div v-for="ev in tradeEvents" :key="ev.id" class="trade-event-item" :class="'trade-event-item--' + ev.type">
            <div class="trade-event-item__header"><span class="trade-event-item__badge" :class="'trade-event-item__badge--' + ev.type">{{ eventTypeLabel(ev.type) }}</span><span class="trade-event-item__time">{{ ev.time }}</span></div>
            <div class="trade-event-item__sensor">{{ ev.sensorName }}</div>
            <div class="trade-event-item__reason">{{ ev.reason }}</div>
          </div>
        </div>
      </div>
    </aside>
  </div>
</template>

<style scoped>
/* ═══════════════════════════════════════════════════════
   TRADES VIEW v2 — FULL CSS
   ═══════════════════════════════════════════════════════ */
.trades-view{display:flex;flex:1;overflow:hidden;gap:1px;background:var(--bg-primary)}
.trades-nav{width:230px;flex-shrink:0;background:var(--bg-panel);border-right:1px solid var(--border-primary);display:flex;flex-direction:column;overflow-y:auto}
.trades-nav__header{padding:var(--gap-lg);border-bottom:1px solid var(--border-primary)}
.trades-nav__label{font-size:11px;text-transform:uppercase;letter-spacing:1.5px;color:var(--text-muted);font-family:var(--font-mono)}
.trades-nav__list{padding:var(--gap-sm);display:flex;flex-direction:column;gap:4px}
.trade-btn{display:flex;flex-direction:column;gap:6px;padding:var(--gap-md) var(--gap-lg);border:1px solid transparent;border-radius:var(--radius-md);background:transparent;color:var(--text-secondary);cursor:pointer;transition:all .2s ease;text-align:left;font-family:var(--font-sans);font-size:13px}
.trade-btn:hover{background:var(--bg-hover);color:var(--text-primary);border-color:var(--border-accent)}
.trade-btn--active{background:var(--accent-blue-dim);color:var(--accent-blue);border-color:rgba(0,170,255,.25)}
.trade-btn--has-fault{border-left:3px solid var(--status-fault)}
.trade-btn--has-alert:not(.trade-btn--has-fault){border-left:3px solid var(--status-alert)}
.trade-btn__top{display:flex;align-items:center;justify-content:space-between}
.trade-btn__header{display:flex;align-items:center;gap:var(--gap-sm)}
.trade-btn__icon{font-size:16px;width:22px;text-align:center}
.trade-btn__label{font-weight:500}
.trade-btn__badges{display:flex;gap:3px}
.trade-btn__badge{font-size:10px;font-family:var(--font-mono);font-weight:700;padding:1px 6px;border-radius:8px;min-width:20px;text-align:center}
.trade-btn__badge--fault{background:var(--status-fault);color:#fff}
.trade-btn__badge--alert{background:var(--status-alert);color:#1a1a1a}
.trade-btn__health-bar{height:3px;background:var(--border-primary);border-radius:2px;overflow:hidden}
.trade-btn__health-fill{height:100%;border-radius:2px;transition:width .5s ease,background .3s ease}
.trade-btn__health-text{display:flex;justify-content:space-between;font-size:9px;color:var(--text-muted);font-family:var(--font-mono)}
.trades-nav__overview{margin-top:auto;padding:var(--gap-lg);border-top:1px solid var(--border-primary)}
.trades-nav__overview-title{font-size:10px;text-transform:uppercase;letter-spacing:1px;color:var(--text-muted);font-family:var(--font-mono);margin-bottom:var(--gap-sm)}
.trades-nav__overview-row{display:flex;align-items:center;gap:var(--gap-sm);padding:3px 0;font-size:11px;color:var(--text-secondary)}
.trades-nav__overview-icon{font-size:12px;width:18px;text-align:center}
.trades-nav__overview-label{flex:1}
.trades-nav__overview-dot{width:8px;height:8px;border-radius:50%}
.trades-nav__overview-dot--ok{background:var(--status-ok)}
.trades-nav__overview-dot--fault{background:var(--status-fault);box-shadow:0 0 6px var(--status-fault)}
.trades-nav__overview-dot--alert{background:var(--status-alert)}
.trades-main{flex:1;overflow-y:auto;padding:var(--gap-xl);display:flex;flex-direction:column;gap:var(--gap-lg)}
.trades-main__header{background:var(--bg-panel);border:1px solid var(--border-primary);border-radius:var(--radius-lg);padding:var(--gap-lg) var(--gap-xl);display:flex;align-items:center;justify-content:space-between;gap:var(--gap-lg);border-left:3px solid;flex-wrap:wrap}
.trades-main__title-row{display:flex;align-items:center;gap:var(--gap-lg)}
.trades-main__icon{width:48px;height:48px;border-radius:var(--radius-md);display:flex;align-items:center;justify-content:center;font-size:22px;flex-shrink:0}
.trades-main__title{font-size:18px;font-weight:600;color:var(--text-primary);margin:0}
.trades-main__subtitle{font-size:12px;color:var(--text-muted);margin:2px 0 0 0}
.trades-header-right{display:flex;align-items:center;gap:var(--gap-lg);flex-wrap:wrap}
.health-gauge{position:relative;width:72px;height:72px}
.health-gauge__svg{width:100%;height:100%}
.health-gauge__arc{transition:stroke-dashoffset .6s ease,stroke .3s ease}
.health-gauge__text{position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);display:flex;align-items:baseline;gap:1px}
.health-gauge__pct{font-size:20px;font-weight:700;font-family:var(--font-mono);line-height:1}
.health-gauge__unit{font-size:10px;color:var(--text-muted);font-family:var(--font-mono)}
.trades-stats{display:flex;gap:var(--gap-sm);flex-wrap:wrap}
.trades-stat{text-align:center;padding:var(--gap-sm) var(--gap-md);border-radius:var(--radius-md);min-width:48px}
.trades-stat--ok{background:var(--status-ok-bg)}
.trades-stat--fault{background:var(--status-fault-bg)}
.trades-stat--alert{background:var(--status-alert-bg)}
.trades-stat--offline{background:var(--status-offline-bg)}
.trades-stat__count{display:block;font-size:18px;font-weight:700;font-family:var(--font-mono);line-height:1}
.trades-stat--ok .trades-stat__count{color:var(--status-ok)}
.trades-stat--fault .trades-stat__count{color:var(--status-fault)}
.trades-stat--alert .trades-stat__count{color:var(--status-alert)}
.trades-stat--offline .trades-stat__count{color:var(--status-offline)}
.trades-stat__label{display:block;font-size:9px;color:var(--text-muted);font-family:var(--font-mono);text-transform:uppercase;margin-top:2px}
.subtrades-bar{display:flex;gap:6px;flex-wrap:wrap}
.subtrade-btn{display:flex;align-items:center;gap:4px;padding:6px 14px;border:1px solid var(--border-primary);border-radius:20px;background:var(--bg-panel);color:var(--text-secondary);font-size:12px;font-family:var(--font-sans);cursor:pointer;transition:all .15s ease}
.subtrade-btn:hover{background:var(--bg-hover);color:var(--text-primary)}
.subtrade-btn--active{background:var(--accent-blue-dim);color:var(--accent-blue);border-color:rgba(0,170,255,.3)}
.subtrade-btn__icon{font-size:13px}
.problems-section{background:var(--bg-panel);border:1px solid var(--border-primary);border-radius:var(--radius-lg);overflow:visible}
.problems-section__header{padding:var(--gap-lg);border-bottom:1px solid var(--border-primary);display:flex;align-items:flex-start;justify-content:space-between;gap:var(--gap-md);background:rgba(255,61,113,.04);flex-wrap:wrap}
.problems-section__title-row{display:flex;gap:var(--gap-md)}
.problems-section__icon{font-size:20px;color:var(--status-fault);margin-top:2px}
.problems-section__title{font-size:15px;font-weight:600;color:var(--text-primary)}
.problems-section__impact{font-size:11px;color:var(--text-muted);font-family:var(--font-mono);margin-top:2px}
.problems-section__view-modes{display:flex;gap:2px;background:var(--bg-primary);border-radius:var(--radius-md);padding:2px;flex-shrink:0}
.view-mode-btn{padding:4px 10px;border:none;border-radius:var(--radius-sm);background:transparent;color:var(--text-muted);font-size:11px;font-family:var(--font-sans);cursor:pointer;transition:all .15s ease;white-space:nowrap}
.view-mode-btn:hover{color:var(--text-primary)}
.view-mode-btn--active{background:var(--bg-panel);color:var(--accent-blue);box-shadow:var(--shadow-card)}
.problems-list{padding:var(--gap-lg)}
.severity-group{margin-bottom:var(--gap-md)}
.severity-group__header{display:flex;align-items:center;gap:var(--gap-sm);padding:var(--gap-sm)}
.severity-group__badge{font-size:10px;font-family:var(--font-mono);font-weight:700;padding:2px 8px;border-radius:4px;text-transform:uppercase;letter-spacing:.5px}
.severity-group__badge--fault{background:var(--status-fault);color:#fff}
.severity-group__badge--alert{background:var(--status-alert);color:#1a1a1a}
.severity-group__badge--offline{background:var(--status-offline);color:#fff}
.severity-group__count{font-size:11px;color:var(--text-muted);font-family:var(--font-mono)}
.severity-group__items{display:flex;flex-direction:column;gap:8px}
.zone-group{margin-bottom:var(--gap-lg)}
.zone-group__header{display:flex;align-items:center;gap:var(--gap-sm);padding:var(--gap-sm);border-bottom:1px solid var(--border-primary);margin-bottom:var(--gap-sm)}
.zone-group__floor{font-size:12px;font-weight:600;color:var(--accent-blue);font-family:var(--font-mono)}
.zone-group__room{font-size:12px;color:var(--text-secondary)}
.zone-group__count{margin-left:auto;font-size:10px;color:var(--text-muted);font-family:var(--font-mono)}
.zone-group__items{display:flex;flex-direction:column;gap:8px}
.type-group{margin-bottom:var(--gap-lg)}
.type-group__header{display:flex;align-items:center;gap:var(--gap-sm);padding:var(--gap-sm);border-bottom:1px solid var(--border-primary);margin-bottom:var(--gap-sm)}
.type-group__icon{font-size:16px}
.type-group__label{font-size:12px;font-weight:600;color:var(--text-primary)}
.type-group__count{margin-left:auto;font-size:10px;color:var(--text-muted);font-family:var(--font-mono);background:var(--bg-card);padding:1px 6px;border-radius:8px}
.type-group__items{display:flex;flex-direction:column;gap:8px}
/* ═══ PROBLEM CARD (pcard) — stacked vertical layout ═══ */
.pcard{display:flex;flex-direction:column;gap:6px;padding:var(--gap-md) var(--gap-lg);border-radius:var(--radius-md);cursor:pointer;transition:all .15s ease;border:1px solid transparent;background:var(--bg-card)}
.pcard:hover{border-color:var(--border-accent);background:var(--bg-hover)}
.pcard--selected{border-color:rgba(0,170,255,.4);background:var(--accent-blue-dim)}
.pcard--fault{border-left:3px solid var(--status-fault)}
.pcard--alert{border-left:3px solid var(--status-alert)}
.pcard--offline{border-left:3px solid var(--status-offline);opacity:.7}
.pcard__row1{display:flex;align-items:center;gap:var(--gap-sm)}
.pcard__icon{font-size:16px;flex-shrink:0}
.pcard__name{flex:1;font-size:13px;font-weight:600;color:var(--text-primary);white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
.pcard__value{font-size:14px;font-weight:700;font-family:var(--font-mono);flex-shrink:0}
.pcard__value--fault{color:var(--status-fault)}
.pcard__value--alert{color:var(--status-alert)}
.pcard__value--offline{color:var(--status-offline)}
.pcard__badge-mini{font-size:9px;font-family:var(--font-mono);font-weight:700;padding:2px 7px;border-radius:3px;text-transform:uppercase;flex-shrink:0}
.pcard__badge-mini--fault{background:var(--status-fault);color:#fff}
.pcard__badge-mini--alert{background:var(--status-alert);color:#1a1a1a}
.pcard__badge-mini--offline{background:var(--status-offline);color:#fff}
.pcard__row2{display:flex;align-items:center;justify-content:space-between;gap:var(--gap-sm)}
.pcard__location{font-size:11px;color:var(--text-muted);font-family:var(--font-mono)}
.pcard__time{font-size:10px;color:var(--text-muted);font-family:var(--font-mono);flex-shrink:0}
.pcard__reason{font-size:12px;padding:8px 12px;border-radius:var(--radius-sm);line-height:1.4}
.pcard__reason--fault{color:var(--status-fault);background:var(--status-fault-bg);border:1px solid var(--status-fault-border)}
.pcard__reason--alert{color:var(--status-alert);background:var(--status-alert-bg);border:1px solid var(--status-alert-border)}
.pcard__reason--offline{color:var(--status-offline);background:var(--status-offline-bg);border:1px solid var(--status-offline-border)}
.pcard__reason-label{font-weight:600;font-size:10px;text-transform:uppercase;font-family:var(--font-mono);letter-spacing:.3px}
.pcard__bottom{display:flex;align-items:center;justify-content:space-between;gap:var(--gap-md);margin-top:2px}
.pcard__action{font-size:11px;color:var(--accent-blue);font-family:var(--font-mono);flex:1;min-width:0}
.pcard__resolve{padding:5px 14px;border:1px solid var(--status-ok-border);border-radius:var(--radius-sm);background:var(--status-ok-bg);color:var(--status-ok);font-size:11px;font-weight:600;font-family:var(--font-sans);cursor:pointer;transition:all .15s ease;white-space:nowrap;flex-shrink:0}
.pcard__resolve:hover{background:var(--status-ok);color:var(--bg-primary)}
.no-problems{text-align:center;padding:40px;background:var(--bg-panel);border:1px solid var(--status-ok-border);border-radius:var(--radius-lg)}
.no-problems__icon{font-size:36px;color:var(--status-ok);margin-bottom:var(--gap-sm)}
.no-problems__title{font-size:16px;font-weight:600;color:var(--status-ok)}
.no-problems__subtitle{font-size:12px;color:var(--text-muted);margin-top:4px;font-family:var(--font-mono)}
.floor-groups{display:flex;flex-direction:column;gap:var(--gap-md)}
.floor-groups__title{font-size:11px;text-transform:uppercase;letter-spacing:1.5px;color:var(--text-muted);font-family:var(--font-mono)}
.floor-group{background:var(--bg-panel);border:1px solid var(--border-primary);border-radius:var(--radius-lg);overflow:hidden}
.floor-group--has-fault{border-left:3px solid var(--status-fault)}
.floor-group--has-alert:not(.floor-group--has-fault){border-left:3px solid var(--status-alert)}
.floor-group__header{display:flex;align-items:center;justify-content:space-between;padding:var(--gap-md) var(--gap-lg);cursor:pointer;transition:background .15s ease;user-select:none}
.floor-group__header:hover{background:var(--bg-hover)}
.floor-group__left{display:flex;align-items:center;gap:var(--gap-sm)}
.floor-group__chevron{font-size:12px;color:var(--text-muted);transition:transform .2s ease;display:inline-block}
.floor-group__chevron--open{transform:rotate(90deg)}
.floor-group__label{font-size:13px;font-weight:600;color:var(--accent-blue);font-family:var(--font-mono)}
.floor-group__name{font-size:12px;color:var(--text-secondary)}
.floor-group__right{display:flex;align-items:center;gap:var(--gap-sm)}
.floor-group__mini-bar{width:48px;height:4px;background:var(--border-primary);border-radius:2px;overflow:hidden}
.floor-group__mini-fill{height:100%;background:var(--status-ok);border-radius:2px;transition:width .4s ease}
.floor-group__count{font-size:10px;color:var(--text-muted);font-family:var(--font-mono)}
.floor-group__badge{font-size:9px;font-family:var(--font-mono);font-weight:700;padding:1px 6px;border-radius:8px}
.floor-group__badge--fault{background:var(--status-fault);color:#fff}
.floor-group__badge--alert{background:var(--status-alert);color:#1a1a1a}
.floor-group__body{padding:0 var(--gap-lg) var(--gap-lg);border-top:1px solid var(--border-primary)}
.sensor-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(195px,1fr));gap:var(--gap-md);padding-top:var(--gap-md)}
.sensor-card{background:var(--bg-card);border:1px solid var(--border-primary);border-radius:var(--radius-md);padding:var(--gap-md);cursor:pointer;transition:all .2s ease}
.sensor-card:hover{background:var(--bg-hover);border-color:var(--border-accent);transform:translateY(-1px);box-shadow:var(--shadow-card)}
.sensor-card--selected{border-color:rgba(0,170,255,.4);background:var(--accent-blue-dim)}
.sensor-card--fault{border-color:var(--status-fault-border);background:var(--status-fault-bg)}
.sensor-card--alert{border-color:var(--status-alert-border);background:var(--status-alert-bg)}
.sensor-card--offline{opacity:.6}
.sensor-card__header{display:flex;align-items:center;justify-content:space-between;margin-bottom:var(--gap-sm)}
.sensor-card__icon{font-size:18px}
.sensor-card__status-dot{width:8px;height:8px;border-radius:50%}
.sensor-card__status-dot--ok{background:var(--status-ok)}
.sensor-card__status-dot--fault{background:var(--status-fault);box-shadow:0 0 6px var(--status-fault);animation:pulse-dot 1.5s ease-in-out infinite}
.sensor-card__status-dot--alert{background:var(--status-alert)}
.sensor-card__status-dot--offline{background:var(--status-offline)}
@keyframes pulse-dot{0%,100%{opacity:1}50%{opacity:.3}}
.sensor-card__name{font-size:12px;font-weight:500;color:var(--text-primary);margin-bottom:4px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
.sensor-card__value{font-size:16px;font-weight:700;font-family:var(--font-mono);color:var(--text-bright);margin-bottom:4px}
.sensor-card__value--fault{color:var(--status-fault)}
.sensor-card__value--alert{color:var(--status-alert)}
.sensor-card__meta{display:flex;justify-content:space-between;font-size:10px;color:var(--text-muted);font-family:var(--font-mono)}
.sensor-card__reason{font-size:10px;color:var(--status-fault);margin-top:4px;padding-top:4px;border-top:1px solid var(--border-primary)}
.sensor-card--alert .sensor-card__reason{color:var(--status-alert)}
.trades-empty{text-align:center;color:var(--text-muted);padding:48px;font-size:13px}
.trades-detail{width:290px;flex-shrink:0;background:var(--bg-panel);border-left:1px solid var(--border-primary);display:flex;flex-direction:column;overflow-y:auto}
.detail-panel{border-bottom:1px solid var(--border-primary)}
.detail-panel__header{display:flex;align-items:center;justify-content:space-between;padding:var(--gap-lg);border-bottom:1px solid var(--border-primary)}
.detail-panel__label{font-size:11px;text-transform:uppercase;letter-spacing:1.5px;color:var(--text-muted);font-family:var(--font-mono)}
.detail-panel__close{width:24px;height:24px;border:1px solid var(--border-primary);border-radius:var(--radius-sm);background:transparent;color:var(--text-muted);cursor:pointer;font-size:12px;display:flex;align-items:center;justify-content:center;transition:all .15s ease}
.detail-panel__close:hover{background:var(--bg-hover);color:var(--text-primary)}
.detail-panel__body{padding:var(--gap-lg)}
.detail-status-banner{display:flex;align-items:center;gap:var(--gap-sm);padding:var(--gap-sm) var(--gap-md);border-radius:var(--radius-md);margin-bottom:var(--gap-md)}
.detail-status-banner--ok{background:var(--status-ok-bg);border:1px solid var(--status-ok-border)}
.detail-status-banner--fault{background:var(--status-fault-bg);border:1px solid var(--status-fault-border)}
.detail-status-banner--alert{background:var(--status-alert-bg);border:1px solid var(--status-alert-border)}
.detail-status-banner--offline{background:var(--status-offline-bg);border:1px solid var(--status-offline-border)}
.detail-status-banner__icon{font-size:18px}
.detail-status-banner__badge{font-size:11px;font-family:var(--font-mono);font-weight:700;text-transform:uppercase;letter-spacing:.5px}
.detail-status-banner--ok .detail-status-banner__badge{color:var(--status-ok)}
.detail-status-banner--fault .detail-status-banner__badge{color:var(--status-fault)}
.detail-status-banner--alert .detail-status-banner__badge{color:var(--status-alert)}
.detail-status-banner--offline .detail-status-banner__badge{color:var(--status-offline)}
.detail-panel__name{font-size:15px;font-weight:600;color:var(--text-primary);margin:0 0 2px 0}
.detail-panel__type{font-size:11px;color:var(--text-muted);font-family:var(--font-mono);margin-bottom:var(--gap-md)}
.detail-value-box{text-align:center;padding:var(--gap-md);border-radius:var(--radius-md);margin-bottom:var(--gap-md)}
.detail-value-box--ok{background:var(--status-ok-bg);border:1px solid var(--status-ok-border)}
.detail-value-box--fault{background:var(--status-fault-bg);border:1px solid var(--status-fault-border)}
.detail-value-box--alert{background:var(--status-alert-bg);border:1px solid var(--status-alert-border)}
.detail-value-box--offline{background:var(--status-offline-bg);border:1px solid var(--status-offline-border)}
.detail-value-box__label{display:block;font-size:9px;text-transform:uppercase;color:var(--text-muted);font-family:var(--font-mono);margin-bottom:2px}
.detail-value-box__value{font-size:22px;font-weight:700;font-family:var(--font-mono)}
.detail-value-box--ok .detail-value-box__value{color:var(--status-ok)}
.detail-value-box--fault .detail-value-box__value{color:var(--status-fault)}
.detail-value-box--alert .detail-value-box__value{color:var(--status-alert)}
.detail-value-box--offline .detail-value-box__value{color:var(--status-offline)}
.detail-panel__fields{display:flex;flex-direction:column;gap:4px;margin-bottom:var(--gap-md)}
.detail-field{display:flex;justify-content:space-between;align-items:center;padding:3px 0;border-bottom:1px solid rgba(30,42,58,.3)}
.detail-field__label{font-size:10px;color:var(--text-muted);font-family:var(--font-mono);text-transform:uppercase}
.detail-field__value{font-size:12px;color:var(--text-primary);font-family:var(--font-mono)}
.detail-problem-box{padding:var(--gap-md);border-radius:var(--radius-md)}
.detail-problem-box--fault{background:var(--status-fault-bg);border:1px solid var(--status-fault-border)}
.detail-problem-box--alert{background:var(--status-alert-bg);border:1px solid var(--status-alert-border)}
.detail-problem-box--offline{background:var(--status-offline-bg);border:1px solid var(--status-offline-border)}
.detail-problem-box__reason{font-size:12px;color:var(--text-primary);margin-bottom:var(--gap-md)}
.detail-problem-box__reason-label{display:block;font-size:9px;text-transform:uppercase;font-family:var(--font-mono);color:var(--text-muted);margin-bottom:2px}
.detail-problem-box__action{font-size:12px;color:var(--accent-blue);margin-bottom:var(--gap-md)}
.detail-problem-box__action-label{display:block;font-size:9px;text-transform:uppercase;font-family:var(--font-mono);color:var(--text-muted);margin-bottom:2px}
.detail-panel__resolve-btn{width:100%;padding:var(--gap-sm) var(--gap-md);border:1px solid var(--status-ok-border);border-radius:var(--radius-md);background:var(--status-ok-bg);color:var(--status-ok);font-size:12px;font-weight:600;cursor:pointer;transition:all .15s ease}
.detail-panel__resolve-btn:hover{background:var(--status-ok);color:var(--bg-primary)}
.detail-empty{display:flex;flex-direction:column;align-items:center;justify-content:center;padding:48px var(--gap-lg);color:var(--text-muted);font-size:12px;text-align:center;gap:var(--gap-sm)}
.detail-empty__icon{font-size:28px;opacity:.4}
.trade-events{flex:1;display:flex;flex-direction:column;border-top:1px solid var(--border-primary)}
.trade-events__header{display:flex;align-items:center;justify-content:space-between;padding:var(--gap-lg);border-bottom:1px solid var(--border-primary)}
.trade-events__label{font-size:11px;text-transform:uppercase;letter-spacing:1px;color:var(--text-muted);font-family:var(--font-mono)}
.trade-events__count{font-size:10px;font-family:var(--font-mono);padding:1px 6px;border-radius:8px;background:var(--bg-card);color:var(--text-muted)}
.trade-events__list{padding:var(--gap-sm);overflow-y:auto;flex:1}
.trade-events__empty{text-align:center;color:var(--text-muted);padding:32px;font-size:12px}
.trade-event-item{padding:var(--gap-sm) var(--gap-md);border-radius:var(--radius-md);margin-bottom:4px;border-left:2px solid transparent}
.trade-event-item--fault{border-left-color:var(--status-fault)}
.trade-event-item--alert{border-left-color:var(--status-alert)}
.trade-event-item--resolved{border-left-color:var(--status-ok)}
.trade-event-item__header{display:flex;align-items:center;justify-content:space-between;margin-bottom:2px}
.trade-event-item__badge{font-size:9px;font-family:var(--font-mono);font-weight:600;padding:1px 5px;border-radius:3px;text-transform:uppercase}
.trade-event-item__badge--fault{background:var(--status-fault-bg);color:var(--status-fault)}
.trade-event-item__badge--alert{background:var(--status-alert-bg);color:var(--status-alert)}
.trade-event-item__badge--resolved{background:var(--status-ok-bg);color:var(--status-ok)}
.trade-event-item__time{font-size:10px;color:var(--text-muted);font-family:var(--font-mono)}
.trade-event-item__sensor{font-size:12px;color:var(--text-primary);font-weight:500}
.trade-event-item__reason{font-size:10px;color:var(--text-muted)}
</style>