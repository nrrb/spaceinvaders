<script setup>
import { onBeforeUnmount, onMounted, reactive, ref } from 'vue'

const canvasRef = ref(null)
const state = reactive({
  score: 0,
  lives: 3,
  wave: 1,
  status: 'ready', // ready | running | over
})

const player = reactive({
  x: 0,
  y: 0,
  width: 60,
  height: 18,
  speed: 380,
  cooldown: 0,
})

const keys = reactive({
  left: false,
  right: false,
  shoot: false,
})

let invaders = []
let playerBullets = []
let invaderBullets = []
let stars = []
let invaderDirection = 1
let invaderSpeed = 80
let invaderFireTimer = 1
let animationFrameId
let lastTime = 0

const margins = {
  left: 40,
  right: 40,
  top: 60,
}

const invaderSize = {
  width: 28,
  height: 18,
}

const invaderPattern = [
  ' ## # # ##  ### ##  # #  #  # # # # ',
  '#   # # # # #   # # ### # # # # # # ',
  '#    #  ##  ##  ##  ### ###  #   #  ',
  '#    #  # # #   # # # # # # # # # # ',
  ' ##  #  ##  ### # # # # # # # # # # ',
]

const bulletSizes = {
  player: { w: 3, h: 12, dy: -540 },
  invader: { w: 4, h: 16, dyBase: 240 },
}

const clamp = (value, min, max) => Math.min(Math.max(value, min), max)
const intersects = (a, b) =>
  a.x < b.x + b.width &&
  a.x + a.width > b.x &&
  a.y < b.y + b.height &&
  a.height + a.y > b.y

const resizeCanvas = () => {
  const canvas = canvasRef.value
  if (!canvas) return
  canvas.width = window.innerWidth
  canvas.height = window.innerHeight
  player.y = canvas.height - 90
  if (!stars.length) {
    stars = Array.from({ length: 90 }, () => ({
      x: Math.random() * canvas.width,
      y: Math.random() * canvas.height,
      size: Math.random() * 2 + 0.5,
      speed: 10 + Math.random() * 30,
    }))
  } else {
    stars = stars.map((star) => ({
      ...star,
      x: Math.random() * canvas.width,
      y: Math.random() * canvas.height,
    }))
  }
  if (state.status === 'ready') {
    centerPlayer()
  }
}

const centerPlayer = () => {
  const canvas = canvasRef.value
  if (!canvas) return
  player.x = canvas.width / 2 - player.width / 2
}

const resetGame = () => {
  state.score = 0
  state.lives = 3
  state.wave = 1
  invaderSpeed = 80
  invaderDirection = 1
  state.status = 'running'
  playerBullets = []
  invaderBullets = []
  createWave()
  centerPlayer()
}

const startGame = () => {
  if (state.status === 'running') return
  if (state.status === 'ready') {
    resetGame()
    return
  }
  if (state.status === 'over') {
    resetGame()
  }
}

const createWave = () => {
  const canvas = canvasRef.value
  if (!canvas) return
  invaders = []
  const spacingX = 28
  const spacingY = 34
  const patternWidth = invaderPattern[0].length * spacingX
  const startX = Math.max(
    margins.left,
    (canvas.width - patternWidth) / 2,
  )
  const startY = margins.top

  for (let row = 0; row < invaderPattern.length; row += 1) {
    for (let col = 0; col < invaderPattern[row].length; col += 1) {
      if (invaderPattern[row][col] === '#') {
        invaders.push({
          x: startX + col * spacingX,
          y: startY + row * spacingY,
          width: invaderSize.width,
          height: invaderSize.height,
        })
      }
    }
  }
}

