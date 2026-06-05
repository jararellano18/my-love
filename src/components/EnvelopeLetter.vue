<script setup lang="ts">
import { ref, nextTick } from 'vue'

// ── Letters ────────────────────────────────────────────────────────────────
interface Letter {
  greeting: string
  lines: string[]
  signature: string
}

const letters: Letter[] = [
  {
    greeting: 'My Love,',
    lines: [
      'Three months of you — and I would choose every single day again.',
      'Happy 3rd monthsary, my love. ♡',
    ],
    signature: 'Yours, always',
  },
  {
    greeting: 'To My Everything,',
    lines: [
      'You showed up and made ordinary days feel like something worth keeping.',
      'Happy 3rd monthsary. Here\'s to every month ahead. ♡',
    ],
    signature: 'With all my love',
  },
  {
    greeting: 'Dearest Joy,',
    lines: [
      'It has always felt right with you — from the very first day.',
      'Happy 3rd monthsary. Thank you for being exactly who you are. ♡',
    ],
    signature: 'Forever yours',
  },
  {
    greeting: 'My Dearest,',
    lines: [
      'Loving you feels less like falling and more like finally arriving home.',
      'Happy 3rd monthsary. I love you more than words can hold. ♡',
    ],
    signature: 'All of me is yours',
  },
  {
    greeting: 'To the One I Choose Every Day,',
    lines: [
      'Three months in, and all I feel is lucky — lucky that it gets to be you.',
      'Happy 3rd monthsary, my love. This is only the beginning. ♡',
    ],
    signature: 'Completely yours',
  },
]

let letter: Letter = letters[Math.floor(Math.random() * letters.length)] ?? letters[0]!

// ── Photo rain ─────────────────────────────────────────────────────────────
const photoModules = import.meta.glob<{ default: string }>('@/assets/images/*.jpg', { eager: true })
const photos: string[] = Object.values(photoModules).map((m) => m.default)

// Shown when no real photos are loaded yet
const placeholderColors = [
  '#ffd6d6', '#ffdeba', '#fff4ba', '#d4f4d4',
  '#d4e8ff', '#e8d4ff', '#ffc4c4', '#c4e8c4',
]

interface FallingPhoto {
  id: number
  src: string | null
  color: string
  size: number      // px
  // Confetti trajectory (all px, relative to startX/startY)
  startX: number    // fixed left of photo when launched
  startY: number    // fixed top of photo when launched
  txPeak: number    // x translation at arc peak
  tyPeak: number    // y translation at arc peak (negative = up)
  txEnd: number     // x translation at landing
  tyEnd: number     // y translation at landing
  rotMid: number    // rotation at peak (deg)
  rotEnd: number    // rotation at landing (deg)
  duration: number  // seconds
  delay: number     // seconds
  // Drag state — null = controlled by CSS animation
  posX: number | null
  posY: number | null
  isDragging: boolean
  // Leave animation
  isLeaving: boolean
  leaveDelay: number
}

const fallingPhotos = ref<FallingPhoto[]>([])
let spawnTimer: ReturnType<typeof setInterval> | null = null
let photoCounter = 0

// ── Drag ───────────────────────────────────────────────────────────────────
// Keyed by pointerId so multiple touches can drag different photos at once
const activeDrags = new Map<number, {
  photo: FallingPhoto
  startPointerX: number
  startPointerY: number
  startPhotoX: number
  startPhotoY: number
}>()

