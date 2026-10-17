<template>
  <div
    class="wallpaper"
    :style="{ '--move-ms': `${moveMs}ms`, '--flip-ms': `${flipMs}ms` }"
  >
    <!-- 所有牌放在同一层，靠 transform 决定位置，移动时会自动补间 -->
    <div class="table-layer">
      <div
        v-for="card in cards"
        :key="card.id"
        class="card"
        :class="{
          'card--dragging': isCardDragging(card),
          'card--draggable': isCardDraggable(card),
        }"
        :style="cardStyle(card)"
        @pointerdown.prevent="onCardPointerDown($event, card)"
        @pointermove="onCardPointerMove"
        @pointerup="onCardPointerUp"
        @pointercancel="cancelCardDrag"
      >
        <div class="card-inner" :class="{ 'is-flipped': card.flipped }">
          <!-- 牌背 -->
          <div class="card-face card-face--back">
            <img :src="cardBackUrl" alt="" draggable="false" />
          </div>
          <!-- 牌面：名字叠在卡面上 -->
          <div class="card-face card-face--front">
            <img :src="cardFrontUrl" alt="" draggable="false" />
            <span class="card-name">{{ card.name }}</span>
          </div>
        </div>
      </div>
    </div>

    <!-- 平铺查看时，点击任意位置收起 -->
    <div v-if="spreadOpen" class="spread-backdrop" @click="closeSpread"></div>
    <p v-if="spreadOpen" class="spread-hint">
      点击空白处收起 · 把牌拖回左上角牌堆可还原
    </p>

    <!-- 拖动卡片时，左上角显示还原区域 -->
    <div v-if="spreadDrag || stackDrag" class="deck-drop">
      <span>拖到这里还原</span>
    </div>

    <!-- 左上角牌堆：把最上面那张拖出来即可抽牌 -->
    <button
      v-if="deckIds.length > 0 && !spreadOpen"
      class="deck-hit"
      title="把牌拖出来即可抽取"
      @pointerdown.prevent="onDeckPointerDown"
      @pointermove="onDeckPointerMove"
      @pointerup="onDeckPointerUp"
      @pointercancel="cancelDeckDrag"
    ></button>

    <!-- 右上角已抽取牌堆：点击或拖出即可平铺 -->
    <button
      v-if="drawnIds.length > 0 && !spreadOpen"
      class="drawn-hit"
      title="点击或拖出平铺；拖回左上角牌堆可全部还原"
      @pointerdown.prevent="onStackPointerDown"
      @pointermove="onStackPointerMove"
      @pointerup="onStackPointerUp"
      @pointercancel="cancelStackDrag"
    >
      <span class="drawn-count">{{ drawnIds.length }}</span>
    </button>

    <!-- 底部控制栏 -->
    <div class="controls">
      <button
        class="ghost"
        title="重新洗牌，并重新读取外部 names.json"
        @click="reset"
      >
        重开
      </button>
      <span class="status">剩余 {{ deckIds.length }} 张</span>
    </div>
  </div>
</template>

<script setup>
import { computed, onMounted, onUnmounted, ref } from 'vue'
import {
  BaseDirectory,
  exists,
  mkdir,
  readTextFile,
  writeTextFile,
} from '@tauri-apps/plugin-fs'
import cardBackUrl from './assets/card_back.svg'
import cardFrontUrl from './assets/card_front.svg'
import pickCard1Url from './assets/pick_card_1.mp3'
import pickCard2Url from './assets/pick_card_2.mp3'
import placeCardUrl from './assets/place_card.mp3'
import defaultConfig from './config.json'

/* ------------- 名字列表：优先读取程序外部配置文件 ------------- */
const CONFIG_FILE = 'names.json'
const defaultNames = defaultConfig.names

// 支持两种写法：
//   1. "张三"                          → 权重 1
//   2. { "name": "张三", "weight": 2 } → 权重 2，更容易被抽到
function parseNames(json) {
  const list = Array.isArray(json) ? json : json?.names
  if (!Array.isArray(list) || list.length === 0) {
    throw new Error('names 必须是非空数组')
  }

  return list.map((item) => {
    if (typeof item === 'string') {
      return { name: item, weight: 1 }
    }

    const name = String(item?.name ?? '').trim()
    if (!name) throw new Error('每个名字都必须填写 name')

    const rawWeight = Number(item?.weight ?? 1)
    const weight =
      Number.isFinite(rawWeight) && rawWeight > 0 ? rawWeight : 1

    return { name, weight }
  })
}

