<template>
  <div class="page" @click="onPageClick">
    <div class="playfield"></div>

    <div class="hud">
      <h1>广告净化大师 3000</h1>
      <p>
        Click banners to remove them. After the first banner, roaches start annoying you.
        After the fifth, banners dodge the cursor. After the tenth, chaos mode starts.
        Remove all 50 to win.
      </p>

      <div class="stats">
        <span>Difficulty: {{ difficultyLabel }}</span>
        <span>Removed: {{ removedCount }} / {{ maxSpawnTotal }}</span>
        <span>On screen: {{ visibleBanners.length }}</span>
        <span>Spawned: {{ spawnedTotal }} / {{ maxSpawnTotal }}</span>
        <span>Phase: {{ phaseLabel }}</span>
        <span>Slow charges: {{ slowCharges }}</span>
      </div>

      <div class="controls">
        <button class="spell-btn" :class="{ disabled: !canCastSlow }" @click.stop="castSlow">
          ❄ Slow Spell
        </button>
        <button class="mode-btn" @click.stop="toggleDifficulty">
          Switch to {{ nextDifficultyLabel }}
        </button>
        <span class="hint">Every 10 removed banners grants 1 slow charge.</span>
      </div>
    </div>

    <div v-if="slowTimer > 0" class="slow-overlay">
      <div class="slow-text">❄ SLOW {{ slowTimer.toFixed(1) }}s</div>
    </div>

    <TransitionGroup name="banner-pop" tag="div">
      <button
        v-for="banner in visibleBanners"
        :key="banner.id"
        class="banner"
        :class="{
          panic: banner.panic,
          slowed: isSlowActive,
          chaos: currentPhase >= 3,
          popping: banner.popping,
        }"
        :style="bannerStyle(banner)"
        @click.stop="popBanner(banner.id)"
        @mouseenter="markCursorActive"
      >
        <div class="close fake">点</div>
        <div class="badge">限时</div>
        <div class="banner-content">
          <div class="title">{{ banner.title }}</div>
          <div class="subtitle">{{ banner.subtitle }}</div>
          <div class="cta">{{ banner.cta }}</div>
        </div>

        <div class="roach-zone">
          <div
            v-for="roach in banner.roaches"
            :key="roach.id"
            class="roach"
            :class="{
              blocker: roach.mode === 'block',
              carrier: roach.mode === 'carry',
              berserk: roach.mode === 'chaos',
              slowed: isSlowActive,
            }"
            :style="roachStyle(roach)"
          >
            🪳
          </div>
        </div>
      </button>
    </TransitionGroup>

    <div v-if="gameState !== 'playing'" class="overlay-card">
      <h2>{{ gameState === 'won' ? '你赢了！' : 'Game Over' }}</h2>
      <p v-if="gameState === 'won'">
        You removed all {{ maxSpawnTotal }} banners. Peace has returned to the browser.
      </p>
      <p v-else>
        The ad swarm trapped the screen.
      </p>
      <button @click="spawnInitialWave">Play Again</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, ref } from 'vue'

interface Roach {
  id: number
  angle: number
  radius: number
  phase: number
  speed: number
  x: number
  y: number
  mode: 'idle' | 'block' | 'carry' | 'chaos'
}

interface Banner {
  id: number
  x: number
  y: number
  width: number
  height: number
  title: string
  subtitle: string
  cta: string
  panic: boolean
  vx: number
  vy: number
  roaches: Roach[]
  justSpawned: boolean
  popping: boolean
}

type Difficulty = 'normal' | 'nightmare'

const ads = [
  { title: '恭喜！你被选中！', subtitle: '点击领取 9999 节数学作业套餐', cta: '立即变强' },
  { title: '超快提分秘诀', subtitle: '3 秒学会一学期内容，老师看了沉默', cta: '马上冲刺' },
  { title: '神秘校长推荐', subtitle: '每天背 200 个单词，快乐到飞起', cta: '现在开始' },
  { title: '全自动写作业仪', subtitle: '仅限今天，买一送九十九', cta: '火速下单' },
  { title: '高能学习模式', subtitle: '开启后连周一早晨都不可怕', cta: '进入模式' },
  { title: '绝密考试护符', subtitle: '据说看一眼就会做选择题', cta: '偷偷领取' },
  { title: '沉浸式补习体验', subtitle: '作业越多，成长越快，真的真的', cta: '立刻体验' },
  { title: '宇宙级课程优惠', subtitle: '错过今天，再等 3 分钟', cta: '别再犹豫' },
]

