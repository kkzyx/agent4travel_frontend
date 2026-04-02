<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue'

const API_BASE = 'http://127.0.0.1:8000/api'
const AMAP_KEY = import.meta.env.VITE_AMAP_KEY || ''
const AMAP_SECURITY_CODE = import.meta.env.VITE_AMAP_SECURITY_CODE || ''
const currentOrigin = typeof window !== 'undefined' ? window.location.origin : ''

const form = reactive({
  destination: '北京',
  start_date: '2026-05-01',
  days: 3,
  travelers: 2,
  budget_level: 'medium',
  hotel_style: 'business',
  preferences: ['历史文化', '美食'],
  notes: '希望节奏舒适，包含一晚夜游。'
})

const preferenceOptions = ['历史文化', '自然风光', '美食', '亲子', '拍照打卡', '城市漫步']
const plan = ref(null)
const loading = ref(false)
const savingPlan = ref(false)
const exportLoading = ref('')
const amapStatus = ref('idle')
const amapMessage = ref('')
const mapContainer = ref(null)

let mapInstance = null
let AMapRef = null
let labelsLayer = null
let mapOverlays = []
let plannerRequestId = 0
let serviceCache = null
let baseTileLayer = null

const dragState = reactive({
  itemId: '',
  fromDay: null
})

const newAttraction = reactive({
  id: 'custom-stop',
  name: '',
  category: '自定义',
  description: '',
  duration_hours: 2,
  ticket_cost: 0,
  meal_cost: 0,
  transport_cost: 0,
  hotel_cost: 0,
  lat: 39.92,
  lng: 116.4
})

const colorMap = {
  sight: '#0f766e',
  meal: '#f97316',
  hotel: '#7c3aed',
  transport: '#0f172a'
}

const routeColors = ['#f97316', '#0284c7', '#0f766e', '#dc2626', '#7c3aed']

const totalStops = computed(() => {
  if (!plan.value) return 0
  return plan.value.days.reduce((sum, day) => sum + day.items.filter((item) => item.type === 'sight').length, 0)
})

const hasAmapKey = computed(() => Boolean(AMAP_KEY))

function toMinutes(text) {
  if (!text || text.includes('次日')) return null
  const [hour, minute] = text.split(':').map(Number)
  if (Number.isNaN(hour) || Number.isNaN(minute)) return null
  return hour * 60 + minute
}

function formatTime(minutes) {
  if (minutes == null) return ''
  const normalized = Math.max(0, Math.min(23 * 60 + 59, minutes))
  const hour = Math.floor(normalized / 60)
  const minute = normalized % 60
  return `${String(hour).padStart(2, '0')}:${String(minute).padStart(2, '0')}`
}

function sortDayItems(day) {
  day.items.sort((a, b) => {
    const aMinutes = toMinutes(a.start_time)
    const bMinutes = toMinutes(b.start_time)
    if (aMinutes == null && bMinutes == null) return 0
    if (aMinutes == null) return 1
    if (bMinutes == null) return -1
    return aMinutes - bMinutes
  })
}

function normalizePlan() {
  if (!plan.value) return
  plan.value.days.forEach((day) => {
    day.items.forEach((item) => {
      item.day = day.day
    })
    sortDayItems(day)
  })
}

async function generatePlan() {
  loading.value = true
  try {
    const response = await fetch(`${API_BASE}/plan`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(form)
    })
    plan.value = await response.json()
    normalizePlan()
  } finally {
    loading.value = false
  }
}

async function syncPlan() {
  if (!plan.value) return
  savingPlan.value = true
  try {
    normalizePlan()
    const response = await fetch(`${API_BASE}/plan/update`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        plan: plan.value,
        action: 'replace_plan'
      })
    })
    plan.value = await response.json()
    normalizePlan()
  } finally {
    savingPlan.value = false
  }
}

async function updatePlan(payload) {
  if (!plan.value) return
  savingPlan.value = true
  try {
    const response = await fetch(`${API_BASE}/plan/update`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        plan: plan.value,
        ...payload
      })
    })
    plan.value = await response.json()
    normalizePlan()
  } finally {
    savingPlan.value = false
  }
}

function togglePreference(option) {
  if (form.preferences.includes(option)) {
    const index = form.preferences.indexOf(option)
    form.preferences.splice(index, 1)
    return
  }
  form.preferences.push(option)
}