if (!Array.isArray(defaultNames) || defaultNames.length === 0) {
  throw new Error('src/config.json 中至少要有一个默认名字')
}

// 先用内置名单把界面渲染出来，外部配置在后台异步读取
const names = ref(parseNames(defaultConfig))

// 外部文件不存在时，用内置配置生成一份，方便用户直接编辑
async function ensureConfigFile() {
  const fileExists = await exists(CONFIG_FILE, {
    baseDir: BaseDirectory.AppConfig,
  })
  if (fileExists) return

  await mkdir('', { baseDir: BaseDirectory.AppConfig, recursive: true }).catch(
    () => {},
  )
  await writeTextFile(
    CONFIG_FILE,
    `${JSON.stringify(defaultConfig, null, 2)}\n`,
    { baseDir: BaseDirectory.AppConfig },
  )
}

async function loadNames() {
  try {
    await ensureConfigFile()
    const text = await readTextFile(CONFIG_FILE, {
      baseDir: BaseDirectory.AppConfig,
    })
    names.value = parseNames(JSON.parse(text))
  } catch (error) {
    // 浏览器的 npm run dev 里没有 Tauri API，这里会自然回退到内置名单
    console.error('读取外部配置失败，使用内置名单：', error)
    names.value = parseNames(defaultConfig)
  }

  initCards()
}

/* ---------------- 尺寸与动画参数 ---------------- */
const CARD_W = 120
const CARD_H = 168 // 跟卡面 200x280 的比例一致
const CENTER_SCALE = 1.4
// 抽牌节奏：慢点时的基础速度（单位毫秒）
// MOVE_MS = 牌飞到中间的时间；FLIP_MS = 翻面时间
const MOVE_MS = 400
const FLIP_MS = 350

// 自适应速度：两次点击的间隔越短，动画越快
const SLOW_INTERVAL_MS = 900 // 间隔 ≥ 它：按基础速度
const MIN_MOVE_MS = 110 // 最快时的移动时间
const MIN_FLIP_MS = 90 // 最快时的翻面时间

// 连续抽牌：手速超过动画速度时，请求先排队，牌一张张累积在中间
const BURST_INTERVAL_MS = 200 // 点击间隔小于它，直接进入连续抽牌
const BURST_MAX_PENDING = 2 // 队列积压到这么多，也算连续抽牌
const BURST_IDLE_MS = 700 // 手停下来后，中间平铺停留多久再一起飞走

const DECK_LEFT = 24
const DECK_TOP = 24

// 右上角已抽牌堆
const DRAWN_SCALE = 0.62
const DRAWN_MARGIN = 24
const DRAWN_STACK_STEP = 2 // 每张牌错开 2px，形成堆叠厚度
const DRAWN_W = CARD_W * DRAWN_SCALE
const DRAWN_MAX_OFFSET = 6 // 偏移量最多叠加 6 层，避免戳出屏幕

// 平铺查看
const SPREAD_SCALE = 0.95
const SPREAD_MARGIN = 48
const SPREAD_MAX_STEP = 180 // 牌少时两张牌左上角的距离，越大越稀疏
const SPREAD_MIN_STEP = 24 // 牌多时最密能叠到多近

// 拖动超过这个距离才算“拖出来”，否则当成点击
const DRAG_THRESHOLD = 28

// 判定“拖回牌堆”时，牌堆周围额外宽容的范围
const DECK_DROP_PADDING = 40

/* ---------------- 牌堆偏移（沿用之前的堆叠效果） ---------------- */
const FAN_STEP = 1
const PILE_STEP = 1
const SIDE_STEP = 3

function stackOffsetX(index) {
  return index * SIDE_STEP
}

function stackOffsetY(index) {
  if (index < 3) return index * FAN_STEP
  return 2 * FAN_STEP + (index - 2) * PILE_STEP
}