const maxSpawnTotal = 50
const initialWave = 4
const difficulty = ref<Difficulty>('normal')

const banners = ref<Banner[]>([])
const removedCount = ref(0)
const spawnedTotal = ref(0)
const nextId = ref(1)
const nextRoachId = ref(1)
const slowCharges = ref(0)
const slowTimer = ref(0)
const gameState = ref<'playing' | 'won' | 'lost'>('playing')

const cursor = reactive({
  x: typeof window !== 'undefined' ? window.innerWidth / 2 : 0,
  y: typeof window !== 'undefined' ? window.innerHeight / 2 : 0,
  vx: 0,
  vy: 0,
  speed: 0,
  active: false,
  lastX: typeof window !== 'undefined' ? window.innerWidth / 2 : 0,
  lastY: typeof window !== 'undefined' ? window.innerHeight / 2 : 0,
  lastTime: typeof performance !== 'undefined' ? performance.now() : 0,
})

let animationFrame = 0
let lastAnimationTime = 0

const visibleBanners = computed(() => banners.value)
const difficultyLabel = computed(() => (difficulty.value === 'normal' ? 'Normal' : 'Nightmare'))
const nextDifficultyLabel = computed(() => (difficulty.value === 'normal' ? 'Nightmare' : 'Normal'))
const maxVisibleAtOnce = computed(() => (difficulty.value === 'normal' ? 9 : 12))
const difficultySpeedBoost = computed(() => (difficulty.value === 'normal' ? 1 : 1.3))
const isSlowActive = computed(() => slowTimer.value > 0)
const slowFactor = computed(() => (isSlowActive.value ? 0.18 : 1))
const currentPhase = computed(() => {
  if (removedCount.value >= 10) return 3
  if (removedCount.value >= 5) return 2
  if (removedCount.value >= 1) return 1
  return 0
})
const phaseLabel = computed(() => {
  switch (currentPhase.value) {
    case 0:
      return 'Calm'
    case 1:
      return 'Interference'
    case 2:
      return 'Dodging'
    case 3:
      return 'MAYHEM'
    default:
      return '???'
  }
})
const canCastSlow = computed(() => slowCharges.value > 0 && !isSlowActive.value && gameState.value === 'playing')

function clamp(value: number, min: number, max: number) {
  return Math.max(min, Math.min(max, value))
}

function getInsets() {
  const mobile = window.innerWidth <= 900
  return {
    left: 18,
    right: mobile ? 18 : 24,
    top: 128,
    bottom: 28,
  }
}

function getPlayBounds(width: number, height: number) {
  const inset = getInsets()
  const minX = inset.left
  const maxX = Math.max(minX, window.innerWidth - width - inset.right)
  const minY = inset.top
  const maxY = Math.max(minY, window.innerHeight - height - inset.bottom)
  return { minX, maxX, minY, maxY }
}

function randomFromAds() {
  return ads[Math.floor(Math.random() * ads.length)]
}

function overlapsExisting(x: number, y: number, width: number, height: number, ignoreId?: number) {
  return banners.value.some((banner) => {
    if (ignoreId && banner.id === ignoreId) return false
    const padding = 28
    return !(
      x + width + padding < banner.x ||
      x > banner.x + banner.width + padding ||
      y + height + padding < banner.y ||
      y > banner.y + banner.height + padding
    )
  })
}

function createRoach(): Roach {
  return {
    id: nextRoachId.value++,
    angle: Math.random() * Math.PI * 2,
    radius: 28 + Math.random() * 24,
    phase: Math.random() * Math.PI * 2,
    speed: 0.02 + Math.random() * 0.03,
    x: 0,
    y: 0,
    mode: 'idle',
  }
}