const update = (dt) => {
  const canvas = canvasRef.value
  if (!canvas || state.status !== 'running') return

  player.cooldown = Math.max(0, player.cooldown - dt)

  if (keys.left) {
    player.x -= player.speed * dt
  }
  if (keys.right) {
    player.x += player.speed * dt
  }
  player.x = clamp(player.x, margins.left, canvas.width - player.width - margins.right)

  if (keys.shoot && player.cooldown <= 0) {
    playerBullets.push({
      x: player.x + player.width / 2 - bulletSizes.player.w / 2,
      y: player.y - 6,
      width: bulletSizes.player.w,
      height: bulletSizes.player.h,
      dy: bulletSizes.player.dy,
    })
    player.cooldown = 0.28
  }

  playerBullets.forEach((bullet) => {
    bullet.y += bullet.dy * dt
  })
  playerBullets = playerBullets.filter((b) => b.y + b.height > 0)

  invaders.forEach((invader) => {
    invader.x += invaderDirection * invaderSpeed * dt
  })

  let minX = Infinity
  let maxX = -Infinity
  invaders.forEach((inv) => {
    minX = Math.min(minX, inv.x)
    maxX = Math.max(maxX, inv.x + inv.width)
  })

  if (minX < margins.left || maxX > canvas.width - margins.right) {
    invaderDirection *= -1
    invaders.forEach((inv) => {
      inv.y += 22
    })
  }

  invaderFireTimer -= dt
  if (invaderFireTimer <= 0 && invaders.length) {
    const shooter = pickShooter()
    if (shooter) {
      invaderBullets.push({
        x: shooter.x + shooter.width / 2 - bulletSizes.invader.w / 2,
        y: shooter.y + shooter.height,
        width: bulletSizes.invader.w,
        height: bulletSizes.invader.h,
        dy: bulletSizes.invader.dyBase + state.wave * 12,
      })
    }
    invaderFireTimer = Math.max(0.45, 1.4 - state.wave * 0.12)
  }

  invaderBullets.forEach((bullet) => {
    bullet.y += bullet.dy * dt
  })
  invaderBullets = invaderBullets.filter((b) => b.y < canvas.height + 30)

  for (let i = playerBullets.length - 1; i >= 0; i -= 1) {
    const bullet = playerBullets[i]
    for (let j = invaders.length - 1; j >= 0; j -= 1) {
      const inv = invaders[j]
      if (intersects(bullet, inv)) {
        playerBullets.splice(i, 1)
        invaders.splice(j, 1)
        state.score += 100
        invaderSpeed += 1.5
        break
      }
    }
  }

  for (let i = invaderBullets.length - 1; i >= 0; i -= 1) {
    const bullet = invaderBullets[i]
    if (intersects(bullet, player)) {
      invaderBullets.splice(i, 1)
      state.lives -= 1
      centerPlayer()
      player.cooldown = 0.4
      if (state.lives <= 0) {
        state.status = 'over'
      }
    }
  }

  invaders.forEach((inv) => {
    if (inv.y + inv.height >= player.y - 6) {
      state.status = 'over'
    }
  })

  if (!invaders.length && state.status === 'running') {
    state.wave += 1
    invaderSpeed += 12
    state.lives = Math.min(5, state.lives + 1)
    createWave()
  }
}

const pickShooter = () => {
  if (!invaders.length) return null
  const sorted = [...invaders].sort((a, b) => b.y - a.y)
  const slice = sorted.slice(0, Math.max(1, Math.floor(sorted.length * 0.5)))
  return slice[Math.floor(Math.random() * slice.length)]
}

const drawShip = (ctx) => {
  ctx.fillStyle = '#8ef8ff'
  ctx.fillRect(player.x, player.y, player.width, player.height)
  ctx.fillStyle = '#b1ff66'
  ctx.fillRect(player.x + player.width / 2 - 4, player.y - 8, 8, 8)
}

const drawInvader = (ctx, inv) => {
  ctx.fillStyle = '#ff7c9d'
  ctx.fillRect(inv.x, inv.y, inv.width, inv.height)
  ctx.fillStyle = '#ffe98a'
  ctx.fillRect(inv.x + 8, inv.y + 6, inv.width - 16, inv.height - 12)
}

const drawBackground = (ctx, canvas, dt) => {
  const gradient = ctx.createLinearGradient(0, 0, 0, canvas.height)
  gradient.addColorStop(0, '#0a0f1f')
  gradient.addColorStop(1, '#05060e')
  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, canvas.width, canvas.height)

  ctx.fillStyle = '#ffffff10'
  ctx.fillRect(0, 0, canvas.width, canvas.height)

  ctx.fillStyle = '#b5d8ff'
  stars.forEach((star) => {
    ctx.beginPath()
    ctx.arc(star.x, star.y, star.size, 0, Math.PI * 2)
    ctx.fill()
    star.y += star.speed * dt
    if (star.y > canvas.height) {
      star.y = -2
      star.x = Math.random() * canvas.width
    }
  })
}

const draw = (dt) => {
  const canvas = canvasRef.value
  if (!canvas) return
  const ctx = canvas.getContext('2d')
  drawBackground(ctx, canvas, dt)

  playerBullets.forEach((bullet) => {
    ctx.fillStyle = '#9efcff'
    ctx.fillRect(bullet.x, bullet.y, bullet.width, bullet.height)
  })

  invaderBullets.forEach((bullet) => {
    ctx.fillStyle = '#ff9a3c'
    ctx.fillRect(bullet.x, bullet.y, bullet.width, bullet.height)
  })

  invaders.forEach((inv) => drawInvader(ctx, inv))
  drawShip(ctx)

  if (state.status !== 'running') {
    ctx.fillStyle = '#00000080'
    ctx.fillRect(0, 0, canvas.width, canvas.height)
  }
}

