<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import { useRouter } from 'vue-router'
import heroBg from '@/assets/hero.png'

const router = useRouter()

// ─── Canvas 墨迹聯灰效果（移植自 mimo.xiaomi.com）─────────────────
const canvasRef = ref<HTMLCanvasElement | null>(null)

// 各项常量（对齐 mimo 原始参数）
const R_START    = 8        // 墨点初始半径
const R_END      = 140      // 墨点最大半径
const R_VARY     = 0.45     // 每个墨点大小随机波动幅度
const LIFETIME   = 560      // 墨点存活时间 ms
const STAMP_STEP = 12       // 每隔多少像素投一个墨点
const MAX_STAMPS = 160      // 同时存活墨点上限
const DPR        = Math.min(window.devicePixelRatio || 1, 2)

type Stamp = { x: number; y: number; born: number; seed: number; rmax: number }

let canvasW = 0
let canvasH = 0
let stamps: Stamp[] = []
let trailLastX: number | null = null
let trailLastY: number | null = null
let running = false
let animId  = 0

// 向路径上按间距投放墨点
function addStamp(x: number, y: number) {
  if (stamps.length >= MAX_STAMPS) stamps.shift()
  stamps.push({
    x, y,
    born: performance.now(),
    seed: Math.random() * Math.PI * 2,
    rmax: R_END * (1 - R_VARY + Math.random() * R_VARY),
  })
}

function stampAlong(x: number, y: number) {
  if (trailLastX === null) {
    addStamp(x, y)
  } else {
    const dx = x - trailLastX
    const dy = y - trailLastY!
    const dist = Math.hypot(dx, dy)
    const steps = Math.max(1, Math.ceil(dist / STAMP_STEP))
    for (let i = 1; i <= steps; i++) {
      addStamp(trailLastX + (dx * i) / steps, trailLastY! + (dy * i) / steps)
    }
  }
  trailLastX = x
  trailLastY = y
}

// 画一个不规则墨渍圆（模拟墨水晕散边缘）
function carveInk(ctx: CanvasRenderingContext2D, x: number, y: number, r: number, alpha: number, seed: number) {
  const g = ctx.createRadialGradient(x, y, r * 0.25, x, y, r)
  g.addColorStop(0,    `rgba(0,0,0,${0.95 * alpha})`)
  g.addColorStop(0.55, `rgba(0,0,0,${0.88 * alpha})`)
  g.addColorStop(1,    'rgba(0,0,0,0)')
  ctx.fillStyle = g
  ctx.beginPath()
  const segs = 32
  for (let i = 0; i <= segs; i++) {
    const a   = (i / segs) * Math.PI * 2
    const wob = 0.78
      + 0.14 * Math.sin(a * 3  + seed)
      + 0.08 * Math.sin(a * 7  + seed * 2.1)
      + 0.05 * Math.sin(a * 13 + seed * 0.7)
    const rr = r * wob
    const px = x + Math.cos(a) * rr
    const py = y + Math.sin(a) * rr
    if (i === 0) ctx.moveTo(px, py)
    else         ctx.lineTo(px, py)
  }
  ctx.closePath()
  ctx.fill()
}

function loop() {
  const canvas = canvasRef.value
  if (!canvas) return
  const ctx = canvas.getContext('2d')
  if (!ctx) return

  const now = performance.now()

  // 重新铺满黑色蒙层
ctx.globalCompositeOperation = 'source-over'
  ctx.fillStyle = '#000'
  ctx.fillRect(0, 0, canvasW, canvasH)

  // 逐一在蒙层上挖出墨点小洞
  ctx.globalCompositeOperation = 'destination-out'
  for (let i = stamps.length - 1; i >= 0; i--) {
    const t = (now - stamps[i].born) / LIFETIME
    if (t >= 1) { stamps.splice(i, 1); continue }
    const ease  = 1 - Math.pow(1 - t, 3)          // easeOutCubic 扭大
    const r     = R_START + (stamps[i].rmax - R_START) * ease
    const alpha = 1 - t * t                        // 渐渐消隐
    carveInk(ctx, stamps[i].x, stamps[i].y, r, alpha, stamps[i].seed)
  }

  if (stamps.length > 0) {
    animId = requestAnimationFrame(loop)
  } else {
    running = false
  }
}

function startLoop() {
  if (!running) {
    running = true
    animId = requestAnimationFrame(loop)
  }
}

