<script setup>
import { ref, computed, watch } from 'vue'
import SensorMarker from './SensorMarker.vue'

const props = defineProps({
  floorInfo: { type: Object, default: null },
  sensors: { type: Array, required: true },
  selectedSensor: { type: Object, default: null }
})

const emit = defineEmits(['select-sensor'])

const floorImages = {
   0: { src: './plans/rdc.png',  viewBox: { w: 1200, h: 800 } },
   1: { src: '/plans/r+1.png',   viewBox: { w: 1200, h: 800 } },
}

const fallbackLayouts = {
  0: {
    rooms: [
      { x: 40, y: 40, w: 340, h: 180, label: 'Hall Principal' },
      { x: 40, y: 240, w: 180, h: 150, label: 'Accueil' },
      { x: 240, y: 240, w: 140, h: 150, label: 'Sécurité' },
      { x: 420, y: 40, w: 200, h: 180, label: 'Espace Attente' },
      { x: 420, y: 240, w: 200, h: 150, label: 'Local Technique' },
      { x: 640, y: 40, w: 160, h: 350, label: 'Couloir Est' }
    ],
    doors: [
      { x1: 120, y1: 240, x2: 160, y2: 240 },
      { x1: 380, y1: 130, x2: 420, y2: 130 },
      { x1: 300, y1: 240, x2: 340, y2: 240 },
      { x1: 620, y1: 130, x2: 640, y2: 130 }
    ]
  },
  1: {
    rooms: [
      { x: 40, y: 40, w: 300, h: 160, label: 'Open Space Nord' },
      { x: 40, y: 220, w: 160, h: 170, label: 'Bureau 101' },
      { x: 220, y: 220, w: 120, h: 170, label: 'Kitchenette' },
      { x: 360, y: 40, w: 260, h: 160, label: 'Open Space Centre' },
      { x: 360, y: 220, w: 160, h: 170, label: 'Bureau 102' },
      { x: 540, y: 220, w: 80, h: 170, label: 'WC' },
      { x: 640, y: 40, w: 160, h: 350, label: 'Couloir Est' }
    ],
    doors: [
      { x1: 340, y1: 120, x2: 360, y2: 120 },
      { x1: 120, y1: 220, x2: 160, y2: 220 },
      { x1: 440, y1: 220, x2: 480, y2: 220 },
      { x1: 620, y1: 120, x2: 640, y2: 120 }
    ]
  },
  2: {
    rooms: [
      { x: 40, y: 40, w: 280, h: 190, label: 'Salle Panorama' },
      { x: 40, y: 250, w: 280, h: 140, label: 'Salle Créativité' },
      { x: 340, y: 40, w: 140, h: 350, label: 'Couloir Central' },
      { x: 500, y: 40, w: 200, h: 190, label: 'Salle Innovation' },
      { x: 500, y: 250, w: 120, h: 140, label: 'Archive' },
      { x: 720, y: 40, w: 80, h: 350, label: 'Local Serveur' }
    ],
    doors: [
      { x1: 320, y1: 130, x2: 340, y2: 130 },
      { x1: 480, y1: 130, x2: 500, y2: 130 },
      { x1: 320, y1: 310, x2: 340, y2: 310 },
      { x1: 700, y1: 210, x2: 720, y2: 210 }
    ]
  },
  3: {
    rooms: [
      { x: 40, y: 40, w: 300, h: 200, label: 'Bureau Direction' },
      { x: 40, y: 260, w: 300, h: 130, label: 'Secrétariat' },
      { x: 360, y: 40, w: 100, h: 350, label: 'Hall Privé' },
      { x: 480, y: 40, w: 240, h: 200, label: 'Salle du Conseil' },
      { x: 480, y: 260, w: 240, h: 130, label: 'Lounge Direction' },
      { x: 740, y: 40, w: 60, h: 350, label: 'Terr.' }
    ],
    doors: [
      { x1: 340, y1: 140, x2: 360, y2: 140 },
      { x1: 460, y1: 140, x2: 480, y2: 140 },
      { x1: 340, y1: 320, x2: 360, y2: 320 },
      { x1: 720, y1: 140, x2: 740, y2: 140 }
    ]
  }
}

// ── Computed state ──
const floorId = computed(() => props.floorInfo?.id ?? 0)
const hasImage = computed(() => !!floorImages[floorId.value])
const floorImage = computed(() => floorImages[floorId.value] || null)
const fallbackLayout = computed(() => fallbackLayouts[floorId.value] || fallbackLayouts[0])

