<script setup>
import { ref, reactive, computed, onMounted, onUnmounted, watch } from 'vue'
import sensorsData from '../data/sensors.json'
import FloorSelector from './FloorSelector.vue'
import FloorPlan from './FloorPlan.vue'
import StatusSidebar from './StatusSidebar.vue'
import TradesView from './TradesView.vue'

const viewMode = ref('plans') // plans | trades
const selectedTrade = ref('cvc')
const selectedSubTrade = ref(null)

const building = sensorsData.building
const rooms = sensorsData.rooms

// ── Reactive sensors with numValue for simulation ──
const allSensors = reactive(sensorsData.sensors.map(s => ({
  ...s,
  reason: null,
  numValue: s.numValue ?? 0
})))

const selectedFloor = ref(0)
const selectedSensor = ref(null)
const leftPanelOpen = ref(true)
const rightPanelOpen = ref(true)
const eventLog = reactive([])

// ═══════════════════════════════════════════════════════
// ROOM COMMAND STATE — one setpoint per room
// ═══════════════════════════════════════════════════════
const roomCommands = reactive({})
rooms.forEach(room => {
  // Find initial values from sensors in this room
  const roomSensors = sensorsData.sensors.filter(s => s.room === room.id)
  const tempSensor = roomSensors.find(s => s.type === 'temperature')
  const lightSensor = roomSensors.find(s => s.type === 'light')
  const hvacSensor = roomSensors.find(s => s.type === 'hvac')

  roomCommands[room.id] = {
    tempSetpoint: tempSensor ? Math.round(tempSensor.numValue) : 22,
    hvacMode: 'auto',      // auto | heat | cool | off
    lightLevel: lightSensor ? lightSensor.numValue : 80,
    lightMode: 'auto',     // auto | manual | eco | off
    ventSpeed: 'medium',   // low | medium | high | auto
    doorLocked: true
  }
})

// ═══════════════════════════════════════════════════════
// SIMULATION ENGINE — runs every 5s, drifts values toward setpoints
// ═══════════════════════════════════════════════════════
const SIMULATION_TICK = 5000

function getNow() {
  const d = new Date()
  return d.toLocaleDateString('fr-FR') + ' ' + d.toLocaleTimeString('fr-FR', { hour: '2-digit', minute: '2-digit' })
}