function resizeCanvas() {
  const canvas = canvasRef.value
  if (!canvas) return
  const rect = canvas.parentElement!.getBoundingClientRect()
  canvasW = rect.width
  canvasH = rect.height
  canvas.width  = Math.round(canvasW * DPR)
  canvas.height = Math.round(canvasH * DPR)
  canvas.style.width  = canvasW + 'px'
  canvas.style.height = canvasH + 'px'
  const ctx = canvas.getContext('2d')
  if (!ctx) return
  ctx.setTransform(DPR, 0, 0, DPR, 0, 0)
  // 初始全黑
  ctx.globalCompositeOperation = 'source-over'
  ctx.fillStyle = '#000'
  ctx.fillRect(0, 0, canvasW, canvasH)
}

function onMouseEnter(e: MouseEvent) {
  const rect = (e.currentTarget as HTMLElement).getBoundingClientRect()
  trailLastX = e.clientX - rect.left
  trailLastY = e.clientY - rect.top
  stampAlong(trailLastX, trailLastY)
  startLoop()
}

function onMouseMove(e: MouseEvent) {
  const rect = (e.currentTarget as HTMLElement).getBoundingClientRect()
  stampAlong(e.clientX - rect.left, e.clientY - rect.top)
  startLoop()
}

function onMouseLeave() {
  trailLastX = null
  trailLastY = null
}

onMounted(() => {
  resizeCanvas()
  window.addEventListener('resize', resizeCanvas)
})
onUnmounted(() => {
  cancelAnimationFrame(animId)
  window.removeEventListener('resize', resizeCanvas)
})

// ─── 登录表单 ─────────────────────────────────────────────
// ─── Mock 用户库 ────────────────────────────────────────
interface MockUser { username: string; password: string }

function getUsers(): MockUser[] {
  const raw = localStorage.getItem('mock_users')
  if (raw) return JSON.parse(raw) as MockUser[]
  const defaults: MockUser[] = [
    { username: 'admin', password: '123456' },
    { username: 'test',  password: '123456' },
  ]
  localStorage.setItem('mock_users', JSON.stringify(defaults))
  return defaults
}

function saveUsers(users: MockUser[]) {
  localStorage.setItem('mock_users', JSON.stringify(users))
}

// ─── 表单状态 ────────────────────────────────────────
type Mode = 'login' | 'register'
const mode    = ref<Mode>('login')

const loginForm = ref({ username: '', password: '' })
const regForm   = ref({ username: '', password: '', confirm: '' })

const loading    = ref(false)
const shake      = ref(false)
const errorMsg   = ref('')
const successMsg = ref('')

function showError(msg: string) {
  errorMsg.value = msg
  successMsg.value = ''
  shake.value = true
  setTimeout(() => (shake.value = false), 500)
  setTimeout(() => (errorMsg.value = ''), 3000)
}

function showSuccess(msg: string) {
  successMsg.value = msg
  errorMsg.value = ''
  setTimeout(() => (successMsg.value = ''), 3000)
}

function switchMode(m: Mode) {
  mode.value = m
  errorMsg.value = ''
  successMsg.value = ''
  loginForm.value = { username: '', password: '' }
  regForm.value   = { username: '', password: '', confirm: '' }
}

function handleLogin() {
  const { username, password } = loginForm.value
  if (!username || !password) { showError('请完整填写账号和密码'); return }
  loading.value = true
  setTimeout(() => {
    const users = getUsers()
    const match = users.find(u => u.username === username && u.password === password)
    if (match) {
      localStorage.setItem('token', 'mock-token')
      router.push('/dashboard')
    } else {
      showError('账号或密码错误，请重试')
    }
    loading.value = false
  }, 700)
}

function handleRegister() {
  const { username, password, confirm } = regForm.value
  if (!username || !password || !confirm) { showError('请完整填写所有字段'); return }
  if (username.length < 3)  { showError('账号至少 3 个字符'); return }
  if (password.length < 6)  { showError('密码至少 6 个字符'); return }
  if (password !== confirm)  { showError('两次密码输入不一致'); return }
  loading.value = true
  setTimeout(() => {
    const users = getUsers()
    if (users.find(u => u.username === username)) {
      showError('该账号已存在')
    } else {
      users.push({ username, password })
      saveUsers(users)
      showSuccess('注册成功！请登录')
      setTimeout(() => switchMode('login'), 1200)
    }
    loading.value = false
  }, 700)
}
</script>

