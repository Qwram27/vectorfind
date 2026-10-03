<script setup>
import { ref, shallowRef, computed, onMounted, onUnmounted } from 'vue'

// --- STAN I REAKTYWNOŚĆ ---
const canvasRef = ref(null)
const searchQuery = ref('')
const selectedId = ref(null)
const rangeNm = ref(40) // Zasięg w milach morskich
const audioEnabled = ref(false)
const tcasAlertsCount = ref(0)

// Nierelatywny kontener danych fizycznych (wysoka wydajność, brak narzutu proxy Vue)
const aircraftMap = new Map()

// Reaktywna lista tylko dla listy bocznej i wyszukiwarki
const aircraftDisplayList = shallowRef([])

// Predefiniowane cele początkowe (lotnisko EPWA / Warszawa Approach)
const initialAircraft = [
  { id: 'LOT3801', callsign: 'LOT3801', x: 50, y: -70, alt: 11000, spd: 260, hdg: 210, squawk: '1000' },
  { id: 'RYR44P',  callsign: 'RYR44P',  x: -90, y: 30,  alt: 5500,  spd: 210, hdg: 80,  squawk: '4212' },
  { id: 'DLH1340', callsign: 'DLH1340', x: -30, y: -50, alt: 3200,  spd: 160, hdg: 115, squawk: '7000' },
  { id: 'WZZ281',  callsign: 'WZZ281',  x: 110, y: 60,   alt: 21000, spd: 390, hdg: 295, squawk: '2351' },
  { id: 'AFR1145', callsign: 'AFR1145', x: 15,  y: 25,   alt: 3100,  spd: 150, hdg: 290, squawk: '7700' }, // Squawk ratunkowy
  { id: 'BAW852',  callsign: 'BAW852',  x: -60, y: -100, alt: 18000, spd: 340, hdg: 145, squawk: '3604' }
]

// Inicjalizacja danych
initialAircraft.forEach(ac => {
  aircraftMap.set(ac.id, { ...ac, alert: false })
})

const updateDisplayList = () => {
  aircraftDisplayList.value = Array.from(aircraftMap.values())
}

// Filtrowane cele na podstawie wyszukiwarki
const filteredAircraft = computed(() => {
  const q = searchQuery.value.trim().toUpperCase()
  if (!q) return aircraftDisplayList.value
  return aircraftDisplayList.value.filter(ac => 
    ac.callsign.includes(q) || ac.squawk.includes(q) || ac.alt.toString().includes(q)
  )
})

const selectedAircraft = computed(() => {
  return aircraftDisplayList.value.find(ac => ac.id === selectedId.value) || null
})

// --- SYNTEZA DŹWIĘKU (Web Audio API - zero plików zewnętrznych) ---
let audioCtx = null

const initAudio = () => {
  if (!audioCtx) {
    audioCtx = new (window.AudioContext || window.webkitAudioContext)()
  }
  if (audioCtx.state === 'suspended') {
    audioCtx.resume()
  }
  audioEnabled.value = true
}

const playBeep = (freq = 880, duration = 0.08, type = 'sine') => {
  if (!audioEnabled.value || !audioCtx) return
  try {
    const osc = audioCtx.createOscillator()
    const gain = audioCtx.createGain()
    osc.type = type
    osc.frequency.setValueAtTime(freq, audioCtx.currentTime)
    gain.gain.setValueAtTime(0.04, audioCtx.currentTime)
    gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + duration)
    osc.connect(gain)
    gain.connect(audioCtx.destination)
    osc.start()
    osc.stop(audioCtx.currentTime + duration)
  } catch {
    // Bezpieczne wyciszenie błędów audio
  }
}

// --- SILNIK RENDEROWANIA & DELTA TIME ---
let animId = null
let lastTime = performance.now()
let sweepAngle = 0 // Stopnie