function simulateTick() {
  const now = getNow()

  allSensors.forEach(sensor => {
    if (sensor.status !== 'ok') return // Don't simulate broken sensors
    const cmd = roomCommands[sensor.room]
    if (!cmd) return

    // ── Temperature simulation ──
    if (sensor.type === 'temperature') {
      const target = cmd.tempSetpoint
      const current = sensor.numValue
      let drift = 0

      if (cmd.hvacMode === 'off') {
        // Drift toward 18°C (ambient without HVAC)
        drift = current > 18 ? -0.1 : (current < 18 ? 0.05 : 0)
      } else {
        const diff = target - current
        if (Math.abs(diff) > 0.1) {
          // Speed depends on mode
          const speed = cmd.hvacMode === 'auto' ? 0.2 : 0.3
          drift = Math.sign(diff) * Math.min(speed, Math.abs(diff))
          // Add small random noise
          drift += (Math.random() - 0.5) * 0.05
        } else {
          // Near target, small oscillation
          drift = (Math.random() - 0.5) * 0.1
        }
      }

      sensor.numValue = Math.round((current + drift) * 10) / 10
      sensor.value = sensor.numValue.toFixed(1) + '°C'
      sensor.lastUpdate = now
    }

    // ── Humidity simulation ──
    if (sensor.type === 'humidity') {
      // Humidity naturally drifts 40-55%, HVAC mode affects it
      const target = cmd.hvacMode === 'cool' ? 40 : (cmd.hvacMode === 'heat' ? 30 : 45)
      const diff = target - sensor.numValue
      const drift = Math.sign(diff) * Math.min(0.5, Math.abs(diff)) + (Math.random() - 0.5) * 0.3
      sensor.numValue = Math.round(Math.max(15, Math.min(90, sensor.numValue + drift)))
      sensor.value = sensor.numValue + '%'
      sensor.lastUpdate = now
    }

    // ── CO2 simulation ──
    if (sensor.type === 'co2') {
      let target = 450 // good air
      if (cmd.ventSpeed === 'low') target = 900
      else if (cmd.ventSpeed === 'medium') target = 650
      else if (cmd.ventSpeed === 'high') target = 400
      else if (cmd.ventSpeed === 'auto') target = 550

      // Presence increases CO2
      const presenceSensors = allSensors.filter(s => s.room === sensor.room && s.type === 'presence' && s.numValue > 0)
      if (presenceSensors.length > 0) target += 150

      const diff = target - sensor.numValue
      const drift = Math.sign(diff) * Math.min(15, Math.abs(diff)) + (Math.random() - 0.5) * 5
      sensor.numValue = Math.round(Math.max(350, Math.min(2000, sensor.numValue + drift)))
      sensor.value = sensor.numValue + ' ppm'
      sensor.lastUpdate = now

      // Auto-alert if CO2 too high
      if (sensor.numValue > 1000 && sensor.status === 'ok') {
        sensor.status = 'alert'
        sensor.reason = 'CO₂ au-dessus de 1000 ppm — ventilation insuffisante'
      } else if (sensor.numValue <= 900 && sensor.status === 'alert' && sensor.reason?.includes('CO₂')) {
        sensor.status = 'ok'
        sensor.reason = null
      }
    }

    // ── HVAC display ──
    if (sensor.type === 'hvac') {
      const modeLabels = { auto: 'Auto', heat: 'Chauffage', cool: 'Climatisation', off: 'Arrêt' }
      if (cmd.hvacMode === 'off') {
        sensor.value = 'Arrêt'
      } else {
        sensor.value = modeLabels[cmd.hvacMode] + ' — ' + cmd.tempSetpoint + '°C'
      }
      sensor.numValue = cmd.tempSetpoint
      sensor.lastUpdate = now
    }

    // ── Light simulation ──
    if (sensor.type === 'light') {
      let targetLight = cmd.lightLevel
      if (cmd.lightMode === 'off') targetLight = 0
      else if (cmd.lightMode === 'eco') targetLight = Math.min(cmd.lightLevel, 40)

      // Instant change for lights
      sensor.numValue = targetLight
      sensor.value = targetLight + '%'
      sensor.lastUpdate = now
    }

    // ── Door simulation ──
    if (sensor.type === 'door') {
      if (cmd.doorLocked) {
        sensor.value = 'Verrouillée'
        sensor.numValue = 0
      } else {
        sensor.value = 'Déverrouillée'
        sensor.numValue = 1
      }
      sensor.lastUpdate = now
    }

    // ── Energy simulation ──
    if (sensor.type === 'energy') {
      // Energy increases based on HVAC + light activity
      let consumption = 0.01
      if (cmd.hvacMode !== 'off') consumption += 0.03
      if (cmd.lightLevel > 50) consumption += 0.01
      sensor.numValue = Math.round((sensor.numValue + consumption) * 100) / 100
      sensor.value = sensor.numValue.toFixed(1) + ' kWh'
      sensor.lastUpdate = now
    }
  })

  // Update selected sensor reference if it changed
  if (selectedSensor.value) {
    const updated = allSensors.find(s => s.id === selectedSensor.value.id)
    if (updated) selectedSensor.value = { ...updated }
  }
}