function getPhotoStyle(photo: FallingPhoto): Record<string, string> {
  // Static position (dragged or frozen for leave animation)
  if (photo.posX !== null && photo.posY !== null) {
    if (photo.isLeaving) {
      return {
        width: photo.size + 'px',
        left: photo.posX + 'px',
        top: photo.posY + 'px',
        opacity: '0',
        transform: `translateY(-80px) rotate(${photo.rotEnd}deg) scale(0.85)`,
        animation: 'none',
        transition: 'opacity 1.4s ease-out, transform 1.4s ease-out',
        transitionDelay: photo.leaveDelay + 's',
        zIndex: '200',
        pointerEvents: 'none',
        touchAction: 'none',
      }
    }
    return {
      width: photo.size + 'px',
      left: photo.posX + 'px',
      top: photo.posY + 'px',
      transform: `rotate(${photo.rotEnd}deg)`,
      animation: 'none',
      transition: 'none',
      opacity: '1',
      zIndex: photo.isDragging ? '200' : '6',
      cursor: photo.isDragging ? 'grabbing' : 'grab',
      touchAction: 'none',
    }
  }
  // In-flight via CSS animation (confetti trajectory)
  return {
    width: photo.size + 'px',
    left: photo.startX + 'px',
    top: photo.startY + 'px',
    '--tx-peak': photo.txPeak + 'px',
    '--ty-peak': photo.tyPeak + 'px',
    '--tx-end': photo.txEnd + 'px',
    '--ty-end': photo.tyEnd + 'px',
    '--rot-mid': photo.rotMid + 'deg',
    '--rot-end': photo.rotEnd + 'deg',
    '--dur': photo.duration + 's',
    '--delay': photo.delay + 's',
    cursor: 'grab',
    touchAction: 'none',
  }
}

function onPointerDown(event: PointerEvent, photo: FallingPhoto) {
  event.preventDefault()
  event.stopPropagation()
  if (!isOpen.value || photo.isLeaving || busy) return

  const el = event.currentTarget as HTMLElement

  let startPhotoX: number
  let startPhotoY: number

  if (photo.posX !== null && photo.posY !== null) {
    // Already in static position — reuse existing coords directly
    startPhotoX = photo.posX
    startPhotoY = photo.posY
  } else {
    // Still in CSS animation — extract pure translation from the computed matrix
    // (avoids bounding-rect distortion caused by the element's rotation)
    const matrix = new DOMMatrix(window.getComputedStyle(el).transform)
    startPhotoX = photo.startX + matrix.m41
    startPhotoY = photo.startY + matrix.m42
  }

  photo.posX = startPhotoX
  photo.posY = startPhotoY
  photo.isDragging = true

  el.setPointerCapture(event.pointerId)

  activeDrags.set(event.pointerId, {
    photo,
    startPointerX: event.clientX,
    startPointerY: event.clientY,
    startPhotoX,
    startPhotoY,
  })
}

function onPointerMove(event: PointerEvent) {
  const drag = activeDrags.get(event.pointerId)
  if (!drag) return
  drag.photo.posX = drag.startPhotoX + (event.clientX - drag.startPointerX)
  drag.photo.posY = drag.startPhotoY + (event.clientY - drag.startPointerY)
}

function onPointerUp(event: PointerEvent) {
  const drag = activeDrags.get(event.pointerId)
  if (drag) drag.photo.isDragging = false
  activeDrags.delete(event.pointerId)
}

