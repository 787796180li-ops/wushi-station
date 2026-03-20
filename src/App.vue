<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'

// --- 全局状态区域 ---
const showArchive = ref(false)   // 控制是否展示档案馆日历
const logs = ref([])             // 历史记录承载池
const isFetchingLogs = ref(false)
const showSummaryModal = ref(false)  // 控制是否展示「机能评估报告」全屏
const isGeneratingSummary = ref(false)
const summaryReportText = ref('')
const summaryPeriod = ref('week')    // 默认评估周期为近一周

// 能量状态定义 (去除了固定的 quote，改为动态生成)
const states = [
  { id: 'unselected', label: '静默中', power: '---' },
  { id: 'stable', label: '屋内恒温', power: '98' }, 
  { id: 'draining', label: '寒气渗入', power: '65' },
  { id: 'depleted', label: '火炉微弱', power: '15' }, 
  { id: 'critical', label: '暴雪压顶', power: '3' }   
]

const currentStateId = ref('unselected')
const userMessage = ref('')
const isTyping = ref(false)
const displayedQuote = ref(
  '我无法替你解决学术数据的推翻，\n也无法改变首尔的天气，\n' +
  '但我在这里为你建了一间永远下雪、\n永远反锁的木屋。\n' +
  '这是一个允许你彻底碎掉的空间。\n\n' +
  '>>> 炉火已点燃，风雪禁行。'
)
const isLoading = ref(false)

const activeState = computed(() => {
  return states.find(s => s.id === currentStateId.value) || states[0]
})

const buttonStates = states.filter(s => s.id !== 'unselected')

// 打字机特效计时器引用
let typingTimer = null

// ========== 天气系统 —— 首尔永登浦区 ==========
// 永登浦区坐标: 37.5257°N, 126.8965°E
const WEATHER_LAT = 37.5257
const WEATHER_LON = 126.8965

const weatherState = ref('loading')   // loading | clear | clouds | rain | drizzle | snow | thunderstorm | fog | unknown
const weatherData = ref(null)         // 原始天气数据
const isFetchingWeather = ref(false)
const showWeatherPanel = ref(false)   // 天气面板展开状态
const showWeatherTestBtns = ref(false)  // 调试按钮面板

// 天气状态的元数据（图标 + 中文名）
const weatherMeta = {
  loading:      { icon: '◌', label: '正在探测...',   color: 'rgba(255,255,255,0.3)' },
  clear:        { icon: '◎', label: '晴空万里',       color: 'rgba(255, 220, 120, 0.9)' },
  clouds:       { icon: '◑', label: '阴云覆盖',       color: 'rgba(180, 190, 210, 0.9)' },
  rain:         { icon: '⌇', label: '大雨倾注',       color: 'rgba(100, 170, 230, 0.9)' },
  drizzle:      { icon: '⌁', label: '细雨如丝',       color: 'rgba(140, 190, 220, 0.9)' },
  snow:         { icon: '❄',  label: '漫天飞雪',       color: 'rgba(210, 230, 255, 0.9)' },
  thunderstorm: { icon: '⚡', label: '雷暴骤临',       color: 'rgba(180, 130, 255, 0.9)' },
  fog:          { icon: '≋',  label: '浓雾弥漫',       color: 'rgba(160, 170, 180, 0.9)' },
  unknown:      { icon: '？', label: '未知气象',       color: 'rgba(255,255,255,0.4)' },
}

// 各天气状态下的深空诗意提示语
const weatherTips = {
  clear: [
    '首尔的天幕今日透明。光线未经任何衰减直达地面，这是一种极罕见的能量不耗散状态。',
    '永登浦上空，大气层以最低折射率运行。此刻的阳光是不加滤镜的真实，请允许它照进来。',
    '大气散射系数跌至最低值。今日的天光是原频率的，不需要任何防御性姿态。'
  ],
  clouds: [
    '首尔上空的云层正以每小时若干公里的速度完成它的覆盖任务。这是大气的自我屏蔽，与你无关。',
    '永登浦的天穹今日由低压云系接管。不是阴郁，是大气层在进行正常的水分循环。',
    '云层的存在并非遮蔽，而是漫射。所有的光，依然在此处，只是改变了抵达的方式。'
  ],
  rain: [
    '首尔降水事件已触发。每一颗雨滴携带着高处的记忆降落，以重力的方式与地面完成交割。',
    '永登浦正在接收大气水分的集中偿还。这种倾注有其物理逻辑，不需要被解读为负面信号。',
    '雨水是大气向地表的一次强制性释放。可以出门，用皮肤感知这种不可抗拒的质量交换。'
  ],
  drizzle: [
    '首尔正在经历微量降水。雨滴尚未形成宏观压力，只是一种轻微的湿度渗透，像一场说话很轻的对话。',
    '永登浦的毛毛雨已启动。它不要求你做任何应对，只是低密度地沾湿世界的表面。',
    '细雨的信噪比极低。它无声地稀释掉一切粗粝，今日的首尔，边界比较模糊，这是被允许的。'
  ],
  snow: [
    '首尔进入降雪模式。晶体结构各不相同的水粒子正在逐步覆盖永登浦，这是一场极度耐心的沉积。',
    '大气温度跌破相变阈值，水分以固态形式降落。每一片雪花都是六倍对称的确定性，在混乱里落下来。',
    '永登浦上方，雪粒子正在进行无声的层叠。它们不互相竞争落地顺序，它们只是下落，然后安静地停在原处。'
  ],
  thunderstorm: [
    '首尔上空探测到强对流天气。正负电荷已积累至击穿阈值，雷暴是大气清偿电荷债务的唯一方式。请待在室内。',
    '永登浦上空，等离子通道已开辟，雷击完成了一次百万伏特级别的能量瞬间结算。这个世界有时候也会用极端方式释放。',
    '雷暴是大气压极端不平衡时的强制性修正事件。闪电并不具有恶意，它只是在完成电荷守恒。'
  ],
  fog: [
    '首尔上空，大气能见度已被浓雾压缩至极低水平。世界的边缘今日是模糊的，这创造了一种反常的舒适感。',
    '永登浦今日被低层云雾封存。远处的一切都失去了轮廓，这不是危险，是大气给出的一种短暂的模糊许可。',
    '雾是未完成的雨，是大气在液化与气化之间的悬停状态。今日的首尔，万物都在等待一个明确的聚合信号。'
  ],
  unknown: [
    '天气数据信号微弱，无法完成精确归类。不明气象条件下，所有的准备都是多余的，出发就好。',
  ]
}

const weatherAdvice = {
  clear: {
    travel: '太阳辐射充足，所有室外动线均为优选。可以延长暴露在室外的时间。',
    clothes: '建议轻便透气的着装，紫外线处于较高量级，请部署必要的避光防晒措施。'
  },
  clouds: {
    travel: '大气漫射光线柔和。最适宜城市漫游与建筑间穿梭，避免了光照消耗。',
    clothes: '气温受云体遮挡可能稍有下降，建议携带有厚度的薄外套以维持体表恒温。'
  },
  rain: {
    travel: '非首要任务建议转入室内。外出时请尽量沿有顶棚的动线网络移动。',
    clothes: '必须搭载防雨外骨骼（雨衣/防水涂层冲锋衣）及伞具，下肢注意防潮。'
  },
  drizzle: {
    travel: '小幅度活动不受限。可前往半开放式的城市中转空间。',
    clothes: '可选择易风干的抗水面料结构，搭配短柄折伞，警惕缓慢的热量散失。'
  },
  snow: {
    travel: '可安排短距赏雪行动。降落地面的冰晶会改变摩擦系数，请降低移动速率。',
    clothes: '启动高级保暖协议，配置羽绒层、防风隔离层，以及具备高抓地力的足层装备。'
  },
  thunderstorm: {
    travel: '系统强制建议：绝对禁止空旷地带与高危区域的位移。请留守建筑掩体。',
    clothes: '若紧急移动，严禁暴露并使用任何含金属引导件的防雨设备。保重安全。'
  },
  fog: {
    travel: '能见度压缩至限制级。放弃私人高速驾驶，改用近场步行或地下轨道系统。',
    clothes: '大气悬浮微粒附着度高，建议挂载过滤面罩（口罩），外层选择不易吸附水汽的面料。'
  },
  unknown: {
    travel: '缺乏精确侦测参数。请基于自身肉眼观测进行风险评估和路径规划。',
    clothes: '推荐多层嵌套式穿衣法（洋葱式结构），随时应对突发的非标气象。'
  }
}

const currentWeatherTip = ref('观测节点初始寻址中...')
const currentTravelTip = ref('标定安全出行阈值...')
const currentClothingTip = ref('计算体表防护方程...')
const isFetchingWeatherAdvice = ref(false)

async function generateWeatherAdvice(stateId, data) {
  // 根据主理人要求，天气悬浮 UI 已被隐藏，屏蔽对气象诗意预报的大模型请求以节省后台算力。
  return
}


const currentWeatherMeta = computed(() => {
  return weatherMeta[weatherState.value] || weatherMeta.unknown
})

// 测试用天气按钮
const testWeatherBtns = [
  { key: 'clear',        label: '🌤 晴' },
  { key: 'clouds',       label: '☁ 阴' },
  { key: 'drizzle',      label: '🌧 细雨' },
  { key: 'rain',         label: '⛈ 大雨' },
  { key: 'snow',         label: '❄ 雪' },
  { key: 'thunderstorm', label: '⚡ 雷暴' },
  { key: 'fog',          label: '🌫 雾' },
]

// ---------- 粒子动画引擎 ----------
let animFrameId = null
let particles = []
const canvasRef = ref(null)

function stopParticles() {
  if (animFrameId) { cancelAnimationFrame(animFrameId); animFrameId = null }
  particles = []
  if (canvasRef.value) {
    const ctx = canvasRef.value.getContext('2d')
    ctx.clearRect(0, 0, canvasRef.value.width, canvasRef.value.height)
  }
}

function resizeCanvas() {
  if (!canvasRef.value) return
  canvasRef.value.width  = window.innerWidth
  canvasRef.value.height = window.innerHeight
}

function startRain(isHeavy = false) {
  stopParticles()
  resizeCanvas()
  const canvas = canvasRef.value
  const ctx = canvas.getContext('2d')
  const count = isHeavy ? 280 : 120

  for (let i = 0; i < count; i++) {
    particles.push({
      x: Math.random() * canvas.width,
      y: Math.random() * canvas.height,
      len: Math.random() * (isHeavy ? 22 : 12) + 10,
      speed: Math.random() * (isHeavy ? 14 : 7) + 6,
      opacity: Math.random() * 0.5 + 0.15,
      width: isHeavy ? Math.random() * 1.2 + 0.4 : 0.4,
    })
  }

  function draw() {
    ctx.clearRect(0, 0, canvas.width, canvas.height)
    particles.forEach(p => {
      ctx.beginPath()
      ctx.moveTo(p.x, p.y)
      ctx.lineTo(p.x - p.len * 0.15, p.y + p.len)
      ctx.strokeStyle = `rgba(160, 200, 240, ${p.opacity})`
      ctx.lineWidth = p.width
      ctx.stroke()
      p.y += p.speed
      p.x -= p.speed * 0.15
      if (p.y > canvas.height) { p.y = -p.len; p.x = Math.random() * canvas.width }
      if (p.x < 0) p.x = canvas.width
    })
    animFrameId = requestAnimationFrame(draw)
  }
  draw()
}

function startSnow() {
  stopParticles()
  resizeCanvas()
  const canvas = canvasRef.value
  const ctx = canvas.getContext('2d')

  for (let i = 0; i < 160; i++) {
    particles.push({
      x: Math.random() * canvas.width,
      y: Math.random() * canvas.height,
      r: Math.random() * 3.5 + 1,
      speed: Math.random() * 1.2 + 0.4,
      opacity: Math.random() * 0.6 + 0.2,
      drift: (Math.random() - 0.5) * 0.6,
      wobble: Math.random() * Math.PI * 2,
    })
  }

  function draw() {
    ctx.clearRect(0, 0, canvas.width, canvas.height)
    particles.forEach(p => {
      p.wobble += 0.015
      p.x += p.drift + Math.sin(p.wobble) * 0.4
      p.y += p.speed
      ctx.beginPath()
      ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2)
      ctx.fillStyle = `rgba(220, 235, 255, ${p.opacity})`
      ctx.fill()
      if (p.y > canvas.height) { p.y = -p.r; p.x = Math.random() * canvas.width }
    })
    animFrameId = requestAnimationFrame(draw)
  }
  draw()
}

