<script setup>
import { ref, computed, watch } from 'vue'
import SensorMarker from './SensorMarker.vue'

const props = defineProps({
  floorInfo: { type: Object, default: null },
  sensors: { type: Array, required: true },
  selectedSensor: { type: Object, default: null }
})

const emit = defineEmits(['select-sensor'])

// ═══════════════════════════════════════════════════════
// CONFIGURATION DES PLANS D'ÉTAGE
// ═══════════════════════════════════════════════════════
// Remplacez les chemins ci-dessous par vos fichiers PNG/JPG.
// Placez vos images dans /src/assets/plans/ ou /public/plans/
//
// Largeur × Hauteur du viewBox = résolution de référence.
// Les coordonnées x,y des capteurs dans sensors.json doivent
// correspondre à cette résolution.
//
// Exemple : si votre image fait 1200×800 px, mettez :
//   viewBox: { w: 1200, h: 800 }
// et positionnez les capteurs avec x entre 0-1200 et y entre 0-800.
//
// Pour trouver les coordonnées, ouvrez votre image dans un éditeur
// (Paint, GIMP, Photoshop...) et relevez les x,y au pixel.
// ═══════════════════════════════════════════════════════

const floorImages = {
  // id_étage: { src: 'chemin/vers/image.png', viewBox: { w: largeur, h: hauteur } }
  //
  // Exemples (décommenter et adapter) :
   0: { src: './plans/rdc.png',  viewBox: { w: 1200, h: 800 } },
   1: { src: '/plans/r+1.png',   viewBox: { w: 1200, h: 800 } },
  // 2: { src: '/plans/r2.png',   viewBox: { w: 1200, h: 800 } },
  // 3: { src: '/plans/r3.jpg',   viewBox: { w: 1200, h: 800 } },
}

// ═══════════════════════════════════════════════════════
// PLANS SVG GÉNÉRÉS (FALLBACK)
// Utilisés quand aucune image n'est configurée pour un étage.
// ═══════════════════════════════════════════════════════
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

// ViewBox dimensions: use image config if available, otherwise 840×440
const vbW = computed(() => floorImage.value?.viewBox?.w || 840)
const vbH = computed(() => floorImage.value?.viewBox?.h || 440)
const viewBox = computed(() => `0 0 ${vbW.value} ${vbH.value}`)

// Image load error tracking
const imageError = ref(false)
watch(floorId, () => { imageError.value = false })

function onImageError() {
  imageError.value = true
  console.warn(`[GTB] Image introuvable pour l'étage ${floorId.value}. Fallback SVG utilisé.`)
}

// Show fallback if no image configured or image failed to load
const showFallback = computed(() => !hasImage.value || imageError.value)

// Mouse position for coordinate helper (dev mode)
const showCoords = ref(false)
const mouseX = ref(0)
const mouseY = ref(0)

function onSvgMouseMove(e) {
  if (!showCoords.value) return
  const svg = e.currentTarget
  const rect = svg.getBoundingClientRect()
  const scaleX = vbW.value / rect.width
  const scaleY = vbH.value / rect.height
  mouseX.value = Math.round((e.clientX - rect.left) * scaleX)
  mouseY.value = Math.round((e.clientY - rect.top) * scaleY)
}
</script>

<template>
  <main class="floorplan-container">
    <div class="floorplan__header">
      <span class="floorplan__title">{{ floorInfo?.name || 'Plan d\'étage' }}</span>
      <div class="floorplan__header-right">
        <!-- Bouton aide coordonnées (dev) -->
        <button
          class="coords-toggle"
          :class="{ 'coords-toggle--active': showCoords }"
          @click="showCoords = !showCoords"
          title="Afficher les coordonnées au survol (aide au placement)"
        >
          📐
        </button>
        <span class="floorplan__badge">{{ sensors.length }} capteurs</span>
      </div>
    </div>

    <div class="floorplan__canvas">
      <div class="floorplan__svg-wrapper" :style="{ aspectRatio: vbW + ' / ' + vbH }">
        <svg
          :viewBox="viewBox"
          xmlns="http://www.w3.org/2000/svg"
          preserveAspectRatio="xMidYMid meet"
          @mousemove="onSvgMouseMove"
        >
          <!-- ═══ MODE IMAGE : plan PNG/JPG en fond ═══ -->
          <template v-if="!showFallback">
            <!-- Image de fond -->
            <image
              :href="floorImage.src"
              x="0" y="0"
              :width="vbW"
              :height="vbH"
              preserveAspectRatio="xMidYMid meet"
              class="floorplan__bg-image"
              @error="onImageError"
            />
            <!-- Overlay sombre léger pour contraste des marqueurs -->
            <rect
              x="0" y="0"
              :width="vbW" :height="vbH"
              fill="rgba(10, 14, 20, 0.15)"
              pointer-events="none"
            />
          </template>

          <!-- ═══ MODE FALLBACK : plan SVG généré ═══ -->
          <template v-if="showFallback">
            <defs>
              <pattern id="grid" width="20" height="20" patternUnits="userSpaceOnUse">
                <path d="M 20 0 L 0 0 0 20" fill="none" class="floor-svg-grid" />
              </pattern>
            </defs>
            <rect :width="vbW" :height="vbH" fill="url(#grid)" opacity="0.5" />
            <rect
              x="30" y="30"
              :width="vbW - 60" :height="vbH - 60"
              fill="none" stroke="#2a3a50" stroke-width="2.5" rx="4"
            />
            <g v-for="(room, idx) in fallbackLayout.rooms" :key="'room-' + idx">
              <rect :x="room.x" :y="room.y" :width="room.w" :height="room.h" class="floor-svg-room" rx="2" />
              <text :x="room.x + room.w / 2" :y="room.y + room.h / 2" class="floor-svg-label">{{ room.label }}</text>
            </g>
            <g v-for="(door, idx) in fallbackLayout.doors" :key="'door-' + idx">
              <line :x1="door.x1" :y1="door.y1" :x2="door.x2" :y2="door.y2" stroke="#00aaff" stroke-width="3" opacity="0.35" stroke-linecap="round" />
            </g>
          </template>

          <!-- ═══ MARQUEURS CAPTEURS (superposés sur les deux modes) ═══ -->
          <SensorMarker
            v-for="sensor in sensors"
            :key="sensor.id"
            :sensor="sensor"
            :isSelected="selectedSensor?.id === sensor.id"
            @click="emit('select-sensor', sensor)"
          />

          <!-- ═══ COORDONNÉES AU SURVOL (aide placement) ═══ -->
          <template v-if="showCoords">
            <!-- Crosshair -->
            <line :x1="mouseX" y1="0" :x2="mouseX" :y2="vbH" stroke="#00aaff" stroke-width="0.5" opacity="0.4" stroke-dasharray="4 4" pointer-events="none" />
            <line x1="0" :y1="mouseY" :x2="vbW" :y2="mouseY" stroke="#00aaff" stroke-width="0.5" opacity="0.4" stroke-dasharray="4 4" pointer-events="none" />
            <!-- Label -->
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