/* ---------------- 抽牌音效 ---------------- */
const pickSounds = [new Audio(pickCard1Url), new Audio(pickCard2Url)]
pickSounds.forEach((audio) => {
  audio.preload = 'auto'
})

function playPickSound() {
  const sound = pickSounds[Math.floor(Math.random() * pickSounds.length)]
  sound.currentTime = 0
  sound.play().catch(() => {})
}

const placeSound = new Audio(placeCardUrl)
placeSound.preload = 'auto'

function playPlaceSound() {
  placeSound.currentTime = 0
  placeSound.play().catch(() => {})
}

/* ---------------- 游戏状态 ---------------- */
const cards = ref([]) // 所有牌
const deckIds = ref([]) // 牌堆顺序，第 0 个是最上面那张
const drawnIds = ref([]) // 已抽到右上角的牌
const centerIds = ref([]) // 中间展示的牌（连续抽牌时会累积多张）
const isBusy = ref(false) // 动画进行中（不再阻止新的抽牌请求）
const spreadOpen = ref(false) // 已抽取名单是否平铺展示
const dragCard = ref(null) // 正从牌堆拖出的牌 { id, startX, startY, dx, dy }
const stackDrag = ref(null) // 正被拖动的已抽牌堆 { startX, startY, dx, dy }
const spreadDrag = ref(null) // 正被拖动的单张已抽牌 { id, startX, startY, dx, dy }

// 连续抽牌的队列与速度
const drawQueue = ref([]) // 等待处理的抽牌请求
const processing = ref(false) // 是否正在逐张处理队列
const burstActive = ref(false) // 是否处于连续抽牌阶段
const moveMs = ref(MOVE_MS) // 当前移动动画时长（CSS 变量会跟着变）
const flipMs = ref(FLIP_MS) // 当前翻面动画时长

let lastRequestAt = 0 // 上一次抽牌请求的时间
let finalizeTimer = null // 连续抽牌结束后，延迟收尾的定时器

const viewportWidth = ref(window.innerWidth)

function updateViewport() {
  viewportWidth.value = window.innerWidth
}

// 平铺时，已抽牌 + 中间所有牌，一起参与排布
const spreadCount = computed(
  () => drawnIds.value.length + centerIds.value.length,
)

function openSpread() {
  if (drawnIds.value.length === 0) return
  spreadOpen.value = true
}

function closeSpread() {
  spreadOpen.value = false
}

/* ---------------- 拖动交互 ---------------- */
function releasePointer(el, event) {
  if (el?.hasPointerCapture?.(event.pointerId)) {
    el.releasePointerCapture(event.pointerId)
  }
}

// 左上角牌堆：按住最上面的牌拖出去 → 抽牌
function onDeckPointerDown(event) {
  if (spreadOpen.value || deckIds.value.length === 0) return

  dragCard.value = {
    id: deckIds.value[0],
    startX: event.clientX,
    startY: event.clientY,
    dx: 0,
    dy: 0,
  }
  event.currentTarget.setPointerCapture(event.pointerId)
}

function onDeckPointerMove(event) {
  const drag = dragCard.value
  if (!drag) return

  drag.dx = event.clientX - drag.startX
  drag.dy = event.clientY - drag.startY
}

function onDeckPointerUp(event) {
  const drag = dragCard.value
  if (!drag) return

  dragCard.value = null
  releasePointer(event.currentTarget, event)

  // 原地点击和拖出来都算一次抽牌请求
  drawCard()
}

function cancelDeckDrag(event) {
  dragCard.value = null
  releasePointer(event.currentTarget, event)
}

// 右上角牌堆：拖出来 → 平铺已抽名单
function onStackPointerDown(event) {
  if (isBusy.value || spreadOpen.value || drawnIds.value.length === 0) return

  stackDrag.value = {
    startX: event.clientX,
    startY: event.clientY,
    dx: 0,
    dy: 0,
  }
  event.currentTarget.setPointerCapture(event.pointerId)
}