function createBanner(preferredX?: number, preferredY?: number): Banner {
  const width = 250 + Math.round(Math.random() * 70)
  const height = 132 + Math.round(Math.random() * 24)
  const bounds = getPlayBounds(width, height)

  let x = preferredX ?? bounds.minX + Math.random() * Math.max(1, bounds.maxX - bounds.minX)
  let y = preferredY ?? bounds.minY + Math.random() * Math.max(1, bounds.maxY - bounds.minY)
  x = clamp(x, bounds.minX, bounds.maxX)
  y = clamp(y, bounds.minY, bounds.maxY)

  let attempts = 0
  while (overlapsExisting(x, y, width, height) && attempts < 60) {
    x = bounds.minX + Math.random() * Math.max(1, bounds.maxX - bounds.minX)
    y = bounds.minY + Math.random() * Math.max(1, bounds.maxY - bounds.minY)
    attempts++
  }

  const roachCount = currentPhase.value >= 3 ? 4 : currentPhase.value >= 2 ? 3 : 2
  const copy = randomFromAds()

  return {
    id: nextId.value++,
    x,
    y,
    width,
    height,
    title: copy.title,
    subtitle: copy.subtitle,
    cta: copy.cta,
    panic: false,
    vx: 0,
    vy: 0,
    roaches: Array.from({ length: roachCount }, () => createRoach()),
    justSpawned: true,
    popping: false,
  }
}

function spawnBanner(preferredX?: number, preferredY?: number) {
  if (spawnedTotal.value >= maxSpawnTotal || gameState.value !== 'playing') return false
  if (banners.value.length >= maxVisibleAtOnce.value) return false

  const banner = createBanner(preferredX, preferredY)
  banners.value.push(banner)
  spawnedTotal.value += 1

  window.setTimeout(() => {
    const found = banners.value.find((item) => item.id === banner.id)
    if (found) found.justSpawned = false
  }, 260)

  return true
}

function fillVisibleSlots() {
  if (gameState.value !== 'playing') return
  while (banners.value.length < maxVisibleAtOnce.value && spawnedTotal.value < maxSpawnTotal) {
    if (!spawnBanner()) break
  }
}

function spawnInitialWave() {
  banners.value = []
  removedCount.value = 0
  spawnedTotal.value = 0
  slowCharges.value = 0
  slowTimer.value = 0
  nextId.value = 1
  nextRoachId.value = 1
  gameState.value = 'playing'

  for (let i = 0; i < initialWave; i++) {
    spawnBanner()
  }

  nextTick(() => {
    onResize()
  })
}

function toggleDifficulty() {
  difficulty.value = difficulty.value === 'normal' ? 'nightmare' : 'normal'
  spawnInitialWave()
}

function rebalanceRoaches() {
  const target = currentPhase.value >= 3 ? 4 : currentPhase.value >= 2 ? 3 : 2
  banners.value.forEach((banner) => {
    while (banner.roaches.length < target) banner.roaches.push(createRoach())
    if (banner.roaches.length > target) banner.roaches.splice(target)
  })
}

function popBanner(id: number) {
  if (gameState.value !== 'playing') return
  const banner = banners.value.find((item) => item.id === id)
  if (!banner || banner.popping) return
  banner.popping = true

  window.setTimeout(() => {
    const index = banners.value.findIndex((item) => item.id === id)
    if (index === -1) return
    banners.value.splice(index, 1)
    removedCount.value += 1

    if (removedCount.value % 10 === 0) slowCharges.value += 1
    rebalanceRoaches()

    if (removedCount.value >= maxSpawnTotal) {
      gameState.value = 'won'
      return
    }

    window.setTimeout(() => fillVisibleSlots(), 120)
  }, 130)
}

function castSlow() {
  if (!canCastSlow.value) return
  slowCharges.value -= 1
  slowTimer.value = 5.5
}

function updateCursor(event: MouseEvent) {
  const now = performance.now()
  const dt = Math.max(16, now - cursor.lastTime)
  cursor.x = event.clientX
  cursor.y = event.clientY
  cursor.vx = ((event.clientX - cursor.lastX) / dt) * 16
  cursor.vy = ((event.clientY - cursor.lastY) / dt) * 16
  cursor.speed = Math.sqrt(cursor.vx * cursor.vx + cursor.vy * cursor.vy)
  cursor.lastX = event.clientX
  cursor.lastY = event.clientY
  cursor.lastTime = now
  cursor.active = true
}

function onMouseMove(event: MouseEvent) {
  updateCursor(event)
}

function markCursorActive() {
  cursor.active = true
}

function onMouseLeave() {
  cursor.active = false
}

function onPageClick() {
  cursor.active = true
}

