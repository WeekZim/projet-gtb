<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const props = defineProps({
  sensor: { type: Object, default: null }
})

const emit = defineEmits(['close'])

const activePage = ref('syn')
const currentTime = ref('—')
const toastMsg = ref('')
const toastType = ref('ok')
const toastVisible = ref(false)

const modalOpen = ref(false)
const modalTitle = ref('')
const modalRows = ref([])

const consigne = ref(20)
const ventSouf = ref(62)
const ventRep = ref(63)
const battFroid = ref(12)
const currentMode = ref('normal')
const toggles = ref({ on: true, byp: false, dg: false, bfa: true, bca: true })

const planInitialized = ref(false)
const planSlots = ref({})
const defaultPlan = [
  [0,0,0,0,0,0,1,1,1,1,1,1,1,1,1,1,1,1,1,2,2,2,0,0],
  [0,0,0,0,0,0,1,1,1,1,1,1,1,1,1,1,1,1,1,2,2,2,0,0],
  [0,0,0,0,0,0,1,1,1,1,1,1,1,1,1,1,1,1,1,2,2,2,0,0],
  [0,0,0,0,0,0,1,1,1,1,1,1,1,1,1,1,1,1,1,2,2,2,0,0],
  [0,0,0,0,0,0,1,1,1,1,1,1,1,1,1,1,1,1,1,2,2,2,0,0],
  [0,0,0,0,0,0,0,2,2,2,2,2,2,0,0,0,0,0,0,0,0,0,0,0],
  [0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
]

const modalData = {
  'volet-neuf': { t: 'Volet air neuf', r: [['État', 'Ouvert', 'ok'], ['Position', '100 %', 'ok'], ['Type', 'Motorisé 24V', null], ['Alarme', 'Aucune', 'ok']] },
  'filtre-f7': { t: 'Filtre F7 — air neuf', r: [['Classe', 'F7 ePM1 ≥ 50%', null], ['Δp mesuré', '24 Pa', 'ok'], ['Seuil alarme', '100 Pa', null], ['État', 'Propre', 'ok'], ['Remplacement', '6 mois', null]] },
  'vent-souf': { t: 'Ventilateur soufflage EC', r: [['Fréquence', '62 %', 'ok'], ['Pression diff.', '198 Pa', null], ['Débit', '2 880 m³/h', null], ['Type', 'EC brushless', null], ['Alarme', 'Non', 'ok']] },
  'echangeur': { t: 'Échangeur contre-courant', r: [['Rendement EN308', '84 %', 'ok'], ['T° neuf entrée', '-0,4 °C', null], ['T° souf. sortie', '16,7 °C', null], ['T° reprise', '35,0 °C', null], ['Bypass', 'Fermé', null], ['Pas de mélange', 'Certifié ✓', 'ok']] },
  'batterie-hc': { t: 'Batterie eau chaude', r: [['Vanne', '0 %', 'off'], ['Mode', 'Arrêt (récup. suffisante)', 'off'], ['Protection gel', '5 °C active', 'ok'], ['Alarme gel', 'Non', 'ok']] },
  'batterie-froid': { t: 'Batterie eau froide CWC', r: [['Vanne', '12 %', 'ok'], ['Mode', 'Auto — refroidissement', null], ['T° départ eau glacée', '6 °C', null], ['T° retour eau glacée', '12 °C', null], ['Alarme', 'Non', 'ok']] },
  'volet-souf': { t: 'Volet soufflage', r: [['Position', '12 %', null], ['T° soufflage', '22,1 °C', null], ['Pression aval', '198 Pa', null], ['Alarme', 'Non', 'ok']] },
  'filtre-m5': { t: 'Filtre M5 — reprise', r: [['Classe', 'M5 Coarse ≥ 65%', null], ['Δp mesuré', '54 Pa', 'warn'], ['Seuil alarme', '80 Pa', null], ['État', 'Légèrement encrassé', 'warn'], ['Remplacement', '3 mois', null]] },
  'vent-rep': { t: 'Ventilateur reprise EC', r: [['Fréquence', '63 %', 'ok'], ['Pression diff.', '199 Pa', null], ['Débit', '2 880 m³/h', null], ['Alarme', 'Non', 'ok']] },
  'volet-rep': { t: 'Volet reprise', r: [['État', 'Ouvert', 'ok'], ['T° reprise', '35,0 °C', null], ['Alarme', 'Non', 'ok']] },
  'volet-rejet': { t: 'Volet extraction', r: [['Position', '0 % (fermé)', 'off'], ['Mode', 'Auto', null]] },
  'humidif': { t: 'Humidificateur', r: [['Vanne', '0 %', 'off'], ['HR cible', '42 %', null], ['État', 'Arrêt', 'off']] }
}

const debit = computed(() => Math.round(ventSouf.value / 100 * 6500).toLocaleString('fr-FR') + ' m³/h')
const tSouf = computed(() => (consigne.value + 2.1).toFixed(1) + ' °C')
const modeLabels = { arret: 'Arrêt', eco: 'Éco', normal: 'Normale', haute: 'Haute vitesse', max: 'Max débit', nuit: 'Nuit' }
const modeSpeeds = { arret: 0, eco: 30, normal: 62, haute: 80, max: 100, nuit: 25 }
const statusLabel = computed(() => toggles.value.on ? 'En marche' : 'Arrêtée')
const statusDotClass = computed(() => toggles.value.on ? 'dot-ok' : 'dot-err')
const modePillLabel = computed(() => modeLabels[currentMode.value] || 'Normale')

function toast(msg, type = 'ok') {
  toastMsg.value = msg
  toastType.value = type
  toastVisible.value = true
  setTimeout(() => toastVisible.value = false, 2200)
}

function openModal(key) {
  const d = modalData[key]
  if (!d) return
  modalTitle.value = d.t
  modalRows.value = d.r
  modalOpen.value = true
}

function closeModal() { modalOpen.value = false }

function goPage(id) {
  activePage.value = id
  if (id === 'plan') initPlan()
}

function setMode(m) {
  currentMode.value = m
  ventSouf.value = modeSpeeds[m]
  ventRep.value = Math.min(modeSpeeds[m] + 1, 100)
  toast('Mode ' + modeLabels[m] + ' activé ✔')
}

function doOn() { toast(toggles.value.on ? 'CTA démarrée ✔' : 'CTA arrêtée', toggles.value.on ? 'ok' : 'warn') }

function doReset() {
  toggles.value = { on: true, byp: false, dg: false, bfa: true, bca: true }
  ventSouf.value = 62; ventRep.value = 63; consigne.value = 20; battFroid.value = 12; currentMode.value = 'normal'
  toast('Paramètres réinitialisés ✔')
}

function urgStop() { toggles.value.on = false; toast("Arrêt d'urgence activé", 'err') }

const slotClasses = { 0: '', 1: 'on', 2: 'eco' }
const slotLabels = { 0: '', 1: 'N', 2: 'É' }
const days = ['Lun', 'Mar', 'Mer', 'Jeu', 'Ven', 'Sam', 'Dim']
const hours = Array.from({ length: 24 }, (_, i) => i)

function initPlan() {
  if (planInitialized.value) return
  planInitialized.value = true
  for (let d = 0; d < 7; d++) planSlots.value[d] = [...defaultPlan[d]]
}

function toggleSlot(d, h) {
  planSlots.value[d][h] = (planSlots.value[d][h] + 1) % 3
}

let clockTimer
function tick() {
  const n = new Date()
  currentTime.value = n.toLocaleDateString('fr-FR', { day: '2-digit', month: 'short' }) + ' ' +
    n.toLocaleTimeString('fr-FR', { hour: '2-digit', minute: '2-digit' })
}

function onKeydown(e) { if (e.key === 'Escape') emit('close') }

onMounted(() => { tick(); clockTimer = setInterval(tick, 1000); window.addEventListener('keydown', onKeydown) })
onUnmounted(() => { clearInterval(clockTimer); window.removeEventListener('keydown', onKeydown) })
</script>

<template>
  <div class="cta-overlay" @click.self="emit('close')">
    <div class="cta-view">
      <!-- HEADER -->
      <div class="cta-hdr">
        <button class="cta-back" @click="emit('close')" title="Retour">← Retour</button>
        <div class="cta-hdr-logo">S</div>
        <div class="cta-hdr-info">
          <div class="cta-hdr-name">Topvex SC60-L</div>
          <div class="cta-hdr-sub">SKU 461747 · Systemair</div>
        </div>
        <div class="cta-hdr-right">
          <div class="status-pill">
            <div class="status-dot" :class="statusDotClass"></div>
            <span :style="{ color: toggles.on ? 'var(--ok)' : 'var(--err)' }">{{ statusLabel }}</span>
          </div>
          <div class="status-pill" style="color:var(--info)">{{ modePillLabel }}</div>
          <span class="cta-hdr-clock">{{ currentTime }}</span>
        </div>
      </div>

      <!-- BODY -->
      <div class="cta-body">
        <!-- ═══ SYNOPTIQUE ═══ -->
        <div v-show="activePage === 'syn'" class="cta-page syn-page">
          <div class="kpi-strip">
            <div class="kpi-cell"><div class="kpi-lbl">T. soufflage</div><div class="kpi-val c-warn">{{ tSouf }}</div></div>
            <div class="kpi-cell"><div class="kpi-lbl">T. reprise</div><div class="kpi-val c-info">35,0 °C</div></div>
            <div class="kpi-cell"><div class="kpi-lbl">T. air neuf</div><div class="kpi-val c-err">-0,4 °C</div></div>
            <div class="kpi-cell"><div class="kpi-lbl">Récupérateur</div><div class="kpi-val c-ok">84 %</div></div>
            <div class="kpi-cell"><div class="kpi-lbl">Débit souf.</div><div class="kpi-val">{{ debit }}</div></div>
            <div class="kpi-cell"><div class="kpi-lbl">CO₂</div><div class="kpi-val c-ok">435 ppm</div></div>
            <div class="kpi-cell"><div class="kpi-lbl">Filtre M5</div><div class="kpi-val c-warn">⚠ 54 Pa</div></div>
          </div>
          <div class="syn-svg-wrap">
            <svg class="syn-svg" viewBox="0 0 900 310" preserveAspectRatio="xMidYMid meet">
              <defs>
                <pattern id="grid" width="40" height="40" patternUnits="userSpaceOnUse">
                  <path d="M40,0 L0,0 0,40" fill="none" stroke="rgba(255,255,255,.03)" stroke-width="1"/>
                </pattern>
              </defs>
              <rect width="900" height="310" fill="url(#grid)"/>

              <!-- GAINES -->
              <rect x="0" y="74" width="464" height="18" fill="#a0522d" opacity=".7"/>
              <rect x="462" y="74" width="438" height="18" fill="#c8960a" opacity=".75"/>
              <polygon points="18,68 0,83 18,98" fill="#c87840" opacity=".9"/>
              <polygon points="882,68 900,83 882,98" fill="#b8860a" opacity=".9"/>
              <rect x="0" y="214" width="464" height="18" fill="#3a7d44" opacity=".7"/>
              <rect x="462" y="214" width="438" height="18" fill="#1e6bb8" opacity=".75"/>
              <polygon points="18,208 0,223 18,238" fill="#2d7a35" opacity=".9"/>
              <polygon points="882,208 900,223 882,238" fill="#1855a0" opacity=".9"/>
              <text x="26" y="70" font-family="DM Sans" font-size="10" fill="#c87840" font-weight="700" opacity=".9">AIR NEUF</text>
              <text x="874" y="70" text-anchor="end" font-family="DM Sans" font-size="10" fill="#b8860a" font-weight="700" opacity=".9">SOUFFLAGE</text>
              <text x="26" y="244" font-family="DM Sans" font-size="10" fill="#3a9e48" font-weight="700" opacity=".9">REJET</text>
              <text x="874" y="244" text-anchor="end" font-family="DM Sans" font-size="10" fill="#4090d8" font-weight="700" opacity=".9">REPRISE</text>

              <!-- VOLET AIR NEUF -->
              <g class="clk" @click="openModal('volet-neuf')">
                <rect x="28" y="60" width="48" height="36" rx="5" fill="#1c2333" stroke="#a0522d" stroke-width="1.5"/>
                <line x1="36" y1="64" x2="68" y2="92" stroke="#ccc" stroke-width="2.5" stroke-linecap="round"/>
                <rect x="24" y="42" width="56" height="16" rx="3" fill="rgba(63,185,80,.15)" stroke="rgba(63,185,80,.3)" stroke-width="1"/>
                <text x="52" y="53" text-anchor="middle" font-family="JetBrains Mono" font-size="10" fill="#3fb950" font-weight="600">Ouvert</text>
                <text x="52" y="36" text-anchor="middle" font-family="JetBrains Mono" font-size="11" fill="#f85149" font-weight="600">-0,4 °C</text>
                <text x="52" y="112" text-anchor="middle" font-family="DM Sans" font-size="10" fill="#7d8fa8">Volet neuf</text>
              </g>

              <!-- FILTRE F7 -->
              <g class="clk" @click="openModal('filtre-f7')">
                <rect x="94" y="60" width="48" height="36" rx="5" fill="#1c2333" stroke="#58a6ff" stroke-width="1.5"/>
                <line x1="102" y1="64" x2="102" y2="92" stroke="#58a6ff" stroke-width="2"/>
                <line x1="110" y1="64" x2="110" y2="92" stroke="#58a6ff" stroke-width="2"/>
                <line x1="118" y1="64" x2="118" y2="92" stroke="#58a6ff" stroke-width="2"/>
                <line x1="126" y1="64" x2="126" y2="92" stroke="#58a6ff" stroke-width="2"/>
                <line x1="134" y1="64" x2="134" y2="92" stroke="#58a6ff" stroke-width="2"/>
                <rect x="90" y="42" width="56" height="16" rx="3" fill="rgba(88,166,255,.12)" stroke="rgba(88,166,255,.3)" stroke-width="1"/>
                <text x="118" y="53" text-anchor="middle" font-family="JetBrains Mono" font-size="10" fill="#58a6ff" font-weight="600">F7  OK</text>
                <text x="118" y="112" text-anchor="middle" font-family="DM Sans" font-size="10" fill="#7d8fa8">Filtre F7</text>
              </g>

              <!-- VENTILATEUR SOUFFLAGE -->
              <g class="clk" @click="openModal('vent-souf')">
                <circle cx="210" cy="83" r="24" fill="#1c2333" stroke="#c8960a" stroke-width="2"/>
                <path d="M200,76 Q210,66 220,76 Q226,82 220,90" fill="none" stroke="#c8960a" stroke-width="2.5" stroke-linecap="round"/>
                <polygon points="218,87 223,95 226,87" fill="#c8960a"/>
                <circle cx="210" cy="83" r="4" fill="#c8960a"/>
                <rect x="186" y="46" width="48" height="16" rx="3" fill="rgba(210,153,34,.12)" stroke="rgba(210,153,34,.3)" stroke-width="1"/>
                <text x="210" y="57" text-anchor="middle" font-family="JetBrains Mono" font-size="11" fill="#d29922" font-weight="600">{{ ventSouf }} %</text>
                <text x="210" y="122" text-anchor="middle" font-family="DM Sans" font-size="10" fill="#7d8fa8">Ventil. souf.</text>
              </g>

              <!-- ÉCHANGEUR -->
              <g class="clk" @click="openModal('echangeur')">
                <rect x="270" y="36" width="40" height="234" rx="5" fill="#1a2030" stroke="#7d8fa8" stroke-width="1.5"/>
                <line x1="270" y1="58" x2="310" y2="78" stroke="#2a3a50" stroke-width="1.5"/>
                <line x1="270" y1="78" x2="310" y2="98" stroke="#2a3a50" stroke-width="1.5"/>
                <line x1="270" y1="98" x2="310" y2="118" stroke="#2a3a50" stroke-width="1.5"/>
                <line x1="270" y1="118" x2="310" y2="138" stroke="#2a3a50" stroke-width="1.5"/>
                <line x1="270" y1="138" x2="310" y2="158" stroke="#3d5070" stroke-width="1.5"/>
                <line x1="270" y1="158" x2="310" y2="178" stroke="#2a3a50" stroke-width="1.5"/>
                <line x1="270" y1="178" x2="310" y2="198" stroke="#2a3a50" stroke-width="1.5"/>
                <line x1="270" y1="198" x2="310" y2="218" stroke="#2a3a50" stroke-width="1.5"/>
                <line x1="270" y1="218" x2="310" y2="238" stroke="#2a3a50" stroke-width="1.5"/>
                <text x="290" y="155" text-anchor="middle" font-family="DM Sans" font-size="10" fill="#7d8fa8" transform="rotate(-90,290,155)">RÉCUPÉRATEUR  84 %</text>
                <text x="290" y="285" text-anchor="middle" font-family="JetBrains Mono" font-size="11" fill="#7d8fa8" font-weight="600">16,7 °C</text>
              </g>

              <!-- BATT. EAU CHAUDE -->
              <g class="clk" @click="openModal('batterie-hc')">
                <rect x="328" y="60" width="48" height="36" rx="5" fill="#1c2333" stroke="#f85149" stroke-width="1.5"/>
                <path d="M334,68 Q340,63 346,68 Q352,73 358,68 Q364,63 370,68 Q370,76 364,81 Q358,86 352,81 Q346,76 340,81 Q334,76 334,81" fill="none" stroke="#f85149" stroke-width="1.8"/>
                <rect x="318" y="42" width="68" height="16" rx="3" fill="rgba(248,81,73,.1)" stroke="rgba(248,81,73,.25)" stroke-width="1"/>
                <text x="352" y="53" text-anchor="middle" font-family="JetBrains Mono" font-size="11" fill="#f85149" font-weight="600">0 %</text>
                <text x="352" y="112" text-anchor="middle" font-family="DM Sans" font-size="10" fill="#7d8fa8">Batt. chaude</text>
              </g>

              <!-- BATT. EAU FROIDE -->
              <g class="clk" @click="openModal('batterie-froid')">
                <rect x="392" y="60" width="48" height="36" rx="5" fill="#1c2333" stroke="#58a6ff" stroke-width="1.5"/>
                <path d="M398,68 Q404,63 410,68 Q416,73 422,68 Q428,63 434,68 Q434,76 428,81 Q422,86 416,81 Q410,76 404,81 Q398,76 398,81" fill="none" stroke="#58a6ff" stroke-width="1.8"/>
                <circle cx="404" cy="90" r="2" fill="#58a6ff" opacity=".5"/>
                <circle cx="413" cy="92" r="2" fill="#58a6ff" opacity=".5"/>
                <circle cx="422" cy="90" r="2" fill="#58a6ff" opacity=".5"/>
                <rect x="376" y="42" width="80" height="16" rx="3" fill="rgba(88,166,255,.12)" stroke="rgba(88,166,255,.3)" stroke-width="1"/>
                <text x="416" y="53" text-anchor="middle" font-family="JetBrains Mono" font-size="11" fill="#58a6ff" font-weight="600">{{ battFroid }} %  6/12°</text>
                <text x="416" y="112" text-anchor="middle" font-family="DM Sans" font-size="10" fill="#7d8fa8">Batt. froide</text>
              </g>

              <!-- T° SOUFFLAGE -->
              <text x="528" y="43" text-anchor="middle" font-family="JetBrains Mono" font-size="11" fill="#d29922" font-weight="600">{{ tSouf }}</text>

              <!-- VOLET SOUFFLAGE -->
              <g class="clk" @click="openModal('volet-souf')">
                <rect x="556" y="60" width="48" height="36" rx="5" fill="#1c2333" stroke="#c8960a" stroke-width="1.5"/>
                <line x1="564" y1="64" x2="596" y2="92" stroke="#ccc" stroke-width="2.5" stroke-linecap="round"/>
                <text x="580" y="112" text-anchor="middle" font-family="DM Sans" font-size="10" fill="#7d8fa8">Volet souf.</text>
              </g>

              <!-- FILTRE M5 REPRISE -->
              <g class="clk" @click="openModal('filtre-m5')">
                <rect x="650" y="200" width="48" height="36" rx="5" fill="#1c2333" stroke="#d29922" stroke-width="1.5"/>
                <line x1="658" y1="204" x2="658" y2="232" stroke="#d29922" stroke-width="2"/>
                <line x1="666" y1="204" x2="666" y2="232" stroke="#d29922" stroke-width="2"/>
                <line x1="674" y1="204" x2="674" y2="232" stroke="#d29922" stroke-width="2"/>
                <line x1="682" y1="204" x2="682" y2="232" stroke="#d29922" stroke-width="2"/>
                <line x1="690" y1="204" x2="690" y2="232" stroke="#d29922" stroke-width="2"/>
                <rect x="642" y="180" width="64" height="16" rx="3" fill="rgba(210,153,34,.12)" stroke="rgba(210,153,34,.3)" stroke-width="1"/>
                <text x="674" y="191" text-anchor="middle" font-family="JetBrains Mono" font-size="10" fill="#d29922" font-weight="600">M5 ⚠ 54Pa</text>
                <text x="674" y="252" text-anchor="middle" font-family="DM Sans" font-size="10" fill="#7d8fa8">Filtre M5</text>
              </g>

              <!-- VENTILATEUR REPRISE -->
              <g class="clk" @click="openModal('vent-rep')">
                <circle cx="580" cy="223" r="24" fill="#1c2333" stroke="#1e6bb8" stroke-width="2"/>
                <path d="M570,216 Q580,206 590,216 Q596,222 590,230" fill="none" stroke="#1e6bb8" stroke-width="2.5" stroke-linecap="round"/>
                <polygon points="588,227 593,235 596,227" fill="#1e6bb8"/>
                <circle cx="580" cy="223" r="4" fill="#1e6bb8"/>
                <rect x="556" y="186" width="48" height="16" rx="3" fill="rgba(30,107,184,.12)" stroke="rgba(30,107,184,.3)" stroke-width="1"/>
                <text x="580" y="197" text-anchor="middle" font-family="JetBrains Mono" font-size="11" fill="#58a6ff" font-weight="600">{{ ventRep }} %</text>
                <text x="580" y="262" text-anchor="middle" font-family="DM Sans" font-size="10" fill="#7d8fa8">Ventil. rep.</text>
              </g>

              <!-- VOLET REPRISE -->
              <g class="clk" @click="openModal('volet-rep')">
                <rect x="780" y="200" width="48" height="36" rx="5" fill="#1c2333" stroke="#1e6bb8" stroke-width="1.5"/>
                <line x1="788" y1="204" x2="820" y2="232" stroke="#ccc" stroke-width="2.5" stroke-linecap="round"/>
                <rect x="772" y="180" width="64" height="16" rx="3" fill="rgba(63,185,80,.15)" stroke="rgba(63,185,80,.3)" stroke-width="1"/>
                <text x="804" y="191" text-anchor="middle" font-family="JetBrains Mono" font-size="10" fill="#3fb950" font-weight="600">Ouvert</text>
                <text x="804" y="172" text-anchor="middle" font-family="JetBrains Mono" font-size="11" fill="#58a6ff" font-weight="600">35,0 °C</text>
                <text x="804" y="252" text-anchor="middle" font-family="DM Sans" font-size="10" fill="#7d8fa8">Volet rep.</text>
              </g>

              <!-- VOLET REJET -->
              <g class="clk" @click="openModal('volet-rejet')">
                <rect x="28" y="200" width="48" height="36" rx="5" fill="#1c2333" stroke="#3a7d44" stroke-width="1.5"/>
                <line x1="52" y1="204" x2="52" y2="232" stroke="#ccc" stroke-width="2.5" stroke-linecap="round"/>
                <rect x="24" y="180" width="56" height="16" rx="3" fill="rgba(125,143,168,.1)" stroke="rgba(125,143,168,.2)" stroke-width="1"/>
                <text x="52" y="191" text-anchor="middle" font-family="JetBrains Mono" font-size="10" fill="#7d8fa8" font-weight="600">Fermé</text>
                <text x="52" y="252" text-anchor="middle" font-family="DM Sans" font-size="10" fill="#7d8fa8">Volet rejet</text>
              </g>
            </svg>
          </div>
        </div>

        <!-- ═══ MESURES ═══ -->
        <div v-show="activePage === 'mes'" class="cta-page mes-page">
          <div class="mes-card">
            <div class="mes-head">Températures</div>
            <div class="mes-row"><span class="mes-lbl">Soufflage</span><span class="mes-val c-warn">{{ tSouf }}</span></div>
            <div class="mes-row"><span class="mes-lbl">Reprise</span><span class="mes-val c-info">35,0<span class="mes-unit">°C</span></span></div>
            <div class="mes-row"><span class="mes-lbl">Air neuf</span><span class="mes-val c-err">-0,4<span class="mes-unit">°C</span></span></div>
            <div class="mes-row"><span class="mes-lbl">Après récup.</span><span class="mes-val">16,7<span class="mes-unit">°C</span></span></div>
          </div>
          <div class="mes-card">
            <div class="mes-head">Ventilateurs</div>
            <div class="mes-row"><span class="mes-lbl">Fréq. soufflage</span><span class="mes-val c-ok">{{ ventSouf }}<span class="mes-unit">%</span></span></div>
            <div class="mes-row"><span class="mes-lbl">Fréq. reprise</span><span class="mes-val c-ok">{{ ventRep }}<span class="mes-unit">%</span></span></div>
            <div class="mes-row"><span class="mes-lbl">Δp soufflage</span><span class="mes-val">198<span class="mes-unit">Pa</span></span></div>
            <div class="mes-row"><span class="mes-lbl">Δp reprise</span><span class="mes-val">199<span class="mes-unit">Pa</span></span></div>
          </div>
          <div class="mes-card">
            <div class="mes-head">Batteries</div>
            <div class="mes-row"><span class="mes-lbl">Vanne chaude</span><span class="mes-val c-muted">0<span class="mes-unit">%</span></span></div>
            <div class="mes-row"><span class="mes-lbl">Vanne froide</span><span class="mes-val c-info">{{ battFroid }}<span class="mes-unit">%</span></span></div>
            <div class="mes-row"><span class="mes-lbl">Eau glacée</span><span class="mes-val">6 / 12<span class="mes-unit">°C</span></span></div>
          </div>
          <div class="mes-card">
            <div class="mes-head">Filtres & Qualité</div>
            <div class="mes-row"><span class="mes-lbl">Δp Filtre F7</span><span class="mes-val c-ok">24<span class="mes-unit">Pa</span></span></div>
            <div class="mes-row"><span class="mes-lbl">Δp Filtre M5</span><span class="mes-val c-warn">54<span class="mes-unit">Pa</span></span></div>
            <div class="mes-row"><span class="mes-lbl">CO₂ reprise</span><span class="mes-val c-ok">435<span class="mes-unit">ppm</span></span></div>
            <div class="mes-row"><span class="mes-lbl">Récupérateur</span><span class="mes-val c-ok">84<span class="mes-unit">%</span></span></div>
          </div>
        </div>

        <!-- ═══ CONTRÔLE ═══ -->
        <div v-show="activePage === 'ctrl'" class="cta-page ctrl-page">
          <div class="ccard">
            <div class="ccard-title">Marche / Arrêt</div>
            <div class="crow">
              <span class="clbl">Centrale</span>
              <label class="tog"><input type="checkbox" v-model="toggles.on" @change="doOn"><span class="tog-tr"></span><span class="tog-th"></span></label>
              <span class="tstate" :class="toggles.on ? 'on' : 'off'">{{ toggles.on ? 'ON' : 'OFF' }}</span>
            </div>
          </div>
          <div class="ccard">
            <div class="ccard-title">Mode de fonctionnement</div>
            <div class="msel">
              <button v-for="m in ['arret', 'eco', 'normal', 'haute', 'max', 'nuit']" :key="m" class="mopt" :class="{ sel: currentMode === m }" @click="setMode(m)">{{ modeLabels[m] }}</button>
            </div>
            <div class="crow" style="margin-top:10px"><span class="clbl">Mode actif</span><span class="cval c-info">{{ modePillLabel }}</span></div>
          </div>
          <div class="ccard">
            <div class="ccard-title">Consigne température</div>
            <div class="crow"><span class="clbl">Soufflage</span><span class="cval">{{ consigne }} °C</span></div>
            <input type="range" class="cslider" min="14" max="28" v-model.number="consigne">
            <div class="crow"><span class="clbl">T° actuelle</span><span class="cval c-warn">{{ tSouf }}</span></div>
          </div>
          <div class="ccard">
            <div class="ccard-title">Ventilateurs</div>
            <div class="crow"><span class="clbl">Soufflage</span><span class="cval">{{ ventSouf }} %</span></div>
            <input type="range" class="cslider oj" min="0" max="100" v-model.number="ventSouf">
            <div class="crow"><span class="clbl">Reprise</span><span class="cval">{{ ventRep }} %</span></div>
            <input type="range" class="cslider" min="0" max="100" v-model.number="ventRep">
            <div class="crow"><span class="clbl">Débit soufflage</span><span class="cval">{{ debit }}</span></div>
          </div>
          <div class="ccard">
            <div class="ccard-title">Batterie froide</div>
            <div class="crow"><span class="clbl">Vanne</span><span class="cval">{{ battFroid }} %</span></div>
            <input type="range" class="cslider gn" min="0" max="100" v-model.number="battFroid">
            <div class="crow">
              <span class="clbl">Auto</span>
              <label class="tog"><input type="checkbox" v-model="toggles.bfa"><span class="tog-tr"></span><span class="tog-th"></span></label>
              <span class="tstate" :class="toggles.bfa ? 'on' : 'off'">{{ toggles.bfa ? 'AUTO' : 'MANUEL' }}</span>
            </div>
          </div>
          <div class="ccard">
            <div class="ccard-title">Options</div>
            <div class="crow">
              <span class="clbl">Bypass récup.</span>
              <label class="tog"><input type="checkbox" v-model="toggles.byp"><span class="tog-tr"></span><span class="tog-th"></span></label>
              <span class="tstate" :class="toggles.byp ? 'warn' : 'off'">{{ toggles.byp ? 'ON' : 'OFF' }}</span>
            </div>
            <div class="crow">
              <span class="clbl">Dégivrage forcé</span>
              <label class="tog"><input type="checkbox" v-model="toggles.dg"><span class="tog-tr"></span><span class="tog-th"></span></label>
              <span class="tstate" :class="toggles.dg ? 'warn' : 'off'">{{ toggles.dg ? 'ON' : 'OFF' }}</span>
            </div>
          </div>
          <div class="ccard" style="grid-column: span 2;">
            <div class="ccard-title">Actions</div>
            <div style="display:flex;gap:12px">
              <button class="cbtn info" @click="doReset">↺ Réinitialiser</button>
              <button class="cbtn err" @click="urgStop">⚠ Arrêt urgence</button>
            </div>
          </div>
        </div>

        <!-- ═══ ALARMES ═══ -->
        <div v-show="activePage === 'alm'" class="cta-page alm-page">
          <div class="alm-card">
            <div class="alm-head"><span>Alarmes actives</span><span class="c-warn">1</span></div>
            <div class="alm-row">
              <div class="alm-sev" style="background:var(--warn)"></div>
              <div class="alm-info">
                <div class="alm-time">{{ currentTime }}</div>
                <div class="alm-msg">Filtre M5 encrassé — Δp 54 Pa (seuil 80 Pa)</div>
              </div>
              <button class="alm-btn" @click="toast('Alarme acquittée ✔')">Acquitter</button>
            </div>
          </div>
          <div class="alm-card">
            <div class="alm-head"><span>Historique (24h)</span><span class="c-muted">2</span></div>
            <div class="alm-row" style="opacity:.5">
              <div class="alm-sev" style="background:var(--ok)"></div>
              <div class="alm-info">
                <div class="alm-time">Hier 16:42</div>
                <div class="alm-msg">Défaut communication bus — résolu automatiquement</div>
              </div>
              <span class="alm-ack">ACK</span>
            </div>
            <div class="alm-row" style="opacity:.5">
              <div class="alm-sev" style="background:var(--ok)"></div>
              <div class="alm-info">
                <div class="alm-time">Hier 08:15</div>
                <div class="alm-msg">Démarrage à froid — préchauffage terminé</div>
              </div>
              <span class="alm-ack">ACK</span>
            </div>
          </div>
        </div>

        <!-- ═══ PLANNING ═══ -->
        <div v-show="activePage === 'plan'" class="cta-page plan-page">
          <div style="display:flex;align-items:center;gap:12px;flex-wrap:wrap">
            <div class="plan-legend">
              <span style="display:flex;align-items:center;gap:6px"><span class="plan-dot" style="background:var(--info)"></span>Normal</span>
              <span style="display:flex;align-items:center;gap:6px"><span class="plan-dot" style="background:var(--ok)"></span>Éco</span>
              <span style="display:flex;align-items:center;gap:6px"><span class="plan-dot" style="background:var(--surf3);border:1px solid var(--border2)"></span>Arrêt</span>
            </div>
            <button class="cbtn ok" style="margin-left:auto;width:auto;padding:0 20px;height:44px" @click="toast('Planning sauvegardé ✔')">Sauvegarder</button>
          </div>
          <div class="wgrid">
            <div class="wc hdr"></div>
            <div v-for="d in days" :key="d" class="wc hdr">{{ d }}</div>
            <template v-for="h in hours" :key="h">
              <div class="wc time">{{ String(h).padStart(2, '0') }}h</div>
              <div v-for="(day, dIdx) in days" :key="dIdx + '-' + h" class="wc slot" :class="slotClasses[planSlots[dIdx]?.[h] ?? 0]" @click="toggleSlot(dIdx, h)">
                {{ slotLabels[planSlots[dIdx]?.[h] ?? 0] }}
              </div>
            </template>
          </div>
        </div>
      </div>

      <!-- TAB BAR -->
      <div class="cta-tabbar">
        <button class="tab" :class="{ active: activePage === 'syn' }" @click="goPage('syn')"><span class="tab-icon">⌂</span>Synoptique</button>
        <button class="tab" :class="{ active: activePage === 'mes' }" @click="goPage('mes')"><span class="tab-icon">📊</span>Mesures</button>
        <button class="tab" :class="{ active: activePage === 'ctrl' }" @click="goPage('ctrl')"><span class="tab-icon">🎛</span>Contrôle</button>
        <button class="tab" :class="{ active: activePage === 'alm' }" @click="goPage('alm')" style="position:relative"><span class="tab-icon">🔔</span>Alarmes<div class="tab-badge">1</div></button>
        <button class="tab" :class="{ active: activePage === 'plan' }" @click="goPage('plan')"><span class="tab-icon">📅</span>Planning</button>
      </div>

      <!-- MODAL COMPOSANT -->
      <div class="cta-modal-overlay" :class="{ open: modalOpen }" @click.self="closeModal">
        <div class="cta-modal">
          <div class="mhdr"><span class="mhdr-t">{{ modalTitle }}</span><button class="mclose" @click="closeModal">✕</button></div>
          <div class="mbody">
            <div v-for="(row, idx) in modalRows" :key="idx" class="mrow">
              <span class="mk">{{ row[0] }}</span>
              <span class="mv" :class="row[2] || ''">{{ row[1] }}</span>
            </div>
          </div>
        </div>
      </div>

      <!-- TOAST -->
      <div class="cta-toast" :class="{ show: toastVisible }" :style="{ borderColor: toastType === 'ok' ? 'var(--ok)' : toastType === 'warn' ? 'var(--warn)' : 'var(--err)', color: toastType === 'ok' ? 'var(--ok)' : toastType === 'warn' ? 'var(--warn)' : 'var(--err)' }">{{ toastMsg }}</div>
    </div>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700&family=JetBrains+Mono:wght@400;600&display=swap');

.cta-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,.85);
  z-index: 1000;
  display: flex;
  align-items: center;
  justify-content: center;
}

