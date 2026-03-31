<script setup>
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue'
import sensorsData from '../data/sensors.json'
import FloorSelector from './FloorSelector.vue'
import FloorPlan from './FloorPlan.vue'
import StatusSidebar from './StatusSidebar.vue'
import TradesView from './TradesView.vue'
import CtaDetailView from './CtaDetailView.vue'

const viewMode = ref('plans') // plans | trades

const building = sensorsData.building
const rooms    = sensorsData.rooms

// ── Reactive sensors ──
const allSensors = reactive(sensorsData.sensors.map(s => ({
  ...s,
  reason:   null,
  numValue: s.numValue ?? 0
})))

const selectedFloor   = ref(0)
const selectedSensor  = ref(null)
const leftPanelOpen   = ref(true)
const rightPanelOpen  = ref(true)
const eventLog        = reactive([])

// ── CTA Detail View ──
const showCtaView = ref(false)
const ctaSensor   = ref(null)

// ═══════════════════════════════════════════════════════
// ROOM COMMAND STATE
// ═══════════════════════════════════════════════════════
const roomCommands = reactive({})
rooms.forEach(room => {
  const roomSensors = sensorsData.sensors.filter(s => s.room === room.id)
  const tempSensor  = roomSensors.find(s => s.type === 'temperature')
  const lightSensor = roomSensors.find(s => s.type === 'light')

  // Legere variance sur le setpoint pour rendre la simulation visible
  const baseTemp = tempSensor ? Math.round(tempSensor.numValue) : 22
  const variance = Math.random() > 0.5 ? 1 : -1

  roomCommands[room.id] = {
    tempSetpoint: baseTemp + variance,
    hvacMode:    'auto',
    lightLevel:  lightSensor ? lightSensor.numValue : 80,
    lightMode:   'auto',
    ventSpeed:   'medium',
    doorLocked:  true
  }
})

// ═══════════════════════════════════════════════════════
// SIMULATION ENGINE — toutes les 5s
// ═══════════════════════════════════════════════════════
const SIMULATION_TICK = 5000

function getNow() {
  const d = new Date()
  return d.toLocaleDateString('fr-FR') + ' ' + d.toLocaleTimeString('fr-FR', { hour: '2-digit', minute: '2-digit' })
}