function bannerStyle(banner: Banner) {
  return {
    width: `${banner.width}px`,
    height: `${banner.height}px`,
    transform: `translate3d(${banner.x}px, ${banner.y}px, 0) scale(${banner.popping ? 0.84 : 1})`,
    opacity: banner.popping ? '0' : '1',
  }
}

function roachStyle(roach: Roach) {
  return {
    transform: `translate3d(${roach.x}px, ${roach.y}px, 0) rotate(${roach.angle + Math.PI / 2}rad)`,
  }
}

function animate(now: number) {
  const deltaMs = lastAnimationTime ? Math.min(34, now - lastAnimationTime) : 16
  lastAnimationTime = now
  const dt = deltaMs / 16.666
  const phaseSlow = slowFactor.value
  const difficultyBoost = difficultySpeedBoost.value

  if (slowTimer.value > 0) {
    slowTimer.value = Math.max(0, slowTimer.value - deltaMs / 1000)
  }

  if (gameState.value === 'playing') {
    banners.value.forEach((banner) => {
      const centerX = banner.x + banner.width / 2
      const centerY = banner.y + banner.height / 2
      const dx = centerX - cursor.x
      const dy = centerY - cursor.y
      const distance = Math.max(1, Math.sqrt(dx * dx + dy * dy))
      const nearCursor = distance < 230
      banner.panic = currentPhase.value >= 1 && nearCursor

      banner.roaches.forEach((roach, index) => {
        roach.angle += roach.speed * dt * (currentPhase.value >= 3 ? 1.6 : 1) * phaseSlow * difficultyBoost
        roach.phase += (0.045 + index * 0.003) * dt * phaseSlow * difficultyBoost
        const orbit = roach.radius + Math.sin(roach.phase * 2) * 6
        roach.x = Math.cos(roach.angle) * orbit
        roach.y = Math.sin(roach.angle) * orbit * 0.65

        if (currentPhase.value === 0) {
          roach.mode = 'idle'
          return
        }

        if (currentPhase.value === 1) {
          roach.mode = 'block'
          if (nearCursor && !banner.justSpawned && !banner.popping) {
            banner.vx += (Math.random() - 0.5) * 0.45 * dt * phaseSlow * difficultyBoost
            banner.vy += (Math.random() - 0.5) * 0.25 * dt * phaseSlow * difficultyBoost
          }
          return
        }

        const strength = clamp((290 - distance) / 290, 0, 1)
        const speedBoost = clamp(cursor.speed / 22, 0.8, currentPhase.value >= 3 ? 3.1 : 2.2)
        const push = strength * speedBoost * (currentPhase.value >= 3 ? 4.8 : 2.9) * dt * phaseSlow * difficultyBoost

        if (currentPhase.value === 2) {
          roach.mode = 'carry'
          if (nearCursor && !banner.justSpawned && !banner.popping) {
            banner.vx += (dx / distance) * push
            banner.vy += (dy / distance) * push
          }
        }

        if (currentPhase.value >= 3) {
          roach.mode = 'chaos'
          if (!banner.justSpawned && !banner.popping) {
            banner.vx += (dx / distance) * push
            banner.vy += (dy / distance) * push
            banner.vx += (Math.random() - 0.5) * 0.85 * dt * phaseSlow * difficultyBoost
            banner.vy += (Math.random() - 0.5) * 0.55 * dt * phaseSlow * difficultyBoost
          }
        }
      })

      const friction = isSlowActive.value ? 0.72 : currentPhase.value >= 3 ? 0.95 : 0.9
      banner.vx *= friction
      banner.vy *= friction
      banner.x += banner.vx
      banner.y += banner.vy

      const bounds = getPlayBounds(banner.width, banner.height)
      banner.x = clamp(banner.x, bounds.minX, bounds.maxX)
      banner.y = clamp(banner.y, bounds.minY, bounds.maxY)

      if (banner.x === bounds.minX || banner.x === bounds.maxX) banner.vx *= 0.35
      if (banner.y === bounds.minY || banner.y === bounds.maxY) banner.vy *= 0.35
    })
  }

  animationFrame = requestAnimationFrame(animate)
}

function onResize() {
  banners.value.forEach((banner) => {
    const bounds = getPlayBounds(banner.width, banner.height)
    banner.x = clamp(banner.x, bounds.minX, bounds.maxX)
    banner.y = clamp(banner.y, bounds.minY, bounds.maxY)
    banner.vx = 0
    banner.vy = 0
  })
}