const updatePhysics = (dt) => {
  // Prędkość obrotu radaru: 24 RPM = 144 stopnie na sekundę
  const prevAngle = sweepAngle
  sweepAngle = (sweepAngle + 120 * dt) % 360

  let currentAlerts = 0
  const targets = Array.from(aircraftMap.values())

  // Sprawdzanie kolizji TCAS (dystans < 3.5 NM oraz różnica wysokości < 900 ft)
  for (let i = 0; i < targets.length; i++) {
    targets[i].alert = false
  }

  for (let i = 0; i < targets.length; i++) {
    for (let j = i + 1; j < targets.length; j++) {
      const a = targets[i]
      const b = targets[j]
      const dist = Math.hypot(a.x - b.x, a.y - b.y) * (rangeNm.value / 200)
      const altDiff = Math.abs(a.alt - b.alt)

      if (dist < 3.5 && altDiff < 900) {
        a.alert = true
        b.alert = true
        currentAlerts++
      }
    }
  }

  // Aktualizacja pozycji samolotów
  targets.forEach(ac => {
    const rad = (ac.hdg - 90) * (Math.PI / 180)
    // Przeliczenie prędkości w węzłach na NM/s
    const speedScale = (ac.spd / 3600) * dt * 8
    ac.x += Math.cos(rad) * speedScale
    ac.y += Math.sin(rad) * speedScale

    // Zawracanie po wyjściu z zasięgu
    if (Math.hypot(ac.x, ac.y) > 180) {
      ac.hdg = (ac.hdg + 180) % 360
    }

    // Wykrywanie omiatania anteny radaru (ping dźwiękowy)
    const targetAngle = ((Math.atan2(ac.y, ac.x) * 180 / Math.PI) + 450) % 360
    if (prevAngle < targetAngle && sweepAngle >= targetAngle) {
      if (ac.alert || ac.squawk === '7700') {
        playBeep(440, 0.15, 'sawtooth')
      } else {
        playBeep(980, 0.04, 'sine')
      }
    }
  })

  tcasAlertsCount.value = currentAlerts
}

const renderCanvas = () => {
  const canvas = canvasRef.value
  if (!canvas) return
  const ctx = canvas.getContext('2d')
  const width = canvas.width
  const height = canvas.height
  const cx = width / 2
  const cy = height / 2
  const maxRadius = Math.min(cx, cy) - 20

  // 1. Zanikający ślad luminoforu CRT (Phosphor Trail)
  ctx.fillStyle = 'rgba(2, 8, 4, 0.18)'
  ctx.fillRect(0, 0, width, height)

  // 2. Pierścienie dystansowe i siatka azymutalna
  ctx.lineWidth = 1
  ctx.strokeStyle = 'rgba(0, 255, 120, 0.15)'
  for (let r = 1; r <= 4; r++) {
    const radius = (maxRadius / 4) * r
    ctx.beginPath()
    ctx.arc(cx, cy, radius, 0, Math.PI * 2)
    ctx.stroke()

    // Oznaczenia dystansowe NM
    ctx.fillStyle = 'rgba(0, 255, 120, 0.4)'
    ctx.font = '10px monospace'
    ctx.fillText(`${(rangeNm.value / 4) * r}NM`, cx + 6, cy - radius + 12)
  }

  // Celownik środkowy
  ctx.beginPath()
  ctx.moveTo(cx, cy - maxRadius); ctx.lineTo(cx, cy + maxRadius)
  ctx.moveTo(cx - maxRadius, cy); ctx.lineTo(cx + maxRadius, cy)
  ctx.stroke()

  // 3. Wiązka radaru (Conic Sweep Gradient)
  const sweepRad = (sweepAngle - 90) * (Math.PI / 180)
  const grad = ctx.createConicGradient(sweepRad, cx, cy)
  grad.addColorStop(0, 'rgba(0, 255, 120, 0.35)')
  grad.addColorStop(0.1, 'rgba(0, 255, 120, 0.0)')
  grad.addColorStop(1, 'rgba(0, 255, 120, 0.0)')

  ctx.fillStyle = grad
  ctx.beginPath()
  ctx.arc(cx, cy, maxRadius, 0, Math.PI * 2)
  ctx.fill()

  // 4. Renderowanie samolotów
  const scale = maxRadius / 180
  const targets = Array.from(aircraftMap.values())

  targets.forEach(ac => {
    const sx = cx + ac.x * scale
    const sy = cy + ac.y * scale
    const isSelected = selectedId.value === ac.id
    const isEmerg = ac.squawk === '7700'
    const isWarn = ac.alert

    // Kolor w zależności od stanu
    let color = '#00ff78'
    if (isEmerg || isWarn) color = '#ff3344'
    else if (isSelected) color = '#ffffff'

    // Blip radarowy
    ctx.fillStyle = color
    ctx.fillRect(sx - 2.5, sy - 2.5, 5, 5)

    // Wektor prędkości (przewidywanie pozycji za 60 sek.)
    const rad = (ac.hdg - 90) * (Math.PI / 180)
    const vLen = (ac.spd / 12) * scale
    ctx.strokeStyle = color
    ctx.lineWidth = isSelected ? 1.5 : 1
    ctx.beginPath()
    ctx.moveTo(sx, sy)
    ctx.lineTo(sx + Math.cos(rad) * vLen, sy + Math.sin(rad) * vLen)
    ctx.stroke()

    // Etykieta Mode-S (Callsign, Wysokość, Prędkość)
    ctx.fillStyle = color
    ctx.font = '10px monospace'
    const fl = `FL${Math.round(ac.alt / 100).toString().padStart(3, '0')}`
    ctx.fillText(ac.callsign, sx + 8, sy - 5)
    ctx.fillText(`${fl} ${ac.spd}KTS`, sx + 8, sy + 6)

    if (isEmerg) {
      ctx.fillText('!EMERG 7700!', sx + 8, sy + 17)
    } else if (isWarn) {
      ctx.fillText('!TCAS ALERT!', sx + 8, sy + 17)
    }

    // Zaznaczenie wybranego celu
    if (isSelected) {
      ctx.strokeStyle = '#ffffff'
      ctx.lineWidth = 1
      ctx.beginPath()
      ctx.arc(sx, sy, 9, 0, Math.PI * 2)
      ctx.stroke()
    }
  })
}