.cta-view {
  --bg: #0d1117;
  --surf1: #161b24;
  --surf2: #1c2333;
  --surf3: #242e42;
  --border: rgba(255,255,255,.08);
  --border2: rgba(255,255,255,.14);
  --text: #e6edf3;
  --muted: #7d8fa8;
  --dim: #3d4d60;
  --ok: #3fb950;
  --warn: #d29922;
  --err: #f85149;
  --info: #58a6ff;
  --tap: 52px;
  --tap-sm: 44px;
  --r: 10px;
  --r-sm: 7px;

  background: var(--bg);
  width: 100%;
  max-width: 1200px;
  height: 90vh;
  max-height: 800px;
  border-radius: 16px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  font-family: 'DM Sans', sans-serif;
  color: var(--text);
  user-select: none;
  box-shadow: 0 25px 80px rgba(0,0,0,.6);
}

/* HEADER */
.cta-hdr {
  background: var(--surf1);
  border-bottom: 1px solid var(--border);
  height: 56px;
  display: flex;
  align-items: center;
  padding: 0 20px;
  gap: 16px;
  flex-shrink: 0;
}

.cta-back {
  background: var(--surf2);
  border: 1px solid var(--border2);
  color: var(--muted);
  padding: 8px 14px;
  border-radius: 8px;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  transition: all .15s;
}
.cta-back:hover { color: var(--text); border-color: var(--info); }

