<script setup>
import { ref, computed } from 'vue'

const props = defineProps({
  rooms: { type: Array, required: true },
  sensors: { type: Array, required: true },
  roomCommands: { type: Object, required: true }
})

const emit = defineEmits(['update-room-command'])

const selectedRoomId = ref(null)

// Auto-select first room
if (props.rooms.length > 0 && !selectedRoomId.value) {
  selectedRoomId.value = props.rooms[0].id
}

const selectedRoom = computed(() => props.rooms.find(r => r.id === selectedRoomId.value))
const cmd = computed(() => props.roomCommands[selectedRoomId.value] || {})

const roomSensors = computed(() => props.sensors.filter(s => s.room === selectedRoomId.value))
const tempSensor = computed(() => roomSensors.value.find(s => s.type === 'temperature'))
const co2Sensor = computed(() => roomSensors.value.find(s => s.type === 'co2'))
const humSensor = computed(() => roomSensors.value.find(s => s.type === 'humidity'))
const lightSensor = computed(() => roomSensors.value.find(s => s.type === 'light'))
const hvacSensor = computed(() => roomSensors.value.find(s => s.type === 'hvac'))
const doorSensor = computed(() => roomSensors.value.find(s => s.type === 'door'))

const hasTemp = computed(() => !!tempSensor.value)
const hasHvac = computed(() => !!hvacSensor.value)
const hasLight = computed(() => !!lightSensor.value)
const hasCo2 = computed(() => !!co2Sensor.value)
const hasDoor = computed(() => !!doorSensor.value)

function send(key, value) {
  emit('update-room-command', { roomId: selectedRoomId.value, key, value })
}

function adjustTemp(delta) {
  const next = (cmd.value.tempSetpoint || 22) + delta
  if (next >= 15 && next <= 30) send('tempSetpoint', next)
}

// Status indicator for the room
const roomStatus = computed(() => {
  const s = roomSensors.value
  if (s.some(x => x.status === 'fault')) return 'fault'
  if (s.some(x => x.status === 'alert')) return 'alert'
  if (s.some(x => x.status === 'offline')) return 'offline'
  return 'ok'
})
</script>