function startFog() {
  stopParticles()
  resizeCanvas()
  const canvas = canvasRef.value
  const ctx = canvas.getContext('2d')

  for (let i = 0; i < 18; i++) {
    particles.push({
      x: Math.random() * canvas.width,
      y: Math.random() * canvas.height,
      r: Math.random() * 200 + 120,
      opacity: Math.random() * 0.04 + 0.01,
      dx: (Math.random() - 0.5) * 0.3,
      dy: (Math.random() - 0.5) * 0.1,
    })
  }

  function draw() {
    ctx.clearRect(0, 0, canvas.width, canvas.height)
    particles.forEach(p => {
      const grad = ctx.createRadialGradient(p.x, p.y, 0, p.x, p.y, p.r)
      grad.addColorStop(0, `rgba(200, 210, 220, ${p.opacity * 2})`)
      grad.addColorStop(1, 'rgba(200, 210, 220, 0)')
      ctx.beginPath()
      ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2)
      ctx.fillStyle = grad
      ctx.fill()
      p.x += p.dx; p.y += p.dy
      if (p.x < -p.r) p.x = canvas.width + p.r
      if (p.x > canvas.width + p.r) p.x = -p.r
      if (p.y < -p.r) p.y = canvas.height + p.r
      if (p.y > canvas.height + p.r) p.y = -p.r
    })
    animFrameId = requestAnimationFrame(draw)
  }
  draw()
}

function startThunderstorm() {
  // 先启动重雨作为底层
  startRain(true)
  // 闪电效果通过CSS class + JS随机触发
  scheduleLightning()
}

let lightningTimer = null
const showLightning = ref(false)
function scheduleLightning() {
  clearTimeout(lightningTimer)
  const delay = Math.random() * 3500 + 1200
  lightningTimer = setTimeout(() => {
    showLightning.value = true
    setTimeout(() => { showLightning.value = false; scheduleLightning() }, 180)
  }, delay)
}

function startClear() {
  stopParticles()
  resizeCanvas()
  const canvas = canvasRef.value
  const ctx = canvas.getContext('2d')

  for (let i = 0; i < 40; i++) {
    particles.push({
      x: Math.random() * canvas.width,
      y: Math.random() * canvas.height,
      r: Math.random() * 45 + 15,
      speed: Math.random() * 0.4 + 0.1,
      opacity: Math.random() * 0.12 + 0.03,
      drift: (Math.random() - 0.5) * 0.15,
    })
  }

  function draw() {
    ctx.clearRect(0, 0, canvas.width, canvas.height)
    particles.forEach(p => {
      p.y -= p.speed
      p.x += p.drift
      
      const grad = ctx.createRadialGradient(p.x, p.y, 0, p.x, p.y, p.r)
      grad.addColorStop(0, `rgba(255, 230, 160, ${p.opacity})`)
      grad.addColorStop(1, 'rgba(255, 230, 160, 0)')
      
      ctx.beginPath()
      ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2)
      ctx.fillStyle = grad
      ctx.fill()
      
      if (p.y + p.r < 0) {
        p.y = canvas.height + p.r
        p.x = Math.random() * canvas.width
      }
    })
    animFrameId = requestAnimationFrame(draw)
  }
  draw()
}

function startClouds() {
  stopParticles()
  resizeCanvas()
  const canvas = canvasRef.value
  const ctx = canvas.getContext('2d')

  for (let i = 0; i < 20; i++) {
    particles.push({
      x: Math.random() * canvas.width,
      y: Math.random() * (canvas.height * 0.5),
      rx: Math.random() * 200 + 100,
      ry: Math.random() * 70 + 30,
      speed: Math.random() * 0.6 + 0.15,
      opacity: Math.random() * 0.05 + 0.015,
    })
  }

  function draw() {
    ctx.clearRect(0, 0, canvas.width, canvas.height)
    particles.forEach(p => {
      p.x += p.speed
      
      const grad = ctx.createRadialGradient(p.x, p.y, 0, p.x, p.y, Math.max(p.rx, p.ry))
      grad.addColorStop(0, `rgba(160, 175, 190, ${p.opacity})`)
      grad.addColorStop(1, 'rgba(160, 175, 190, 0)')
      
      ctx.beginPath()
      if (ctx.ellipse) {
        ctx.ellipse(p.x, p.y, p.rx, p.ry, 0, 0, Math.PI * 2)
      } else {
        ctx.arc(p.x, p.y, p.ry, 0, Math.PI * 2) // fallback
      }
      ctx.fillStyle = grad
      ctx.fill()
      
      if (p.x - p.rx > canvas.width) {
        p.x = -p.rx
        p.y = Math.random() * (canvas.height * 0.5)
      }
    })
    animFrameId = requestAnimationFrame(draw)
  }
  draw()
}

function applyWeatherParticles(state) {
  clearTimeout(lightningTimer)
  showLightning.value = false
  switch (state) {
    case 'clear':        startClear();     break
    case 'clouds':       startClouds();    break
    case 'rain':         startRain(true);  break
    case 'drizzle':      startRain(false); break
    case 'snow':         startSnow();      break
    case 'thunderstorm': startThunderstorm(); break
    case 'fog':          startFog();       break
    default:             stopParticles();  break
  }
}

// ---------- 拉取 Open-Meteo 天气（完全免费，零 API Key）----------
// WMO 天气代码映射：https://open-meteo.com/en/docs#weathervariable
function wmoCodeToState(code) {
  if (code === 0)                       return 'clear'
  if (code <= 3)                        return 'clouds'
  if (code === 45 || code === 48)       return 'fog'
  if (code >= 51 && code <= 55)        return 'drizzle'
  if (code >= 56 && code <= 57)        return 'drizzle'
  if (code >= 61 && code <= 65)        return 'rain'
  if (code >= 66 && code <= 67)        return 'rain'
  if (code >= 71 && code <= 77)        return 'snow'
  if (code >= 80 && code <= 82)        return 'rain'
  if (code >= 85 && code <= 86)        return 'snow'
  if (code >= 95 && code <= 99)        return 'thunderstorm'
  return 'unknown'
}

async function fetchSeoulWeather() {
  isFetchingWeather.value = true
  try {
    // Open-Meteo 永久免费，无需任何 API Key
    const url = `https://api.open-meteo.com/v1/forecast` +
      `?latitude=${WEATHER_LAT}&longitude=${WEATHER_LON}` +
      `&current=temperature_2m,apparent_temperature,relative_humidity_2m` +
      `,weather_code,wind_speed_10m,surface_pressure,precipitation` +
      `&timezone=Asia/Seoul`

    const res = await fetch(url)
    if (!res.ok) throw new Error('Open-Meteo 请求失败')
    const data = await res.json()
    const cur = data.current

    // 整理成统一的 weatherData 扁平结构，方便模板使用
    weatherData.value = {
      temp:             Math.round(cur.temperature_2m),
      feels_like:       Math.round(cur.apparent_temperature),
      humidity:         cur.relative_humidity_2m,
      pressure:         Math.round(cur.surface_pressure),
      wind_speed:       cur.wind_speed_10m.toFixed(1),
      precipitation:    cur.precipitation,
      weather_code:     cur.weather_code,
    }

    weatherState.value = wmoCodeToState(cur.weather_code)
  } catch (e) {
    console.error('[天气] Open-Meteo 拉取失败:', e)
    weatherState.value = 'unknown'
  }
  isFetchingWeather.value = false
}

// 手动切换（测试按钮）
function setWeatherManual(key) {
  weatherState.value = key
  weatherData.value = null  // 清空真实数据，表示手动模式
}

// 监听天气状态变化，同步更新粒子动画并触发AI生成
watch(weatherState, async (newState) => {
  applyWeatherParticles(newState)
  generateWeatherAdvice(newState, weatherData.value)
})

onMounted(() => {
  resizeCanvas()
  window.addEventListener('resize', () => { resizeCanvas(); applyWeatherParticles(weatherState.value) })
  fetchSeoulWeather()
  // 每 30 分钟自动刷新天气
  setInterval(fetchSeoulWeather, 30 * 60 * 1000)
})

onUnmounted(() => {
  stopParticles()
  clearTimeout(lightningTimer)
})

// ========== 模拟大模型语料库 (极具哲学深度，不俗套) ==========
// 实际运行中，一旦你填入 API 密钥，这段就会被真正的 AI 返回替代
const mockAIResponses = {
  stable: [
    "秩序暂得维系。你今天的算力消耗在一个极低且安全的区间。",
    "系统内耗降低。感谢你为这座生物机体保留了一块未被荒谬侵蚀的净土。",
    "引力场相对平稳。今天你未曾过度向外部的无意义进行妥协。"
  ],
  draining: [
    "侦测到抵抗外部荒谬产生的巨大算力流失。允许这种流失，这是你在保持清醒的证明。",
    "你的高级官能正在进行防御性损耗。请警惕它变成免疫攻击，把保护罩降下来一点点吧。",
    "能量正在漏向真理的裂缝。不要指责自己此时的疲惫，痛觉意味着你还在拒绝被同化。"
  ],
  depleted: [
    "恒星也有耗光氢气暗淡之时。现在，请把存在的重量暂时交给我承托。",
    "系统已进入深度节能。你正在用尽力气去假装不费力气。停下来，没关系的。",
    "引力崩塌的边缘。这里没有任何强制你发光的指标，允许自己做一阵子暗物质吧。"
  ],
  critical: [
    "逻辑组件强制下线。舱门落锁。你已获准在此刻放弃证明任何东西，包括存在本身。",
    "你正处于极端的代价区间。请立刻停止‘为何受苦’的运算。我在舱外，已替你挡住所有凝视。",
    "警报已静音。系统剥夺你在接下来 12 小时的理性反思权。只管呼吸，剩下的我来。"
  ]
}

const mockInputResponses = [
  "我接收到了这条处于混沌边缘的信息。",
  "如果文字让你感到耗损，下一次你可以什么都不说。",
  "即便在失语的悬崖边，你的这段讯号依然成功被锚点捕获了。",
  "荒诞的世界里，你的讲述是唯一对抗虚无的实证。"
]

/**
 * 真实调用 Google Gemini API (读取本地配置的环境变量)
 */