.cta-hdr-logo {
  width: 36px; height: 36px;
  background: var(--info);
  border-radius: 8px;
  display: flex; align-items: center; justify-content: center;
  font-size: 18px; font-weight: 700; color: white;
}
.cta-hdr-info { display: flex; flex-direction: column; }
.cta-hdr-name { font-size: 15px; font-weight: 700; }
.cta-hdr-sub { font-size: 12px; color: var(--muted); }
.cta-hdr-right { margin-left: auto; display: flex; align-items: center; gap: 12px; }
.cta-hdr-clock { font-family: 'JetBrains Mono', monospace; font-size: 14px; color: var(--muted); }

.status-pill {
  display: flex; align-items: center; gap: 7px;
  background: var(--surf2); border: 1px solid var(--border2);
  border-radius: 20px; padding: 6px 14px;
  font-size: 13px; font-weight: 600;
}
.status-dot { width: 8px; height: 8px; border-radius: 50%; }
.dot-ok { background: var(--ok); box-shadow: 0 0 6px var(--ok); }
.dot-err { background: var(--err); box-shadow: 0 0 6px var(--err); }

/* BODY */
.cta-body { flex: 1; overflow: hidden; display: flex; flex-direction: column; min-height: 0; }
.cta-page { flex: 1; display: flex; flex-direction: column; overflow: hidden; min-height: 0; }