function onStackPointerMove(event) {
  const drag = stackDrag.value
  if (!drag) return

  drag.dx = event.clientX - drag.startX
  drag.dy = event.clientY - drag.startY
}

function onStackPointerUp(event) {
  const drag = stackDrag.value
  if (!drag) return

  const distance = Math.hypot(drag.dx, drag.dy)
  const overDeck = isPointOverDeck(event.clientX, event.clientY)
  stackDrag.value = null
  releasePointer(event.currentTarget, event)

  // 拖回左上角牌堆 = 把已抽出的牌全部还原；否则点击/拖出都平铺
  if (distance >= DRAG_THRESHOLD && overDeck) {
    restoreAllDrawn()
  } else {
    openSpread()
  }
}

function cancelStackDrag(event) {
  stackDrag.value = null
  releasePointer(event.currentTarget, event)
}

/* ---------------- 单张牌的拖动：拖回牌堆即还原 ---------------- */
function isPointOverDeck(x, y) {
  return (
    x >= DECK_LEFT - DECK_DROP_PADDING &&
    x <= DECK_LEFT + CARD_W + DECK_DROP_PADDING &&
    y >= DECK_TOP - DECK_DROP_PADDING &&
    y <= DECK_TOP + CARD_H + DECK_DROP_PADDING
  )
}

// 平铺中的牌、以及中间展示的牌，都可以拖回牌堆
function isCardDraggable(card) {
  if (spreadDrag.value?.id === card.id) return true
  if (isBusy.value) return false
  if (spreadOpen.value) {
    return centerIds.value.includes(card.id) || drawnIds.value.includes(card.id)
  }
  return centerIds.value.includes(card.id)
}

function onCardPointerDown(event, card) {
  if (!isCardDraggable(card)) return

  spreadDrag.value = {
    id: card.id,
    startX: event.clientX,
    startY: event.clientY,
    dx: 0,
    dy: 0,
  }
  event.currentTarget.setPointerCapture(event.pointerId)
}

function onCardPointerMove(event) {
  const drag = spreadDrag.value
  if (!drag) return

  drag.dx = event.clientX - drag.startX
  drag.dy = event.clientY - drag.startY
}

function onCardPointerUp(event) {
  const drag = spreadDrag.value
  if (!drag) return

  const distance = Math.hypot(drag.dx, drag.dy)
  const overDeck = isPointOverDeck(event.clientX, event.clientY)
  const tapped = distance < DRAG_THRESHOLD

  spreadDrag.value = null
  releasePointer(event.currentTarget, event)

  if (tapped) {
    // 平铺时点一下牌，等于点击空白处，直接收起
    if (spreadOpen.value) closeSpread()
    return
  }

  if (overDeck) restoreCard(drag.id)
}

function cancelCardDrag(event) {
  spreadDrag.value = null
  releasePointer(event.currentTarget, event)
}

// 把一张牌放回牌堆顶部，并翻回背面
function restoreCard(id) {
  const card = cards.value.find((item) => item.id === id)
  if (!card) return

  const centerIndex = centerIds.value.indexOf(id)
  if (centerIndex !== -1) {
    centerIds.value.splice(centerIndex, 1)
  } else {
    const index = drawnIds.value.indexOf(id)
    if (index === -1) return
    drawnIds.value.splice(index, 1)
  }

  card.flipped = false
  deckIds.value.unshift(id)
  playPlaceSound()

  // 中间已经没有连续抽牌留下的牌了，结束这一轮
  if (centerIds.value.length === 0 && burstActive.value) {
    burstActive.value = false
    clearFinalizeTimer()
  }

  // 收起平铺，让牌飞回牌堆的过程完整可见
  if (spreadOpen.value) closeSpread()
}

// 把右上角整叠已抽牌全部放回牌堆
function restoreAllDrawn() {
  if (drawnIds.value.length === 0) return

  drawnIds.value.forEach((id) => {
    const card = cards.value.find((item) => item.id === id)
    if (card) card.flipped = false
    deckIds.value.unshift(id)
  })
  drawnIds.value = []
  playPlaceSound()
}

function onKeydown(event) {
  if (event.key === 'Escape' && spreadOpen.value) {
    closeSpread()
  }
}