function simulateTick() {
  const now = getNow()

  allSensors.forEach(sensor => {
    if (sensor.status !== 'ok') return
    const cmd = roomCommands[sensor.room]
    if (!cmd) return

    // ── Temperature ──
    if (sensor.type === 'temperature') {
      const target  = cmd.tempSetpoint
      const current = sensor.numValue
      let drift = 0
      if (cmd.hvacMode === 'off') {
        drift = current > 18 ? -0.1 : (current < 18 ? 0.05 : 0)
      } else {
        const diff  = target - current
        const speed = cmd.hvacMode === 'auto' ? 0.2 : 0.3
        if (Math.abs(diff) > 0.1) {
          drift = Math.sign(diff) * Math.min(speed, Math.abs(diff))
          drift += (Math.random() - 0.5) * 0.05
        } else {
          drift = (Math.random() - 0.5) * 0.1
        }
      }
      sensor.numValue   = Math.round((current + drift) * 10) / 10
      sensor.value      = sensor.numValue.toFixed(1) + 'C'
      sensor.lastUpdate = now
    }

    // ── Humidite ──
    if (sensor.type === 'humidity') {
      const target = cmd.hvacMode === 'cool' ? 40 : (cmd.hvacMode === 'heat' ? 30 : 45)
      const diff   = target - sensor.numValue
      const drift  = Math.sign(diff) * Math.min(0.5, Math.abs(diff)) + (Math.random() - 0.5) * 0.3
      sensor.numValue   = Math.round(Math.max(15, Math.min(90, sensor.numValue + drift)))
      sensor.value      = sensor.numValue + '%'
      sensor.lastUpdate = now
    }

    // ── CO2 ──
    if (sensor.type === 'co2') {
      let target = 450
      if (cmd.ventSpeed === 'low')         target = 900
      else if (cmd.ventSpeed === 'medium') target = 650
      else if (cmd.ventSpeed === 'high')   target = 400
      else if (cmd.ventSpeed === 'auto')   target = 550

      const presenceSensors = allSensors.filter(s => s.room === sensor.room && s.type === 'presence' && s.numValue > 0)
      if (presenceSensors.length > 0) target += 150

      const diff  = target - sensor.numValue
      const drift = Math.sign(diff) * Math.min(15, Math.abs(diff)) + (Math.random() - 0.5) * 5
      sensor.numValue   = Math.round(Math.max(350, Math.min(2000, sensor.numValue + drift)))
      sensor.value      = sensor.numValue + ' ppm'
      sensor.lastUpdate = now

      if (sensor.numValue > 1000 && sensor.status === 'ok') {
        sensor.status = 'alert'
        sensor.reason = 'CO2 au-dessus de 1000 ppm - ventilation insuffisante'
        eventLog.unshift({ id: Date.now(), type: 'alert', sensorId: sensor.id, sensorName: sensor.name, reason: sensor.reason, time: now })
      } else if (sensor.numValue <= 900 && sensor.status === 'alert' && sensor.reason?.includes('CO2')) {
        sensor.status = 'ok'
        sensor.reason = null
      }
    }

    // ── HVAC ──
    if (sensor.type === 'hvac') {
      const modeLabels  = { auto: 'Auto', heat: 'Chauffage', cool: 'Climatisation', off: 'Arret' }
      sensor.value      = cmd.hvacMode === 'off' ? 'Arret' : modeLabels[cmd.hvacMode] + ' - ' + cmd.tempSetpoint + 'C'
      sensor.numValue   = cmd.tempSetpoint
      sensor.lastUpdate = now
    }

    // ── Eclairage ──
    if (sensor.type === 'light') {
      let targetLight = cmd.lightLevel
      if (cmd.lightMode === 'off')       targetLight = 0
      else if (cmd.lightMode === 'eco')  targetLight = Math.min(cmd.lightLevel, 40)
      sensor.numValue   = targetLight
      sensor.value      = targetLight + '%'
      sensor.lastUpdate = now
    }

    // ── Porte ──
    if (sensor.type === 'door') {
      sensor.value      = cmd.doorLocked ? 'Verrouillee' : 'Deverrouillee'
      sensor.numValue   = cmd.doorLocked ? 0 : 1
      sensor.lastUpdate = now
    }

    // ── Energie ──
    if (sensor.type === 'energy') {
      let consumption = 0.01
      if (cmd.hvacMode !== 'off') consumption += 0.03
      if (cmd.lightLevel > 50)    consumption += 0.01
      sensor.numValue   = Math.round((sensor.numValue + consumption) * 100) / 100
      sensor.value      = sensor.numValue.toFixed(1) + ' kWh'
      sensor.lastUpdate = now
    }
  })

  // Rafraichir le capteur selectionne
  if (selectedSensor.value) {
    const updated = allSensors.find(s => s.id === selectedSensor.value.id)
    if (updated) selectedSensor.value = { ...updated }
  }
}

// ═══════════════════════════════════════════════════════
// MOTEUR DEFAUTS / ALERTES ALEATOIRES
// ═══════════════════════════════════════════════════════
const faultReasons = {
  temperature: [
    { value: 'ERR',        reason: 'Perte de signal - cablage defectueux' },
    { value: 'ERR',        reason: 'Sonde HS - remplacement necessaire' }
  ],
  humidity:    [{ value: 'ERR',          reason: 'Sonde HS - remplacement necessaire' }],
  presence:    [{ value: 'ERR',          reason: 'Defaut capteur PIR - zone aveugle' }],
  door:        [{ value: 'Bloquee',      reason: 'Gache electrique bloquee - intervention requise' }],
  smoke:       [
    { value: 'ALARME',     reason: 'Detection fumee - verification requise' },
    { value: 'ERR',        reason: 'Auto-test echoue - capteur encrosse' }
  ],
  hvac:        [
    { value: 'Arret force', reason: 'Defaut compresseur - surcharge moteur' },
    { value: 'ERR',         reason: 'Perte communication bus CVC' }
  ],
  light:       [
    { value: '0%',  reason: 'Ballast defectueux - circuit ouvert' },
    { value: 'ERR', reason: 'Defaut driver DALI' }
  ],
  energy:      [{ value: 'ERR',         reason: 'Communication Modbus interrompue' }],
  co2:         [{ value: 'ERR',         reason: 'Sonde CO2 hors service - etalonnage requis' }],
  camera:      [{ value: 'Hors ligne',  reason: 'Perte flux video - verifier alimentation PoE' }],
  meter:       [
    { value: 'Hors ligne', reason: 'Perte communication reseau - verifier cable Ethernet' },
    { value: 'ERR',        reason: 'Timeout Modbus - centrale mesure injoignable' }
  ]
}