/* KPI STRIP */
.kpi-strip {
  display: flex;
  background: var(--surf1);
  border-bottom: 1px solid var(--border);
  flex-shrink: 0;
}
.kpi-cell {
  flex: 1;
  padding: 8px 14px;
  border-right: 1px solid var(--border);
  display: flex; flex-direction: column; gap: 2px;
}
.kpi-cell:last-child { border-right: none; }
.kpi-lbl { font-size: 9px; font-weight: 600; color: var(--muted); text-transform: uppercase; letter-spacing: .07em; }
.kpi-val { font-family: 'JetBrains Mono', monospace; font-size: 17px; font-weight: 600; }

/* SYNOPTIQUE */
.syn-page { flex: 1; display: flex; flex-direction: column; overflow: hidden; }
.syn-svg-wrap { flex: 1; overflow: hidden; background: var(--surf2); min-height: 0; }
.syn-svg { width: 100%; height: 100%; display: block; }
.clk { cursor: pointer; }
.clk:active rect, .clk:active circle { opacity: .75; }

/* MESURES */
.mes-page {
  flex: 1; overflow-y: auto; padding: 16px;
  background: var(--bg);
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 12px;
  align-content: start;
}
.mes-card { background: var(--surf1); border: 1px solid var(--border); border-radius: var(--r); overflow: hidden; }
.mes-head { padding: 10px 14px; background: var(--surf2); border-bottom: 1px solid var(--border); font-size: 11px; font-weight: 700; color: var(--muted); text-transform: uppercase; letter-spacing: .08em; }
.mes-row { display: flex; align-items: center; justify-content: space-between; padding: 12px 16px; border-bottom: 1px solid var(--border); min-height: var(--tap-sm); }
.mes-row:last-child { border-bottom: none; }
.mes-lbl { font-size: 14px; color: var(--muted); }
.mes-val { font-family: 'JetBrains Mono', monospace; font-size: 18px; font-weight: 600; }
.mes-unit { font-size: 11px; color: var(--muted); margin-left: 4px; }