onMounted(() => {
  window.addEventListener('resize', updateViewport)
  window.addEventListener('keydown', onKeydown)
})

onUnmounted(() => {
  window.removeEventListener('resize', updateViewport)
  window.removeEventListener('keydown', onKeydown)
})

/* ---------------- 洗牌与初始化 ---------------- */
// 按权重洗牌：权重越大，越容易排在牌堆前面，也就越容易被抽到。
// 用指数竞赛法（Efraimidis–Spirakis），保证每次抽牌时，
// 每张剩余牌被抽中的概率都正比于它的权重。
function weightedShuffle(entries) {
  return entries
    .map((entry) => ({
      entry,
      key: Math.pow(Math.random(), 1 / entry.weight),
    }))
    .sort((a, b) => b.key - a.key)
    .map((item) => item.entry)
}

function initCards() {
  cards.value = weightedShuffle(names.value).map((entry, index) => ({
    id: index + 1,
    name: entry.name,
    weight: entry.weight,
    flipped: false,
  }))
  deckIds.value = cards.value.map((card) => card.id)
  drawnIds.value = []
  centerIds.value = []
}

// 先用内置名单立即初始化，保证界面不会空白
initCards()

// 再异步读取外部配置；读到了会重新洗牌
loadNames()

/* ---------------- 平铺位置：数量越多，间距越小 ---------------- */
function spreadTransform(index, count, dragX = 0, dragY = 0) {
  const cardW = CARD_W * SPREAD_SCALE
  const cardH = CARD_H * SPREAD_SCALE

  // 先看“牌少时”的大间距能不能放下；放不下就自动压缩
  const available = viewportWidth.value - SPREAD_MARGIN * 2
  const fitStep = count > 1 ? (available - cardW) / (count - 1) : 0
  const step =
    count > 1
      ? Math.max(Math.min(SPREAD_MAX_STEP, fitStep), SPREAD_MIN_STEP)
      : 0

  // 以屏幕中心为基准，左右对称排开
  const offsetX = (index - (count - 1) / 2) * step

  return `translate(calc(50vw + ${
    offsetX - cardW / 2 + dragX
  }px), calc(50vh + ${dragY - cardH / 2}px)) scale(${SPREAD_SCALE})`
}

// 中间区域：单张时放大居中；连续抽牌有多张时横向平铺
function centerTransform(index, count, dragX = 0, dragY = 0) {
  if (count === 1) {
    const halfW = (CARD_W * CENTER_SCALE) / 2
    const halfH = (CARD_H * CENTER_SCALE) / 2
    return `translate(calc(50vw + ${dragX - halfW}px), calc(50vh + ${
      dragY - halfH
    }px)) scale(${CENTER_SCALE})`
  }

  return spreadTransform(index, count, dragX, dragY)
}