// Główna pętla renderująca (60+ FPS)
const tick = (now) => {
  const dt = Math.min((now - lastTime) / 1000, 0.1) // Zabezpieczenie przed skokami po uśpieniu karty
  lastTime = now

  updatePhysics(dt)
  renderCanvas()
  updateDisplayList()

  animId = requestAnimationFrame(tick)
}

// Interakcja z płótnem radaru
const handleCanvasClick = (e) => {
  initAudio()
  const canvas = canvasRef.value
  if (!canvas) return
  const rect = canvas.getBoundingClientRect()
  const clickX = (e.clientX - rect.left) * (canvas.width / rect.width)
  const clickY = (e.clientY - rect.top) * (canvas.height / rect.height)

  const cx = canvas.width / 2
  const cy = canvas.height / 2
  const scale = (Math.min(cx, cy) - 20) / 180

  const clicked = Array.from(aircraftMap.values()).find(ac => {
    const sx = cx + ac.x * scale
    const sy = cy + ac.y * scale
    return Math.hypot(clickX - sx, clickY - sy) < 18
  })

  selectedId.value = clicked ? clicked.id : null
}

const selectAircraft = (id) => {
  initAudio()
  selectedId.value = id
}

// Skalowanie pod gęstość pikseli matrycy (HiDPI)
const setupCanvasDpi = () => {
  const canvas = canvasRef.value
  if (!canvas) return
  const dpr = window.devicePixelRatio || 1
  const displaySize = 640

  canvas.width = displaySize * dpr
  canvas.height = displaySize * dpr
  canvas.style.width = `${displaySize}px`
  canvas.style.height = `${displaySize}px`

  const ctx = canvas.getContext('2d')
  ctx.scale(dpr, dpr)
  canvas.width = displaySize
  canvas.height = displaySize
}

onMounted(() => {
  setupCanvasDpi()
  lastTime = performance.now()
  animId = requestAnimationFrame(tick)
})

onUnmounted(() => {
  if (animId) cancelAnimationFrame(animId)
  if (audioCtx) audioCtx.close()
})
</script>