/* CONTRÔLE */
.ctrl-page {
  flex: 1; overflow-y: auto; padding: 16px;
  background: var(--bg);
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 12px;
  align-content: start;
}
.ccard { background: var(--surf1); border: 1px solid var(--border); border-radius: var(--r); padding: 16px; display: flex; flex-direction: column; gap: 10px; }
.ccard-title { font-size: 11px; font-weight: 700; color: var(--muted); text-transform: uppercase; letter-spacing: .09em; padding-bottom: 10px; border-bottom: 1px solid var(--border); }
.crow { display: flex; align-items: center; justify-content: space-between; gap: 10px; min-height: var(--tap-sm); }
.clbl { font-size: 14px; color: var(--text); flex: 1; }
.cval { font-family: 'JetBrains Mono', monospace; font-size: 15px; font-weight: 600; min-width: 60px; text-align: right; }

.cslider { -webkit-appearance: none; appearance: none; width: 100%; height: 6px; border-radius: 3px; background: var(--surf3); outline: none; cursor: pointer; }
.cslider::-webkit-slider-thumb { -webkit-appearance: none; width: 26px; height: 26px; border-radius: 50%; background: var(--info); border: 3px solid var(--surf1); cursor: pointer; box-shadow: 0 0 8px rgba(88,166,255,.4); }
.cslider.oj::-webkit-slider-thumb { background: var(--warn); box-shadow: 0 0 8px rgba(210,153,34,.4); }
.cslider.gn::-webkit-slider-thumb { background: var(--ok); box-shadow: 0 0 8px rgba(63,185,80,.4); }