/* ---------------- 每张牌的位置 ---------------- */
function cardStyle(card) {
  const centerIndex = centerIds.value.indexOf(card.id)
  const drawnIndex = drawnIds.value.indexOf(card.id)
  const deckIndex = deckIds.value.indexOf(card.id)

  // 单张牌拖动时，把它跟着指针的偏移量算进去
  const spreadOffset = spreadDrag.value?.id === card.id ? spreadDrag.value : null
  const cardDx = spreadOffset?.dx ?? 0
  const cardDy = spreadOffset?.dy ?? 0

  let transform
  if (centerIndex !== -1) {
    if (spreadOpen.value) {
      // 平铺查看时，中间的牌排在已抽牌后面
      transform = spreadTransform(
        drawnIds.value.length + centerIndex,
        spreadCount.value,
        cardDx,
        cardDy,
      )
    } else {
      transform = centerTransform(
        centerIndex,
        centerIds.value.length,
        cardDx,
        cardDy,
      )
    }
  } else if (drawnIndex !== -1) {
    if (spreadOpen.value) {
      // 平铺：横向排开
      transform = spreadTransform(
        drawnIndex,
        spreadCount.value,
        cardDx,
        cardDy,
      )
    } else {
      // 右上角：所有牌叠在同一个位置，只错开一点厚度
      const offset = Math.min(drawnIndex, DRAWN_MAX_OFFSET) * DRAWN_STACK_STEP
      const dragX = stackDrag.value?.dx ?? 0
      const dragY = stackDrag.value?.dy ?? 0
      const x = viewportWidth.value - DRAWN_MARGIN - DRAWN_W + offset + dragX
      const y = DRAWN_MARGIN + offset + dragY
      transform = `translate(${x}px, ${y}px) scale(${DRAWN_SCALE})`
    }
  } else {
    // 左上角牌堆（拖动时跟随指针）
    const drag = dragCard.value?.id === card.id ? dragCard.value : null
    const x = DECK_LEFT + stackOffsetX(deckIndex) + (drag?.dx ?? 0)
    const y = DECK_TOP + stackOffsetY(deckIndex) + (drag?.dy ?? 0)
    transform = `translate(${x}px, ${y}px) scale(1)`
  }

  let zIndex = 1
  if (spreadOpen.value && (centerIndex !== -1 || drawnIndex !== -1)) {
    // 平铺的牌要盖在遮罩上方，越新的越靠上
    zIndex =
      400 +
      (centerIndex !== -1 ? drawnIds.value.length + centerIndex : drawnIndex)
  } else {
    if (drawnIndex !== -1) zIndex = 50 + drawnIndex
    if (deckIndex !== -1) zIndex = 100 - deckIndex
    if (centerIndex !== -1) zIndex = 500 + centerIndex
  }

  // 拖动中的牌要浮在最上面
  if (dragCard.value?.id === card.id) zIndex = 550
  if (stackDrag.value && drawnIndex !== -1) zIndex = 560 + drawnIndex
  if (spreadDrag.value?.id === card.id) zIndex = 700

  return { transform, zIndex }
}

function isCardDragging(card) {
  if (dragCard.value?.id === card.id) return true
  if (spreadDrag.value?.id === card.id) return true
  return Boolean(stackDrag.value) && drawnIds.value.includes(card.id)
}

/* ---------------- 动画小工具 ---------------- */
function wait(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms))
}

// 重置时作废旧动画，避免回调继续执行
let runId = 0

function clearFinalizeTimer() {
  if (finalizeTimer !== null) {
    clearTimeout(finalizeTimer)
    finalizeTimer = null
  }
}

/* ---------------- 抽牌请求：慢点原速，快点加速，再快就排队 ---------------- */
function drawCard() {
  if (deckIds.value.length === 0 && centerIds.value.length === 0) return

  const now = performance.now()
  const interval = lastRequestAt === 0 ? Infinity : now - lastRequestAt
  lastRequestAt = now

  // 手速比动画快 → 进入连续抽牌
  if (
    interval < BURST_INTERVAL_MS ||
    drawQueue.value.length >= BURST_MAX_PENDING
  ) {
    burstActive.value = true
  }

  spreadOpen.value = false
  drawQueue.value.push({ interval })
  clearFinalizeTimer()

  if (!processing.value) processQueue()
}

async function processQueue() {
  if (processing.value) return

  const myRun = runId
  processing.value = true
  isBusy.value = true

  try {
    while (drawQueue.value.length > 0) {
      if (myRun !== runId) return
      const request = drawQueue.value.shift()
      await performDraw(request)
      if (myRun !== runId) return
    }
  } finally {
    if (myRun === runId) {
      processing.value = false
      isBusy.value = false
      // 队列处理完后恢复正常速度，供后续平铺 / 还原等动画使用
      moveMs.value = MOVE_MS
      flipMs.value = FLIP_MS
      scheduleFinalize()
    }
  }
}