const alertReasons = {
  temperature: [
    { reason: 'Temperature elevee - seuil haut approche' },
    { reason: 'Temperature basse - chauffage insuffisant' }
  ],
  humidity:    [{ reason: 'Humidite elevee - ventilation recommandee' }],
  door:        [{ reason: 'Porte ouverte prolongee - verifier ferme-porte' }],
  hvac:        [
    { reason: 'Filtre encrosse - remplacement a planifier' },
    { reason: 'Performance degradee - maintenance preventive' }
  ],
  light:       [{ reason: 'Luminosite reduite - fin de vie lampe' }],
  energy:      [{ reason: 'Consommation anormalement elevee' }],
  camera:      [{ reason: 'Qualite image degradee - lentille sale' }],
  smoke:       [{ reason: 'Niveau de particules en hausse' }],
  presence:    [{ reason: 'Signal instable - nettoyage lentille requis' }],
  co2:         [{ reason: 'CO2 eleve - qualite air mediocre' }],
  meter:       [{ reason: 'Latence elevee - congestion reseau detectee' }]
}

function pickRandom(a) { return a[Math.floor(Math.random() * a.length)] }

function checkAndGenerateEvents() {
  const now        = getNow()
  const faultCount = allSensors.filter(s => s.status === 'fault').length
  const alertCount = allSensors.filter(s => s.status === 'alert').length

  // Jusqu'a 2 defauts simultanes
  if (faultCount < 2) {
    const ok = allSensors.filter(s => s.status === 'ok')
    if (ok.length) {
      const s    = pickRandom(ok)
      const pool = faultReasons[s.type] || [{ value: 'ERR', reason: 'Defaut inconnu' }]
      const p    = pickRandom(pool)
      s.status    = 'fault'
      s.value     = p.value
      s.reason    = p.reason
      s.lastUpdate = now
      eventLog.unshift({ id: Date.now(), type: 'fault', sensorId: s.id, sensorName: s.name, reason: p.reason, time: now })
    }
  }

  // Jusqu'a 4 alertes simultanees
  if (alertCount < 4) {
    const n = Math.min(Math.floor(Math.random() * 2) + 1, 4 - alertCount)
    for (let i = 0; i < n; i++) {
      const ok = allSensors.filter(s => s.status === 'ok')
      if (!ok.length) break
      const s    = pickRandom(ok)
      const pool = alertReasons[s.type] || [{ reason: 'Anomalie detectee' }]
      const p    = pickRandom(pool)
      const alertVal = s.type === 'temperature'
        ? ((s.numValue + (Math.random() > 0.5 ? 6 : -5)).toFixed(1) + 'C')
        : s.value
      s.status    = 'alert'
      s.value     = alertVal
      s.reason    = p.reason
      s.lastUpdate = now
      eventLog.unshift({ id: Date.now() + i + 1, type: 'alert', sensorId: s.id, sensorName: s.name, reason: p.reason, time: now })
    }
  }
}

// ── Resolution capteur ──
const originalValues = {}
sensorsData.sensors.forEach(s => {
  originalValues[s.id] = { value: s.value, numValue: s.numValue ?? 0, status: 'ok' }
})

function resolveSensor(sensorId) {
  const sensor = allSensors.find(s => s.id === sensorId)
  if (!sensor) return
  const orig       = originalValues[sensorId]
  sensor.status    = 'ok'
  sensor.value     = orig?.value || '-'
  sensor.numValue  = orig?.numValue ?? 0
  sensor.reason    = null
  sensor.lastUpdate = getNow()
  eventLog.unshift({
    id: Date.now(), type: 'resolved',
    sensorId: sensor.id, sensorName: sensor.name,
    reason: 'Resolu par operateur', time: getNow()
  })
  if (selectedSensor.value?.id === sensorId) selectedSensor.value = { ...sensor }
}

// ── Commandes salle ──
function updateRoomCommand(roomId, key, value) {
  if (roomCommands[roomId]) roomCommands[roomId][key] = value
}