async function removeItem(itemId) {
  await updatePlan({ action: 'delete', item_id: itemId })
}

async function moveItem(itemId, targetDay) {
  if (!targetDay) return
  await updatePlan({ action: 'move', item_id: itemId, target_day: Number(targetDay) })
}

async function addCustomAttraction(targetDay) {
  if (!newAttraction.name.trim()) return
  await updatePlan({ action: 'add', target_day: targetDay, attraction: { ...newAttraction } })
  newAttraction.name = ''
  newAttraction.description = ''
}

function onDragStart(item, day) {
  dragState.itemId = item.id
  dragState.fromDay = day.day
}

function onDragEnd() {
  dragState.itemId = ''
  dragState.fromDay = null
}

async function onDropItem(targetDay, targetIndex) {
  if (!plan.value || !dragState.itemId) return
  const sourceDay = plan.value.days.find((day) => day.day === dragState.fromDay)
  const destinationDay = plan.value.days.find((day) => day.day === targetDay.day)
  if (!sourceDay || !destinationDay) return
  const sourceIndex = sourceDay.items.findIndex((item) => item.id === dragState.itemId)
  if (sourceIndex < 0) return
  const [moved] = sourceDay.items.splice(sourceIndex, 1)
  moved.day = destinationDay.day
  let insertIndex = targetIndex
  if (sourceDay.day === destinationDay.day && sourceIndex < targetIndex) {
    insertIndex -= 1
  }
  destinationDay.items.splice(Math.max(0, insertIndex), 0, moved)
  normalizePlan()
  onDragEnd()
  await syncPlan()
}

async function onDropDayEnd(day) {
  if (!plan.value || !dragState.itemId) return
  await onDropItem(day, day.items.length)
}

async function updateItemTime(day, item) {
  if (!item.start_time || !item.end_time) return
  if (toMinutes(item.start_time) != null && toMinutes(item.end_time) != null && toMinutes(item.end_time) < toMinutes(item.start_time)) {
    item.end_time = item.start_time
  }
  sortDayItems(day)
  await syncPlan()
}

async function exportFile(type) {
  if (!plan.value) return
  exportLoading.value = type
  try {
    const response = await fetch(`${API_BASE}/export/${type}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(plan.value)
    })
    const blob = await response.blob()
    const url = URL.createObjectURL(blob)
    const link = document.createElement('a')
    link.href = url
    link.download = type === 'pdf' ? 'travel-plan.pdf' : 'travel-plan-map.svg'
    link.click()
    URL.revokeObjectURL(url)
  } finally {
    exportLoading.value = ''
  }
}

function ensureAmapScript() {
  if (window.AMap) return Promise.resolve(window.AMap)
  if (!hasAmapKey.value) {
    return Promise.reject(new Error('missing-key'))
  }
  amapStatus.value = 'loading'
  if (AMAP_SECURITY_CODE) {
    window._AMapSecurityConfig = { securityJsCode: AMAP_SECURITY_CODE }
  }
  return new Promise((resolve, reject) => {
    const existing = document.querySelector('script[data-amap-sdk="true"]')
    if (existing) {
      existing.addEventListener('load', () => resolve(window.AMap), { once: true })
      existing.addEventListener('error', () => reject(new Error('load-failed')), { once: true })
      return
    }
    const script = document.createElement('script')
    script.src = `https://webapi.amap.com/maps?v=2.0&key=${AMAP_KEY}&plugin=AMap.Scale,AMap.ToolBar,AMap.PlaceSearch,AMap.Walking,AMap.Driving`
    script.async = true
    script.defer = true
    script.dataset.amapSdk = 'true'
    script.onload = () => {
      if (window.AMap) {
        resolve(window.AMap)
      } else {
        reject(new Error('sdk-loaded-but-amap-missing'))
      }
    }
    script.onerror = () => reject(new Error('load-failed'))
    document.head.appendChild(script)
  })
}

function initLabelsLayer() {
  if (!mapInstance || !AMapRef) return
  if (labelsLayer) {
    mapInstance.remove(labelsLayer)
  }
  labelsLayer = new AMapRef.LabelsLayer({
    collision: true,
    allowCollision: false,
    animation: true
  })
  mapInstance.add(labelsLayer)
}