async function performDraw(request) {
  // 两次点击间隔越短，动画越快；队列有积压时直接用最快速度追
  const interval = Number.isFinite(request.interval)
    ? request.interval
    : SLOW_INTERVAL_MS
  const ratio =
    drawQueue.value.length > 0
      ? 0.3
      : Math.min(1, Math.max(0.3, interval / SLOW_INTERVAL_MS))

  const speedMove = Math.max(MIN_MOVE_MS, Math.round(MOVE_MS * ratio))
  const speedFlip = Math.max(MIN_FLIP_MS, Math.round(FLIP_MS * ratio))
  moveMs.value = speedMove
  flipMs.value = speedFlip

  // 连续抽牌：牌只往中间平铺，先不送去右上角
  if (burstActive.value) {
    if (deckIds.value.length === 0) return

    playPickSound()
    const id = deckIds.value.shift()
    const card = cards.value.find((item) => item.id === id)
    card.flipped = false
    centerIds.value.push(id)

    // 连续抽牌时不落地再翻，直接边飞边翻，节省时间
    card.flipped = true
    await wait(Math.max(speedMove, speedFlip))
    return
  }

  // 普通模式：上一张保持正面，先飞去右上角
  if (centerIds.value.length > 0) {
    drawnIds.value.push(...centerIds.value)
    centerIds.value = []
  }

  if (deckIds.value.length > 0) {
    playPickSound()

    const id = deckIds.value.shift()
    const card = cards.value.find((item) => item.id === id)
    card.flipped = false
    centerIds.value = [id]

    // 等它到位后翻面
    await wait(speedMove)
    card.flipped = true
    await wait(speedFlip)
  } else {
    // 牌堆空了，只把中间的牌送去右上角
    await wait(speedMove)
  }
}

function scheduleFinalize() {
  clearFinalizeTimer()
  if (!burstActive.value) return

  // 中间已经没牌了，直接结束这一轮连续抽牌
  if (centerIds.value.length === 0) {
    burstActive.value = false
    return
  }

  // 用户正在平铺查看时先不打断，等收起后再收尾
  if (spreadOpen.value) {
    finalizeTimer = setTimeout(scheduleFinalize, 400)
    return
  }

  finalizeTimer = setTimeout(finalizeBurst, BURST_IDLE_MS)
}

// 连续抽牌结束：中间累积的牌一起飞到右上角
function finalizeBurst() {
  finalizeTimer = null

  if (spreadOpen.value) {
    finalizeTimer = setTimeout(scheduleFinalize, 400)
    return
  }

  if (centerIds.value.length > 0) {
    // 收尾这一段用正常速度，让“一起飞走”看得清楚
    moveMs.value = MOVE_MS
    flipMs.value = FLIP_MS
    drawnIds.value.push(...centerIds.value)
    centerIds.value = []
    playPlaceSound()
  }

  burstActive.value = false
  lastRequestAt = 0
}

async function reset() {
  runId++
  clearFinalizeTimer()
  drawQueue.value = []
  burstActive.value = false
  processing.value = false
  isBusy.value = false
  lastRequestAt = 0
  moveMs.value = MOVE_MS
  flipMs.value = FLIP_MS
  spreadOpen.value = false

  // 重新读取外部文件，用户改完 names.json 后点「重开」即可生效
  await loadNames()
}
</script>

<style>
.wallpaper {
  position: relative;
  min-height: 100vh;
  overflow: hidden;
  background-image: url('./assets/background.svg');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  background-attachment: fixed;
}

.table-layer {
  position: absolute;
  inset: 0;
  pointer-events: none;
}

/* 卡片外壳：只负责移动和缩放 */
.card {
  position: absolute;
  top: 0;
  left: 0;
  width: 120px;
  height: 168px;
  transform-origin: top left;
  transition: transform var(--move-ms, 400ms) cubic-bezier(0.22, 1, 0.36, 1);
  will-change: transform;
  perspective: 1200px;
}

/* 卡片内层：只负责翻面 */
.card-inner {
  position: relative;
  width: 100%;
  height: 100%;
  transform-style: preserve-3d;
  transition: transform var(--flip-ms, 350ms) cubic-bezier(0.22, 1, 0.36, 1);
}

.card-inner.is-flipped {
  transform: rotateY(180deg);
}

.card-face {
  position: absolute;
  inset: 0;
  overflow: hidden;
  border-radius: 10px;
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
  box-shadow: 0 10px 22px rgb(0 0 0 / 35%);
}

.card-face img {
  display: block;
  width: 100%;
  height: 100%;
  user-select: none;
  -webkit-user-drag: none;
}

/* 牌面默认朝后，翻面时才转到正面 */
.card-face--front {
  transform: rotateY(180deg);
  background: #f7f3e8;
}