function startPhotoRain() {
  fallingPhotos.value = []
  photoCounter = 0

  const isMobile = window.innerWidth <= 540
  const count = isMobile
    ? Math.floor(Math.random() * 2) + 3   // 3–4 on mobile
    : Math.floor(Math.random() * 5) + 12  // 12–16 on desktop

  // Coordinates are now relative to .envelope-container since .photo-rain lives inside it
  const envEl = document.querySelector<HTMLElement>('.envelope-container')
  const envRect = envEl?.getBoundingClientRect()
  const envW = envRect?.width ?? 500
  const envH = envRect?.height ?? 320
  const envLeft = envRect?.left ?? (window.innerWidth - envW) / 2
  const envTop = envRect?.top ?? (window.innerHeight - envH) / 2

  const W = window.innerWidth
  const H = window.innerHeight

  // Shuffle photos so each open shows a different random selection
  const pool = [...photos].sort(() => Math.random() - 0.5)

  spawnTimer = setInterval(() => {
    if (photoCounter >= count) {
      if (spawnTimer) clearInterval(spawnTimer)
      return
    }

    const src = pool.length ? (pool[photoCounter % pool.length] ?? null) : null
    const color = placeholderColors[photoCounter % placeholderColors.length] ?? '#ffd6d6'
    const size = Math.floor(Math.random() * 40 + 130)

    // Photo starts at the center of the container (container-relative coords)
    const startX = envW / 2 - size / 2
    const startY = envH / 2 - size / 2

    // Landing spot in viewport coords → convert to container-relative translation
    const landViewportX = Math.random() * (W - size)
    const landViewportY = H - size - 16
    const txEnd = (landViewportX - envLeft) - startX
    const tyEnd = (landViewportY - envTop) - startY

    // Arc peak — upward burst before gravity takes over.
    // Clamp horizontal peak to viewport bounds; vertical is unclamped (fine to go above screen).
    const startViewportX = envLeft + startX
    const txPeakRaw = txEnd * 0.35 + (Math.random() - 0.5) * 180
    const peakViewportX = Math.max(0, Math.min(W - size, startViewportX + txPeakRaw))
    const txPeak = peakViewportX - startViewportX
    const tyPeak = -(Math.random() * 400 + envTop + envH)

    const rotMid = (Math.random() - 0.5) * 420
    const rotEnd = (Math.random() - 0.5) * 720

    fallingPhotos.value.push({
      id: Date.now() + photoCounter,
      src,
      color,
      size,
      startX,
      startY,
      txPeak,
      tyPeak,
      txEnd,
      tyEnd,
      rotMid,
      rotEnd,
      duration: Math.random() * 1 + 2,       // 2–3 s
      delay: photoCounter * 0.06 + Math.random() * 0.08, // tight stagger burst
      posX: null,
      posY: null,
      isDragging: false,
      isLeaving: false,
      leaveDelay: 0,
    })

    photoCounter++
  }, 150)
  // Photos stay on screen until the envelope is closed
}

async function stopPhotoRain() {
  if (spawnTimer) { clearInterval(spawnTimer); spawnTimer = null }

  // Step 1: freeze any photo still in CSS animation at its current visual position
  const photoEls = Array.from(
    document.querySelectorAll<HTMLElement>('.falling-photo'),
  )
  fallingPhotos.value.forEach((photo, i) => {
    if (photo.posX === null) {
      const el = photoEls[i]
      if (el) {
        // Read the exact translation from the computed matrix — avoids the
        // bounding-rect distortion caused by rotation on the element.
        const matrix = new DOMMatrix(window.getComputedStyle(el).transform)
        photo.posX = photo.startX + matrix.m41
        photo.posY = photo.startY + matrix.m42
      }
    }
    // Random stagger so photos don't all vanish at the same moment
    photo.leaveDelay = Math.random() * 0.5
  })

  // Step 2: let Vue render the frozen positions (opacity 1, no animation)
  await nextTick()

  // Step 3: one rAF later — browser has painted frame 1, now trigger the transition
  requestAnimationFrame(() => {
    fallingPhotos.value.forEach(photo => {
      photo.isLeaving = true
    })
  })

  // Step 4: remove from DOM after the longest possible transition finishes
  setTimeout(() => {
    fallingPhotos.value = []
  }, 2100) // 1400ms transition + 500ms max delay + buffer
}

// ── Envelope ───────────────────────────────────────────────────────────────
const letterEl = ref<HTMLElement>()
const envelopeFlapEl = ref<HTMLElement>()
const isOpen = ref(false)
let busy = false

function toggle() {
  if (busy) return
  busy = true

  if (!isOpen.value) {
    isOpen.value = true
    letter = letters[Math.floor(Math.random() * letters.length)] ?? letters[0]!

    setTimeout(() => {
      if (envelopeFlapEl.value) envelopeFlapEl.value.style.zIndex = '1'
      if (letterEl.value) {
        letterEl.value.style.transform = 'translateY(-88%)'
        letterEl.value.style.zIndex = '2'
      }
      startPhotoRain()
      busy = false
    }, 400)
  } else {
    stopPhotoRain()
    if (letterEl.value) letterEl.value.style.transform = ''

    setTimeout(() => {
      if (envelopeFlapEl.value) envelopeFlapEl.value.style.zIndex = '5'
      if (letterEl.value) letterEl.value.style.zIndex = ''
      isOpen.value = false
      busy = false
    }, 700)
  }
}
</script>