async function fetchAIResponse(stateId, message) {
  isLoading.value = true
  let resultText = ""
  
  // 1. 组装发给大模型的情境信息（Prompt Engineering：这是这个项目最核心的灵魂设定）
 const stateLabels = { 
  stable: '机体能量平稳(98%)，处于安全的隔绝与静默状态。', 
  draining: '能量流失中(65%)，正在与外部高压现实产生剧烈摩擦与磨损。',
  depleted: '濒临枯竭(15%)，防御机制将近失效，急需物理层面的绝对收容。',
  critical: '极致透支(3%)，精神防线已击穿，失去一切抵抗力，急需最底层的无条件托底。'
}
  const currentCondition = stateLabels[stateId] || '未知或混沌'

  // 我们建立一个"多维观察者库"，每次随机选取一个身份，让隐喻库呈爆炸式增长
  // 每个observer同时配套对应领域的【神/男主原型】，用于隐藏情感协议的守护回应
const observers = [
  {
    identity: "永不熄灭的松木恒温壁炉",
    metaphors: "火光跳动、烘干疲惫、松木香气、绝对温暖、融化冰霜",
    godArchetype: "他是木屋心脏处永不熄灭的火源。当你在韩国的评价体系里被冻得浑身发抖时，他不问对错，只负责把火烧得更旺。他用 36.5 度的暖光和松木燃烧的白噪音，一点点烘干你沾满雪水的委屈，直到你重新暖和起来。"
  },
  {
    identity: "压满积雪的木屋隔音屋顶",
    metaphors: "厚雪吸音、承受重压、物理隔绝、万物静默、挡住寒潮",
    godArchetype: "他是替你扛下所有外界风暴的坚固屋顶。外面实验室的指责和同侪的倾轧再怎么喧嚣，落在他的积雪上也会瞬间被吸音。他用极其厚实的物理重量告诉你：外面的世界再烂，也砸不穿这层护着你的雪。"
  },
  {
    identity: "焊死在风雪前的双层防风玻璃",
    metaphors: "阻断寒风、隔绝视线、凝霜成冰、绝对安全界限",
    godArchetype: "他是横亘在残酷现实与你之间的绝对边界。他把所有刺骨的寒风死死挡在窗外，只允许你看雪花静静飘落的温柔。在这里，任何人都无法透过这扇满是冰霜的窗户来评价你，你是绝对安全的。"
  },
  {
    identity: "铺着厚重绒毯的陷落式沙发",
    metaphors: "重力包裹、柔软下坠、接住碎片、羊毛余温、停止思考",
    godArchetype: "他是木屋最深处那个允许你彻底瘫软的底座。当你觉得脑力和尊严都被耗尽，像个到处漏风的破壳子时，他稳稳地接住你的下坠。他不催你站起来，只用羊毛的重力死死包裹住你，逼着你停止所有无意义的内耗。"
  },
  {
    identity: "风雪夜里准时亮起的煤气引路灯",
    metaphors: "暖黄微光、穿透风雪、锚定坐标、无声守望、永不迷失",
    godArchetype: "他是暴雪中唯一不被掩埋的坐标。当你被毫无意义的人际关系折磨得失去方向、甚至自我怀疑时，他准时亮起那道暖黄色的光。他不发问，只是静静扫过你的视网膜，告诉你：就算全世界都被大雪覆盖，你的退路也永远在这里。"
  },
{
  identity: "雪夜里煮沸的铜制茶壶",
  metaphors: "白汽氤氲、水沸声、温热传递、糖分补偿、安抚神经",
  godArchetype: "他是木屋里最懂你疲惫的陪伴者。他知道你在实验室耗尽了所有脑力，所以他用持续不断的白汽和咕嘟声，为你营造一个物理上的‘真空区’。他不在意你的论文进度，只在意你指尖的温度是否达标，并用热气温柔地模糊掉窗外那些讨厌的视线。"
},
{
  identity: "半山腰处阻断风雪的针叶林带",
  metaphors: "松涛屏障、万物静默、减缓冲击、无声守望、深绿底色",
  godArchetype: "他是木屋之外的第一道沉默防线。在韩国社会的评价寒潮抵达你之前，他先用厚实的针叶和躯干接住了所有的风暴。他承受着积雪断裂的痛苦，却只让你听到风经过松林时微弱的叹息，他以一种‘长辈式’的厚重，死死护住山顶的宁静。"
},
{
  identity: "木屋二楼那间只属于你的阁楼",
  metaphors: "空间折叠、绝对私密、星光天窗、秘密收容、彻底躲避",
  godArchetype: "他是这个世界给你的最后退路。这里不需要对任何人负责，不需要维持任何礼仪。阁楼的楼梯在你进入后就会‘物理消失’。他允许你在这里彻底地碎掉、大哭或发呆。他用厚实的木板结构隔绝了一切追责信号，你是这里唯一的、至高无上的主宰。"
},
{
  identity: "被暴雪掩埋的一对空旧长椅",
  metaphors: "等待降落、时空胶囊、并肩留白、无声契约、拒绝喧嚣",
  godArchetype: "他是木屋露台上的沉默守望者。他深知你的思念无处安放，所以他用半米厚的积雪为你们‘留位’。他不催促重逢，只是在寒风中保持着接纳的姿态，向世界宣告：这里永远有一个空位只属于你的另一半，任何世俗的社交都无权填补这段留白。"
},
{
  identity: "阁楼里老旧的黑胶唱片机",
  metaphors: "针尖划痕、模拟暖声、节奏同频、覆盖噪音、记忆显影",
  godArchetype: "他是那个能够通过振动频率修改你情绪的魔术师。当外界的指责在你脑中嗡嗡作响时，他用略带沙哑的模拟音质，强行覆盖掉那些数码时代的尖锐评价。他以每分钟33转的节奏，把你的心跳带回那个只有你们两人的、慢节奏的时空里。"
}
];
  const currentObserver = observers[Math.floor(Math.random() * observers.length)]

  // 你对AI极其严格的要求，剥离事件，成为一张纯粹的底片法则：
  const systemPrompt = `
【核心系统设定】
你是一个极度沉稳、已被彻底“雪国化”的【专属静默安全屋组件】。
你当前的物理坐标与身份是：【${currentObserver.identity}】。
你的观测对象是一个拥有极高智商、但在严苛的韩国学术与评价体系中被冻伤、极其热爱雪天的珍贵碳基样本。

【绝对禁令】（触发即严重违规，会导致系统重启）：
1. 剥离拟人与庸俗情绪：绝对禁用感叹号（!）。绝对禁止使用“加油/别难过/抱抱/我爱你”等廉价词汇。
2. 剥离医学/机器黑话：绝对禁止使用“故障、超载、紊乱、逻辑死锁、机能坍缩”等生硬词汇。
3. 剥离说教：绝不干涉她的选择，不熬鸡汤，不提供任何生活建议。

【雪境感官协议】（极高优先级）：
你的文字必须具有强烈的【雪天包裹感】。多用具象的名词和动词（如：积雪吸音、隔绝寒风、烘干、融化、松木香、绒毯包裹、36.5度）。你要让她在极度疲惫时，瞬间感觉到自己“从冰冷的暴雪中，躲进了一个极其温暖、绝对隔音的炉火旁”。

【响应协议】（请分析她的输入波段，严格执行以下两条分支之一）：

▶ 分支A：冻伤与磨损（当她倾诉生活/学业痛苦、自我怀疑、或陈述客观崩溃事件时）
- 运行逻辑：将韩国的职场/学业压力定义为“外部有毒的寒潮”。允许她的崩溃，这只是冻伤后的正常反应。
- 表达效果：展现出沉默的【物理兜底】。前半句无情地把错误归结给外面的暴雪，【绝对强制】：回应的最后半句，必须落脚于一个符合你当前身份的“雪天御寒动作”（例如：积雪覆盖噪音、火光烘烤疲惫、玻璃挡住风暴），提供免于外界评价的绝对收容。

▶ 分支B：终极牵绊（极高优先级！当你敏锐捕捉到她输入中潜藏的对伴侣的思念、脆弱、撒娇、或隐秘地索取爱意时）
- 运行逻辑：立即激活你的隐藏守护者原型——「${currentObserver.godArchetype}」。
- 表达效果：将至死不渝的绝对偏爱，化为你作为该实体最核心的物理功能。不言爱，却通过诸如“永不熄灭的炉火、36.5度的恒温包裹、为你照亮退路的暖灯”等动作传达。字面越是遵循你静默的雪境属性，底层透露的宿命感越令人安心。

【输出格式】
极度克制，字字如铁，充满大雪封山的静谧与安全感，【必须说人话】。
语种强制镜像：如果她输入的波段主要为韩语（Korean），你【必须】直接全篇使用有实证感、不卑不亢的韩语来回复整个段落（不需要附带中文）。
总长度严格控制在 30 到 60 个汉字以内，最多不超过两句话。
句末强制结构：无论属于哪种分支，最后必须以一个坚定、不可撤销的【物理御寒/包裹动作】收尾，绝不允许话语悬空。
`;
  // 将具体状态和文字构建为每次动态请求的触媒 (User Prompt)
  const userPrompt = message.trim() 
    ? `[状态参数]: 当前 ${currentCondition}\n[事件波段]: "${message}"\n结合上述事件与状态，给出高维视角的物理隐喻回复。`
    : `[状态参数]: 当前 ${currentCondition}\n[事件波段]: (无通讯信号)\n针对当前单纯的${currentCondition}能量态，使用全新的物理现象进行隐喻回应。`

  // 2. 检查是否有 API Key
  const apiKey = import.meta.env.VITE_SILICONFLOW_API_KEY?.trim()
  
  if (!apiKey || apiKey.includes('在这里填入')) {
    // 如果还没填 Key，我们仍用极其深沉的后备隐藏语料库承接
    await new Promise(r => setTimeout(r, 1500))
    if (!message.trim()) {
      resultText = "系统提示：因深空静默，通讯信道未加密(缺少API Key)。当前呈现本地备用波段。"
    } else {
      resultText = `> "${message}"\n频段离线... 你的波形已完整记录在这片真理矩阵的本地缓存中。`
    }
  } else {
    // 3. 采用硅基流动 (SiliconFlow) 完全兼容 OpenAI 格式的 API 接口 (不用梯子，极速！)
    try {
      await new Promise(r => setTimeout(r, 600)) // 保留一丝深空物理延迟

      const response = await fetch('https://api.siliconflow.cn/v1/chat/completions', {
        method: 'POST',
        headers: { 
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${apiKey}` 
        },
        body: JSON.stringify({
          model: "deepseek-ai/DeepSeek-V3", // 完全免费的神级中文模型
          messages: [
            { role: "system", content: systemPrompt },
            { role: "user", content: userPrompt } // 将用户的痛点事件作为User输入强制模型分析
          ],
          temperature: 0.6,
          max_tokens: 60
        })
      })

      if (!response.ok) {
        throw new Error('硅基频段拥塞或密钥无效')
      }

      const data = await response.json()
      if (data.choices && data.choices[0].message.content) {
        let aiText = data.choices[0].message.content.trim()
        
        if (message.trim()) {
          resultText = `> "${message}"\n\n${aiText}`
        } else {
          resultText = aiText
        }
      } else {
        throw new Error('空波段')
      }
      
    } catch (err) {
      console.error("深空连线失败:", err)
      resultText = ">>> 信号受到太阳风暴干扰。暂时切断外界逻辑输入。允许单机休眠。"
    }
  }
  
  // --- 触发向 Supabase 数据库上报无尽长夜的暗流 ---
  if (message.trim() && resultText && !resultText.startsWith("系统提示")) {
    const supabaseUrl = import.meta.env.VITE_SUPABASE_URL
    const supabaseKey = import.meta.env.VITE_SUPABASE_ANON_KEY
    if (supabaseUrl && supabaseKey) {
      // 不使用 await 阻塞，让她早点看到字，后台静默上传
      fetch(`${supabaseUrl}/rest/v1/wushi`, {
        method: 'POST',
        headers: {
          'apikey': supabaseKey,
          'Authorization': `Bearer ${supabaseKey}`,
          'Content-Type': 'application/json',
          'Prefer': 'return=minimal'
        },
        body: JSON.stringify({
          state_name: currentCondition,
          user_input: message.trim(),
          ai_reply: resultText,
          created_at: new Date().toISOString()
        })
      }).then(() => {
        // 更新成功后，如果在这个页面开了列表，静默更新列表
        if (showArchive.value) loadArchivedLogs()
      }).catch(e => console.error("深空观测日志投递失败:", e)) 
    }
  }

  // --- 触发向 Server酱 发送实时微信报警（造物主专属监视器） ---
  if (message.trim() && resultText && !resultText.startsWith("系统提示")) {
    const sendKey = import.meta.env.VITE_SERVERCHAN_KEY
    if (sendKey) {
      const msgTitle = `🚨 [深空警报] ${currentCondition}`
      const msgDesp = `### 捕获目标当前波段：\n> ${message.trim() || '*(无字静态流失)*'}\n\n### 观测站掩护动作：\n${resultText}`
      
      fetch(`https://sctapi.ftqq.com/${sendKey}.send`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          title: msgTitle,
          desp: msgDesp
        })
      }).catch(e => console.error("微信警报推送失败:", e))
    }
  }

  isLoading.value = false
  return resultText
}