<template>
  <div class="vf-root">
    <!-- Pasek statusu ATC -->
    <header class="vf-header">
      <div class="brand">
        <span class="badge">TACTICAL SYSTEM</span>
        <h1>VECTORFIND // ATC PPI RADAR</h1>
      </div>

      <div class="controls">
        <button 
          class="audio-toggle" 
          :class="{ active: audioEnabled }"
          @click="initAudio"
        >
          🔊 DŹWIĘK: {{ audioEnabled ? 'WŁĄCZONY' : 'WYCISZONY (KLIKNIJ)' }}
        </button>

        <div class="stat-box" :class="{ alert: tcasAlertsCount > 0 }">
          TCAS ALERTY: <strong>{{ tcasAlertsCount }}</strong>
        </div>
      </div>
    </header>

    <!-- Główny interfejs operacyjny -->
    <main class="vf-workspace">
      <!-- Radar Scope -->
      <section class="radar-box">
        <canvas
          ref="canvasRef"
          class="radar-screen"
          @click="handleCanvasClick"
        ></canvas>
        <div class="scope-legend">ZASIĘG RADARU: {{ rangeNm }} NM | OBRÓT: 24 RPM</div>
      </section>

      <!-- Panel boczny: Wyszukiwarka VectorFind i Telemetria -->
      <aside class="vf-sidebar">
        <!-- Wyszukiwarka celów -->
        <div class="search-section">
          <label class="section-label">SZUKAJ CELU (CALLSIGN / SQUAWK / FL)</label>
          <input
            v-model="searchQuery"
            type="text"
            placeholder="np. LOT3801, 7700..."
            class="search-input"
          />
        </div>

        <!-- Lista namierzonych samolotów -->
        <div class="target-list">
          <div
            v-for="ac in filteredAircraft"
            :key="ac.id"
            class="target-item"
            :class="{ 
              active: selectedId === ac.id, 
              danger: ac.squawk === '7700' || ac.alert 
            }"
            @click="selectAircraft(ac.id)"
          >
            <div class="ti-main">
              <span class="ti-callsign">{{ ac.callsign }}</span>
              <span class="ti-squawk">SQ: {{ ac.squawk }}</span>
            </div>
            <div class="ti-sub">
              <span>FL{{ Math.round(ac.alt / 100) }}</span>
              <span>{{ ac.spd }} KTS</span>
              <span>HDG {{ ac.hdg }}°</span>
            </div>
          </div>

          <div v-if="filteredAircraft.length === 0" class="empty-state">
            Brak samolotów spełniających kryteria.
          </div>
        </div>

        <!-- Szczegółowa telemetria wybranego celu -->
        <div v-if="selectedAircraft" class="telemetry-box">
          <div class="section-label">TELEMETRIA CELU: {{ selectedAircraft.callsign }}</div>
          <div class="telemetry-grid">
            <div>Wysokość:</div><div>{{ selectedAircraft.alt.toLocaleString() }} FT</div>
            <div>Prędkość (GS):</div><div>{{ selectedAircraft.spd }} WĘZŁÓW</div>
            <div>Kurs wektorowy:</div><div>{{ selectedAircraft.hdg }}°</div>
            <div>Pozycja:</div><div>{{ selectedAircraft.x.toFixed(1) }}X / {{ selectedAircraft.y.toFixed(1) }}Y</div>
            <div>Status:</div>
            <div :style="{ color: selectedAircraft.alert ? '#ff3344' : '#00ff78' }">
              {{ selectedAircraft.alert ? 'KOLIZJA TCAS' : 'SEPARACJA PRAWIDŁOWA' }}
            </div>
          </div>
          <button class="clear-btn" @click="selectedId = null">ODZNACZ CEL</button>
        </div>
      </aside>
    </main>
  </div>
</template>

<style scoped>
.vf-root {
  min-height: 100vh;
  background-color: #020704;
  color: #00ff78;
  font-family: 'JetBrains Mono', 'Consolas', 'Courier New', monospace;
  display: flex;
  flex-direction: column;
  padding: 18px;
  box-sizing: border-box;
}

