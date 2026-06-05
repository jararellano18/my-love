<script setup lang="ts">
import { ref } from 'vue'

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

      busy = false
    }, 400)
  } else {

    if (letterEl.value) {
      letterEl.value.style.transform = ''
    }
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