async function setupMap() {
  if (!mapContainer.value) return
  if (!hasAmapKey.value) {
    amapStatus.value = 'missing-key'
    amapMessage.value = '请在 frontend/.env.local 中配置 VITE_AMAP_KEY 和 VITE_AMAP_SECURITY_CODE。'
    return
  }
  try {
    AMapRef = await ensureAmapScript()
    if (!mapInstance) {
      baseTileLayer = new AMapRef.TileLayer()
      mapInstance = new AMapRef.Map(mapContainer.value, {
        viewMode: '2D',
        zoom: 11,
        resizeEnable: true,
        center: [116.397428, 39.90923],
        layers: [baseTileLayer]
      })
      mapInstance.addControl(new AMapRef.Scale())
      mapInstance.addControl(new AMapRef.ToolBar({ position: 'RB' }))
      initLabelsLayer()
    }
    amapStatus.value = 'ready'
    amapMessage.value = ''
    await resolvePoisAndRoutes()
  } catch (error) {
    amapStatus.value = 'error'
    amapMessage.value = `高德地图加载失败：${error?.message || 'unknown-error'}`
  }
}

function clearAmapOverlays() {
  if (mapInstance && mapOverlays.length) {
    mapInstance.remove(mapOverlays)
  }
  mapOverlays = []
  if (labelsLayer) {
    labelsLayer.clear()
  }
}

function getServices() {
  if (!AMapRef) throw new Error('amap-not-ready')
  if (serviceCache) return serviceCache
  serviceCache = {
    placeSearch: new AMapRef.PlaceSearch({
      pageSize: 1,
      pageIndex: 1,
      citylimit: true
    }),
    walking: new AMapRef.Walking({ hideMarkers: true, autoFitView: false }),
    driving: new AMapRef.Driving({ hideMarkers: true, autoFitView: false })
  }
  return serviceCache
}

function searchPoi(point, city) {
  return new Promise((resolve, reject) => {
    const { placeSearch } = getServices()
    placeSearch.setCity(city)
    placeSearch.search(`${city}${point.name}`, (status, result) => {
      if (status === 'complete' && result?.poiList?.pois?.length) {
        const poi = result.poiList.pois[0]
        const location = poi.location
        resolve({
          id: point.id,
          name: point.name,
          day: point.day,
          lng: location.lng,
          lat: location.lat,
          address: poi.address || poi.name
        })
        return
      }
      placeSearch.search(point.name, (fallbackStatus, fallbackResult) => {
        if (fallbackStatus === 'complete' && fallbackResult?.poiList?.pois?.length) {
          const poi = fallbackResult.poiList.pois[0]
          resolve({
            id: point.id,
            name: point.name,
            day: point.day,
            lng: poi.location.lng,
            lat: poi.location.lat,
            address: poi.address || poi.name
          })
          return
        }
        reject(new Error(`poi-not-found:${point.name}`))
      })
    })
  })
}

function flattenRoutePath(result) {
  const route = result?.routes?.[0]
  if (!route) return []
  const steps = route.steps || route.segments || []
  const path = []
  steps.forEach((step) => {
    const segmentPath = step.path || []
    segmentPath.forEach((lnglat) => {
      path.push([lnglat.lng, lnglat.lat])
    })
  })
  return path
}

function planWalking(start, end) {
  return new Promise((resolve) => {
    const { walking } = getServices()
    walking.search([start.lng, start.lat], [end.lng, end.lat], (status, result) => {
      if (status === 'complete') {
        resolve({ type: 'walking', path: flattenRoutePath(result) })
        return
      }
      resolve({ type: 'walking', path: [] })
    })
  })
}

function planDriving(start, end) {
  return new Promise((resolve) => {
    const { driving } = getServices()
    driving.search([start.lng, start.lat], [end.lng, end.lat], (status, result) => {
      if (status === 'complete') {
        resolve({ type: 'driving', path: flattenRoutePath(result) })
        return
      }
      resolve({ type: 'driving', path: [] })
    })
  })
}

async function buildSegmentPath(start, end) {
  const walkingResult = await planWalking(start, end)
  if (walkingResult.path.length > 1) return walkingResult
  const drivingResult = await planDriving(start, end)
  if (drivingResult.path.length > 1) return drivingResult
  return {
    type: 'straight',
    path: [
      [start.lng, start.lat],
      [end.lng, end.lat]
    ]
  }
}