<template>
  <div class="command-panel">
    <div class="command-panel__header">
      <span class="section-label">Commandes par salle</span>
    </div>

    <!-- Room selector -->
    <div class="room-selector">
      <button
        v-for="room in rooms"
        :key="room.id"
        class="room-tab"
        :class="{ 'room-tab--active': selectedRoomId === room.id }"
        @click="selectedRoomId = room.id"
      >
        {{ room.label }}
      </button>
    </div>

    <div v-if="selectedRoom" class="command-room">
      <!-- Room live readout -->
      <div class="room-readout">
        <div v-if="tempSensor" class="room-readout__item">
          <span class="room-readout__icon">🌡</span>
          <span class="room-readout__value" :class="{ 'room-readout__value--alert': tempSensor.status !== 'ok' }">
            {{ tempSensor.status === 'ok' ? tempSensor.value : tempSensor.status === 'fault' ? 'ERR' : tempSensor.value }}
          </span>
        </div>
        <div v-if="humSensor" class="room-readout__item">
          <span class="room-readout__icon">💧</span>
          <span class="room-readout__value">{{ humSensor.value }}</span>
        </div>
        <div v-if="co2Sensor" class="room-readout__item">
          <span class="room-readout__icon">🌬</span>
          <span class="room-readout__value" :class="{ 'room-readout__value--alert': co2Sensor.numValue > 1000 }">
            {{ co2Sensor.value }}
          </span>
        </div>
        <div v-if="lightSensor" class="room-readout__item">
          <span class="room-readout__icon">💡</span>
          <span class="room-readout__value">{{ lightSensor.value }}</span>
        </div>
      </div>

      <!-- Temperature Setpoint -->
      <div v-if="hasTemp" class="command-section">
        <div class="command-section__title">
          <span>🌡 Consigne Température</span>
          <span class="command-section__live">
            Réel: <strong>{{ tempSensor?.value || '—' }}</strong>
          </span>
        </div>
        <div class="temp-control">
          <button class="temp-btn" @click="adjustTemp(-0.5)">−</button>
          <div class="temp-display">
            <span class="temp-display__value">{{ (cmd.tempSetpoint || 22).toFixed(1) }}</span>
            <span class="temp-display__unit">°C</span>
          </div>
          <button class="temp-btn" @click="adjustTemp(0.5)">+</button>
        </div>
        <div class="temp-bar">
          <div class="temp-bar__track">
            <div class="temp-bar__target" :style="{ left: ((cmd.tempSetpoint - 15) / 15 * 100) + '%' }"></div>
            <div v-if="tempSensor?.status === 'ok'" class="temp-bar__actual" :style="{ left: ((tempSensor.numValue - 15) / 15 * 100) + '%' }"></div>
          </div>
          <div class="temp-bar__labels"><span>15°C</span><span>22.5°C</span><span>30°C</span></div>
        </div>
      </div>

      <!-- HVAC Mode -->
      <div v-if="hasHvac || hasTemp" class="command-section">
        <div class="command-section__title">
          <span>❄ Mode CVC</span>
          <span class="command-section__live">{{ hvacSensor?.value || '—' }}</span>
        </div>
        <div class="mode-selector">
          <button v-for="m in [{key:'auto',label:'Auto',icon:'⟳'},{key:'heat',label:'Chauf.',icon:'🔥'},{key:'cool',label:'Clim',icon:'❄'},{key:'off',label:'Arrêt',icon:'⏻'}]"
            :key="m.key" class="mode-btn" :class="{ 'mode-btn--active': cmd.hvacMode === m.key }"
            @click="send('hvacMode', m.key)">
            <span class="mode-btn__icon">{{ m.icon }}</span>
            <span class="mode-btn__label">{{ m.label }}</span>
          </button>
        </div>
      </div>

      <!-- Lighting -->
      <div v-if="hasLight" class="command-section">
        <div class="command-section__title">
          <span>💡 Éclairage</span>
          <span class="command-section__live">{{ lightSensor?.value || '—' }}</span>
        </div>
        <div class="slider-control">
          <input type="range" min="0" max="100" step="5" :value="cmd.lightLevel"
            class="slider" @input="send('lightLevel', Number($event.target.value))" />
          <div class="slider-labels"><span>0%</span><span>50%</span><span>100%</span></div>
        </div>
        <div class="mode-selector">
          <button v-for="m in [{key:'auto',label:'Auto'},{key:'manual',label:'Manuel'},{key:'eco',label:'Éco'},{key:'off',label:'Arrêt'}]"
            :key="m.key" class="mode-btn mode-btn--sm" :class="{ 'mode-btn--active': cmd.lightMode === m.key }"
            @click="send('lightMode', m.key)">
            {{ m.label }}
          </button>
        </div>
      </div>

      <!-- Ventilation -->
      <div v-if="hasCo2" class="command-section">
        <div class="command-section__title">
          <span>🌬 Ventilation</span>
          <span class="command-section__live">CO₂: {{ co2Sensor?.value || '—' }}</span>
        </div>
        <div class="mode-selector">
          <button v-for="s in [{key:'low',label:'Faible'},{key:'medium',label:'Moyen'},{key:'high',label:'Fort'},{key:'auto',label:'Auto'}]"
            :key="s.key" class="mode-btn mode-btn--sm" :class="{ 'mode-btn--active': cmd.ventSpeed === s.key }"
            @click="send('ventSpeed', s.key)">
            {{ s.label }}
          </button>
        </div>
      </div>

      <!-- Door -->
      <div v-if="hasDoor" class="command-section">
        <div class="command-section__title">
          <span>🚪 Contrôle d'accès</span>
          <span class="command-section__live">{{ doorSensor?.value || '—' }}</span>
        </div>
        <button class="door-toggle" :class="{ 'door-toggle--locked': cmd.doorLocked }"
          @click="send('doorLocked', !cmd.doorLocked)">
          <span class="door-toggle__icon">{{ cmd.doorLocked ? '🔒' : '🔓' }}</span>
          <span>{{ cmd.doorLocked ? 'Verrouillée' : 'Déverrouillée' }}</span>
        </button>
      </div>

      <!-- Info: sensors with no controls -->
      <div v-if="!hasTemp && !hasLight && !hasCo2 && !hasDoor" class="command-empty">
        Pas de commande disponible pour cette salle.<br />
        Les capteurs présents sont en lecture seule.
      </div>
    </div>
  </div>
</template>