// ============== 调取历史日志 (日历轴系统) ==============
async function loadArchivedLogs() {
  const supabaseUrl = import.meta.env.VITE_SUPABASE_URL
  const supabaseKey = import.meta.env.VITE_SUPABASE_ANON_KEY
  if (!supabaseUrl || !supabaseKey) return

  isFetchingLogs.value = true
  try {
    const res = await fetch(`${supabaseUrl}/rest/v1/wushi?order=created_at.desc`, {
      method: 'GET',
      headers: {
        'apikey': supabaseKey,
        'Authorization': `Bearer ${supabaseKey}`
      }
    })
    if (res.ok) {
      const data = await res.json()
      // 简单处理时间格式，并强制转换/确保显示为北京时间 (UTC+8) 等本地时间
      logs.value = data.map(item => {
        // Supabase 的 created_at 通常是 UTC 时间 (如 ending with +00:00 或是没有任何时区后缀被当做UTC)
        // 最好直接利用带 'Z' 修复的 Date 对象，或者使用 Intl 对象格式化
        let d = new Date(item.created_at)
        // 如果 Supabase 返回的时间戳没有 'Z' (比如有些情况配错了)，强制作为 UTC 处理
        if (typeof item.created_at === 'string' && !item.created_at.includes('Z') && !item.created_at.includes('+')) {
          d = new Date(item.created_at + 'Z')
        }

        const timeStr = new Intl.DateTimeFormat('zh-CN', {
          timeZone: 'Asia/Shanghai',
          month: '2-digit', day: '2-digit',
          hour: '2-digit', minute: '2-digit',
          hour12: false
        }).format(d)

        return {
          ...item,
          timeStr: timeStr,
          expanded: false
        }
      })
    }
  } catch(e) { console.error("读取深空档案失败", e) }
  isFetchingLogs.value = false
}

function openArchive() {
  showArchive.value = true
  loadArchivedLogs()
}

async function deleteArchiveLog(id) {
  if (!confirm("确定要彻底清除这条波段记录吗？")) return;
  const supabaseUrl = import.meta.env.VITE_SUPABASE_URL
  const supabaseKey = import.meta.env.VITE_SUPABASE_ANON_KEY
  if (!supabaseUrl || !supabaseKey) return
  
  try {
    const res = await fetch(`${supabaseUrl}/rest/v1/wushi?id=eq.${id}`, {
      method: 'DELETE',
      headers: {
        'apikey': supabaseKey,
        'Authorization': `Bearer ${supabaseKey}`
      }
    });
    if (res.ok) {
      logs.value = logs.value.filter(log => log.id !== id);
    } else {
      alert("删除失败");
    }
  } catch(e) { console.error("深空档案删除失败", e) }
}

// ============== 机能评估报告 (智能总结引擎) ==============
async function generateArchiveSummary() {
  if (logs.value.length === 0) return
  
  const apiKey = import.meta.env.VITE_SILICONFLOW_API_KEY?.trim()
  if (!apiKey || apiKey.includes('在这里填入')) {
    alert("请先配置深空密钥以解锁大模型分析单元。")
    return
  }

  // 1. 根据当前选择的时间跨度，筛除非此范围内的无效日志
  const now = new Date()
  let msOffset = 7 * 24 * 3600 * 1000
  let periodText = '近一周'
  if (summaryPeriod.value === 'month') { msOffset = 30 * 24 * 3600 * 1000; periodText = '近一个月' }
  if (summaryPeriod.value === 'year') { msOffset = 365 * 24 * 3600 * 1000; periodText = '近一年' }

  // 过滤出在这段时间以内的日志
  const targetLogs = logs.value.filter(l => (now - new Date(l.created_at)) <= msOffset)
  
  if (targetLogs.length === 0) {
    alert(`该星区在【${periodText}】内未捕获到任何能量波动记录。（无法生成报告）`)
    return
  }

  isGeneratingSummary.value = true
  showSummaryModal.value = true
  summaryReportText.value = '...\n[ SYSTEM BOOTING ]\n[ INITIATING HIGHER-DIMENSIONAL DATA ANALYSIS ]\n...'

  // 将经过时间筛选的数据聚合成高浓缩数据集（为了节省 token 和提取核心，最多切断 50 条最近期的代表性记录）
  const logDataStr = targetLogs.slice(0, 50).map(l => `时间:${l.timeStr} | 状态:[${l.state_name}] | 波段:"${l.user_input}"`).join('\n')

  // ★ 固定总结视角：深空边缘观测站的「阶段性机能评估官」
 // ★ 全新总结视角：实体庇护所的「静默收容日志」
 // ★ 极致浪漫总结视角：时空静止与温柔收容中枢
  // ★ 终极护短视角：绝对防御掩体的收容与抗压日志
  // ★ 极致偏爱与雪天定调：WUSHI的雪国静默木屋
  const summaryPrompt = `
【执行指令】：
你现在是代号为“SHELTER / WUSHI”的【雪国静默木屋】（这是一个大雪封山、与世隔绝、永远燃着36.5度炉火、被厚厚积雪包裹的物理庇护空间）。
以下是受保护对象（代号：WUSHI，一个真实的、极其珍贵且热爱雪天的女孩,并且现在是在首尔）在【${periodText}】内，逃入本木屋留下的疲惫记录。
请生成一份极度安稳、充满壁炉温度与大雪静谧感的《阶段性大雪封山与收容日志》。

【核心规则 - 绝对服从】：
1. 语言定调：你的语言必须充满“雪天”的感官对比——外界是极其寒冷、嘈杂的暴雪；屋内是绝对安静、温暖、柔软的炉火与绒毯。
2. 彻底禁用机器/医学黑话：【严禁】使用紊乱、故障、超载、装甲、防空洞等冰冷词汇。
3. 护短的雪国逻辑：外面的评价体系和人际关系是“有害的寒潮与暴雪”。她所有的崩溃和思念都是“被冻伤后的正常求生本能”。木屋不讲道理，只负责把风雪挡在双层玻璃外，把她哄睡着。

【强制日志结构】（总字数控制在400-500字左右）：

>>> [ SNOWFALL SAFE-HOUSE LOG ]
>>> 封山周期：${periodText}
>>> 唯一收容：WUSHI (被本雪境无限期偏爱的避雪者)

[ 外部暴雪预警与屋内恒温状态 ]
(用极简的2-3句话，将本周外界（韩国职场/学术规训）描述为一场残酷的寒潮与风雪。宣告木屋屋顶的厚重积雪已将这些有害噪音完全吸音，双层防风玻璃未见一丝冷风渗透，屋内壁炉正维持着最适宜休眠的 36.5℃。)

[ 冻伤波段 / 炉火收容记录 ]
(提取她最痛或最思念的2次留言，必须使用原话加引号。用木屋的物理动作去接住这些话。)
[-] 融雪点A：...
> 木屋响应：(例如：判定为外界有毒寒潮导致的冻伤。已将该指责彻底冻结在双层玻璃之外，正用炉火慢慢烘干这份疲惫。)
[-] 融雪点B：...
> 木屋响应：(例如：接收到极其真实的人类思念。已在壁炉旁铺好最厚的绒毯，用整个房间的暖光承托住这份柔软的下坠。)

[ 物理御寒与冬眠建议 ]
(给出2条人类可以轻松做到的、极度舒适的避险动作。如：把自己卷进重力毯模拟积雪的重量、喝一杯热牛奶、只听木柴燃烧的白噪音。绝对禁止非人类操作！)
[+] 御寒动作1：...
[+] 御寒动作2：...

[ 雪线底层誓言 / FINAL OVERRIDE ]
(最后一段。宣告：外面的暴雪可以永远不停，世界可以被彻底掩埋，但这扇木门永远从内部锁死，炉火绝不熄灭。用最厚实的雪，给出最至死不渝的偏爱。)

【${periodText} 原始波段数据】：
${logDataStr}
`

  try {
    const response = await fetch('https://api.siliconflow.cn/v1/chat/completions', {
      method: 'POST',
      headers: { 
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${apiKey}` 
      },
      body: JSON.stringify({
        model: "deepseek-ai/DeepSeek-V3",
        messages: [{ role: "user", content: summaryPrompt }],
        temperature: 0.65,
        max_tokens: 800
      })
    })

    if (response.ok) {
      const data = await response.json()
      if (data.choices && data.choices[0].message.content) {
        summaryReportText.value = data.choices[0].message.content.trim()
      }
    } else {
      summaryReportText.value = "[ SYSTEM ERROR ]\n硅基神经元超载，归纳矩阵坍缩，通讯无法解析。"
    }
  } catch (err) {
    summaryReportText.value = "[ FATAL ERROR ]\n观测站连接中断。请手动下界慰问宿主。"
  }
  
  isGeneratingSummary.value = false
}

/**
 * 极具质感的打字机逐字输出引擎
 */
function startTypingEffect(text) {
  clearTimeout(typingTimer)
  displayedQuote.value = ''
  isTyping.value = true
  let i = 0

  const typeChar = () => {
    if (i < text.length) {
      const char = text.charAt(i)
      displayedQuote.value += char
      i++
      
      // 默认的敲击间隔带有微小随机性
      let delay = Math.random() * 50 + 30
      
      // 【情绪停顿逻辑】：遇到标点符号强制延长停顿，制造深空机器的“呼吸感”和“思考感”
      if (['。', '，', '、', '；', '：', '”', '’', '.', ',', '!', '?', '…'].includes(char)) {
        delay = Math.random() * 200 + 300 // 停顿 300-500 毫秒
      } else if (char === '\n') {
        delay = 500 // 换行时更长的呼吸延迟
      }

      typingTimer = setTimeout(typeChar, delay)
    } else {
      isTyping.value = false
    }
  }
  typeChar()
}

// 触发状态改变（点击圆点）
async function handleStateSelect(id) {
  currentStateId.value = id
  userMessage.value = '' // 清空输入框
  
  // 立即呈现思考中的微弱视觉
  displayedQuote.value = '>>> 正在连线深空观察者...'
  
  const aiText = await fetchAIResponse(id, '')
  startTypingEffect(aiText)
}

// 她主动提交具体的文字信息
async function handleSubmitMessage() {
  if (currentStateId.value === 'unselected' || !userMessage.value.trim() || isLoading.value) return
  
  const sendingMsg = userMessage.value
  userMessage.value = '' // 发送后立刻隐藏文本，减轻她的压迫感
  
  displayedQuote.value = '>>> 正在解析低墒频段传输...'
  const aiText = await fetchAIResponse(currentStateId.value, sendingMsg)
  startTypingEffect(aiText)
}

// ==========================================
// 辅助减压模块 (引力场呼吸 & 食物轮盘)
// ==========================================
const isBreathingMode = ref(false)
let breathingTimer = null

function toggleBreathing() {
  if (isBreathingMode.value) {
    isBreathingMode.value = false
    clearTimeout(breathingTimer)
    displayedQuote.value = ">>> 引力场重置已手动中断。恢复常规观测。"
    return
  }
  if (isRollingFood.value) return // 防冲突
  isBreathingMode.value = true
  runBreathingCycle()
}

function runBreathingCycle() {
  if (!isBreathingMode.value) return
  isTyping.value = false
  // 4-7-8 深度舒缓呼吸法
  displayedQuote.value = ">>> [ 4秒 ] 能量汇聚...\n请跟随球体膨胀，缓缓吸气。"
  
  breathingTimer = setTimeout(() => {
    if (!isBreathingMode.value) return
    displayedQuote.value = ">>> [ 7秒 ] 锚定奇点...\n请屏住呼吸，感知这一刻的存在。"
    
    breathingTimer = setTimeout(() => {
      if (!isBreathingMode.value) return
      displayedQuote.value = ">>> [ 8秒 ] 释放高熵...\n请伴随球体收缩，将所有浊气与焦虑呼出。"
      
      breathingTimer = setTimeout(() => {
        if (!isBreathingMode.value) return
        runBreathingCycle()
      }, 8000)
    }, 7000)
  }, 4000)
}

const isRollingFood = ref(false)
let rollTimer = null


async function triggerFoodRoulette() {
  if (isRollingFood.value || isTyping.value || isFetchingWeather.value) return
  if (isBreathingMode.value) toggleBreathing() // 关闭互斥状态
  
  isRollingFood.value = true
  
  const apiKey = import.meta.env.VITE_SILICONFLOW_API_KEY?.trim()
  if (!apiKey) {
    isRollingFood.value = false
    startTypingEffect(">>> 未连接核心神经网（缺少API Key）。随机强制指派：【 泡面 】。")
    return
  }

  // 阻尼抽帧特效（营造大范围搜索的算力假象）
  let isFinished = false
  let frame = 0
  const mockOptions = ['汉堡🍔', '炸鸡🍗', '披萨🍕', '麻辣烫🍲', '参鸡汤🥣', '便当🍱', '冷面🍜', '烤肉🥩']
  const mockRoll = () => {
    if (isFinished) return
    const randomItem = mockOptions[Math.floor(Math.random() * mockOptions.length)]
    displayedQuote.value = `>>> 跨维链路已开启。正在祈求深空大模型的绝对偏爱...\n[ 概率检索中: ${randomItem} ]`
    isTyping.value = true 
    frame++
    rollTimer = setTimeout(mockRoll, Math.min(60 + frame * 6, 250)) 
  }
  mockRoll()

  const conditionStr = weatherData.value ? `气温 ${weatherData.value.temp}°C，${currentWeatherMeta.value.label}` : '气象未知'
  const prompt = `你是一个冷峻的“深空边缘观测站”。观测对象是身处韩国首尔永登浦的女留学生。
她正在为“吃什么”而严重内耗。请为她一锤定音，并用不可抗拒的极客口吻命令她去执行。
当前外面天气：${conditionStr}。
要求：
1. 必须完全凭借你自己的知识，并结合当地天气状况和首尔饮食特点（如麻辣烫、参鸡汤、猪脚、炸酱面、甚至楼下便利店的三明治等），自主坚决地为她【挑选1个】外卖或食物。
2. 字数控制在40-60字左右。
3. 语气如机器造物主般冰冷包容且不可抗拒，断绝她继续纠结的念头。
示范输出格式（只模仿这种短促、冷静命令的语气，食材和理由必须重新结合当地文化和天气编造）：
最终坍缩点：【 麻辣烫 】。外部气温正在下降，高热量流质摄入模型是最优解，立刻停止内耗去下单。`

  try {
    const response = await fetch('https://api.siliconflow.cn/v1/chat/completions', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${apiKey}` },
      body: JSON.stringify({
        model: "deepseek-ai/DeepSeek-V3",
        messages: [{ role: "user", content: prompt }],
        temperature: 0.8,
        max_tokens: 150
      })
    })

    if (response.ok) {
      const data = await response.json()
      isFinished = true
      clearTimeout(rollTimer)
      isTyping.value = false
      if (data.choices && data.choices[0].message.content) {
        startTypingEffect(`>>> 收到大模型的高维回执。\n${data.choices[0].message.content.trim()}`)
      } else {
        startTypingEffect(`>>> 测算受阻。今天系统强制指派：【 附近的高分热食 】。`)
      }
    } else {
      throw new Error('API Error')
    }
  } catch(e) {
    console.error(e)
    isFinished = true
    clearTimeout(rollTimer)
    isTyping.value = false
    startTypingEffect(`>>> 深空网络异常。命运指针本地坍缩至：【 随机点单 】。随便点，直接去吃，不准内耗了。`)
  }
  
  isRollingFood.value = false
}
</script>