.tog { position: relative; width: 52px; height: 30px; cursor: pointer; flex-shrink: 0; }
.tog input { opacity: 0; width: 0; height: 0; position: absolute; }
.tog-tr { position: absolute; inset: 0; background: var(--surf3); border-radius: 15px; transition: background .2s; }
.tog input:checked + .tog-tr { background: var(--ok); }
.tog-th { position: absolute; top: 3px; left: 3px; width: 24px; height: 24px; background: var(--muted); border-radius: 50%; transition: transform .2s, background .2s; box-shadow: 0 2px 4px rgba(0,0,0,.4); }
.tog input:checked ~ .tog-th { transform: translateX(22px); background: white; }
.tstate { font-family: 'JetBrains Mono', monospace; font-size: 13px; font-weight: 600; min-width: 52px; }
.tstate.on { color: var(--ok); }
.tstate.off { color: var(--dim); }
.tstate.warn { color: var(--warn); }

.msel { display: grid; grid-template-columns: repeat(3, 1fr); gap: 6px; }
.mopt { height: var(--tap); border-radius: var(--r-sm); border: 1px solid var(--border2); background: var(--surf2); color: var(--muted); font-size: 13px; font-weight: 600; cursor: pointer; transition: all .12s; }
.mopt:active { transform: scale(.97); }
.mopt.sel { background: rgba(88,166,255,.15); border-color: var(--info); color: var(--info); }