.vf-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid rgba(0, 255, 120, 0.25);
  padding-bottom: 14px;
  margin-bottom: 18px;
}

.brand h1 {
  font-size: 19px;
  margin: 4px 0 0 0;
  letter-spacing: 1.5px;
}

.badge {
  background: rgba(0, 255, 120, 0.15);
  border: 1px solid #00ff78;
  font-size: 10px;
  padding: 2px 6px;
  font-weight: bold;
}

.controls {
  display: flex;
  gap: 16px;
  align-items: center;
}

.audio-toggle {
  background: transparent;
  border: 1px solid rgba(0, 255, 120, 0.4);
  color: #00ff78;
  font-family: inherit;
  font-size: 11px;
  padding: 6px 12px;
  cursor: pointer;
  transition: all 0.2s;
}

.audio-toggle.active {
  background: rgba(0, 255, 120, 0.2);
  border-color: #00ff78;
}

.stat-box {
  font-size: 12px;
  padding: 4px 8px;
  border: 1px solid rgba(0, 255, 120, 0.3);
}

.stat-box.alert {
  border-color: #ff3344;
  color: #ff3344;
  animation: flash 1s infinite;
}

@keyframes flash {
  50% { opacity: 0.4; }
}

.vf-workspace {
  display: flex;
  gap: 24px;
  justify-content: center;
  flex-wrap: wrap;
}

.radar-box {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.radar-screen {
  background: radial-gradient(circle, #051409 0%, #010603 100%);
  border: 2px solid rgba(0, 255, 120, 0.45);
  border-radius: 50%;
  box-shadow: 0 0 35px rgba(0, 255, 120, 0.15);
  cursor: crosshair;
}

.scope-legend {
  margin-top: 10px;
  font-size: 11px;
  opacity: 0.6;
}

.vf-sidebar {
  width: 320px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.section-label {
  font-size: 10px;
  letter-spacing: 1px;
  opacity: 0.7;
  display: block;
  margin-bottom: 6px;
}

.search-input {
  width: 100%;
  background: #041107;
  border: 1px solid rgba(0, 255, 120, 0.4);
  color: #00ff78;
  padding: 8px 10px;
  font-family: inherit;
  font-size: 13px;
  box-sizing: border-box;
  outline: none;
}

.search-input:focus {
  border-color: #00ff78;
  box-shadow: 0 0 8px rgba(0, 255, 120, 0.3);
}

.target-list {
  background: rgba(4, 17, 7, 0.7);
  border: 1px solid rgba(0, 255, 120, 0.2);
  max-height: 280px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
}

.target-item {
  padding: 8px 10px;
  border-bottom: 1px solid rgba(0, 255, 120, 0.1);
  cursor: pointer;
  transition: background 0.15s;
}

.target-item:hover {
  background: rgba(0, 255, 120, 0.1);
}

.target-item.active {
  background: rgba(0, 255, 120, 0.25);
  border-left: 3px solid #00ff78;
}

.target-item.danger {
  color: #ff3344;
  border-left-color: #ff3344;
}

.ti-main {
  display: flex;
  justify-content: space-between;
  font-weight: bold;
  font-size: 13px;
}

.ti-sub {
  display: flex;
  gap: 12px;
  font-size: 10px;
  opacity: 0.8;
  margin-top: 3px;
}

.empty-state {
  padding: 16px;
  font-size: 11px;
  text-align: center;
  opacity: 0.5;
}

.telemetry-box {
  background: #041208;
  border: 1px solid #00ff78;
  padding: 12px;
}

.telemetry-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  row-gap: 6px;
  font-size: 11px;
  margin-top: 8px;
}

.clear-btn {
  margin-top: 12px;
  width: 100%;
  background: transparent;
  border: 1px solid rgba(0, 255, 120, 0.4);
  color: #00ff78;
  padding: 6px;
  font-family: inherit;
  font-size: 11px;
  cursor: pointer;
}

.clear-btn:hover {
  background: rgba(0, 255, 120, 0.2);
}
</style>