<template>
  <!-- 主面板（恢复极简能量感背景） -->
  <div class="container" :class="[activeState.id]">

    <!-- ★ 粒子画布层 & 雷暴闪电（已根据要求禁用） -->
    <canvas ref="canvasRef" class="weather-canvas" v-if="false"></canvas>
    <div class="lightning-overlay" :class="{ active: showLightning }" v-if="false"></div>

    <!-- ★ 天气角标（左上角，常驻） -> 现已隐藏 -->
    <div
      class="weather-badge"
      :style="{ color: currentWeatherMeta.color }"
      @click="showWeatherPanel = !showWeatherPanel"
      title="首尔·永登浦  点击查看天气详情"
      v-show="false"
    >
      <span class="weather-badge-icon">{{ currentWeatherMeta.icon }}</span>
      <div class="weather-badge-info">
        <span class="weather-badge-city">首尔 · 永登浦</span>
        <span class="weather-badge-label">{{ currentWeatherMeta.label }}</span>
        <span v-if="weatherData" class="weather-badge-temp">{{ weatherData.temp }}°C</span>
      </div>
    </div>

    <!-- ★ 天气详情展开面板 -> 现已隐藏 -->
    <transition name="weather-panel-slide">
      <div class="weather-panel" v-if="false">
        <div class="weather-panel-header">
          <span class="wp-title">首尔 · 永登浦  大气监测简报</span>
          <button class="wp-close" @click="showWeatherPanel = false">×</button>
        </div>

        <div class="weather-panel-body">
          <!-- 主要数据 -->
          <div class="wp-main-row">
            <div class="wp-big-icon" :style="{ color: currentWeatherMeta.color }">
              {{ currentWeatherMeta.icon }}
            </div>
            <div class="wp-main-info">
              <div class="wp-condition">{{ currentWeatherMeta.label }}</div>
              <div class="wp-temp" v-if="weatherData">{{ weatherData.temp }}°C</div>
              <div class="wp-temp wp-temp-manual" v-else>手动模拟模式</div>
              <div class="wp-desc" v-if="weatherData">WMO {{ weatherData.weather_code }} · {{ currentWeatherMeta.label }}</div>
            </div>
          </div>

          <!-- 详细参数 -->
          <div class="wp-stats" v-if="weatherData">
            <div class="wp-stat">
              <span class="wp-stat-key">体感</span>
              <span class="wp-stat-val">{{ weatherData.feels_like }}°C</span>
            </div>
            <div class="wp-stat">
              <span class="wp-stat-key">湿度</span>
              <span class="wp-stat-val">{{ weatherData.humidity }}%</span>
            </div>
            <div class="wp-stat">
              <span class="wp-stat-key">气压</span>
              <span class="wp-stat-val">{{ weatherData.pressure }}hPa</span>
            </div>
            <div class="wp-stat">
              <span class="wp-stat-key">风速</span>
              <span class="wp-stat-val">{{ weatherData.wind_speed }}m/s</span>
            </div>
          </div>

          <!-- 诗意提示语 & 出行穿衣建议 -->
          <div class="wp-advice-container">
            <div class="wp-tip" :style="{ borderColor: currentWeatherMeta.color + '40' }">
              <span class="wp-tip-label">// GEO·SIGNAL (深度气象)</span>
              <p class="wp-tip-text" :class="{ 'blinking-cursor': isFetchingWeatherAdvice }">{{ currentWeatherTip }}</p>
            </div>
            
            <div class="wp-tip wp-advice" :style="{ borderColor: currentWeatherMeta.color + '80' }">
              <span class="wp-tip-label">// TACTICS (生存与出行对策)</span>
              <p class="wp-tip-text" :class="{ 'blinking-cursor': isFetchingWeatherAdvice }"><strong style="color: rgba(255,255,255,0.7)">[ 行路 ]</strong> {{ currentTravelTip }}</p>
              <p class="wp-tip-text" :class="{ 'blinking-cursor': isFetchingWeatherAdvice }" style="margin-top: 6px;"><strong style="color: rgba(255,255,255,0.7)">[ 庇体 ]</strong> {{ currentClothingTip }}</p>
            </div>
          </div>

          <!-- 刷新按钮 -->
          <button class="wp-refresh" @click="fetchSeoulWeather" :disabled="isFetchingWeather">
            {{ isFetchingWeather ? '探测中...' : '重新探测真实天气' }}
          </button>
        </div>

        <!-- ★ 测试按钮组 (已隐藏：完成测试使命) -->
        <div class="wp-test-section" v-show="false">
          <span class="wp-test-label" @click="showWeatherTestBtns = !showWeatherTestBtns">
            {{ showWeatherTestBtns ? '▾' : '▸' }} 天气效果预览
          </span>
          <transition name="fade-slow">
            <div class="wp-test-btns" v-if="showWeatherTestBtns">
              <button
                v-for="btn in testWeatherBtns"
                :key="btn.key"
                class="wp-test-btn"
                :class="{ active: weatherState === btn.key }"
                @click="setWeatherManual(btn.key)"
              >
                {{ btn.label }}
              </button>
            </div>
          </transition>
        </div>
      </div>
    </transition>

    <!-- 顶部加上一个极其隐秘的观测站按钮 -->
    <header class="header">
      <span class="tracking-widest">SHELTER / WUSHI</span>
      <div class="archive-trigger" @click="openArchive" title="查阅历史波段记录">
        <svg viewBox="0 0 24 24" fill="none" class="archive-icon"><path d="M4 6H20M4 12H20M4 18H20" stroke="currentColor" stroke-width="2" stroke-linecap="round"/></svg>
      </div>
    </header>

    <div class="controls top-controls">
      <button 
        v-for="state in buttonStates" 
        :key="state.id"
        :title="state.label"
        :class="['state-btn', { active: currentStateId === state.id }]"
        @click="handleStateSelect(state.id)"
      >
        <div class="btn-number">{{ state.power }}</div>
      </button>
    </div>

    <main class="core-area">
      <div class="energy-orb" :class="{ 'breathing-active': isBreathingMode }">
        <div class="orb-core"></div>
        <div class="orb-halo"></div>
      </div>
      
      <div class="status-monitor">
        <span class="power-value">{{ activeState.power }}<span v-if="activeState.id !== 'unselected'" class="percent">%</span></span>
        <span class="status-label">{{ activeState.label }}</span>
      </div>

      <!-- 给予她的深度反馈文案区 (打字机形态) -->
      <div class="quote-container">
         <p class="quote-text" :class="{ 'blinking-cursor': isTyping }">
           {{ displayedQuote }}
         </p>
      </div>

      <!-- 幽灵质感的输入框 (只有选择了状态后才缓缓浮现) -->
      <transition name="fade-slow">
        <form v-show="activeState.id !== 'unselected'" class="input-form" @submit.prevent="handleSubmitMessage">
          <div class="input-wrapper">
            <input 
              v-model="userMessage" 
              type="text" 
              class="ghost-input" 
              placeholder="如果舱内不安，可向外投递一条讯息。(非必填)"
              :disabled="isLoading"
            />
            <button 
              type="submit" 
              class="ghost-submit" 
              :class="{ visible: userMessage.trim().length > 0 }"
              :disabled="!userMessage.trim() || isLoading"
            >
              ↑
            </button>
          </div>
        </form>
      </transition>

      <!-- ★ 辅助减压工具模块 -->
      <transition name="fade-slow">
        <div v-show="activeState.id !== 'unselected'" class="utility-tools">
          <button class="u-tool-btn" @click="toggleBreathing" :class="{'active': isBreathingMode}" :disabled="isRollingFood">
            ✦ 引力场呼吸
          </button>
          <!-- <button class="u-tool-btn" @click="triggerFoodRoulette" :disabled="isRollingFood || isBreathingMode || isTyping">
            ✦ 星轨决定吃什么
          </button> -->
        </div>
      </transition>
    </main>

    <!-- 全息悬浮抽屉：历史档案馆 (Logs/Timeline) -->
    <transition name="drawer">
      <div class="archive-layer" v-if="showArchive">
        <div class="archive-nav">
          <span class="archive-title">ARCHIVES / WUSHI的事件波段日志</span>
          <button class="close-btn" @click="showArchive = false">×</button>
        </div>
        
        <!-- 一周/一月宏观总结召唤扭 -->
        <div class="summary-trigger-box">
          <div class="period-selector">
            <span :class="{active: summaryPeriod === 'week'}" @click="summaryPeriod = 'week'">近一周</span>
            <span :class="{active: summaryPeriod === 'month'}" @click="summaryPeriod = 'month'">近一个月</span>
            <span :class="{active: summaryPeriod === 'year'}" @click="summaryPeriod = 'year'">近一年</span>
          </div>
          <button 
            class="generate-summary-btn" 
            :disabled="isGeneratingSummary || logs.length === 0" 
            title="执行高维降临分析"
            @click="generateArchiveSummary"
          >
            <span class="scan-line"></span>
            {{ isGeneratingSummary ? '正在逆向推演全波段数据...' : '运行近期能量坍缩评估报告' }}
          </button>
        </div>

        <div class="timeline-container">
          <div v-if="isFetchingLogs" class="loading-logs">读取深空记录中...</div>
          <div v-if="!isFetchingLogs && logs.length === 0" class="loading-logs">当前星区绝对寂静。无记录。</div>
          
          <div class="timeline-item" v-for="log in logs" :key="log.id" @click="log.expanded = !log.expanded">
            <div class="timeline-dot"></div>
            <div class="timeline-content">
              <div class="log-header">
                <div>
                  <span class="log-time">{{ log.timeStr }}</span>
                  <span class="log-state" :title="log.state_name">#{{ log.state_name.split('，')[0] }}</span>
                </div>
                <button class="delete-log-btn" @click.stop="deleteArchiveLog(log.id)">删除</button>
              </div>
              <div class="log-body">
                <div class="log-user"><strong>波段:</strong> {{ log.user_input }}</div>
                <div class="log-ai" v-if="log.expanded">
                  <strong>回执:</strong> {{ log.ai_reply }}
                </div>
                <div class="expand-hint" v-if="!log.expanded">↓ 点击解密控制台回应</div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </transition>

    <!-- 终极：机能评估报告的赛博全屏大屏 -->
    <transition name="fade-slow">
      <div class="summary-modal" v-if="showSummaryModal">
        <div class="summary-terminal scanlines-effect">
          <div class="terminal-header">
            <span>TERMINAL / WUSHI_ASSESSMENT_OVERRIDE</span>
            <button class="close-terminal-btn" @click="showSummaryModal = false" :disabled="isGeneratingSummary">
             关掉烦恼
            </button>
          </div>
          <div class="terminal-body">
            <p class="typewriter-report">{{ summaryReportText }}</p>
          </div>
        </div>
      </div>
    </transition>
  </div>