const vbW = computed(() => floorImage.value?.viewBox?.w || 840)
const vbH = computed(() => floorImage.value?.viewBox?.h || 440)
const viewBox = computed(() => `0 0 ${vbW.value} ${vbH.value}`)

const imageError = ref(false)
watch(floorId, () => {
  imageError.value = false
  positionOverrides.value = {}
  draggingSensorId.value = null
  showExportPanel.value = false
})

function onImageError() {
  imageError.value = true
  console.warn(`[GTB] Image introuvable pour l'étage ${floorId.value}. Fallback SVG utilisé.`)
}

const showFallback = computed(() => !hasImage.value || imageError.value)

// ── Coordonnées au survol ──
const showCoords = ref(false)
const mouseX = ref(0)
const mouseY = ref(0)

// ── Référence SVG pour la transformation de coordonnées ──
const svgEl = ref(null)

function toSvgCoords(clientX, clientY) {
  if (!svgEl.value) return { x: 0, y: 0 }
  const pt = svgEl.value.createSVGPoint()
  pt.x = clientX
  pt.y = clientY
  const m = svgEl.value.getScreenCTM()
  if (!m) return { x: 0, y: 0 }
  const transformed = pt.matrixTransform(m.inverse())
  return { x: Math.round(transformed.x), y: Math.round(transformed.y) }
}

function onSvgMouseMove(e) {
  if (!showCoords.value && !editMode.value) return
  const pt = toSvgCoords(e.clientX, e.clientY)
  mouseX.value = pt.x
  mouseY.value = pt.y

  if (draggingSensorId.value) {
    const newX = Math.max(0, Math.min(vbW.value, pt.x - dragOffset.value.x))
    const newY = Math.max(0, Math.min(vbH.value, pt.y - dragOffset.value.y))
    positionOverrides.value[draggingSensorId.value] = { x: newX, y: newY }
  }
}

// ════════════════════════════════════════════════════════
// MODE ÉDITION — DRAG & DROP
// ════════════════════════════════════════════════════════
const editMode = ref(false)
const positionOverrides = ref({})   // { [sensorId]: { x, y } }
const draggingSensorId = ref(null)
const dragOffset = ref({ x: 0, y: 0 })
const showExportPanel = ref(false)
const exportCopied = ref(false)

const changedCount = computed(() => Object.keys(positionOverrides.value).length)

// Capteurs avec positions surchargées pour l'affichage
const displaySensors = computed(() =>
  props.sensors.map(s => {
    const ov = positionOverrides.value[s.id]
    return ov ? { ...s, x: ov.x, y: ov.y } : s
  })
)

function getSensorPos(sensorId) {
  if (positionOverrides.value[sensorId]) return positionOverrides.value[sensorId]
  const s = props.sensors.find(s => s.id === sensorId)
  return s ? { x: s.x, y: s.y } : { x: 0, y: 0 }
}

function onDragHandleMousedown(e, sensor) {
  e.stopPropagation()
  e.preventDefault()
  const pt = toSvgCoords(e.clientX, e.clientY)
  const cur = getSensorPos(sensor.id)
  dragOffset.value = { x: pt.x - cur.x, y: pt.y - cur.y }
  draggingSensorId.value = sensor.id
}

function onSvgMouseUp() {
  draggingSensorId.value = null
}

function toggleEditMode() {
  editMode.value = !editMode.value
  if (!editMode.value) showExportPanel.value = false
}

function resetPositions() {
  positionOverrides.value = {}
}

// Texte affiché dans le panneau export (changements uniquement)
const exportChangesText = computed(() => {
  const changes = props.sensors
    .filter(s => positionOverrides.value[s.id])
    .map(s => ({
      id: s.id,
      name: s.name,
      x: positionOverrides.value[s.id].x,
      y: positionOverrides.value[s.id].y
    }))
  return JSON.stringify(changes, null, 2)
})

async function copyChanges() {
  await navigator.clipboard.writeText(exportChangesText.value)
  exportCopied.value = true
  setTimeout(() => { exportCopied.value = false }, 2000)
}