<template>
  <div class="page">
    <div class="hearts" aria-hidden="true">
      <span v-for="i in 30" :key="i" class="heart" :style="`--i: ${i}`">♡</span>
    </div>

    <div class="scene">
      <div class="envelope-container" :class="{ open: isOpen }" role="button" tabindex="0" aria-label="Open letter"
        @click="toggle" @keydown.enter="toggle" @keydown.space.prevent="toggle">
        <div class="envelope" aria-hidden="true">
          <div class="env-back" />
          <div class="fold fold-bottom" />
          <div class="fold fold-left" />
          <div class="fold fold-right" />
          <div class="flap-wrapper" ref="envelopeFlapEl">
            <div class="flap-face" />
            <span class="seal">❤</span>
          </div>

          <!-- Photos live inside the envelope, same layer as the letter -->
          <div class="photo-rain" aria-hidden="true">
            <div v-for="photo in fallingPhotos" :key="photo.id" class="falling-photo" :style="getPhotoStyle(photo)"
              @pointerdown.prevent.stop="onPointerDown($event, photo)" @pointermove="onPointerMove($event)"
              @pointerup="onPointerUp($event)" @pointercancel="onPointerUp($event)" @click.stop>
              <div class="polaroid">
                <img v-if="photo.src" :src="photo.src" alt="" />
                <div v-else class="photo-placeholder" :style="{ background: photo.color }" />
                <div class="polaroid-caption">♡</div>
              </div>
            </div>
          </div>

          <div class="letter" ref="letterEl">
            <div class="letter-body">
              <p class="greeting">{{ letter.greeting }}</p>
              <p v-for="(line, i) in letter.lines" :key="i">{{ line }}</p>
              <p class="signature">{{ letter.signature }}</p>
            </div>
            <div class="ruled-lines" />
          </div>
        </div>

        <p class="hint" :class="{ hidden: isOpen }">click to open</p>
      </div>
    </div>
  </div>
</template>