onMounted(() => {
  spawnInitialWave()
  lastAnimationTime = performance.now()
  animationFrame = requestAnimationFrame(animate)
  window.addEventListener('resize', onResize)
  window.addEventListener('mousemove', onMouseMove)
  window.addEventListener('mouseleave', onMouseLeave)
})

onBeforeUnmount(() => {
  cancelAnimationFrame(animationFrame)
  window.removeEventListener('resize', onResize)
  window.removeEventListener('mousemove', onMouseMove)
  window.removeEventListener('mouseleave', onMouseLeave)
})
</script>

<style scoped>
:global(*) {
  box-sizing: border-box;
}

.page {
  position: relative;
  min-height: 100vh;
  overflow: hidden;
  font-family: Inter, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  background:
    radial-gradient(circle at top left, rgba(255, 255, 255, 0.24), transparent 30%),
    radial-gradient(circle at bottom right, rgba(255, 255, 255, 0.16), transparent 22%),
    linear-gradient(135deg, #201544 0%, #361e79 48%, #0f1028 100%);
}

.playfield {
  position: absolute;
  left: 18px;
  top: 120px;
  right: 18px;
  bottom: 18px;
  border-radius: 28px;
  border: 1px solid rgba(255, 255, 255, 0.08);
  background: rgba(255, 255, 255, 0.035);
  box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.025);
  pointer-events: none;
}

.hud {
  position: relative;
  z-index: 5;
  max-width: 980px;
  margin: 0 auto;
  padding: 24px 24px 16px;
  text-align: center;
  color: #fff;
}

.hud h1 {
  margin: 0;
  font-size: clamp(2rem, 4vw, 3.2rem);
  line-height: 1.05;
  letter-spacing: 0.03em;
  text-shadow: 0 10px 30px rgba(0, 0, 0, 0.35);
}

.hud p {
  margin: 12px auto 0;
  max-width: 820px;
  font-size: 1rem;
  line-height: 1.55;
  opacity: 0.92;
}

.stats {
  display: inline-flex;
  gap: 12px;
  flex-wrap: wrap;
  justify-content: center;
  margin-top: 16px;
}

.stats span,
.hint {
  border: 1px solid rgba(255, 255, 255, 0.16);
  background: rgba(255, 255, 255, 0.12);
  backdrop-filter: blur(8px);
  padding: 8px 14px;
  border-radius: 999px;
  font-size: 0.95rem;
}

.controls {
  display: flex;
  gap: 12px;
  justify-content: center;
  align-items: center;
  flex-wrap: wrap;
  margin-top: 14px;
}

.spell-btn,
.mode-btn {
  border: 0;
  border-radius: 16px;
  padding: 12px 18px;
  font: inherit;
  font-weight: 900;
  color: white;
  cursor: pointer;
  transition: transform 0.15s ease, filter 0.15s ease, opacity 0.15s ease;
}