// ═══════════════════════════════════════════════════════
// FAULT / ALERT RANDOM ENGINE
// ═══════════════════════════════════════════════════════
const faultReasons = {
  temperature: [{ value: 'ERR', reason: 'Perte de signal — câblage défectueux' }, { value: 'ERR', reason: 'Sonde HS — remplacement nécessaire' }],
  humidity: [{ value: 'ERR', reason: 'Sonde HS — remplacement nécessaire' }],
  presence: [{ value: 'ERR', reason: 'Défaut capteur PIR — zone aveugle' }],
  door: [{ value: 'Bloquée', reason: 'Gâche électrique bloquée — intervention requise' }],
  smoke: [{ value: 'ALARME', reason: 'Détection fumée — vérification requise' }, { value: 'ERR', reason: 'Auto-test échoué — capteur encrassé' }],
  hvac: [{ value: 'Arrêt forcé', reason: 'Défaut compresseur — surcharge moteur' }, { value: 'ERR', reason: 'Perte communication bus CVC' }],
  light: [{ value: '0%', reason: 'Ballast défectueux — circuit ouvert' }, { value: 'ERR', reason: 'Défaut driver DALI' }],
  energy: [{ value: 'ERR', reason: 'Communication Modbus interrompue' }],
  co2: [{ value: 'ERR', reason: 'Sonde CO₂ hors service — étalonnage requis' }],
  camera: [{ value: 'Hors ligne', reason: 'Perte flux vidéo — vérifier alimentation PoE' }]
}
const alertReasons = {
  temperature: [{ reason: 'Température élevée — seuil haut approché' }, { reason: 'Température basse — chauffage insuffisant' }],
  humidity: [{ reason: 'Humidité élevée — ventilation recommandée' }],
  door: [{ reason: 'Porte ouverte prolongée — vérifier ferme-porte' }],
  hvac: [{ reason: 'Filtre encrassé — remplacement à planifier' }, { reason: 'Performance dégradée — maintenance préventive' }],
  light: [{ reason: 'Luminosité réduite — fin de vie lampe' }],
  energy: [{ reason: 'Consommation anormalement élevée' }],
  camera: [{ reason: 'Qualité image dégradée — lentille sale' }],
  smoke: [{ reason: 'Niveau de particules en hausse' }],
  presence: [{ reason: 'Signal instable — nettoyage lentille requis' }],
  co2: [{ reason: 'CO₂ élevé — qualité air médiocre' }]
}

function pickRandom(a) { return a[Math.floor(Math.random() * a.length)] }

function checkAndGenerateEvents() {
  const now = getNow()
  const faultCount = allSensors.filter(s => s.status === 'fault').length
  const alertCount = allSensors.filter(s => s.status === 'alert').length

  if (faultCount === 0) {
    const ok = allSensors.filter(s => s.status === 'ok')
    if (ok.length) {
      const s = pickRandom(ok)
      const pool = faultReasons[s.type] || [{ value: 'ERR', reason: 'Défaut inconnu' }]
      const p = pickRandom(pool)
      s.status = 'fault'; s.value = p.value; s.reason = p.reason; s.lastUpdate = now
      eventLog.unshift({ id: Date.now(), type: 'fault', sensorId: s.id, sensorName: s.name, reason: p.reason, time: now })
    }
  }

  if (alertCount < 3) {
    const n = Math.min(Math.floor(Math.random() * 2) + 1, 3 - alertCount)
    for (let i = 0; i < n; i++) {
      const ok = allSensors.filter(s => s.status === 'ok')
      if (!ok.length) break
      const s = pickRandom(ok)
      const pool = alertReasons[s.type] || [{ reason: 'Anomalie détectée' }]
      const p = pickRandom(pool)
      const alertVal = s.type === 'temperature' ? ((s.numValue + (Math.random() > 0.5 ? 6 : -5)).toFixed(1) + '°C') : s.value
      s.status = 'alert'; s.value = alertVal; s.reason = p.reason; s.lastUpdate = now
      eventLog.unshift({ id: Date.now() + i + 1, type: 'alert', sensorId: s.id, sensorName: s.name, reason: p.reason, time: now })
    }
  }
}

// ── Resolve ──
const originalValues = {}
sensorsData.sensors.forEach(s => { originalValues[s.id] = { value: s.value, numValue: s.numValue, status: s.status } })

function resolveSensor(sensorId) {
  const sensor = allSensors.find(s => s.id === sensorId)
  if (!sensor) return
  const orig = originalValues[sensorId]
  sensor.status = 'ok'
  sensor.value = orig?.value || '—'
  sensor.numValue = orig?.numValue ?? 0
  sensor.reason = null
  sensor.lastUpdate = getNow()
  eventLog.unshift({ id: Date.now(), type: 'resolved', sensorId: sensor.id, sensorName: sensor.name, reason: 'Résolu par opérateur', time: getNow() })
  if (selectedSensor.value?.id === sensorId) selectedSensor.value = { ...sensor }
}

// ── Room command handler ──
function updateRoomCommand(roomId, key, value) {
  if (roomCommands[roomId]) {
    roomCommands[roomId][key] = value
  }
}

// ── Computed ──
const currentFloorInfo = computed(() => building.floors.find(f => f.id === selectedFloor.value))
const floorSensors = computed(() => allSensors.filter(s => s.floor === selectedFloor.value))
const floorRooms = computed(() => rooms.filter(r => r.floor === selectedFloor.value))