<style scoped>
.page {
  min-height: 100vh;
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(160deg, #ffecd2 0%, #fcb69f 100%);
  font-family: Georgia, 'Times New Roman', serif;
  position: relative;
}

/* Floating hearts */
.hearts {
  position: fixed;
  inset: 0;
  pointer-events: none;
  overflow: hidden;
}

.heart {
  position: absolute;
  bottom: -60px;
  font-size: 1.4rem;
  color: rgba(200, 80, 80, 0.2);
  left: calc(var(--i) * 8.3%);
  animation-name: float-up;
  animation-timing-function: ease-in;
  animation-iteration-count: infinite;
  animation-fill-mode: none;
  animation-duration: calc(7s + var(--i) * 0.4s);
  animation-delay: calc(var(--i) * 0.7s);
}

@keyframes float-up {
  0% {
    transform: translateY(0) scale(1);
    opacity: 0;
  }

  10% {
    opacity: 1;
  }

  90% {
    opacity: 0.4;
  }

  100% {
    transform: translateY(-110vh) scale(1.4);
    opacity: 0;
  }
}

/* Scene */
.scene {
  perspective: 1200px;
  padding-bottom: 60px;
}

/* Envelope container — all dimensions flow from these four variables */
.envelope-container {
  --env-w: 500px;
  --env-h: 320px;
  --env-hw: 250px;
  /* half width  */
  --env-hh: 160px;
  /* half height */

  position: relative;
  width: var(--env-w);
  height: var(--env-h);
  cursor: pointer;
  user-select: none;
  outline: none;
}


/* Letter */
.letter {
  position: absolute;
  width: 84%;
  left: 8%;
  top: 8%;
  bottom: 6%;
  background: #fffef5;
  border-radius: 3px;
  padding: 26px 30px;
  z-index: 2;
  box-shadow: 0 2px 14px rgba(0, 0, 0, 0.08);
  transition: transform 0.7s cubic-bezier(0.34, 1.1, 0.64, 1);
  pointer-events: none;
  overflow: hidden;
}

.letter-body {
  position: relative;
  z-index: 1;
}

.letter .greeting {
  font-size: 17px;
  color: #8b4513;
  margin-bottom: 14px;
  font-style: italic;
}

.letter p {
  font-size: 13.5px;
  color: #5a3520;
  line-height: 1.9;
  margin-bottom: 10px;
}

.letter .signature {
  margin-top: 16px;
  font-style: italic;
  color: #8b4513;
  text-align: right;
  font-size: 15px;
}

.ruled-lines {
  position: absolute;
  inset: 0;
  background: repeating-linear-gradient(transparent,
      transparent 30px,
      rgba(139, 69, 19, 0.07) 30px,
      rgba(139, 69, 19, 0.07) 31px);
  pointer-events: none;
}

/* Envelope shell */
.envelope {
  position: absolute;
  inset: 0;
  z-index: 1;
  transition: transform 0.2s ease;
}

.envelope-container:hover:not(:has(.falling-photo:hover)) .envelope {
  transform: translateY(-3px);
}

.envelope-container:hover:not(:has(.falling-photo:hover)) .env-back {
  box-shadow:
    0 18px 40px rgba(0, 0, 0, 0.22),
    0 2px 6px rgba(0, 0, 0, 0.1);
}

.env-back {
  position: absolute;
  inset: 0;
  background: #e8c89a;
  border-radius: 6px;
  box-shadow:
    0 10px 36px rgba(0, 0, 0, 0.18),
    0 2px 6px rgba(0, 0, 0, 0.08);
  z-index: 1;
  transition: box-shadow 0.25s ease;
}

/* Fold triangles */
.fold {
  position: absolute;
  width: 0;
  height: 0;
}

/* bottom-center crease */
.fold-bottom {
  bottom: 0;
  left: 0;
  border-left: var(--env-hw) solid transparent;
  border-right: var(--env-hw) solid transparent;
  border-bottom: var(--env-hh) solid #d4a870;
  z-index: 4;
}

/* left side crease */
.fold-left {
  top: 0;
  left: 0;
  border-top: var(--env-hh) solid transparent;
  border-bottom: var(--env-hh) solid transparent;
  border-left: var(--env-hw) solid #dcb07a;
  z-index: 4;
}

/* right side crease */
.fold-right {
  top: 0;
  right: 0;
  border-top: var(--env-hh) solid transparent;
  border-bottom: var(--env-hh) solid transparent;
  border-right: var(--env-hw) solid #dcb07a;
  z-index: 4;
}

/* Flap */
.flap-wrapper {
  position: absolute;
  top: 0;
  left: 0;
  width: var(--env-w);
  height: var(--env-hh);
  transform-origin: top center;
  transition: transform 0.6s cubic-bezier(0.4, 0, 0.2, 1);
  z-index: 5;
}

.envelope-container.open .flap-wrapper {
  transform: perspective(1000px) rotateX(-180deg);
}

/* downward-pointing triangle = envelope flap face */
.flap-face {
  position: absolute;
  top: 0;
  left: 0;
  width: 0;
  height: 0;
  border-left: var(--env-hw) solid transparent;
  border-right: var(--env-hw) solid transparent;
  border-top: var(--env-hh) solid #c8955a;
}

.seal {
  position: absolute;
  bottom: 0;
  left: 50%;
  transform: translate(-50%, 50%);
  font-size: 26px;
  color: #c0392b;
  filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.25));
  transition:
    opacity 0.3s,
    transform 0.3s;
  z-index: 5;
  line-height: 1;
}

.envelope-container.open .seal {
  opacity: 0;
  transform: translate(-50%, 50%) scale(0.4);
}

/* Hint label */
.hint {
  position: absolute;
  bottom: -42px;
  left: 50%;
  transform: translateX(-50%);
  color: rgba(120, 56, 16, 0.6);
  font-size: 13px;
  font-style: italic;
  white-space: nowrap;
  letter-spacing: 0.5px;
  transition: opacity 0.4s;
}