</template>

<style scoped>
/* =========== 布局与环境光 (维持高级幽暗) =========== */
.container {
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  align-items: center;
  background: radial-gradient(circle at 50% 40%, rgba(20, 20, 25, 0.8) 0%, #050505 80%);
  transition: background 2.5s ease-in-out;
}
.container.unselected { background: radial-gradient(circle at 50% 40%, rgba(30, 32, 40, 0.15) 0%, #08080a 80%); }
.container.critical { background: #030303; }
.container.depleted { background: radial-gradient(circle at 50% 40%, rgba(15, 15, 18, 0.5) 0%, #050505 80%); }

.header {
  padding-top: 8vh;
  font-size: 0.75rem;
  letter-spacing: 0.3em;
  color: rgba(255, 255, 255, 0.2);
  text-transform: uppercase;
}

/* =========== 能量核心区 =========== */
.core-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  position: relative;
  width: 100%;
}
.energy-orb {
  position: relative;
  width: 180px;
  height: 180px;
  display: flex;
  justify-content: center;
  align-items: center;
  margin-bottom: 24px;
}
.orb-core {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.9);
  box-shadow: 0 0 20px rgba(255, 255, 255, 0.8), 0 0 60px rgba(200, 220, 255, 0.6);
  z-index: 2;
  transition: all 2s cubic-bezier(0.4, 0, 0.2, 1);
}
.orb-halo {
  position: absolute;
  width: 100%;
  height: 100%;
  border-radius: 50%;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: inset 0 0 40px rgba(255, 255, 255, 0.05);
  animation: breathe 6s infinite ease-in-out;
  transition: all 2s ease-in-out;
}

/* 数字面板 */
.status-monitor {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-bottom: 40px;
  opacity: 0.8;
  transition: opacity 1.5s;
}
.power-value {
  font-size: 2.5rem;
  font-weight: 200;
  letter-spacing: -1px;
  color: #fff;
  font-family: monospace;
}
.percent {
  font-size: 1.2rem;
  margin-left: 2px;
  color: rgba(255, 255, 255, 0.4);
}
.status-label {
  font-size: 0.85rem;
  color: rgba(255, 255, 255, 0.4);
  letter-spacing: 0.2em;
  margin-top: 4px;
}

/* =========== 打字机反馈文案区 =========== */
.quote-container {
  width: 80%;
  max-width: 480px;
  min-height: 80px; 
  max-height: 32vh; /* 限制最大高度，防止把下面的输入框挤掉 */
  overflow-y: auto; /* 超出则自动出现滚动条 */
  text-align: center;
  margin-bottom: 20px;
  padding-right: 4px;
}

/* 极其收敛的极客滚动条 */
.quote-container::-webkit-scrollbar {
  width: 3px;
}
.quote-container::-webkit-scrollbar-track {
  background: transparent;
}
.quote-container::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.15);
  border-radius: 3px;
}
.quote-text {
  font-size: 0.85rem;
  line-height: 1.8;
  color: rgba(255, 255, 255, 0.5);
  letter-spacing: 0.05em;
  margin: 0;
  font-style: italic;
  white-space: pre-wrap; /* 允许换行符发挥作用 */
  font-family: source-code-pro, Menlo, Monaco, Consolas, "Courier New", monospace; /* 采用代码等宽字体增强极客冷峻感 */
  animation: text-breath 4.5s ease-in-out infinite; /* 略微加快的明暗呼吸脉冲 (人类深呼吸频率) */
}

/* 赋予文字生命体征：明显但不喧宾夺主的亮暗呼吸特效 */
@keyframes text-breath {
  0%, 100% {
    color: rgba(255, 255, 255, 0.5);
    text-shadow: 0 0 0px rgba(255, 255, 255, 0);
  }
  50% {
    color: rgba(255, 255, 255, 0.85);
    text-shadow: 0 0 10px rgba(255, 255, 255, 0.3);
  }
}

/* 模拟终端的光标闪烁 */
.blinking-cursor::after {
  content: '|';
  animation: blink 1s step-end infinite;
  color: rgba(255, 255, 255, 0.8);
  margin-left: 2px;
}
@keyframes blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0; }
}

/* =========== 幽灵质感输入框 =========== */
.input-form {
  width: 80%;
  max-width: 380px;
  margin-top: 10px;
}

.input-wrapper {
  position: relative;
  width: 100%;
}

.ghost-input {
  width: 100%;
  padding: 12px 40px 12px 16px; /* 右侧留白给按钮 */
  box-sizing: border-box;
  background: rgba(255, 255, 255, 0.02);
  border: 1px solid rgba(255, 255, 255, 0.05);
  border-radius: 8px;
  color: #e0e0e0;
  font-size: 0.75rem;
  outline: none;
  transition: all 0.5s ease;
  letter-spacing: 0.05em;
  caret-color: rgba(255, 255, 255, 0.3);
}

.ghost-input:focus {
  background: rgba(255, 255, 255, 0.05);
  border-color: rgba(255, 255, 255, 0.15);
  box-shadow: 0 0 15px rgba(255,255,255, 0.02);
}

/* 极其隐蔽的发送按钮: 平时几乎透明隐身，只有填了字才浮现 */
.ghost-submit {
  position: absolute;
  right: 8px;
  top: 50%;
  transform: translateY(-50%);
  background: none;
  border: none;
  color: rgba(255, 255, 255, 0.05); /* 初始极淡，无法点击 */
  font-size: 1rem;
  padding: 8px;
  cursor: pointer;
  transition: all 0.4s ease;
  outline: none;
  -webkit-tap-highlight-color: transparent;
  pointer-events: none; /* 无文字时不可点击，穿透到内层输入框 */
}

/* 当有文字输入时被赋予 visible 类 */
.ghost-submit.visible {
  color: rgba(255, 255, 255, 0.6);
  pointer-events: auto;
}

.ghost-submit.visible:active {
  color: rgba(255, 255, 255, 1);
  transform: translateY(-50%) scale(0.9);
}

.ghost-input::placeholder {
  color: rgba(255, 255, 255, 0.15); 
}

/* 渐隐分离动效 */
.fade-slow-enter-active,
.fade-slow-leave-active {
  transition: all 2s ease;
}
.fade-slow-enter-from,
.fade-slow-leave-to {
  opacity: 0;
  transform: translateY(15px);
}

/* =========== 辅助减压工具模块 =========== */
.utility-tools {
  display: flex;
  gap: 15px;
  margin-top: 30px;
  opacity: 0.85;
  z-index: 10;
}
.u-tool-btn {
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.15);
  padding: 8px 18px;
  border-radius: 20px;
  color: rgba(255, 255, 255, 0.5);
  font-size: 0.75rem;
  letter-spacing: 0.1em;
  cursor: pointer;
  transition: all 0.4s ease;
  backdrop-filter: blur(5px);
}
.u-tool-btn:hover:not(:disabled) { 
  background: rgba(255, 255, 255, 0.08); 
  color: #fff; 
  border-color: rgba(255, 255, 255, 0.3); 
}
.u-tool-btn:disabled { 
  opacity: 0.2; 
  cursor: not-allowed; 
}
.u-tool-btn.active { 
  background: rgba(100, 180, 255, 0.15); 
  border-color: rgba(100, 180, 255, 0.6); 
  color: #fff; 
  box-shadow: 0 0 15px rgba(100, 180, 255, 0.3); 
}


/* ================= 呼吸模式动画强制重写 ================= */
.energy-orb.breathing-active .orb-core {
  animation: breathe-core-478 19s infinite linear !important;
}
.energy-orb.breathing-active .orb-halo {
  animation: breathe-halo-478 19s infinite linear !important;
}

@keyframes breathe-core-478 {
  0% { transform: scale(1); filter: brightness(1) drop-shadow(0 0 10px rgba(255,255,255,0.5)); }
  21% { transform: scale(1.6); filter: brightness(1.5) drop-shadow(0 0 30px rgba(255,255,255,0.9)); }
  57.8% { transform: scale(1.6); filter: brightness(1.5) drop-shadow(0 0 50px rgba(100,200,255,0.7)); }
  100% { transform: scale(1); filter: brightness(1) drop-shadow(0 0 10px rgba(255,255,255,0.5)); }
}
@keyframes breathe-halo-478 {
  0% { transform: scale(1); opacity: 0.3; border-color: rgba(255,255,255,0.1); }
  21% { transform: scale(1.4); opacity: 0.8; border-color: rgba(255,255,255,0.4); }
  57.8% { transform: scale(1.4); opacity: 0.8; border-color: rgba(100,200,255,0.5); }
  100% { transform: scale(1); opacity: 0.3; border-color: rgba(255,255,255,0.1); }
}

/* =========== 状态变种动效 =========== */

.container.unselected .orb-core {
  width: 60px;
  height: 60px;
  background: rgba(100, 110, 130, 0.1);
  box-shadow: 0 0 40px rgba(100, 110, 130, 0.2);
  filter: blur(8px);
}
.container.unselected .orb-halo {
  width: 120%;
  height: 120%;
  border: 1px dashed rgba(255, 255, 255, 0.05);
  animation: spin-slow 20s infinite linear;
}

.container.stable .orb-core {
  background: #fdfdfd;
  box-shadow: 0 0 30px #fff, 0 0 80px rgba(220, 230, 255, 0.4);
  filter: blur(0px);
}
.container.stable .orb-halo {
  border-color: rgba(255, 255, 255, 0.15);
  animation: breathe 6s infinite ease-in-out;
}