.spell-btn {
  background: linear-gradient(180deg, #6fc7ff 0%, #3f7cff 100%);
  box-shadow: 0 10px 30px rgba(63, 124, 255, 0.35);
}

.mode-btn {
  background: linear-gradient(180deg, #7d6cff 0%, #5b36f2 100%);
  box-shadow: 0 10px 30px rgba(91, 54, 242, 0.28);
}

.spell-btn:hover,
.mode-btn:hover {
  transform: translateY(-1px) scale(1.02);
}

.spell-btn.disabled {
  opacity: 0.45;
  cursor: not-allowed;
  filter: grayscale(0.35);
}

.slow-overlay {
  position: absolute;
  inset: 0;
  z-index: 4;
  pointer-events: none;
  background: radial-gradient(circle at center, rgba(173, 225, 255, 0.08), rgba(120, 180, 255, 0.03));
}

.slow-text {
  position: absolute;
  top: 120px;
  left: 50%;
  transform: translateX(-50%);
  padding: 12px 18px;
  border-radius: 999px;
  color: white;
  font-weight: 900;
  background: rgba(85, 160, 255, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.25);
  backdrop-filter: blur(10px);
  box-shadow: 0 10px 30px rgba(85, 160, 255, 0.2);
}

.banner {
  position: absolute;
  border-radius: 22px;
  overflow: visible;
  user-select: none;
  cursor: pointer;
  background: linear-gradient(180deg, #fff9d8 0%, #ffd65a 100%);
  box-shadow:
    0 18px 40px rgba(0, 0, 0, 0.25),
    0 8px 18px rgba(255, 174, 0, 0.22),
    inset 0 1px 0 rgba(255, 255, 255, 0.85);
  border: 2px solid rgba(255, 255, 255, 0.7);
  transition: box-shadow 0.15s ease, filter 0.15s ease, opacity 0.12s ease, transform 0.12s ease;
  will-change: transform;
}

.banner.popping {
  pointer-events: none;
}

.banner.panic {
  box-shadow:
    0 22px 48px rgba(0, 0, 0, 0.3),
    0 12px 25px rgba(255, 60, 60, 0.2),
    inset 0 1px 0 rgba(255, 255, 255, 0.92);
}

.banner.slowed {
  filter: saturate(0.9) brightness(1.05);
}

.banner.chaos {
  filter: saturate(1.15) contrast(1.04);
}

.banner-content {
  display: flex;
  height: 100%;
  flex-direction: column;
  justify-content: center;
  gap: 8px;
  padding: 18px 18px 18px 20px;
  color: #522d00;
  text-align: left;
}

.title {
  font-size: 1.25rem;
  font-weight: 900;
  line-height: 1.1;
}

.subtitle {
  font-size: 0.92rem;
  line-height: 1.35;
  opacity: 0.88;
}

.cta {
  align-self: flex-start;
  margin-top: 2px;
  padding: 7px 12px;
  border-radius: 999px;
  background: linear-gradient(180deg, #ff4f4f 0%, #d91f52 100%);
  color: white;
  font-size: 0.85rem;
  font-weight: 800;
  letter-spacing: 0.02em;
  box-shadow: 0 6px 15px rgba(217, 31, 82, 0.28);
}

.close.fake {
  position: absolute;
  top: 10px;
  right: 11px;
  min-width: 30px;
  height: 28px;
  display: grid;
  place-items: center;
  border-radius: 999px;
  background: rgba(0, 0, 0, 0.12);
  color: rgba(0, 0, 0, 0.7);
  font-size: 0.8rem;
  font-weight: 800;
  padding: 0 8px;
}

.badge {
  position: absolute;
  top: 12px;
  left: 14px;
  padding: 5px 9px;
  border-radius: 999px;
  background: rgba(139, 41, 255, 0.14);
  color: #6f22d6;
  font-size: 0.72rem;
  font-weight: 800;
  letter-spacing: 0.03em;
}

.roach-zone {
  position: absolute;
  inset: 0;
  pointer-events: none;
}

.roach {
  position: absolute;
  left: 50%;
  top: 50%;
  font-size: 1.75rem;
  opacity: 0.95;
  filter: drop-shadow(0 6px 8px rgba(0, 0, 0, 0.25));
  will-change: transform;
}

.roach.blocker {
  font-size: 1.95rem;
}

.roach.carrier {
  font-size: 2rem;
}

.roach.berserk {
  font-size: 2.15rem;
  filter: drop-shadow(0 8px 12px rgba(255, 50, 50, 0.25));
}

.roach.slowed {
  filter: drop-shadow(0 8px 12px rgba(120, 220, 255, 0.3));
}

.overlay-card {
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  z-index: 20;
  width: min(90vw, 520px);
  text-align: center;
  padding: 28px;
  border-radius: 28px;
  background: rgba(255, 255, 255, 0.14);
  color: #fff;
  backdrop-filter: blur(16px);
  box-shadow: 0 16px 45px rgba(0, 0, 0, 0.28);
  border: 1px solid rgba(255, 255, 255, 0.16);
}

.overlay-card h2 {
  margin: 0 0 10px;
  font-size: 2rem;
}

.overlay-card p {
  margin: 0 0 18px;
  line-height: 1.5;
}

.overlay-card button {
  border: 0;
  border-radius: 14px;
  padding: 12px 18px;
  font: inherit;
  font-weight: 800;
  cursor: pointer;
  color: #28144d;
  background: #fff;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.18);
}

.banner-pop-enter-active,
.banner-pop-leave-active {
  transition: all 0.18s ease;
}

.banner-pop-enter-from,
.banner-pop-leave-to {
  opacity: 0;
  transform: scale(0.84);
}

@media (max-width: 760px) {
  .hud {
    padding-inline: 16px;
  }
}
</style>