function createLabelMarker(point, color, rank) {
  return new AMapRef.LabelMarker({
    name: point.name,
    position: [point.lng, point.lat],
    zIndex: 120 + rank,
    rank: 120 + rank,
    text: {
      content: `Day ${point.day} · ${point.name}`,
      direction: 'right',
      offset: [10, 0],
      style: {
        fontSize: 13,
        fontWeight: '700',
        fillColor: '#0f172a',
        strokeColor: '#ffffff',
        strokeWidth: 2,
        padding: [6, 10],
        backgroundColor: 'rgba(255,255,255,0.96)',
        borderColor: 'rgba(203,213,225,0.95)',
        borderWidth: 1,
        borderRadius: 999
      }
    },
    icon: {
      image: `https://a.amap.com/jsapi_demos/static/demo-center/icons/poi-marker-${(point.day - 1) % 3 + 1}.png`,
      anchor: 'bottom-center',
      size: [22, 30],
      imageSize: [22, 30]
    }
  })
}

async function resolvePoisAndRoutes() {
  if (!mapInstance || !AMapRef || !plan.value) return
  const requestId = ++plannerRequestId
  amapStatus.value = 'loading'
  amapMessage.value = '正在搜索真实景点坐标并规划路线...'
  clearAmapOverlays()

  try {
    const allDayPois = await Promise.all(
      plan.value.days.map(async (day) => {
        const sights = day.items.filter((item) => item.type === 'sight')
        const pois = []
        for (const item of sights) {
          try {
            const poi = await searchPoi(item, plan.value.destination)
            pois.push(poi)
          } catch {
            pois.push({
              id: item.id,
              name: item.title,
              day: day.day,
              lng: item.lng,
              lat: item.lat,
              address: 'fallback'
            })
          }
        }
        return { day: day.day, pois }
      })
    )

    if (requestId !== plannerRequestId) return

    const fitTargets = []
    const labelMarkers = []

    for (const dayResult of allDayPois) {
      const color = routeColors[(dayResult.day - 1) % routeColors.length]
      dayResult.pois.forEach((poi, index) => {
        const marker = new AMapRef.CircleMarker({
          center: [poi.lng, poi.lat],
          radius: 9,
          strokeColor: '#ffffff',
          strokeWeight: 3,
          strokeOpacity: 1,
          fillColor: color,
          fillOpacity: 0.95,
          zIndex: 110
        })
        const infoWindow = new AMapRef.InfoWindow({
          offset: new AMapRef.Pixel(0, -18),
          content: `<div class="amap-info-window"><strong>${poi.name}</strong><div>第 ${dayResult.day} 天</div><div>${poi.address || ''}</div></div>`
        })
        marker.on('click', () => infoWindow.open(mapInstance, [poi.lng, poi.lat]))
        mapOverlays.push(marker)
        fitTargets.push(marker)
        labelMarkers.push(createLabelMarker(poi, color, index + dayResult.day * 10))
      })

      for (let i = 0; i < dayResult.pois.length - 1; i += 1) {
        const start = dayResult.pois[i]
        const end = dayResult.pois[i + 1]
        const segment = await buildSegmentPath(start, end)
        if (requestId !== plannerRequestId) return
        const polyline = new AMapRef.Polyline({
          path: segment.path,
          strokeColor: color,
          strokeWeight: segment.type === 'walking' ? 5 : 6,
          strokeOpacity: 0.9,
          strokeStyle: segment.type === 'walking' ? 'solid' : 'solid',
          lineJoin: 'round',
          lineCap: 'round',
          showDir: true,
          zIndex: 100
        })
        mapOverlays.push(polyline)
        fitTargets.push(polyline)
      }
    }

    if (requestId !== plannerRequestId) return

    if (mapOverlays.length) {
      mapInstance.add(mapOverlays)
    }
    if (labelsLayer) {
      labelsLayer.clear()
      labelsLayer.add(labelMarkers)
    }
    if (fitTargets.length) {
      mapInstance.setFitView(fitTargets, false, [80, 80, 80, 80], 13)
    }
    amapStatus.value = 'ready'
    amapMessage.value = '已切换为真实 POI 搜索和路径规划结果。'
  } catch (error) {
    amapStatus.value = 'error'
    amapMessage.value = `路线规划失败：${error?.message || 'unknown-error'}`
  }
}

watch(
  () => plan.value?.days,
  async () => {
    await nextTick()
    if (plan.value) {
      setupMap()
    }
  },
  { deep: true }
)

onMounted(async () => {
  await generatePlan()
  await nextTick()
  setupMap()
})