function selectFloor(floorId) { selectedFloor.value = floorId; selectedSensor.value = null }
function selectSensor(sensor) { selectedSensor.value = sensor; if (!rightPanelOpen.value) rightPanelOpen.value = true }

// ── Clock ──
const currentTime = ref('')
const currentDate = computed(() => new Date().toLocaleDateString('fr-FR', { weekday: 'long', day: 'numeric', month: 'long', year: 'numeric' }))

let clockTimer, simTimer, eventTimer
onMounted(() => {
  currentTime.value = new Date().toLocaleTimeString('fr-FR', { hour: '2-digit', minute: '2-digit', second: '2-digit' })
  clockTimer = setInterval(() => {
    currentTime.value = new Date().toLocaleTimeString('fr-FR', { hour: '2-digit', minute: '2-digit', second: '2-digit' })
  }, 1000)
  simTimer = setInterval(simulateTick, SIMULATION_TICK)
  setTimeout(checkAndGenerateEvents, 5000)
  eventTimer = setInterval(checkAndGenerateEvents, 30000)
})
onUnmounted(() => { clearInterval(clockTimer); clearInterval(simTimer); clearInterval(eventTimer) })
</script>

<template>
  <div class="topbar">
    <div class="topbar__brand">
      <div class="topbar__logo">GT</div>
      <span class="topbar__title">{{ building.name }}</span>
      <span class="topbar__subtitle">SUPERVISION GTB</span>
    </div>

    <div class="topbar__center">
      <button
        class="topbar__tab"
        :class="{ 'topbar__tab--active': viewMode === 'plans' }"
        @click="viewMode = 'plans'; selectedSensor = null"
      >
        ▦ Vue Plans
      </button>
      <button
        class="topbar__tab"
        :class="{ 'topbar__tab--active': viewMode === 'trades' }"
        @click="viewMode = 'trades'; selectedSensor = null"
      >
        ◉ Vue Métiers
      </button>
    </div>

    <div class="topbar__meta">
      <span class="topbar__clock">{{ currentDate }} — {{ currentTime }}</span>
      <div class="topbar__status-dot" title="Système en ligne"></div>
    </div>
  </div>

<div class="dashboard">
    <!-- ═══ MODE: VUE PLANS ═══ -->
    <template v-if="viewMode === 'plans'">
      <div class="panel-collapsible panel-collapsible--left" :class="{ 'panel-collapsible--closed': !leftPanelOpen }">
        <FloorSelector :floors="building.floors" :sensors="allSensors" :selectedFloor="selectedFloor" @select-floor="selectFloor" />
      </div>
      <button class="panel-toggle panel-toggle--left" :class="{ 'panel-toggle--collapsed': !leftPanelOpen }" @click="leftPanelOpen = !leftPanelOpen">
        <span class="panel-toggle__icon">{{ leftPanelOpen ? '◂' : '▸' }}</span>
      </button>
 
      <FloorPlan :floorInfo="currentFloorInfo" :sensors="floorSensors" :selectedSensor="selectedSensor" @select-sensor="selectSensor" />
 
      <button class="panel-toggle panel-toggle--right" :class="{ 'panel-toggle--collapsed': !rightPanelOpen }" @click="rightPanelOpen = !rightPanelOpen">
        <span class="panel-toggle__icon">{{ rightPanelOpen ? '▸' : '◂' }}</span>
      </button>
      <div class="panel-collapsible panel-collapsible--right" :class="{ 'panel-collapsible--closed': !rightPanelOpen }">
        <StatusSidebar
          :sensors="floorSensors" :allSensors="allSensors" :selectedSensor="selectedSensor"
          :floorInfo="currentFloorInfo" :eventLog="eventLog"
          :floorRooms="floorRooms" :roomCommands="roomCommands"
          @select-sensor="selectSensor" @resolve-sensor="resolveSensor"
          @update-room-command="({ roomId, key, value }) => updateRoomCommand(roomId, key, value)"
        />
      </div>
    </template>
 
    <!-- ═══ MODE: VUE MÉTIERS ═══ -->
    <template v-if="viewMode === 'trades'">
      <TradesView
        :sensors="allSensors"
        :floors="building.floors"
        :rooms="rooms"
        :eventLog="eventLog"
        @select-sensor="selectSensor"
        @resolve-sensor="resolveSensor"
      />
    </template>
  </div>
</template>