<template>
  <div
    class="login-page"
    @mouseenter="onMouseEnter"
    @mousemove="onMouseMove"
    @mouseleave="onMouseLeave"
  >
    <!-- 底层背景图 -->
    <img class="bg-image" :src="heroBg" alt="" />

    <!-- Canvas 聚光灯遗罩 -->
    <canvas ref="canvasRef" class="spotlight-canvas" />

    <!-- 卡片 -->
    <div class="card" :class="{ shake }">
      <p class="brand">后台管理系统</p>

      <!-- 模式切换 Tab -->
      <div class="tabs">
        <button :class="['tab', { active: mode === 'login' }]" @click="switchMode('login')">登 录</button>
        <button :class="['tab', { active: mode === 'register' }]" @click="switchMode('register')">注 册</button>
      </div>

      <!-- 登录表单 -->
      <template v-if="mode === 'login'">
        <div class="field">
          <label>账号</label>
          <input v-model="loginForm.username" type="text" placeholder="请输入账号" autocomplete="username" @keyup.enter="handleLogin" />
        </div>
        <div class="field">
          <label>密码</label>
          <input v-model="loginForm.password" type="password" placeholder="请输入密码" autocomplete="current-password" @keyup.enter="handleLogin" />
        </div>
      </template>

      <!-- 注册表单 -->
      <template v-else>
        <div class="field">
          <label>账号</label>
          <input v-model="regForm.username" type="text" placeholder="至少 3 个字符" autocomplete="username" @keyup.enter="handleRegister" />
        </div>
        <div class="field">
          <label>密码</label>
          <input v-model="regForm.password" type="password" placeholder="至少 6 个字符" autocomplete="new-password" @keyup.enter="handleRegister" />
        </div>
        <div class="field">
          <label>确认密码</label>
          <input v-model="regForm.confirm" type="password" placeholder="再次输入密码" autocomplete="new-password" @keyup.enter="handleRegister" />
        </div>
      </template>

      <!-- 错误 / 成功提示 -->
      <transition name="msg">
        <p v-if="errorMsg" class="msg msg--error">{{ errorMsg }}</p>
        <p v-else-if="successMsg" class="msg msg--success">{{ successMsg }}</p>
      </transition>

      <!-- 操作按鈕 -->
      <button
        class="btn"
        :disabled="loading"
        @click="mode === 'login' ? handleLogin() : handleRegister()"
      >
        <span v-if="!loading" class="btn-text">{{ mode === 'login' ? '登 录' : '注 册' }}</span>
        <span v-else class="spinner" />
        <svg v-if="!loading" class="btn-arrow" viewBox="0 0 16 16" fill="none" aria-hidden="true">
          <path d="M6.333 3.667H12.333V9.667" stroke="currentColor" stroke-width="1.4" stroke-linecap="round" stroke-linejoin="round"/>
          <path d="M3.848 12.152L12.333 3.667" stroke="currentColor" stroke-width="1.4" stroke-linecap="round" stroke-linejoin="round"/>
        </svg>
      </button>

      <!-- 预置账号提示 -->
      <p v-if="mode === 'login'" class="hint">Demo: admin / 123456</p>
    </div>
  </div>
</template>

<style scoped>
/* ── 页面容器 ── */
.login-page {
  position: fixed;
  inset: 0;
  overflow: hidden;
  background: #000;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* ── 背景图 ── */
.bg-image {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  object-fit: contain;
  object-position: center;
  pointer-events: none;
  user-select: none;
}

/* ── Canvas 遮罩 ── */
.spotlight-canvas {
  position: absolute;
  inset: 0;
  pointer-events: none;
  z-index: 1;
}

/* ── 登录卡片 ── */
.card {
  position: relative;
  z-index: 10;
  width: 340px;
  padding: 40px 36px 36px;
  border-radius: 16px;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(18px) saturate(160%);
  -webkit-backdrop-filter: blur(18px) saturate(160%);
  border: 1px solid rgba(255, 255, 255, 0.10);
  box-shadow: 0 12px 48px rgba(0, 0, 0, 0.55);
}

/* 卡片震动反馈 */
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  20%       { transform: translateX(-7px); }
  40%       { transform: translateX(7px); }
  60%       { transform: translateX(-5px); }
  80%       { transform: translateX(5px); }
}
.card.shake { animation: shake 0.45s ease; }