onBeforeUnmount(() => {
  clearAmapOverlays()
  if (mapInstance) {
    mapInstance.destroy()
    mapInstance = null
  }
})
</script>

<template>
  <div class="page-shell">
    <section class="hero">
      <div>
        <p class="eyebrow">Multi-Agent Travel Planner</p>
        <h1>智能旅行助手 Agent 实战项目</h1>
        <p class="hero-copy">把行程规划、预算拆解、拖拽编辑和真实地图路线展示放进一个前后端协作工作台。</p>
      </div>
      <div class="hero-stats">
        <div class="stat-card">
          <span>行程天数</span>
          <strong>{{ form.days }} 天</strong>
        </div>
        <div class="stat-card">
          <span>景点数量</span>
          <strong>{{ totalStops }} 个</strong>
        </div>
        <div class="stat-card" v-if="plan">
          <span>总预算</span>
          <strong>¥ {{ plan.budget.grand_total }}</strong>
        </div>
      </div>
    </section>

    <section class="workspace">
      <aside class="planner-panel">
        <h2>规划输入</h2>
        <label>
          <span>目的地</span>
          <input v-model="form.destination" />
        </label>
        <div class="grid two">
          <label>
            <span>出发日期</span>
            <input v-model="form.start_date" type="date" />
          </label>
          <label>
            <span>旅行天数</span>
            <input v-model="form.days" type="number" min="1" max="7" />
          </label>
        </div>
        <div class="grid two">
          <label>
            <span>出行人数</span>
            <input v-model="form.travelers" type="number" min="1" max="10" />
          </label>
          <label>
            <span>预算级别</span>
            <select v-model="form.budget_level">
              <option value="economy">经济型</option>
              <option value="medium">中等预算</option>
              <option value="premium">高品质</option>
            </select>
          </label>
        </div>
        <label>
          <span>酒店风格</span>
          <select v-model="form.hotel_style">
            <option value="business">商务</option>
            <option value="boutique">精品</option>
            <option value="family">亲子</option>
            <option value="luxury">奢华</option>
          </select>
        </label>
        <div>
          <span class="section-label">旅行偏好</span>
          <div class="chips">
            <button
              v-for="option in preferenceOptions"
              :key="option"
              type="button"
              class="chip"
              :class="{ active: form.preferences.includes(option) }"
              @click="togglePreference(option)"
            >
              {{ option }}
            </button>
          </div>
        </div>
        <label>
          <span>补充说明</span>
          <textarea v-model="form.notes" rows="4" />
        </label>
        <button class="primary-btn" @click="generatePlan" :disabled="loading">
          {{ loading ? '规划中...' : '生成智能行程' }}
        </button>

        <div class="agent-box" v-if="plan">
          <p class="section-label">Agent 分工</p>
          <div class="agent-list">
            <span v-for="agent in plan.agents" :key="agent">{{ agent }}</span>
          </div>
          <p class="sync-hint">{{ savingPlan ? '正在同步编辑...' : '拖拽排序和时间修改会自动同步到后端。' }}</p>
        </div>
      </aside>

      <main class="result-panel" v-if="plan">
        <section class="card intro-card">
          <div>
            <p class="eyebrow">Auto Summary</p>
            <h2>{{ plan.title }}</h2>
            <p>{{ plan.summary }}</p>
          </div>
          <div class="actions">
            <button class="secondary-btn" @click="exportFile('pdf')" :disabled="exportLoading === 'pdf'">
              {{ exportLoading === 'pdf' ? '导出中...' : '导出 PDF' }}
            </button>
            <button class="secondary-btn" @click="exportFile('image')" :disabled="exportLoading === 'image'">
              {{ exportLoading === 'image' ? '导出中...' : '导出图片' }}
            </button>
          </div>
        </section>

        <section class="grid metrics">
          <article class="card budget-card">
            <h3>预算明细</h3>
            <div class="budget-row"><span>门票</span><strong>¥ {{ plan.budget.attraction_total }}</strong></div>
            <div class="budget-row"><span>餐饮</span><strong>¥ {{ plan.budget.meal_total }}</strong></div>
            <div class="budget-row"><span>住宿</span><strong>¥ {{ plan.budget.hotel_total }}</strong></div>
            <div class="budget-row"><span>交通</span><strong>¥ {{ plan.budget.transport_total }}</strong></div>
            <div class="budget-row total"><span>总计</span><strong>¥ {{ plan.budget.grand_total }}</strong></div>
            <div class="budget-row"><span>人均</span><strong>¥ {{ plan.budget.per_person }}</strong></div>
          </article>

          <article class="card map-card">
            <div class="map-header">
              <div>
                <h3>路线可视化</h3>
                <p>基于高德真实景点搜索、官方路径规划和标签避让。</p>
              </div>
              <div class="map-legend" v-if="plan.days.length">
                <span v-for="day in plan.days" :key="day.day">Day {{ day.day }}</span>
              </div>
            </div>
            <div v-if="amapStatus === 'missing-key'" class="map-empty">
              <p>未配置高德地图 Key。</p>
              <code>frontend/.env.local</code>
              <code>VITE_AMAP_KEY=你的高德 Web 端 Key</code>
              <code>VITE_AMAP_SECURITY_CODE=你的安全密钥</code>
            </div>
            <div v-else-if="amapStatus === 'error'" class="map-empty">
              <p>{{ amapMessage }}</p>
              <code>当前来源：{{ currentOrigin }}</code>
            </div>
            <div v-else class="amap-shell">
              <div ref="mapContainer" class="amap-container"></div>
              <div class="map-status" v-if="amapStatus === 'loading'">
                {{ amapMessage || '地图加载中...' }}
              </div>
              <div class="map-note" v-if="amapStatus === 'ready'">{{ amapMessage }}</div>
            </div>
          </article>
        </section>

        <section class="card">
          <div class="section-head">
            <div>
              <p class="eyebrow">Editable Itinerary</p>
              <h3>每日行程</h3>
            </div>
            <div class="tips">
              <span v-for="tip in plan.tips" :key="tip">{{ tip }}</span>
            </div>
          </div>

          <div class="days-grid">
            <article
              class="day-card"
              v-for="day in plan.days"
              :key="day.day"
              @dragover.prevent
              @drop.prevent="onDropDayEnd(day)"
            >
              <div class="day-head">
                <h4>{{ day.theme }}</h4>
                <span class="drag-tip">拖拽卡片可调整顺序</span>
              </div>

              <div class="timeline">
                <div class="drop-slot" @dragover.prevent @drop.prevent="onDropItem(day, 0)">
                  放到最前
                </div>
                <div
                  class="timeline-item"
                  v-for="(item, itemIndex) in day.items"
                  :key="item.id"
                  draggable="true"
                  @dragstart="onDragStart(item, day)"
                  @dragend="onDragEnd"
                >
                  <div class="dot" :style="{ background: colorMap[item.type] }"></div>
                  <div class="timeline-main">
                    <div class="timeline-top">
                      <strong>{{ item.title }}</strong>
                      <span class="item-tag">{{ item.type }}</span>
                    </div>
                    <div class="time-editors">
                      <label>
                        <span>开始</span>
                        <input
                          type="time"
                          :value="item.start_time.includes('次日') ? '' : item.start_time"
                          @input="item.start_time = $event.target.value"
                          @change="updateItemTime(day, item)"
                        />
                      </label>
                      <label>
                        <span>结束</span>
                        <input
                          type="time"
                          :value="item.end_time.includes('次日') ? formatTime(8 * 60) : item.end_time"
                          @input="item.end_time = $event.target.value"
                          @change="updateItemTime(day, item)"
                        />
                      </label>
                    </div>
                    <p>{{ item.notes }}</p>
                    <div class="timeline-actions">
                      <small>¥ {{ item.cost }}</small>
                      <button type="button" @click="removeItem(item.id)">删除</button>
                      <select @change="moveItem(item.id, $event.target.value)">
                        <option value="">移动到...</option>
                        <option v-for="target in plan.days" :key="target.day" :value="target.day">
                          第 {{ target.day }} 天
                        </option>
                      </select>
                    </div>
                  </div>
                  <div class="drop-slot inline" @dragover.prevent @drop.prevent="onDropItem(day, itemIndex + 1)">
                    拖到这里
                  </div>
                </div>
              </div>

              <div class="custom-form">
                <input v-model="newAttraction.name" placeholder="添加自定义景点" />
                <textarea v-model="newAttraction.description" rows="2" placeholder="景点描述"></textarea>
                <button class="secondary-btn" type="button" @click="addCustomAttraction(day.day)">
                  添加到第 {{ day.day }} 天
                </button>
              </div>
            </article>
          </div>
        </section>
      </main>
    </section>
  </div>
</template>