const frame = (timestamp) => {
  const dt = (timestamp - lastTime) / 1000 || 0
  lastTime = timestamp
  update(dt)
  draw(dt)
  animationFrameId = requestAnimationFrame(frame)
}

const handleKeyDown = (event) => {
  if (event.key === 'ArrowLeft' || event.key.toLowerCase() === 'a') keys.left = true
  if (event.key === 'ArrowRight' || event.key.toLowerCase() === 'd') keys.right = true
  if (event.code === 'Space') keys.shoot = true
  if (event.key.toLowerCase() === 'r') startGame()
  if (state.status !== 'running' && event.code === 'Space') {
    startGame()
  }
}

const handleKeyUp = (event) => {
  if (event.key === 'ArrowLeft' || event.key.toLowerCase() === 'a') keys.left = false
  if (event.key === 'ArrowRight' || event.key.toLowerCase() === 'd') keys.right = false
  if (event.code === 'Space') keys.shoot = false
}

onMounted(() => {
  resizeCanvas()
  window.addEventListener('resize', resizeCanvas)
  window.addEventListener('keydown', handleKeyDown)
  window.addEventListener('keyup', handleKeyUp)
  animationFrameId = requestAnimationFrame(frame)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', resizeCanvas)
  window.removeEventListener('keydown', handleKeyDown)
  window.removeEventListener('keyup', handleKeyUp)
  cancelAnimationFrame(animationFrameId)
})
</script>

<template>
  <div class="shell">
    <canvas ref="canvasRef" class="game-canvas"></canvas>
    <div class="hud">
      <div class="hud-left">
        <p class="title">Space Invaders</p>
        <p class="stats">Score {{ state.score }} · Lives {{ state.lives }} · Wave {{ state.wave }}</p>
      </div>
      <div class="hud-right">
        <span>Move: ← → / A D</span>
        <span>Shoot: Space</span>
        <span>Restart: R</span>
      </div>
    </div>
    <div v-if="state.status !== 'running'" class="overlay">
      <div class="overlay-card">
        <p class="overlay-title">
          {{ state.status === 'over' ? 'Game Over' : 'Ready?' }}
        </p>
        <p class="overlay-sub">
          {{ state.status === 'over' ? 'Earth falls. Want another run?' : 'Clear the waves before they reach you.' }}
        </p>
        <button class="btn" @click="startGame">
          {{ state.status === 'over' ? 'Play Again' : 'Start' }}
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.shell {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.game-canvas {
  width: 100%;
  height: 100%;
  display: block;
  background: transparent;
}

.hud {
  position: absolute;
  top: 20px;
  left: 20px;
  right: 20px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  pointer-events: none;
  color: #e8f4ff;
  text-shadow: 0 0 12px #00f6ff55;
}

.title {
  font-size: 1.2rem;
  letter-spacing: 0.06em;
  margin-bottom: 4px;
}

.stats {
  font-size: 0.95rem;
  opacity: 0.9;
}

.hud-right {
  display: grid;
  gap: 2px;
  text-align: right;
  font-size: 0.9rem;
}

.overlay {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  background: radial-gradient(circle at 50% 30%, #0a122b80, #05060fdd 55%, #05060f);
}

.overlay-card {
  background: linear-gradient(145deg, #0e1633, #0a0f22);
  border: 1px solid #2e4a7a;
  box-shadow: 0 10px 40px #01030a;
  padding: 22px 26px;
  width: min(420px, 90vw);
  border-radius: 12px;
  color: #e8f4ff;
  text-align: center;
}

.overlay-title {
  font-size: 1.4rem;
  letter-spacing: 0.08em;
  margin-bottom: 6px;
}

.overlay-sub {
  font-size: 1rem;
  opacity: 0.8;
  margin-bottom: 16px;
}

.btn {
  background: linear-gradient(90deg, #41dfff, #7cffc7);
  color: #041224;
  font-weight: 700;
  border: none;
  border-radius: 999px;
  padding: 10px 18px;
  cursor: pointer;
  box-shadow: 0 8px 20px #00c4ff40;
  transition: transform 0.1s ease, box-shadow 0.1s ease;
  pointer-events: auto;
}

.btn:active {
  transform: translateY(1px);
  box-shadow: 0 4px 12px #00c4ff30;
}

@media (max-width: 640px) {
  .hud {
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }

  .hud-right {
    text-align: left;
  }
}
</style>