.cbtn { height: var(--tap); border-radius: var(--r-sm); border: 1px solid var(--border2); background: var(--surf2); color: var(--muted); font-size: 14px; font-weight: 600; cursor: pointer; width: 100%; display: flex; align-items: center; justify-content: center; gap: 6px; }
.cbtn:active { transform: scale(.98); }
.cbtn.ok { background: rgba(63,185,80,.1); border-color: rgba(63,185,80,.3); color: var(--ok); }
.cbtn.err { background: rgba(248,81,73,.1); border-color: rgba(248,81,73,.3); color: var(--err); }
.cbtn.info { background: rgba(88,166,255,.1); border-color: rgba(88,166,255,.3); color: var(--info); }

/* ALARMES */
.alm-page { flex: 1; overflow-y: auto; padding: 16px; background: var(--bg); display: flex; flex-direction: column; gap: 12px; }
.alm-card { background: var(--surf1); border: 1px solid var(--border); border-radius: var(--r); overflow: hidden; }
.alm-head { padding: 12px 16px; background: var(--surf2); border-bottom: 1px solid var(--border); display: flex; align-items: center; justify-content: space-between; font-size: 13px; font-weight: 700; }
.alm-row { display: flex; align-items: center; gap: 14px; padding: 14px 16px; border-bottom: 1px solid var(--border); min-height: var(--tap); }
.alm-row:last-child { border-bottom: none; }
.alm-sev { width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0; }
.alm-info { flex: 1; }
.alm-time { font-family: 'JetBrains Mono', monospace; font-size: 11px; color: var(--muted); margin-bottom: 2px; }
.alm-msg { font-size: 14px; color: var(--text); }
.alm-btn { height: var(--tap-sm); padding: 0 16px; border-radius: var(--r-sm); border: 1px solid var(--border2); background: var(--surf2); color: var(--muted); font-size: 13px; font-weight: 600; cursor: pointer; }
.alm-btn:hover { border-color: var(--ok); color: var(--ok); }
.alm-ack { font-size: 11px; color: var(--ok); font-family: 'JetBrains Mono', monospace; font-weight: 600; }