/* ── 系统名称 ── */
.brand {
  margin: 0 0 28px;
  font-size: 17px;
  font-weight: 500;
  letter-spacing: 3px;
  color: rgba(255, 255, 255, 0.88);
  text-align: center;
}

/* ── 表单字段 ── */
.field { margin-bottom: 16px; }

.field label {
  display: block;
  margin-bottom: 6px;
  font-size: 11px;
  letter-spacing: 1.2px;
  text-transform: uppercase;
  color: rgba(255, 255, 255, 0.38);
}

.field input {
  width: 100%;
  box-sizing: border-box;
  padding: 10px 14px;
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(255, 255, 255, 0.11);
  border-radius: 8px;
  color: rgba(255, 255, 255, 0.88);
  font-size: 14px;
  outline: none;
  transition: border-color 0.22s, background 0.22s;
  caret-color: #fff;
}
.field input::placeholder { color: rgba(255, 255, 255, 0.2); }
.field input:focus {
  border-color: rgba(255, 255, 255, 0.32);
  background: rgba(255, 255, 255, 0.09);
}

/* ── 胶囊按钮（mimo 风格）── */
.btn {
  margin-top: 24px;
  width: 100%;
  height: 42px;
  padding: 0 20px;
  border-radius: 46px;
  border: 1px solid rgba(255, 255, 255, 0.30);
  background: transparent;
  color: rgba(255, 255, 255, 0.85);
  font-size: 14px;
  font-weight: 500;
  letter-spacing: 2px;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0;
  transition: background 200ms, color 200ms, border-color 200ms, transform 200ms;
  overflow: hidden;
}

.btn:hover:not(:disabled) {
  background: rgba(255, 255, 255, 0.90);
  color: #111;
  border-color: rgba(255, 255, 255, 0.90);
  transform: translateY(-1px);
}

.btn:disabled {
  opacity: 0.55;
  cursor: not-allowed;
}

/* 箭头：默认隐藏，hover 时从左滑入 */
.btn-arrow {
  width: 14px;
  height: 14px;
  flex-shrink: 0;
  opacity: 0;
  margin-left: -14px;
  transition: opacity 200ms, margin-left 200ms;
}
.btn:hover:not(:disabled) .btn-arrow {
  opacity: 1;
  margin-left: 8px;
}

/* ── loading spinner ── */
.spinner {
  width: 16px;
  height: 16px;
  border: 2px solid rgba(255, 255, 255, 0.25);
  border-top-color: rgba(255, 255, 255, 0.85);
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }

/* ── 模式 Tab ── */
.tabs {
  display: flex;
  gap: 4px;
  margin-bottom: 24px;
  background: rgba(255, 255, 255, 0.06);
  border-radius: 10px;
  padding: 4px;
}
.tab {
  flex: 1;
  height: 34px;
  border: none;
  border-radius: 7px;
  background: transparent;
  color: rgba(255, 255, 255, 0.40);
  font-size: 13px;
  font-weight: 500;
  letter-spacing: 2px;
  cursor: pointer;
  transition: background 220ms, color 220ms;
}
.tab.active {
  background: rgba(255, 255, 255, 0.12);
  color: rgba(255, 255, 255, 0.90);
}
.tab:hover:not(.active) {
  color: rgba(255, 255, 255, 0.65);
}

/* ── 提示消息 ── */
.msg {
  margin: 10px 0 0;
  padding: 8px 12px;
  border-radius: 7px;
  font-size: 12px;
  letter-spacing: 0.5px;
  text-align: center;
}
.msg--error {
  background: rgba(239, 68, 68, 0.15);
  border: 1px solid rgba(239, 68, 68, 0.30);
  color: rgba(252, 165, 165, 0.95);
}
.msg--success {
  background: rgba(34, 197, 94, 0.12);
  border: 1px solid rgba(34, 197, 94, 0.28);
  color: rgba(134, 239, 172, 0.95);
}

/* 消息淡入淡出动画 */
.msg-enter-active, .msg-leave-active { transition: opacity 280ms, transform 280ms; }
.msg-enter-from, .msg-leave-to { opacity: 0; transform: translateY(-6px); }

/* ── 提示设计 ── */
.hint {
  margin: 14px 0 0;
  text-align: center;
  font-size: 11px;
  color: rgba(255, 255, 255, 0.22);
  letter-spacing: 0.5px;
}
</style>
