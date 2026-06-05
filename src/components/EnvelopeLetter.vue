<script setup lang="ts">
import { ref } from 'vue'

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
const photoModules = import.meta.glob('@/assets/images/*.jpg', { eager: true })
const photos: string[] = Object.values(photoModules).map((m: any) => m.default)

// Shown when no real photos are loaded yet
const placeholderColors = [
  '#ffd6d6', '#ffdeba', '#fff4ba', '#d4f4d4',
  '#d4e8ff', '#e8d4ff', '#ffc4c4', '#c4e8c4',
]

interface FallingPhoto {
  id: number
  src: string | null
  color: string
  x: number        // vw from left (used while animating)
  rotation: number // degrees
  duration: number // seconds
  delay: number    // seconds
  size: number     // px
  // Drag state — null = controlled by CSS animation
  posX: number | null
  posY: number | null
  isDragging: boolean
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
  // After first drag: static position, no animation
  if (photo.posX !== null && photo.posY !== null) {
    return {
      width: photo.size + 'px',
      left: photo.posX + 'px',
      top: photo.posY + 'px',
      transform: `rotate(${photo.rotation}deg)`,
      animation: 'none',
      zIndex: '200',
      cursor: photo.isDragging ? 'grabbing' : 'grab',
      touchAction: 'none',
    }
  }
  // Still falling via CSS animation
  return {
    width: photo.size + 'px',
    left: photo.x + 'vw',
    '--rot': photo.rotation + 'deg',
    '--dur': photo.duration + 's',
    '--delay': photo.delay + 's',
    cursor: 'grab',
    touchAction: 'none',
  }
}

function onPointerDown(event: PointerEvent, photo: FallingPhoto) {
  event.preventDefault()
  event.stopPropagation()

  const el = event.currentTarget as HTMLElement
  const rect = el.getBoundingClientRect()

  // .photo-rain is position:fixed inset:0, so viewport coords = container coords
  const startPhotoX = rect.left
  const startPhotoY = rect.top

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

  spawnTimer = setInterval(() => {
    if (photoCounter >= 18) {
      if (spawnTimer) clearInterval(spawnTimer)
      return
    }

    const src = photos.length ? (photos[photoCounter % photos.length] ?? null) : null
    const color = placeholderColors[photoCounter % placeholderColors.length] ?? '#ffd6d6'

    fallingPhotos.value.push({
      id: Date.now() + photoCounter,
      src,
      color,
      x: Math.random() * 80 + 5,
      rotation: (Math.random() - 0.5) * 28,
      duration: Math.random() * 2 + 2.5,
      delay: Math.random() * 0.3,
      size: Math.floor(Math.random() * 40 + 130),
      posX: null,
      posY: null,
      isDragging: false,
    })

    photoCounter++
  }, 280)
  // Photos stay on screen until the envelope is closed
}

function stopPhotoRain() {
  if (spawnTimer) { clearInterval(spawnTimer); spawnTimer = null }
  fallingPhotos.value = []
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
      if (envelopeFlapEl.value) envelopeFlapEl.value.style.zIndex = '4'
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

    <!-- Photo rain -->
    <div class="photo-rain" aria-hidden="true">
      <div
        v-for="photo in fallingPhotos"
        :key="photo.id"
        class="falling-photo"
        :style="getPhotoStyle(photo)"
        @pointerdown.prevent.stop="onPointerDown($event, photo)"
        @pointermove="onPointerMove($event)"
        @pointerup="onPointerUp($event)"
        @pointercancel="onPointerUp($event)"
      >
        <div class="polaroid">
          <img v-if="photo.src" :src="photo.src" alt="" />
          <div v-else class="photo-placeholder" :style="{ background: photo.color }" />
          <div class="polaroid-caption">♡</div>
        </div>
      </div>
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
  --env-hw: 250px; /* half width  */
  --env-hh: 160px; /* half height */

  position: relative;
  width: var(--env-w);
  height: var(--env-h);
  cursor: pointer;
  user-select: none;
  outline: none;
  transition: filter 0.25s;
}

.envelope-container:hover {
  filter: drop-shadow(0 18px 36px rgba(0, 0, 0, 0.2));
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
  z-index: 1;
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

.envelope-container:hover .envelope {
  transform: translateY(-3px);
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
  z-index: 3;
}

/* left side crease */
.fold-left {
  top: 0;
  left: 0;
  border-top: var(--env-hh) solid transparent;
  border-bottom: var(--env-hh) solid transparent;
  border-left: var(--env-hw) solid #dcb07a;
  z-index: 3;
}

/* right side crease */
.fold-right {
  top: 0;
  right: 0;
  border-top: var(--env-hh) solid transparent;
  border-bottom: var(--env-hh) solid transparent;
  border-right: var(--env-hw) solid #dcb07a;
  z-index: 3;
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
  z-index: 4;
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

/* ===== Photo rain ===== */
.photo-rain {
  position: fixed;
  inset: 0;
  pointer-events: none;
  overflow: hidden;
  z-index: 100;
}

.falling-photo {
  position: absolute;
  top: -220px;
  pointer-events: auto;
  user-select: none;
  animation: photo-fall var(--dur) ease-in var(--delay) forwards;
}

@keyframes photo-fall {
  0% {
    transform: translateY(0) rotate(var(--rot));
    opacity: 0;
    animation-timing-function: ease-in;
  }
  8%  { opacity: 1; }
  80% {
    transform: translateY(calc(100vh + 54px)) rotate(var(--rot));
    animation-timing-function: ease-out;
  }
  87% {
    transform: translateY(calc(100vh + 20px)) rotate(var(--rot));
    animation-timing-function: ease-in;
  }
  93% {
    transform: translateY(calc(100vh + 60px)) rotate(var(--rot));
    animation-timing-function: ease-out;
  }
  100% {
    transform: translateY(calc(100vh + 54px)) rotate(var(--rot));
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
    padding: 18px 20px;
  }

  .letter .greeting {
    font-size: 15px;
    margin-bottom: 10px;
  }

  .letter p {
    font-size: 12px;
    line-height: 1.8;
    margin-bottom: 8px;
  }

  .letter .signature {
    font-size: 13px;
    margin-top: 12px;
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
    padding: 14px 16px;
  }

  .letter .greeting {
    font-size: 13px;
  }

  .letter p {
    font-size: 11px;
  }
}
</style>