function downloadFullSensors() {
  // On génère le tableau complet des capteurs de cet étage avec les nouvelles positions
  const updated = props.sensors.map(s => {
    const ov = positionOverrides.value[s.id]
    return ov ? { ...s, x: ov.x, y: ov.y } : s
  })
  const blob = new Blob([JSON.stringify(updated, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `sensors_etage${floorId.value}_positions.json`
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}
</script>

<template>
  <main class="floorplan-container">
    <div class="floorplan__header" :class="{ 'floorplan__header--edit': editMode }">
      <span class="floorplan__title">{{ floorInfo?.name || "Plan d'étage" }}</span>

      <!-- ═══ BARRE D'ÉDITION (mode édition actif) ═══ -->
      <div v-if="editMode" class="edit-toolbar">
        <span class="edit-toolbar__status">
          <span class="edit-pulse"></span>
          Mode édition
        </span>
        <span v-if="changedCount > 0" class="edit-toolbar__count">
          {{ changedCount }} capteur{{ changedCount > 1 ? 's' : '' }} déplacé{{ changedCount > 1 ? 's' : '' }}
        </span>
        <div class="edit-toolbar__actions">
          <button class="edit-btn edit-btn--reset" :disabled="changedCount === 0" @click="resetPositions" title="Annuler tous les déplacements">
            ↩ Reset
          </button>
          <button class="edit-btn edit-btn--export" :disabled="changedCount === 0" @click="showExportPanel = !showExportPanel" title="Exporter les nouvelles positions">
            ⬇ Exporter ({{ changedCount }})
          </button>
          <button class="edit-btn edit-btn--done" @click="toggleEditMode">
            ✓ Terminer
          </button>
        </div>
      </div>

      <!-- ═══ CONTRÔLES NORMAUX ═══ -->
      <div v-else class="floorplan__header-right">
        <button
          class="coords-toggle"
          :class="{ 'coords-toggle--active': showCoords }"
          @click="showCoords = !showCoords"
          title="Afficher les coordonnées au survol"
        >📐</button>
        <button
          class="coords-toggle edit-mode-btn"
          @click="toggleEditMode"
          title="Activer le mode édition drag & drop"
        >✏️</button>
        <span class="floorplan__badge">{{ sensors.length }} capteurs</span>
      </div>
    </div>

    <!-- ═══ PANNEAU EXPORT ═══ -->
    <div v-if="showExportPanel && editMode" class="export-panel">
      <div class="export-panel__header">
        <span>📋 Positions modifiées — {{ changedCount }} capteur{{ changedCount > 1 ? 's' : '' }}</span>
        <button class="export-panel__close" @click="showExportPanel = false">✕</button>
      </div>
      <pre class="export-panel__code">{{ exportChangesText }}</pre>
      <div class="export-panel__actions">
        <button class="edit-btn edit-btn--copy" @click="copyChanges">
          {{ exportCopied ? '✓ Copié !' : '⎘ Copier les changements' }}
        </button>
        <button class="edit-btn edit-btn--download" @click="downloadFullSensors">
          ⬇ Télécharger sensors_etage{{ floorId }}.json
        </button>
      </div>
      <p class="export-panel__hint">
        Copiez les nouvelles valeurs <code>x</code> / <code>y</code> dans <code>src/data/sensors.json</code>
      </p>
    </div>

    <div class="floorplan__canvas">
      <div class="floorplan__svg-wrapper" :style="{ aspectRatio: vbW + ' / ' + vbH }">
        <svg
          ref="svgEl"
          :viewBox="viewBox"
          xmlns="http://www.w3.org/2000/svg"
          preserveAspectRatio="xMidYMid meet"
          :style="draggingSensorId ? 'cursor: grabbing' : ''"
          @mousemove="onSvgMouseMove"
          @mouseup="onSvgMouseUp"
          @mouseleave="onSvgMouseUp"
        >
          <!-- ═══ MODE IMAGE ═══ -->
          <template v-if="!showFallback">
            <image
              :href="floorImage.src"
              x="0" y="0"
              :width="vbW"
              :height="vbH"
              preserveAspectRatio="xMidYMid meet"
              class="floorplan__bg-image"
              @error="onImageError"
            />
            <rect x="0" y="0" :width="vbW" :height="vbH" fill="rgba(10, 14, 20, 0.15)" pointer-events="none" />
          </template>

          <!-- ═══ MODE FALLBACK SVG ═══ -->
          <template v-if="showFallback">
            <defs>
              <pattern id="grid" width="20" height="20" patternUnits="userSpaceOnUse">
                <path d="M 20 0 L 0 0 0 20" fill="none" class="floor-svg-grid" />
              </pattern>
            </defs>
            <rect :width="vbW" :height="vbH" fill="url(#grid)" opacity="0.5" />
            <rect x="30" y="30" :width="vbW - 60" :height="vbH - 60" fill="none" stroke="#2a3a50" stroke-width="2.5" rx="4" />
            <g v-for="(room, idx) in fallbackLayout.rooms" :key="'room-' + idx">
              <rect :x="room.x" :y="room.y" :width="room.w" :height="room.h" class="floor-svg-room" rx="2" />
              <text :x="room.x + room.w / 2" :y="room.y + room.h / 2" class="floor-svg-label">{{ room.label }}</text>
            </g>
            <g v-for="(door, idx) in fallbackLayout.doors" :key="'door-' + idx">
              <line :x1="door.x1" :y1="door.y1" :x2="door.x2" :y2="door.y2" stroke="#00aaff" stroke-width="3" opacity="0.35" stroke-linecap="round" />
            </g>
          </template>

          <!-- ═══ MARQUEURS CAPTEURS ═══ -->
          <SensorMarker
            v-for="sensor in displaySensors"
            :key="sensor.id"
            :sensor="sensor"
            :isSelected="!editMode && selectedSensor?.id === sensor.id"
            @click="!editMode && emit('select-sensor', sensor)"
            :size="2"
            :style="editMode ? 'pointer-events: none; opacity: 0.75' : ''"
          />

          <!-- ═══ POIGNÉES DE DRAG (mode édition uniquement) ═══ -->
          <template v-if="editMode">
            <g
              v-for="sensor in displaySensors"
              :key="'drag-' + sensor.id"
              :transform="`translate(${sensor.x}, ${sensor.y})`"
              style="cursor: grab"
              @mousedown="onDragHandleMousedown($event, sensor)"
            >
              <!-- Zone de clic large -->
              <circle
                cx="0" cy="0" r="22"
                fill="transparent"
              />
              <!-- Anneau visuel -->
              <circle
                cx="0" cy="0" r="18"
                fill="none"
                :stroke="positionOverrides[sensor.id] ? '#f0a500' : 'rgba(0,170,255,0.55)'"
                stroke-width="2"
                stroke-dasharray="5 3"
                :class="{ 'drag-ring--moved': !!positionOverrides[sensor.id], 'drag-ring--active': draggingSensorId === sensor.id }"
              />
              <!-- Petit indicateur de déplacement -->
              <circle
                v-if="positionOverrides[sensor.id]"
                cx="13" cy="-13" r="5"
                fill="#f0a500"
              />
              <text
                v-if="positionOverrides[sensor.id]"
                x="13" y="-9"
                fill="#111" font-size="6" text-anchor="middle" dominant-baseline="central"
                font-weight="bold" pointer-events="none"
              >✓</text>
              <!-- Tooltip coordonnées si dragging -->
              <template v-if="draggingSensorId === sensor.id">
                <rect x="20" y="-28" width="96" height="20" rx="3" fill="#0a0e14" stroke="#f0a500" stroke-width="1" opacity="0.95" pointer-events="none" />
                <text x="26" y="-14" fill="#f0a500" font-family="'JetBrains Mono', monospace" font-size="10" pointer-events="none">
                  x:{{ sensor.x }}  y:{{ sensor.y }}
                </text>
              </template>
            </g>

            <!-- Overlay éditeur : fond légèrement teinté pour signaler le mode édition -->
            <rect x="0" y="0" :width="vbW" :height="vbH" fill="rgba(240,165,0,0.03)" pointer-events="none" />
          </template>

          <!-- ═══ COORDONNÉES AU SURVOL ═══ -->
          <template v-if="showCoords || (editMode && !draggingSensorId)">
            <line :x1="mouseX" y1="0" :x2="mouseX" :y2="vbH" stroke="#00aaff" stroke-width="0.5" opacity="0.35" stroke-dasharray="4 4" pointer-events="none" />
            <line x1="0" :y1="mouseY" :x2="vbW" :y2="mouseY" stroke="#00aaff" stroke-width="0.5" opacity="0.35" stroke-dasharray="4 4" pointer-events="none" />
            <rect :x="mouseX + 8" :y="mouseY - 24" width="90" height="20" rx="3" fill="#111820" stroke="#2a3a50" opacity="0.9" pointer-events="none" />
            <text :x="mouseX + 14" :y="mouseY - 10" fill="#00aaff" font-family="'JetBrains Mono', monospace" font-size="11" pointer-events="none">
              x:{{ mouseX }} y:{{ mouseY }}
            </text>
          </template>
        </svg>
      </div>
    </div>
  </main>
</template>

<style scoped>
/* ── Edit mode header ── */
.floorplan__header--edit {
  border-bottom-color: rgba(240, 165, 0, 0.3);
  background: rgba(240, 165, 0, 0.04);
}

/* ── Edit toolbar ── */
.edit-toolbar {
  display: flex;
  align-items: center;
  gap: 10px;
  flex: 1;
  justify-content: flex-end;
}

.edit-toolbar__status {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 11px;
  font-weight: 600;
  color: #f0a500;
  font-family: 'JetBrains Mono', monospace;
  letter-spacing: 0.05em;
  text-transform: uppercase;
}

.edit-pulse {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: #f0a500;
  box-shadow: 0 0 8px #f0a500;
  animation: pulse-edit 1.2s ease-in-out infinite;
}

@keyframes pulse-edit {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.5; transform: scale(0.85); }
}

.edit-toolbar__count {
  font-size: 11px;
  color: #f0a500;
  background: rgba(240, 165, 0, 0.15);
  border: 1px solid rgba(240, 165, 0, 0.3);
  border-radius: 12px;
  padding: 2px 10px;
  font-family: 'JetBrains Mono', monospace;
}

.edit-toolbar__actions {
  display: flex;
  gap: 6px;
}

/* ── Boutons édition ── */
.edit-btn {
  padding: 5px 12px;
  border-radius: 6px;
  border: 1px solid rgba(255,255,255,0.12);
  background: rgba(255,255,255,0.05);
  color: #cdd6e0;
  font-size: 11px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s ease;
  letter-spacing: 0.02em;
}

.edit-btn:hover:not(:disabled) {
  background: rgba(255,255,255,0.1);
  color: #fff;
}

.edit-btn:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.edit-btn--reset {
  border-color: rgba(255, 80, 80, 0.3);
  color: #ff8080;
}

.edit-btn--reset:hover:not(:disabled) {
  background: rgba(255, 80, 80, 0.1);
  border-color: rgba(255, 80, 80, 0.5);
}

.edit-btn--export {
  border-color: rgba(240, 165, 0, 0.4);
  color: #f0a500;
}

.edit-btn--export:hover:not(:disabled) {
  background: rgba(240, 165, 0, 0.1);
  border-color: rgba(240, 165, 0, 0.6);
}

.edit-btn--done {
  border-color: rgba(63, 185, 80, 0.4);
  color: #3fb950;
  background: rgba(63, 185, 80, 0.08);
}

.edit-btn--done:hover {
  background: rgba(63, 185, 80, 0.15);
  border-color: rgba(63, 185, 80, 0.6);
}

.edit-btn--copy {
  border-color: rgba(88, 166, 255, 0.4);
  color: #58a6ff;
}

.edit-btn--copy:hover {
  background: rgba(88, 166, 255, 0.1);
}

.edit-btn--download {
  border-color: rgba(63, 185, 80, 0.4);
  color: #3fb950;
}

.edit-btn--download:hover {
  background: rgba(63, 185, 80, 0.1);
}

/* ── Edit mode button in normal header ── */
.edit-mode-btn {
  font-size: 13px;
}

/* ── Export panel ── */
.export-panel {
  background: #0d1117;
  border-bottom: 1px solid rgba(240, 165, 0, 0.25);
  padding: 12px 16px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.export-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 12px;
  font-weight: 600;
  color: #f0a500;
  font-family: 'JetBrains Mono', monospace;
}

.export-panel__close {
  background: none;
  border: none;
  color: #666;
  cursor: pointer;
  font-size: 14px;
  padding: 0 4px;
  line-height: 1;
}

.export-panel__close:hover { color: #fff; }

.export-panel__code {
  background: #070b0f;
  border: 1px solid rgba(255,255,255,0.08);
  border-radius: 6px;
  padding: 10px 12px;
  font-family: 'JetBrains Mono', monospace;
  font-size: 10px;
  color: #7ee787;
  overflow-x: auto;
  max-height: 160px;
  overflow-y: auto;
  margin: 0;
  line-height: 1.5;
}

.export-panel__actions {
  display: flex;
  gap: 8px;
}

.export-panel__hint {
  font-size: 10px;
  color: #4a5568;
  margin: 0;
}

.export-panel__hint code {
  color: #7ee787;
  background: rgba(126, 231, 135, 0.08);
  padding: 1px 4px;
  border-radius: 3px;
}

/* ── Drag ring animations ── */
.drag-ring--moved {
  animation: rotate-dash 8s linear infinite;
}

.drag-ring--active {
  stroke-dasharray: none !important;
  stroke-width: 2.5 !important;
  filter: drop-shadow(0 0 4px #f0a500);
}

@keyframes rotate-dash {
  from { stroke-dashoffset: 0; }
  to { stroke-dashoffset: -40; }
}
</style>