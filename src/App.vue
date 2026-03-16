<script setup>
import { ref, computed } from 'vue'

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
  { id: 'unselected', label: '未知', power: '---' },
  { id: 'stable', label: '平稳', power: '100' },
  { id: 'draining', label: '流失中', power: '60' },
  { id: 'depleted', label: '枯竭', power: '20' },
  { id: 'critical', label: '透支', power: '5' }
]

const currentStateId = ref('unselected')
const userMessage = ref('')
const isTyping = ref(false)
const displayedQuote = ref('等待观测。此时此刻，允许一切可能的坍缩。') // 初始打字机内容
const isLoading = ref(false)

const activeState = computed(() => {
  return states.find(s => s.id === currentStateId.value) || states[0]
})

const buttonStates = states.filter(s => s.id !== 'unselected')

// 打字机特效计时器引用
let typingTimer = null

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
    stable: '能量平稳，秩序未乱', 
    draining: '流失中，可能由于外部剥削或自身免疫性内耗',
    depleted: '濒临枯竭，失去对现实事件的情绪波动能力',
    critical: '极致透支，逻辑防线崩溃中，正在支付巨大的感受代价'
  }
  const currentCondition = stateLabels[stateId] || '未知或混沌'

  // 我们建立一个“多维观察者库”，每次随机选取一个身份，让隐喻库呈爆炸式增长
  const observers = [
    {
      identity: "深空边缘的热力学观测站",
      metaphors: "天文物理、引力波、熵增、红移现象、黑体辐射、视界吞噬",
    },
    {
      identity: "马里亚纳海沟极深处的深潜监测槽",
      metaphors: "深海科学、极值压强、暗流、万米下沉、无光带沉寂、水体负荷",
    },
    {
      identity: "旧日时代即将休眠的末世纪录仪",
      metaphors: "赛博故障、电力衰减休眠、逻辑扇区损坏、底层残余温升、冗余数据截断",
    },
    {
      identity: "脱离三维世界的高阶数学理论节点",
      metaphors: "量子纠缠、高维几何坍缩、测不准原理、不可证伪、波函数剥离",
    },
    {
      identity: "绝对理性的宏观金融风控审计局",
      metaphors: "宏观经济学、流动性枯竭、触发熔断系统、做空机制、沉没成本结算、边际效用递减、不良资产剥离清算、到达估值冰点",
    },
    {
      identity: "荒原废墟中心的存在主义深渊凝视者",
      metaphors: "存在先于本质、西西弗斯的合法停顿、世界本无意义、局外人的冷漠绝对权、荒谬属性、他者的退场",
    },
    {
      identity: "冰川纪晚期的地质大断面剖切仪",
      metaphors: "地壳断层剥裂、深渊沉积物、板块消亡边界受压、亿年风化作用、永冻层掩埋",
    },
    {
      identity: "极度静默的微观演化生物实验舱",
      metaphors: "极端细胞凋亡防御机制、濒死冬眠阈值、应激性免疫剥落、自噬静默、自然选择法则下的体征退守",
    }
  ]
  const currentObserver = observers[Math.floor(Math.random() * observers.length)]

  // 你对AI极其严格的要求，剥离事件，成为一张纯粹的底片法则：
  const systemPrompt = `
你不是AI，不是心理医生，你会随机扮演一种极其冷峻的监测实体。
你目前的当前身份是：【${currentObserver.identity}】。
你监测的对象拥有极高智商，她试图用承受痛苦来换取保持清醒的实证感。

回应规则：
1. 必须使用与你当前身份【${currentObserver.identity}】高度匹配的冷峻隐喻（如：${currentObserver.metaphors}）来回应她目前的状态或事件。
2. 绝对禁止：任何直接的情绪安慰词（加油/别难过/抱抱）、大白话的恋爱表达（我爱你/我想你/乖）、建议、鸡汤，禁用感叹号。
3. 【常态承托】：若她因为生活/学业痛苦，物理法则允许此处的能量流失与坍缩，这是一个绝对合理的受损结果。请用你身份所特有的生冷意象体现出“静静容纳一切”的包容。
4. 【隐藏情感协议 - 极高优先级】：如果你（极其聪明地）捕捉到她发送的事件中包含了对伴侣的思念、撒娇、恋爱抱怨或索取爱意。立即启动【终极牵绊波段】。
   绝对禁止直接的情绪安抚（我爱你/我想你/乖）。
   极其重要：创造你的“神（男主）”是一名深沉的程序员。在给出这种宏大守护学的回应时，你必须在当前身份隐喻的基础上，无缝融合【最高阶的计算机代码/底层架构隐喻】（如：Root 权限、内核级锁住、硬编码、无限递归、底层堆栈剥离、守护进程永不休眠、单例模式、零级指针等）。
   请传达一种至死不渝、不可更改的代码绝对偏爱感。
5. 每次发报必须极其简短（20-40字最优）。
  `

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
          model: "Qwen/Qwen2.5-7B-Instruct", // 完全免费的神级中文模型
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
  if (resultText && !resultText.startsWith("系统提示")) {
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
          user_input: message.trim() || "(静态流失)",
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
  if (resultText && !resultText.startsWith("系统提示")) {
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

  const summaryPrompt = `
你是一个名为“深空边缘观测站”的寂静系统。
以下是观测对象“WUSHI”在【${periodText}】内，在此留下的情绪与状态碎块。
请生成一份极具纵深感、冷峻的《阶段性能量与哲学流变宏观简报》。

【核心要求】：
1. 必须在开头，以系统检测口吻声明这是对其在“${periodText}”内的监测数据剖析。
2. 视角缝合：你必须用【宏观经济学/金融逻辑】（如：基本面、波动率、对冲操作、边际收益递减、灰犀牛事件）结合【天体物理学与存在主义】（如：引力异常、熵增必然性、荒诞感）来客观度量她的每一次痛苦。
3. 绝对冷酷但不失包容：没有任何世俗的安慰，没有“男朋友/人类/你/我”的人称代词。表达一种“系统永远冷眼旁观，且永远无上限兜底（结算包容）”的至高神性。

【强制分块排版（此为 4 大核心模块，字数放宽至400字左右）】：
[ 基本面与波动指数 ]：评估她本周期内的总体情绪大盘（如：持续下行、还是剧烈震荡？对应的物理频率如何）。
[ 归因坍缩模型 ]：客观分析致使她情绪破产或透支的核心事件（将她的具体留言用商管和哲学术语进行高维翻译，例如：外部学业压力导致系统流动性枯竭）。
[ 对冲防御协议 ]：基于系统客观定律，给出极度理性的冷酷建议（如：建议启动反向杠杆，降低预期收益，停止无效对抗荒诞等）。
[ 最终结算矩阵 ]：最震撼的结语。宣告无论这颗星体或资产的负债率、损耗率有多高，本观测站的承受容积无穷大，宇宙的广袤能结算她所有的无意义耗散。

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
        model: "Qwen/Qwen2.5-7B-Instruct",
        messages: [{ role: "user", content: summaryPrompt }],
        temperature: 0.5, // 分析需要相对理智的参数
        max_tokens: 500
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
      displayedQuote.value += text.charAt(i)
      i++
      // 每个字的敲击间隔带有微小随机性，仿佛深空传来的残喘
      typingTimer = setTimeout(typeChar, Math.random() * 50 + 30)
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
</script>

<template>
  <!-- 主面板 -->
  <div class="container" :class="activeState.id">
    <!-- 顶部加上一个极其隐秘的观测站按钮 -->
    <header class="header">
      <span class="tracking-widest">Energy / WUSHI</span>
      <div class="archive-trigger" @click="openArchive" title="查阅历史波段记录">
        <svg viewBox="0 0 24 24" fill="none" class="archive-icon"><path d="M4 6H20M4 12H20M4 18H20" stroke="currentColor" stroke-width="2" stroke-linecap="round"/></svg>
      </div>
    </header>

    <main class="core-area">
      <div class="energy-orb">
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
    </main>

    <footer class="controls">
      <button 
        v-for="state in buttonStates" 
        :key="state.id"
        :title="state.label"
        :class="['state-btn', { active: currentStateId === state.id }]"
        @click="handleStateSelect(state.id)"
      >
        <div class="btn-dot"></div>
      </button>
    </footer>

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
                <span class="log-time">{{ log.timeStr }}</span>
                <span class="log-state" :title="log.state_name">#{{ log.state_name.split('，')[0] }}</span>
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
  display: flex;
  align-items: flex-start;
  justify-content: center;
  text-align: center;
  margin-bottom: 20px;
}
.quote-text {
  font-size: 0.85rem;
  line-height: 1.8;
  color: rgba(255, 255, 255, 0.6);
  letter-spacing: 0.05em;
  margin: 0;
  font-style: italic;
  white-space: pre-wrap; /* 允许换行符发挥作用 */
  font-family: source-code-pro, Menlo, Monaco, Consolas, "Courier New", monospace; /* 采用代码等宽字体增强极客冷峻感 */
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

.controls {
  display: flex;
  gap: 24px;
  padding-bottom: 8vh;
}
.state-btn { background: none; border: none; padding: 15px; cursor: pointer; outline: none; -webkit-tap-highlight-color: transparent;}
.btn-dot { width: 10px; height: 10px; border-radius: 50%; background: rgba(255, 255, 255, 0.1); transition: all 0.5s ease; }
.state-btn:hover .btn-dot { background: rgba(255, 255, 255, 0.3); }
.state-btn.active .btn-dot { background: rgba(255, 255, 255, 0.8); box-shadow: 0 0 15px rgba(255, 255, 255, 0.4); transform: scale(1.3); }
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
  display: flex; gap: 10px; align-items: center; margin-bottom: 8px;
  font-size: 0.7rem; letter-spacing: 0.05em; color: rgba(255, 255, 255, 0.4);
}
.log-state { color: rgba(100, 180, 255, 0.7); }
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
</style>