// ── Computed ──
const currentFloorInfo = computed(() => building.floors.find(f => f.id === selectedFloor.value))
const floorSensors     = computed(() => allSensors.filter(s => s.floor === selectedFloor.value))
const floorRooms       = computed(() => rooms.filter(r => r.floor === selectedFloor.value))

function selectFloor(floorId) {
  selectedFloor.value  = floorId
  selectedSensor.value = null
}

function selectSensor(sensor) {
  if (sensor.type === 'hvac') {
    ctaSensor.value   = sensor
    showCtaView.value = true
  } else {
    selectedSensor.value = sensor
    if (!rightPanelOpen.value) rightPanelOpen.value = true
  }
}

// ── Horloge ──
const currentTime = ref('')
const currentDate = computed(() =>
  new Date().toLocaleDateString('fr-FR', { weekday: 'long', day: 'numeric', month: 'long', year: 'numeric' })
)

let clockTimer, simTimer, eventTimer

onMounted(() => {
  currentTime.value = new Date().toLocaleTimeString('fr-FR', { hour: '2-digit', minute: '2-digit', second: '2-digit' })
  clockTimer = setInterval(() => {
    currentTime.value = new Date().toLocaleTimeString('fr-FR', { hour: '2-digit', minute: '2-digit', second: '2-digit' })
  }, 1000)

  simTimer = setInterval(simulateTick, SIMULATION_TICK)

  // Premier evenement apres 2s, puis toutes les 20s
  setTimeout(checkAndGenerateEvents, 2000)
  eventTimer = setInterval(checkAndGenerateEvents, 20000)
})

onUnmounted(() => {
  clearInterval(clockTimer)
  clearInterval(simTimer)
  clearInterval(eventTimer)
})
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
        Vue Plans
      </button>
      <button
        class="topbar__tab"
        :class="{ 'topbar__tab--active': viewMode === 'trades' }"
        @click="viewMode = 'trades'; selectedSensor = null"
      >
        Vue Metiers
      </button>
      <button
        class="topbar__tab"
        @click="showCtaView = true"
      >
        Vue CTA
      </button>
    </div>

    <div class="topbar__meta">
      <span class="topbar__clock">{{ currentDate }} - {{ currentTime }}</span>
      <div class="topbar__status-dot" title="Systeme en ligne"></div>
    </div>
  </div>

  <div class="dashboard">
    <!-- VUE PLANS -->
    <template v-if="viewMode === 'plans'">
      <div class="panel-collapsible panel-collapsible--left" :class="{ 'panel-collapsible--closed': !leftPanelOpen }">
        <FloorSelector
          :floors="building.floors"
          :sensors="allSensors"
          :selectedFloor="selectedFloor"
          @select-floor="selectFloor"
        />
      </div>
      <button
        class="panel-toggle panel-toggle--left"
        :class="{ 'panel-toggle--collapsed': !leftPanelOpen }"
        @click="leftPanelOpen = !leftPanelOpen"
      >
        <span class="panel-toggle__icon">{{ leftPanelOpen ? '&#9666;' : '&#9656;' }}</span>
      </button>

      <FloorPlan
        :floorInfo="currentFloorInfo"
        :sensors="floorSensors"
        :selectedSensor="selectedSensor"
        @select-sensor="selectSensor"
      />

      <button
        class="panel-toggle panel-toggle--right"
        :class="{ 'panel-toggle--collapsed': !rightPanelOpen }"
        @click="rightPanelOpen = !rightPanelOpen"
      >
        <span class="panel-toggle__icon">{{ rightPanelOpen ? '&#9656;' : '&#9666;' }}</span>
      </button>
      <div class="panel-collapsible panel-collapsible--right" :class="{ 'panel-collapsible--closed': !rightPanelOpen }">
        <StatusSidebar
          :sensors="floorSensors"
          :allSensors="allSensors"
          :selectedSensor="selectedSensor"
          :floorInfo="currentFloorInfo"
          :eventLog="eventLog"
          :floorRooms="floorRooms"
          :roomCommands="roomCommands"
          @select-sensor="selectSensor"
          @resolve-sensor="resolveSensor"
          @update-room-command="({ roomId, key, value }) => updateRoomCommand(roomId, key, value)"
        />
      </div>
    </template>

    <!-- VUE METIERS -->
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

  <!-- CTA Detail View -->
  <CtaDetailView
    v-if="showCtaView"
    :sensor="ctaSensor"
    @close="showCtaView = false"
  />
</template>