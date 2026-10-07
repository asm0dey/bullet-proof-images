<template>
  <!-- deck-wide footer, per design_rules.footer.pattern '@handle | #conference | #topic'.
       Suppressed on full-bleed art, the cover, the blackout and the bio slide. -->
  <footer v-if="!hideFooter" class="deck-footer">
    @asm0dey <span class="sep">|</span> #DevoxxBE <span class="sep">|</span> #containers
  </footer>

  <!-- Ambient bed. ONE persistent element so it survives slide changes: the bed
       spans 3-4 and returns for 27-29, and a per-slide <audio> would restart it.
       Driven by Slidev's own nav state rather than by DOM poking. Optional by
       design - if the venue has no audio, nothing else in the deck changes. -->
  <audio ref="bed" src="/audio/ambient-sealed-room.mp3" loop preload="auto" />
</template>

<script setup>
import { ref, computed, watch, onUnmounted } from 'vue'
import { useNav } from '@slidev/client'

const { currentSlideRoute, currentPage } = useNav()

const hideFooter = computed(() => {
  const c = currentSlideRoute.value?.meta?.slide?.frontmatter?.class || ''
  return /fullbleed|title-slide|blackout|imgtxt/.test(c)
})

// the two stretches where the room is audible
const BED_RANGES = [[3, 4], [27, 29]]
const TARGET_VOLUME = 0.35
const HARD_CUT_SLIDE = 30      // the blackout: silence must arrive abruptly

const bed = ref(null)
let fade = null

function rampTo(target, ms) {
  const el = bed.value
  if (!el) return
  clearInterval(fade)
  if (ms === 0) { el.volume = target; if (!target) el.pause(); return }
  const step = 40, from = el.volume, delta = target - from
  let t = 0
  fade = setInterval(() => {
    t += step
    const k = Math.min(1, t / ms)
    el.volume = Math.max(0, Math.min(1, from + delta * k))
    if (k >= 1) { clearInterval(fade); if (target === 0) el.pause() }
  }, step)
}

watch(currentPage, (page) => {
  const el = bed.value
  if (!el) return
  const shouldPlay = BED_RANGES.some(([a, b]) => page >= a && page <= b)
  if (shouldPlay) {
    el.volume = el.paused ? 0 : el.volume
    // browsers reject autoplay without a gesture; navigating IS one, and a
    // rejected promise here is harmless - the deck works silently.
    el.play().then(() => rampTo(TARGET_VOLUME, 2000)).catch(() => {})
  } else if (page === HARD_CUT_SLIDE) {
    rampTo(0, 0)               // no fade: the cut is the point
  } else {
    rampTo(0, 1200)
  }
}, { immediate: true })

onUnmounted(() => clearInterval(fade))
</script>

<style scoped>
.deck-footer {
  position: absolute; bottom: 8px; left: 16px;
  font-family: 'JetBrains Mono', ui-monospace, monospace;
  font-size: 11px; letter-spacing: .04em;
  color: #40606f; opacity: .85; z-index: 20;
}
.sep { opacity: .5; margin: 0 .35em; }
</style>