/* PLANNING */
.plan-page { flex: 1; display: flex; flex-direction: column; overflow: hidden; padding: 16px; gap: 12px; background: var(--bg); }
.plan-legend { display: flex; gap: 16px; align-items: center; font-size: 12px; color: var(--muted); }
.plan-dot { width: 10px; height: 10px; border-radius: 2px; }
.wgrid { display: grid; grid-template-columns: 46px repeat(7, 1fr); gap: 1px; background: var(--border); border: 1px solid var(--border); border-radius: var(--r); overflow: hidden; overflow-y: auto; flex: 1; }
.wc { background: var(--surf1); padding: 6px 2px; font-size: 11px; text-align: center; color: var(--muted); display: flex; align-items: center; justify-content: center; }
.wc.hdr { background: var(--surf2); font-weight: 700; color: var(--text); font-size: 12px; }
.wc.time { font-family: 'JetBrains Mono', monospace; font-size: 10px; text-align: right; padding-right: 6px; }
.wc.slot { cursor: pointer; min-height: 28px; font-size: 12px; font-weight: 600; }
.wc.slot:active { opacity: .7; }
.wc.slot.on { background: rgba(88,166,255,.15); color: var(--info); }
.wc.slot.eco { background: rgba(63,185,80,.12); color: var(--ok); }

/* TAB BAR */
.cta-tabbar { background: var(--surf1); border-top: 1px solid var(--border); display: flex; flex-shrink: 0; height: 64px; }
.tab { flex: 1; display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 3px; cursor: pointer; color: var(--dim); font-size: 10px; font-weight: 600; letter-spacing: .04em; text-transform: uppercase; border: none; background: transparent; position: relative; transition: color .15s; }
.tab:active { background: rgba(255,255,255,.04); }
.tab.active { color: var(--info); }
.tab.active::after { content: ''; position: absolute; top: 0; left: 20%; right: 20%; height: 2px; background: var(--info); border-radius: 0 0 2px 2px; }
.tab-icon { font-size: 22px; line-height: 1; }
.tab-badge { position: absolute; top: 8px; right: calc(50% - 18px); background: var(--err); color: white; font-size: 9px; font-weight: 700; width: 16px; height: 16px; border-radius: 50%; display: flex; align-items: center; justify-content: center; border: 2px solid var(--surf1); }

/* MODAL COMPOSANT */
.cta-modal-overlay { display: none; position: absolute; inset: 0; background: rgba(0,0,0,.6); z-index: 100; align-items: center; justify-content: center; padding: 20px; }
.cta-modal-overlay.open { display: flex; }
.cta-modal { background: var(--surf1); border: 1px solid var(--border2); border-radius: 14px; width: 100%; max-width: 420px; overflow: hidden; box-shadow: 0 20px 60px rgba(0,0,0,.6); }
.mhdr { background: var(--surf2); padding: 16px 20px; display: flex; align-items: center; justify-content: space-between; border-bottom: 1px solid var(--border); }
.mhdr-t { font-size: 14px; font-weight: 700; letter-spacing: .03em; }
.mclose { width: var(--tap-sm); height: var(--tap-sm); background: var(--surf3); border: none; border-radius: 8px; color: var(--muted); cursor: pointer; font-size: 18px; display: flex; align-items: center; justify-content: center; }
.mbody { padding: 8px 0; }
.mrow { display: flex; justify-content: space-between; align-items: center; padding: 13px 20px; border-bottom: 1px solid var(--border); min-height: var(--tap-sm); }
.mrow:last-child { border-bottom: none; }
.mk { font-size: 14px; color: var(--muted); }
.mv { font-family: 'JetBrains Mono', monospace; font-size: 14px; font-weight: 600; }
.mv.ok { color: var(--ok); }
.mv.warn { color: var(--warn); }
.mv.off { color: var(--dim); }

/* TOAST */
.cta-toast { position: absolute; bottom: 80px; left: 50%; transform: translateX(-50%) translateY(30px); background: var(--surf2); border: 1px solid var(--ok); color: var(--ok); padding: 10px 24px; border-radius: 30px; font-size: 13px; font-weight: 600; z-index: 200; transition: transform .25s, opacity .25s; opacity: 0; pointer-events: none; white-space: nowrap; }
.cta-toast.show { transform: translateX(-50%) translateY(0); opacity: 1; }

/* COULEURS */
.c-ok { color: var(--ok); }
.c-warn { color: var(--warn); }
.c-err { color: var(--err); }
.c-info { color: var(--info); }
.c-muted { color: var(--muted); }

::-webkit-scrollbar { width: 3px; height: 3px; }
::-webkit-scrollbar-thumb { background: var(--surf3); border-radius: 2px; }
</style>