.hint.hidden {
  opacity: 0;
}

/* ===== Photo confetti — lives inside .envelope-container ===== */
.photo-rain {
  position: absolute;
  inset: 0;
  pointer-events: none;
  overflow: visible;
  /* no z-index → no stacking context → children compete directly with .envelope */
}

.falling-photo {
  position: absolute;
  pointer-events: auto;
  user-select: none;
  animation: confetti-fly var(--dur) linear var(--delay) forwards;
}

@keyframes confetti-fly {

  /* Rising — behind envelope */
  0% {
    z-index: 0;
    transform: translate(0, 0) rotate(0deg) scale(0.1);
    opacity: 0;
    animation-timing-function: cubic-bezier(0.2, 0, 0.4, 1);
  }

  8% {
    z-index: 0;
    opacity: 1;
    transform: translate(calc(var(--tx-peak) * 0.15),
        calc(var(--ty-peak) * 0.3)) rotate(calc(var(--rot-mid) * 0.1)) scale(1);
  }

  /* Arc peak — still rising, still behind */
  32% {
    z-index: 0;
    transform: translate(var(--tx-peak), var(--ty-peak)) rotate(var(--rot-mid));
    animation-timing-function: cubic-bezier(0.4, 0, 0.8, 1);
  }

  /* Falling — now in front */
  33% {
    z-index: 200;
  }

  /* Land */
  80% {
    transform: translate(var(--tx-end), var(--ty-end)) rotate(var(--rot-end));
    animation-timing-function: ease-out;
  }

  /* Bounce */
  87% {
    transform: translate(var(--tx-end), calc(var(--ty-end) - 22px)) rotate(var(--rot-end));
    animation-timing-function: ease-in;
  }

  94% {
    transform: translate(var(--tx-end), var(--ty-end)) rotate(var(--rot-end));
  }

  100% {
    z-index: 200;
    transform: translate(var(--tx-end), var(--ty-end)) rotate(var(--rot-end));
    opacity: 1;
  }
}

.polaroid {
  background: #fff;
  padding: 6px 6px 22px;
  box-shadow: 0 4px 14px rgba(0, 0, 0, 0.22), 0 1px 4px rgba(0, 0, 0, 0.1);
  border-radius: 2px;
}

.polaroid img,
.photo-placeholder {
  width: 100%;
  aspect-ratio: 1;
  object-fit: cover;
  display: block;
  border-radius: 1px;
}

.polaroid-caption {
  text-align: center;
  font-size: 13px;
  color: #c0392b;
  margin-top: 4px;
  line-height: 1;
}

/* ===== Responsive ===== */
@media (max-width: 540px) {
  .envelope-container {
    --env-w: 320px;
    --env-h: 204px;
    --env-hw: 160px;
    --env-hh: 102px;
  }

  .scene {
    padding-bottom: 48px;
  }

  .letter {
    padding: 13px 16px;
    bottom: 2%;
    /* taller card → more room for wrapped text */
  }

  .letter .greeting {
    font-size: 14px;
    margin-bottom: 8px;
  }

  .letter p {
    font-size: 11.5px;
    line-height: 1.65;
    margin-bottom: 5px;
  }

  .letter .signature {
    font-size: 12px;
    margin-top: 8px;
  }

  .hint {
    font-size: 12px;
    bottom: -36px;
  }
}

@media (max-width: 350px) {
  .envelope-container {
    --env-w: 280px;
    --env-h: 178px;
    --env-hw: 140px;
    --env-hh: 89px;
  }

  .letter {
    padding: 10px 13px;
    bottom: 1%;
  }

  .letter .greeting {
    font-size: 13px;
    margin-bottom: 6px;
  }

  .letter p {
    font-size: 10.5px;
    line-height: 1.6;
    margin-bottom: 4px;
  }

  .letter .signature {
    font-size: 11px;
    margin-top: 6px;
  }
}
</style>