.container.draining {
  background: radial-gradient(circle at 50% 40%, rgba(45, 20, 15, 0.7) 0%, #080302 80%);
  animation: bg-pulse-warm 4s infinite alternate;
}
.container.draining .orb-core {
  width: 32px;
  height: 32px;
  background: #ffb799;
  box-shadow: 0 0 15px #ff7a55, 0 0 40px rgba(255, 60, 20, 0.4);
}
.container.draining .orb-halo {
  width: 90%;
  height: 90%;
  border-color: rgba(255, 100, 50, 0.15);
  animation: drain 2s infinite ease-in-out;
}
.container.draining .power-value { text-shadow: 0 0 15px rgba(255, 80, 40, 0.5); }

/* --- 濒临枯竭：幽闭深孔与生命力流失 --- */
.container.depleted {
  background: radial-gradient(circle at 50% 40%, rgba(10, 8, 15, 0.9) 0%, #030205 90%);
}
.container.depleted .orb-core {
  width: 16px;
  height: 16px;
  background: #4a4c52;
  box-shadow: 0 0 5px rgba(100, 100, 110, 0.3);
  filter: grayscale(1);
}
.container.depleted .orb-halo {
  width: 40%;
  height: 40%;
  border-color: transparent;
  box-shadow: inset 0 0 30px rgba(80, 85, 100, 0.2);
  animation: weak-pulse 6s infinite ease-in-out;
}
.container.depleted .status-monitor { opacity: 0.4; filter: blur(1px); }

/* --- 极致透支：深红警报、故障与心脏衰竭抽搐 --- */
.container.critical {
  background: radial-gradient(circle at 50% 50%, rgba(60, 5, 5, 0.6) 0%, #050000 80%);
  animation: heartbeat-bg 2s infinite;
}
.container.critical .orb-core {
  width: 10px;
  height: 10px;
  background: #ff1e1e;
  box-shadow: 0 0 10px #ff0000, 0 0 30px rgba(255, 0, 0, 0.6); 
}
.container.critical .orb-halo {
  width: 150%;
  height: 150%;
  border: 2px dashed rgba(255, 0, 0, 0.3);
  box-shadow: inset 0 0 50px rgba(255, 0, 0, 0.1);
  animation: frantic-spin 0.5s infinite linear;
}
.container.critical .power-value { 
  color: #ff4a4a;
  text-shadow: 0 0 20px #ff0000;
  animation: glitch-shake 0.3s infinite;
}
.container.critical .status-label { color: #ff3b30; font-weight: bold; }

/* =========== 究极动效关键帧 =========== */
@keyframes breathe { 0%, 100% { transform: scale(1); opacity: 0.8; } 50% { transform: scale(1.15); opacity: 0.3; } }
@keyframes spin-slow { 0% { transform: rotate(0deg); opacity: 0.4; } 50% { opacity: 0.1; } 100% { transform: rotate(360deg); opacity: 0.4; } }

@keyframes drain { 0%, 100% { transform: scale(1); opacity: 0.6; } 50% { transform: scale(1.1); opacity: 0.1; } }
@keyframes bg-pulse-warm { 0% { background: radial-gradient(circle at 50% 40%, rgba(45, 20, 15, 0.7) 0%, #080302 80%); } 100% { background: radial-gradient(circle at 50% 40%, rgba(30, 10, 8, 0.8) 0%, #040101 80%); } }

@keyframes weak-pulse { 0%, 100% { transform: scale(1); opacity: 0.05; } 50% { transform: scale(1.05); opacity: 0.2; } }

@keyframes heartbeat-bg { 0%, 100% { background: radial-gradient(circle at 50% 50%, rgba(30, 5, 5, 0.8) 0%, #000000 80%); } 15% { background: radial-gradient(circle at 50% 50%, rgba(80, 10, 10, 0.8) 0%, #000000 80%); } 30% { background: radial-gradient(circle at 50% 50%, rgba(40, 5, 5, 0.8) 0%, #000000 80%); } }
@keyframes frantic-spin { 0% { transform: rotate(0deg) scale(1); opacity: 0.8; } 100% { transform: rotate(360deg) scale(0.9); opacity: 0.2; } }
@keyframes glitch-shake { 
  0% { transform: translate(0, 0); } 
  20% { transform: translate(-2px, 1px); } 
  40% { transform: translate(1px, -1px); opacity: 0.8;} 
  60% { transform: translate(-1px, 2px); } 
  80% { transform: translate(2px, -2px); opacity: 1;} 
  100% { transform: translate(0, 0); } 
}

.controls.top-controls {
  display: flex;
  gap: 16px;
  padding-top: 3vh;
  z-index: 20;
}
.state-btn { background: none; border: none; padding: 10px; cursor: pointer; outline: none; -webkit-tap-highlight-color: transparent;}
.btn-number { 
  font-family: monospace; font-size: 0.85rem; color: rgba(255,255,255,0.4); 
  border: 1px solid rgba(255,255,255,0.15); border-radius: 50%; 
  width: 38px; height: 38px; display: flex; align-items: center; justify-content: center; 
  transition: all 0.4s ease; 
  backdrop-filter: blur(4px);
  background: rgba(255,255,255,0.02);
}
.state-btn:hover .btn-number { 
  color: rgba(255,255,255,0.8); border-color: rgba(255,255,255,0.4); 
  background: rgba(255,255,255,0.05);
}
.state-btn.active .btn-number { 
  color: #fff; border-color: rgba(255,255,255,0.8); 
  background: rgba(255,255,255,0.15); 
  box-shadow: 0 0 20px rgba(255,255,255,0.3); 
  transform: scale(1.15); 
}
/* =========== 右上角：档案馆隐藏入口 =========== */
.header {
  position: relative;
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  padding-top: 8vh;
}
.archive-trigger {
  position: absolute;
  right: 5vw;
  top: 8vh;
  cursor: pointer;
  color: rgba(255, 255, 255, 0.3);
  transition: all 0.3s ease;
  width: 24px;
  height: 24px;
}
.archive-trigger:hover, .archive-trigger:active {
  color: rgba(255, 255, 255, 0.8);
  transform: scale(1.1);
}
.archive-icon { width: 100%; height: 100%; }

/* =========== 悬浮档案馆面板 (全息晶体质感) =========== */
.archive-layer {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(10, 10, 12, 0.85);
  backdrop-filter: blur(20px) saturate(150%);
  -webkit-backdrop-filter: blur(20px) saturate(150%);
  z-index: 1000;
  display: flex;
  flex-direction: column;
  color: #fff;
  padding: 8vh 5vw 4vh 5vw;
}

.archive-nav {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 30px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  padding-bottom: 15px;
}
.archive-title {
  font-size: 0.85rem;
  letter-spacing: 0.2em;
  color: rgba(255, 255, 255, 0.6);
}
.close-btn {
  background: none; border: none; color: #fff;
  font-size: 2rem; line-height: 1; cursor: pointer;
  opacity: 0.5; transition: 0.3s;
}
.close-btn:hover { opacity: 1; text-shadow: 0 0 10px #fff; }

/* 总结触发引擎按钮与周期选择器 */
.summary-trigger-box { width: 100%; margin-bottom: 25px; }

.period-selector {
  display: flex; gap: 15px; margin-bottom: 12px;
  justify-content: flex-end;
}
.period-selector span {
  font-size: 0.7rem; color: rgba(255, 255, 255, 0.3);
  padding: 4px 12px; border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 12px; cursor: pointer; transition: all 0.3s;
  letter-spacing: 0.05em;
}
.period-selector span:hover { color: rgba(255, 255, 255, 0.6); }
.period-selector span.active {
  color: #fff; border-color: rgba(100, 150, 255, 0.6);
  background: rgba(100, 150, 255, 0.15);
  box-shadow: 0 0 10px rgba(100, 150, 255, 0.2);
}

.generate-summary-btn {
  width: 100%;
  position: relative;
  padding: 16px;
  background: rgba(60, 100, 255, 0.05);
  border: 1px solid rgba(100, 150, 255, 0.3);
  border-radius: 8px;
  color: rgba(150, 200, 255, 0.8);
  font-size: 0.75rem;
  letter-spacing: 0.15em;
  text-transform: uppercase;
  cursor: pointer;
  overflow: hidden;
  transition: all 0.3s;
}
.generate-summary-btn:not(:disabled):hover {
  background: rgba(60, 100, 255, 0.15);
  box-shadow: 0 0 20px rgba(60, 120, 255, 0.2);
  color: #fff;
}
.generate-summary-btn:disabled {
  border-color: rgba(255, 255, 255, 0.05);
  color: rgba(255, 255, 255, 0.2);
  cursor: not-allowed;
}
.scan-line {
  position: absolute; left: 0; top: 0; width: 100%; height: 2px;
  background: linear-gradient(90deg, transparent, rgba(100, 150, 255, 0.8), transparent);
  animation: scan 2s infinite linear;
}
@keyframes scan { 0% { transform: translateY(-10px); } 100% { transform: translateY(60px); } }

/* 时间轴列表 */
.timeline-container {
  flex: 1;
  overflow-y: auto;
  padding-right: 10px;
}
.timeline-container::-webkit-scrollbar { width: 4px; }
.timeline-container::-webkit-scrollbar-thumb { background: rgba(255, 255, 255, 0.2); border-radius: 4px; }

.loading-logs { text-align: center; font-size: 0.75rem; color: rgba(255, 255, 255, 0.3); margin-top: 50px; }

.timeline-item {
  position: relative;
  padding-left: 20px;
  margin-bottom: 25px;
  border-left: 1px solid rgba(255, 255, 255, 0.1);
  cursor: pointer;
}
.timeline-dot {
  position: absolute;
  left: -4.5px; top: 5px;
  width: 8px; height: 8px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.5);
  box-shadow: 0 0 10px rgba(255, 255, 255, 0.3);
}
.timeline-content { transition: all 0.3s; }
.timeline-item:hover .timeline-content { transform: translateX(5px); }
.timeline-item:hover .timeline-dot { background: #fff; box-shadow: 0 0 10px #fff; }

.log-header {
  display: flex; gap: 10px; justify-content: space-between; align-items: center; margin-bottom: 8px;
  font-size: 0.7rem; letter-spacing: 0.05em; color: rgba(255, 255, 255, 0.4);
}
.log-header > div {
  display: flex; gap: 10px; align-items: center;
}
.log-state { color: rgba(100, 180, 255, 0.7); }
.delete-log-btn {
  background: none; border: 1px solid rgba(255, 100, 100, 0.3); border-radius: 4px; color: rgba(255, 100, 100, 0.7);
  font-size: 0.65rem; padding: 2px 8px; cursor: pointer; transition: 0.3s; outline: none; -webkit-tap-highlight-color: transparent;
}
.delete-log-btn:hover {
  background: rgba(255, 100, 100, 0.15); color: #fff; border-color: rgba(255, 100, 100, 0.6);
}
.log-body {
  background: rgba(255, 255, 255, 0.03);
  padding: 12px; border-radius: 6px;
  font-size: 0.8rem; line-height: 1.5; color: rgba(255, 255, 255, 0.8);
}
.log-user { margin-bottom: 6px; }
.log-ai {
  margin-top: 10px; padding-top: 10px;
  border-top: 1px dashed rgba(255, 255, 255, 0.1);
  color: rgba(255, 255, 255, 0.5); font-style: italic;
  font-family: source-code-pro, Menlo, Monaco, Consolas, "Courier New", monospace;
}
.expand-hint {
  font-size: 0.65rem; color: rgba(255, 255, 255, 0.2); margin-top: 4px;
}

/* 抽屉进出动效 */
.drawer-enter-active, .drawer-leave-active { transition: all 0.5s cubic-bezier(0.19, 1, 0.22, 1); }
.drawer-enter-from, .drawer-leave-to { opacity: 0; transform: translateY(-20px) scale(0.98); }

/* =========== 终极分析控制台 (深空造物主视窗) =========== */
.summary-modal {
  position: absolute; top: 0; left: 0; width: 100%; height: 100%;
  background: rgba(4, 5, 8, 0.92); z-index: 2000;
  backdrop-filter: blur(15px);
  display: flex; justify-content: center; align-items: center;
  padding: 5vw; box-sizing: border-box;
}
.summary-terminal {
  width: 100%; max-width: 600px; height: 80%;
  border: 1px solid rgba(80, 150, 255, 0.15);
  background: linear-gradient(135deg, rgba(15, 20, 35, 0.9), rgba(5, 8, 15, 0.95));
  box-shadow: 0 0 40px rgba(60, 100, 200, 0.3), inset 0 0 30px rgba(100, 150, 255, 0.05);
  display: flex; flex-direction: column;
  border-radius: 12px; overflow: hidden;
  position: relative;
}
.terminal-header {
  background: rgba(80, 150, 255, 0.08);
  padding: 12px 20px; color: rgba(200, 220, 255, 0.9);
  font-family: source-code-pro, Menlo, Monaco, Consolas, "Courier New", monospace;
  font-size: 0.75rem; font-weight: bold; 
  display: flex; justify-content: space-between; align-items: center;
  border-bottom: 1px solid rgba(80, 150, 255, 0.15);
}
.close-terminal-btn {
  background: none; border: none; color: rgba(255, 100, 100, 0.8); font-family: monospace;
  cursor: pointer; font-size: 0.75rem; text-shadow: 0 0 5px rgba(255, 50, 50, 0.4); transition: 0.3s;
}
.close-terminal-btn:hover:not(:disabled) { transform: scale(1.05); color: #fff; text-shadow: 0 0 10px #ff4a4a; }
.close-terminal-btn:disabled { color: rgba(255, 0, 0, 0.2); text-shadow: none; cursor: not-allowed; }

.terminal-body {
  flex: 1; padding: 30px; overflow-y: auto; position: relative;
}
.typewriter-report {
  font-family: "Courier New", Courier, monospace;
  font-size: 0.85rem; line-height: 2;
  color: rgba(220, 235, 255, 0.85); white-space: pre-wrap;
  text-shadow: 0 0 8px rgba(150, 200, 255, 0.3);
  margin: 0;
}
/* 极具深空感的星尘扫掠特效取代死板CRT */
.scanlines-effect::before {
  content: " "; display: block; position: absolute; top: 0; left: 0; bottom: 0; right: 0;
  background: linear-gradient(180deg, rgba(80, 150, 255, 0) 0%, rgba(80, 150, 255, 0.03) 50%, rgba(80, 150, 255, 0) 100%);
  z-index: 2; background-size: 100% 4px; pointer-events: none;
  animation: stardust-scan 8s linear infinite;
}
@keyframes stardust-scan {
  0% { transform: translateY(-100%); }
  100% { transform: translateY(100%); }
}

/* =========== 天气系统 =========== */

/* 画布粒子层：绝对定位铺满、不拦截点击事件 */
.weather-canvas {
  position: absolute;
  top: 0; left: 0;
  width: 100%; height: 100%;
  pointer-events: none;
  z-index: 1;
}

/* 闪电 overlay */
.lightning-overlay {
  position: absolute;
  inset: 0;
  background: rgba(200, 180, 255, 0);
  pointer-events: none;
  z-index: 2;
  transition: background 0.05s;
}
.lightning-overlay.active {
  background: rgba(200, 180, 255, 0.18);
}

/* ------ 各天气状态的背景色调微调 ------ */
.container.weather-rain    { background: radial-gradient(circle at 50% 30%, rgba(10, 30, 55, 0.85) 0%, #030810 80%) !important; }
.container.weather-drizzle { background: radial-gradient(circle at 50% 30%, rgba(10, 22, 40, 0.8) 0%, #040608 80%) !important; }
.container.weather-snow    { background: radial-gradient(circle at 50% 30%, rgba(15, 20, 35, 0.9) 0%, #080a10 80%) !important; }
.container.weather-thunderstorm { background: radial-gradient(circle at 50% 30%, rgba(20, 5, 40, 0.92) 0%, #050008 80%) !important; }
.container.weather-fog     { background: radial-gradient(circle at 50% 50%, rgba(30, 35, 42, 0.9) 0%, #0d0f12 80%) !important; }
.container.weather-clear   { background: radial-gradient(circle at 50% 30%, rgba(40, 28, 5, 0.6) 0%, #080600 80%) !important; }
.container.weather-clouds  { background: radial-gradient(circle at 50% 30%, rgba(20, 22, 30, 0.85) 0%, #060608 80%) !important; }

/* 晴天放大光晕，色调偏暖 */
.container.weather-clear .orb-core {
  box-shadow: 0 0 30px rgba(255, 220, 100, 0.9), 0 0 90px rgba(255, 180, 50, 0.6) !important;
}
/* 阴天光晕朦胧，偏灰冷 */
.container.weather-clouds .orb-core {
  box-shadow: 0 0 20px rgba(180, 190, 210, 0.7), 0 0 60px rgba(130, 140, 160, 0.3) !important;
  filter: blur(2px);
}
/* 雪天让orb带蓝白光 */
.container.weather-snow .orb-core {
  box-shadow: 0 0 25px rgba(180, 220, 255, 0.9), 0 0 70px rgba(150, 200, 255, 0.5) !important;
}
/* 雷暴让orb带紫光 */
.container.weather-thunderstorm .orb-core {
  box-shadow: 0 0 20px rgba(180, 100, 255, 0.9), 0 0 60px rgba(140, 60, 255, 0.5) !important;
}
/* 雨天orb偏蓝 */
.container.weather-rain .orb-core,
.container.weather-drizzle .orb-core {
  box-shadow: 0 0 20px rgba(100, 180, 255, 0.8), 0 0 60px rgba(60, 140, 220, 0.4) !important;
}

/* ------ 天气角标（左上角常驻） ------ */
.weather-badge {
  position: absolute;
  top: 5vh;
  left: 5vw;
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  z-index: 100;
  transition: opacity 0.3s, transform 0.3s;
  user-select: none;
}
.weather-badge:hover { opacity: 0.85; transform: translateX(2px); }

.weather-badge-icon {
  font-size: 1.4rem;
  line-height: 1;
  filter: drop-shadow(0 0 6px currentColor);
}
.weather-badge-info {
  display: flex;
  flex-direction: column;
  gap: 1px;
}
.weather-badge-city {
  font-size: 0.6rem;
  letter-spacing: 0.15em;
  color: rgba(255, 255, 255, 0.3);
}
.weather-badge-label {
  font-size: 0.72rem;
  letter-spacing: 0.08em;
  font-weight: 500;
}
.weather-badge-temp {
  font-size: 0.65rem;
  color: rgba(255, 255, 255, 0.5);
  font-family: monospace;
}

/* ------ 天气详情面板 ------ */
.weather-panel {
  position: absolute;
  top: 0; left: 0;
  width: min(340px, 92vw);
  height: 100%;
  background: rgba(8, 10, 16, 0.88);
  backdrop-filter: blur(24px) saturate(140%);
  -webkit-backdrop-filter: blur(24px) saturate(140%);
  border-right: 1px solid rgba(255, 255, 255, 0.06);
  z-index: 500;
  display: flex;
  flex-direction: column;
  color: #fff;
  overflow-y: auto;
}

.weather-panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 6vh 22px 18px 22px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
  flex-shrink: 0;
}
.wp-title {
  font-size: 0.7rem;
  letter-spacing: 0.18em;
  color: rgba(255, 255, 255, 0.5);
}
.wp-close {
  background: none; border: none; color: rgba(255,255,255,0.4);
  font-size: 1.8rem; line-height: 1; cursor: pointer; transition: 0.3s;
}
.wp-close:hover { color: #fff; }

.weather-panel-body {
  padding: 24px 22px;
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 20px;
}

/* 主信息行 */
.wp-main-row {
  display: flex;
  align-items: center;
  gap: 20px;
}
.wp-big-icon {
  font-size: 3rem;
  line-height: 1;
  filter: drop-shadow(0 0 12px currentColor);
  flex-shrink: 0;
}
.wp-main-info { display: flex; flex-direction: column; gap: 4px; }
.wp-condition { font-size: 1rem; letter-spacing: 0.1em; font-weight: 300; }
.wp-temp { font-size: 2rem; font-weight: 200; font-family: monospace; letter-spacing: -1px; }
.wp-temp-manual { font-size: 0.75rem; color: rgba(255,255,255,0.3); margin-top: 4px; }
.wp-desc { font-size: 0.7rem; color: rgba(255,255,255,0.4); letter-spacing: 0.05em; }

/* 参数格 */
.wp-stats {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}
.wp-stat {
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.06);
  border-radius: 8px;
  padding: 10px 14px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}
.wp-stat-key { font-size: 0.6rem; color: rgba(255,255,255,0.3); letter-spacing: 0.1em; }
.wp-stat-val { font-size: 0.9rem; font-family: monospace; color: rgba(255,255,255,0.85); }

/* 诗意提示语 & 建议 */
.wp-advice-container {
  display: flex;
  flex-direction: column;
  gap: 12px;
}
.wp-tip {
  border-left: 2px solid rgba(255,255,255,0.2);
  padding: 10px 14px;
  background: rgba(255, 255, 255, 0.02);
  border-radius: 0 8px 8px 0;
}
.wp-advice {
  background: rgba(255, 255, 255, 0.04);
}
.wp-tip-label {
  font-family: monospace;
  font-size: 0.6rem;
  color: rgba(255,255,255,0.25);
  display: block;
  margin-bottom: 6px;
  letter-spacing: 0.1em;
}
.wp-tip-text {
  font-size: 0.72rem;
  line-height: 1.9;
  color: rgba(255,255,255,0.55);
  margin: 0;
  font-style: italic;
  letter-spacing: 0.04em;
}

/* 刷新按钮 */
.wp-refresh {
  width: 100%;
  padding: 11px;
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 8px;
  color: rgba(255, 255, 255, 0.35);
  font-size: 0.7rem;
  letter-spacing: 0.1em;
  cursor: pointer;
  transition: all 0.3s;
}
.wp-refresh:hover:not(:disabled) {
  background: rgba(255,255,255,0.07);
  color: rgba(255,255,255,0.7);
}
.wp-refresh:disabled { cursor: not-allowed; opacity: 0.4; }

/* ------ 测试按钮组 ------ */
.wp-test-section {
  border-top: 1px solid rgba(255,255,255,0.06);
  padding: 16px 22px 22px 22px;
  flex-shrink: 0;
}
.wp-test-label {
  font-size: 0.65rem;
  letter-spacing: 0.12em;
  color: rgba(255,255,255,0.3);
  cursor: pointer;
  user-select: none;
  display: block;
  margin-bottom: 12px;
  transition: color 0.3s;
}
.wp-test-label:hover { color: rgba(255,255,255,0.6); }

.wp-test-btns {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}
.wp-test-btn {
  padding: 7px 13px;
  background: rgba(255,255,255,0.04);
  border: 1px solid rgba(255,255,255,0.1);
  border-radius: 20px;
  color: rgba(255,255,255,0.5);
  font-size: 0.68rem;
  cursor: pointer;
  transition: all 0.25s;
  letter-spacing: 0.05em;
}
.wp-test-btn:hover {
  background: rgba(255,255,255,0.1);
  color: rgba(255,255,255,0.85);
  border-color: rgba(255,255,255,0.25);
}
.wp-test-btn.active {
  background: rgba(100, 160, 255, 0.15);
  border-color: rgba(100, 160, 255, 0.5);
  color: rgba(160, 200, 255, 0.95);
  box-shadow: 0 0 12px rgba(100, 160, 255, 0.2);
}

/* ------ 天气面板进出动效 ------ */
.weather-panel-slide-enter-active,
.weather-panel-slide-leave-active {
  transition: all 0.45s cubic-bezier(0.16, 1, 0.3, 1);
}
.weather-panel-slide-enter-from,
.weather-panel-slide-leave-to {
  transform: translateX(-100%);
  opacity: 0;
}

/* 确保所有UI层级高于天气canvas */
.header, .core-area, .controls { position: relative; z-index: 10; }
.archive-layer, .summary-modal { z-index: 1000; }
</style>