/* 名字叠在牌面上 */
.card-name {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 14px;
  text-align: center;
  font-family: var(--font-card);
  font-size: 30px;
  font-weight: 700;
  line-height: 1.2;
  letter-spacing: 1px;
  color: #362f24;
  overflow-wrap: anywhere;
}

/* ---------------- 平铺查看 ---------------- */
.spread-backdrop {
  position: fixed;
  inset: 0;
  z-index: 300;
  background: rgb(5 8 14 / 45%);
  backdrop-filter: blur(3px);
  animation: backdrop-in 220ms ease both;
}

@keyframes backdrop-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

.spread-hint {
  position: fixed;
  left: 50%;
  bottom: 24px;
  z-index: 450;
  margin: 0;
  transform: translateX(-50%);
  color: var(--color-muted);
  font-size: 12px;
  letter-spacing: 1px;
  pointer-events: none;
  animation: hint-in 300ms ease both;
}

@keyframes hint-in {
  from {
    opacity: 0;
    transform: translate(-50%, 8px);
  }
  to {
    opacity: 1;
    transform: translate(-50%, 0);
  }
}

/* 拖动中的牌不需要过渡，直接跟手 */
.card--dragging {
  transition: none;
}

/* 只有能拖动的牌才接收指针事件（table-layer 默认关闭了） */
.card--draggable {
  pointer-events: auto;
  cursor: grab;
  touch-action: none;
}

.card--draggable:active {
  cursor: grabbing;
}

/* 拖动单张 / 整叠牌时，左上角高亮“还原落点” */
.deck-drop {
  position: absolute;
  top: 8px;
  left: 8px;
  z-index: 620;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 152px;
  height: 200px;
  border: 2px dashed rgb(201 162 39 / 85%);
  border-radius: 14px;
  background: rgb(201 162 39 / 12%);
  color: #f2ead8;
  font-size: 12px;
  letter-spacing: 1px;
  pointer-events: none;
  animation: drop-zone-in 160ms ease both;
}

@keyframes drop-zone-in {
  from {
    opacity: 0;
    transform: scale(0.96);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

/* ---------------- 牌堆拖动热区 ---------------- */
.deck-hit {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 600;
  width: 128px;
  height: 176px;
  padding: 0;
  border: 0;
  background: transparent;
  cursor: grab;
  touch-action: none;
}

.deck-hit:active {
  cursor: grabbing;
}

/* ---------------- 右上角牌堆的点击热区 ---------------- */
.drawn-hit {
  position: absolute;
  top: 20px;
  right: 16px;
  z-index: 600;
  width: 96px;
  height: 126px;
  padding: 0;
  border: 0;
  background: transparent;
  cursor: grab;
  touch-action: none;
}

.drawn-hit:active {
  cursor: grabbing;
}

.drawn-count {
  position: absolute;
  right: 0;
  bottom: 0;
  display: grid;
  place-items: center;
  min-width: 26px;
  height: 26px;
  padding: 0 7px;
  border-radius: 13px;
  background: rgb(201 162 39 / 92%);
  color: #14171f;
  font-size: 13px;
  font-weight: 700;
  box-shadow: 0 4px 10px rgb(0 0 0 / 35%);
}

/* ---------------- 底部控制栏 ---------------- */
.controls {
  position: absolute;
  left: 24px;
  bottom: 24px;
  display: flex;
  align-items: center;
  gap: 10px;
}

.controls button {
  padding: 8px 18px;
  border: 1px solid rgb(201 162 39 / 45%);
  border-radius: 8px;
  background: rgb(20 23 31 / 75%);
  color: #f2ead8;
  font: inherit;
  cursor: pointer;
  transition:
    background 150ms ease,
    transform 150ms ease,
    opacity 150ms ease;
}

.controls button:hover:not(:disabled) {
  background: rgb(30 35 46 / 90%);
}

.controls button:active:not(:disabled) {
  transform: scale(0.96);
}

.controls button:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}

.controls .status {
  margin-left: 4px;
  color: var(--color-muted);
  font-size: 13px;
}
</style>
