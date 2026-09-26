<template>
  <div class="posts">
    <div class="toolbar">
      <form class="filters" @submit.prevent="load">
        <select v-model="f.sentiment"><option value="all">全部情感</option><option value="positive">正面</option><option value="neutral">中性</option><option value="negative">负面</option></select>
        <select v-model="f.source"><option value="all">全部渠道</option><option v-for="s in store.sources" :key="s.id" :value="s.id">{{ s.name }}</option></select>
        <input v-model="f.q" placeholder="搜索关键词…" />
        <button class="btn" type="submit">查询</button>
      </form>
      <button class="add" @click="showAdd=!showAdd">＋ 录入舆情</button>
      <button class="batch" @click="showBatch=!showBatch">📥 批量导入</button>
    </div>

    <form v-if="showAdd" class="add-form" @submit.prevent="submit">
      <input v-model="form.title" placeholder="标题" required />
      <textarea v-model="form.content" placeholder="舆情正文（将自动进行情感分析）" required></textarea>
      <div class="row">
        <select v-model="form.source_id"><option v-for="s in store.sources" :key="s.id" :value="s.id">{{ s.name }}</option></select>
        <input v-model="form.topic" placeholder="话题分类" />
        <input v-model="form.media" placeholder="来源媒体，如 澎湃新闻" />
      </div>
      <div class="row">
        <button class="save" type="submit">收录并分析</button>
        <button type="button" class="ghost" @click="showAdd=false">取消</button>
      </div>
    </form>

    <div v-if="showBatch" class="batch-panel">
      <div class="hint">
        每行一条，格式 <code>标题|正文|话题|来源媒体</code>（话题、媒体可省）。统一渠道：
        <select v-model="batchSource" :disabled="importing"><option v-for="s in store.sources" :key="s.id" :value="s.id">{{ s.name }}</option></select>
        <span class="cnt">共 {{ batchCount }} 条 · 任务化导入：断点续传 · 失败重试 · 重复提交自动去重</span>
      </div>
      <textarea v-model="batchText" rows="6" :disabled="importing" placeholder="某品牌售后拖延引投诉|多位用户反映客服响应慢，投诉量上升。|产品体验|澎湃新闻"></textarea>
      <div class="row">
        <button class="save" :disabled="importing || !batchCount" @click="submitBatch">
          {{ importing ? `导入中 ${job ? job.processed + '/' + job.total : ''}…` : interrupted ? '继续导入' : '校验并导入' }}
        </button>
        <button v-if="job && job.failed && !importing" class="retry" @click="retryFailed">重试失败项（{{ job.failed }}）</button>
        <button class="ghost" :disabled="importing" @click="resetBatch">取消</button>
      </div>
      <div v-if="job" class="progress">
        <div class="bar"><i :style="{ width: pct + '%' }" :class="{ bad: job.failed }"></i></div>
        <span class="ptext">
          任务 #{{ job.id }} · {{ statusText(job.status) }} · 进度 {{ job.processed }}/{{ job.total }}
          （成功 {{ job.succeeded }}<template v-if="job.failed"> · <b class="pfail">失败 {{ job.failed }}</b></template>）
        </span>
      </div>
      <div v-if="batchError" class="err">
        ❌ {{ batchError }}
        <ul v-if="batchErrDetails.length"><li v-for="d in batchErrDetails" :key="d">{{ d }}</li></ul>
      </div>
      <div v-if="jobDetail" class="result">
        <div class="sum">
          <template v-if="jobDetail.status === 'done'">✅ 任务 #{{ jobDetail.id }} 全部完成，共导入 {{ jobDetail.succeeded }} 条，统计已刷新</template>
          <template v-else>⚠️ 任务 #{{ jobDetail.id }} 完成 {{ jobDetail.succeeded }}/{{ jobDetail.total }} 条，{{ jobDetail.failed }} 条失败（可重试）</template>
          <template v-if="jobDetail.summary && jobDetail.summary.alerts">
            · 触发预警 {{ jobDetail.summary.alerts }} 次（自动建档 {{ jobDetail.summary.crisesCreated }} · 并入 {{ jobDetail.summary.crisesMerged }}）
          </template>
        </div>
        <div v-for="it in jobDetail.items" :key="it.seq" class="ritem">
          <span class="no">#{{ it.seq + 1 }}</span>
          <template v-if="it.status === 'done' && it.result">
            <span class="chip sent" :class="it.result.sentiment">{{ sentText(it.result.sentiment) }}</span>
            <span class="heat">热度 {{ it.result.heat }}</span>
            <span class="rt">{{ it.title }}</span>
            <span v-if="it.result.triggered.length" class="trig">⚠️ {{ trigText(it.result.triggered) }}</span>
          </template>
          <template v-else-if="it.status === 'failed'">
            <span class="chip failed">失败</span>
            <span class="rt">{{ it.title }}</span>
            <span class="ferr">{{ it.error }}（已试 {{ it.attempts }} 次）</span>
          </template>
          <template v-else>
            <span class="chip pending">待处理</span>
            <span class="rt">{{ it.title }}</span>
          </template>
        </div>
      </div>
    </div>

    <div class="list">
      <div v-for="p in posts" :key="p.id" class="post" :class="p.sentiment">
        <div class="head">
          <span class="chip sent" :class="p.sentiment">{{ sentText(p.sentiment) }}</span>
          <span class="score"><i :style="scoreBar(p.sentiment_score)"></i>{{ (p.sentiment_score>=0?'+':'')+p.sentiment_score.toFixed(2) }}</span>
          <span v-if="p.hot" class="hot">🔥 热点</span>
          <span class="heat">热度 {{ p.heat }}</span>
          <span class="time">{{ p.published }}</span>
        </div>
        <b class="title">{{ p.title }}</b>
        <p class="content">{{ p.content }}</p>
        <div class="meta">
          <span class="src">{{ srcName(p.source_id) }}</span>
          <span class="topic">#{{ p.topic }}</span>
          <span class="media">{{ p.media }}</span>
        </div>
      </div>
      <div v-if="!posts.length" class="none">没有匹配的舆情</div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { usePubStore } from '@/store/pub'
const store = usePubStore()
const posts = ref([])
const f = ref({ sentiment: 'all', source: 'all', q: '' })
const showAdd = ref(false)
const form = ref({ title: '', content: '', source_id: null, topic: '', media: '' })
const showBatch = ref(false)
const batchText = ref('')
const batchSource = ref(1)
const importing = ref(false)
const interrupted = ref(false) // 导入中断（网络/服务异常）：任务进度已落库，可断点续传
const job = ref(null)          // 当前导入任务进度快照
const jobDetail = ref(null)    // 任务完成后的逐条结果（结果回写）
const batchError = ref('')
const batchErrDetails = ref([])

const batchCount = computed(() => batchText.value.split('\n').filter((l) => l.trim()).length)
const pct = computed(() => (job.value && job.value.total ? Math.round((job.value.processed / job.value.total) * 100) : 0))

// 幂等标识：由导入内容哈希生成 —— 相同内容重复提交/中断重提自动对应同一任务，不产生重复批次
function hashKey(s) {
  let h = 5381
  for (let i = 0; i < s.length; i++) h = ((h << 5) + h + s.charCodeAt(i)) >>> 0
  return `web-${h.toString(36)}`
}
const batchKey = computed(() => hashKey(`${batchSource.value}\n${batchText.value.trim()}`))

async function load() {
  const qs = {}
  if (f.value.sentiment !== 'all') qs.sentiment = f.value.sentiment
  if (f.value.source !== 'all') qs.source = f.value.source
  if (f.value.q) qs.q = f.value.q
  posts.value = await store.fetchPosts(qs)
}
async function submit() {
  try {
    await store.addPost({ ...form.value, source_id: Number(form.value.source_id || 1) })
    form.value = { title: '', content: '', source_id: null, topic: '', media: '' }
    showAdd.value = false
    load()
  } catch (e) { store.msg(e.message, 'warn') }
}
// 解析批量文本：每行 标题|正文|话题|来源媒体，行级校验
function parseBatch() {
  const items = [], errs = []
  batchText.value.split('\n').forEach((line, i) => {
    const t = line.trim()
    if (!t) return
    const [title, content, topic, media] = t.split('|').map((s) => (s || '').trim())
    if (!title || !content) { errs.push(`第 ${i + 1} 行：标题与正文不能为空`); return }
    items.push({ title, content, topic, media, source_id: Number(batchSource.value) || 1 })
  })
  return { items, errs }
}
const IMPORT_CHUNK = 50 // 每个分片处理条数：分片推进，进度实时可见、中断可续
async function submitBatch() {
  batchError.value = ''; batchErrDetails.value = []
  if (interrupted.value && job.value) { await runLoop(job.value.id); return } // 断点续传：直接恢复既有任务
  const { items, errs } = parseBatch()
  if (errs.length) { batchError.value = '格式校验未通过，未创建导入任务'; batchErrDetails.value = errs; return }
  if (!items.length) { batchError.value = '没有可导入的数据'; return }
  importing.value = true
  try {
    const r = await store.createImportJob(batchKey.value, items) // 幂等：同内容重复提交返回同一任务
    job.value = r.job
    jobDetail.value = null
    await runLoop(r.job.id)
  } catch (e) {
    batchError.value = e.message
    if (e.details) batchErrDetails.value = e.details
  } finally { importing.value = false }
}
// 分片执行直至任务收敛（done/partial/failed）；异常时保留断点，可继续导入
async function runLoop(id) {
  importing.value = true
  interrupted.value = false
  try {
    for (let guard = 0; guard < 1000; guard++) {
      const j = await store.runImportJob(id, IMPORT_CHUNK)
      job.value = j
      if (j.status !== 'running' && j.status !== 'pending') break
    }
    jobDetail.value = await store.fetchImportJob(id) // 结果回写：逐条明细 + 触发汇总
    await store.finishImportJob(job.value)
    if (job.value.status === 'done') batchText.value = ''
    load()
  } catch (e) {
    interrupted.value = true
    batchError.value = `导入中断：${e.message}。已完成部分不会重复导入，点击「继续导入」从断点恢复。`
  } finally { importing.value = false }
}
// 失败重试：重置失败条目后断点续跑
async function retryFailed() {
  if (!job.value) return
  batchError.value = ''
  importing.value = true
  try {
    await store.retryImportJob(job.value.id)
    await runLoop(job.value.id)
  } catch (e) { batchError.value = e.message; interrupted.value = true }
  finally { importing.value = false }
}
function resetBatch() {
  showBatch.value = false
  batchError.value = ''; batchErrDetails.value = []
  job.value = null; jobDetail.value = null; interrupted.value = false
}
function statusText(s) {
  return { pending: '待执行', running: '执行中', done: '已完成', partial: '部分完成', failed: '失败' }[s] || s
}
function trigText(triggered) {
  return triggered.map((t) =>
    t.deduped ? `${t.alert}（并入危机 #${t.crisisId}）`
      : t.crisisId ? `${t.alert}（自动建档 #${t.crisisId}）` : t.alert).join('、')
}
function sentText(x) { return x === 'positive' ? '😊 正面' : x === 'negative' ? '😟 负面' : '😐 中性' }
function scoreBar(score) { const w = Math.min(100, Math.abs(score) * 100); return { width: w + '%', background: score >= 0 ? '#66bb6a' : '#ef5350' } }
function srcName(id) { return store.sources.find((s) => s.id === id)?.name || '未知' }
onMounted(load)
</script>

<style scoped>
.posts{display:flex;flex-direction:column;gap:12px;}
.toolbar{display:flex;gap:10px;align-items:center;flex-wrap:wrap;}
.filters{display:flex;gap:8px;flex-wrap:wrap;}
select,input,textarea,button{font-family:inherit;background:#13233f;border:1px solid rgba(120,160,220,0.2);color:#dbe4f3;border-radius:8px;padding:8px 10px;font-size:12px;}
textarea{resize:vertical;min-height:56px;}
.btn{background:#2962ff;border:none;color:#fff;cursor:pointer;font-weight:600;}
.add{background:linear-gradient(135deg,#43a047,#2e7d32);border:none;color:#fff;font-weight:600;cursor:pointer;}
.batch{background:linear-gradient(135deg,#00897b,#00695c);border:none;color:#fff;font-weight:600;cursor:pointer;}
.batch-panel{background:#0f1b38;border:1px solid rgba(120,160,220,0.16);border-radius:12px;padding:14px;display:flex;flex-direction:column;gap:8px;}
.batch-panel .hint{font-size:12px;color:#8ba2c8;display:flex;align-items:center;gap:8px;flex-wrap:wrap;}
.batch-panel .hint code{background:#0c1730;padding:2px 6px;border-radius:4px;color:#90caf9;}
.batch-panel .cnt{margin-left:auto;color:#5b6f94;font-size:11px;}
.batch-panel textarea{width:100%;box-sizing:border-box;}
.err{background:#3a1215;border:1px solid #b71c1c;color:#ef9a9a;border-radius:8px;padding:10px 12px;font-size:12px;}
.err ul{margin:6px 0 0;padding-left:18px;}
.retry{background:linear-gradient(135deg,#f9a825,#f57f17);border:none;color:#fff;font-weight:600;cursor:pointer;}
.progress{display:flex;align-items:center;gap:10px;}
.progress .bar{flex:0 0 180px;height:8px;background:#0c1730;border-radius:4px;overflow:hidden;}
.progress .bar i{display:block;height:100%;background:linear-gradient(90deg,#00897b,#26a69a);transition:width .2s;}
.progress .bar i.bad{background:linear-gradient(90deg,#00897b,#f9a825);}
.progress .ptext{font-size:11px;color:#8ba2c8;}
.progress .pfail{color:#ef9a9a;}
.chip.failed{background:#b71c1c;color:#ffcdd2;}
.chip.pending{background:#37474f;color:#b0bec5;}
.ferr{color:#ef9a9a;font-size:11px;}
.result{background:#0c1730;border:1px solid rgba(120,160,220,0.16);border-radius:8px;padding:10px 12px;display:flex;flex-direction:column;gap:6px;}
.result .sum{color:#a5d6a7;font-size:12px;font-weight:600;}
.ritem{display:flex;align-items:center;gap:8px;flex-wrap:wrap;font-size:12px;color:#aebadd;border-top:1px dashed rgba(120,160,220,0.12);padding-top:6px;}
.ritem .no{color:#5b6f94;font-size:11px;}
.ritem .rt{color:#dbe4f3;}
.ritem .trig{color:#ffb300;font-size:11px;}
.add-form{background:#0f1b38;border:1px solid rgba(120,160,220,0.16);border-radius:12px;padding:14px;display:flex;flex-direction:column;gap:8px;}
.row{display:flex;gap:8px;flex-wrap:wrap;}
.add-form .row:last-child{margin-top:4px;}
.save{background:#2962ff;border:none;color:#fff;font-weight:600;cursor:pointer;}
.ghost{background:#16263f;color:#8ba2c8;cursor:pointer;}
.list{display:flex;flex-direction:column;gap:12px;}
.post{background:#0f1b38;border:1px solid rgba(120,160,220,0.16);border-radius:12px;padding:14px;border-left:4px solid #90a4ae;}
.post.negative{border-left-color:#ef5350;}.post.positive{border-left-color:#66bb6a;}.post.neutral{border-left-color:#90a4ae;}
.head{display:flex;align-items:center;gap:10px;flex-wrap:wrap;margin-bottom:8px;}
.chip{font-size:11px;padding:2px 8px;border-radius:6px;}
.chip.positive{background:#1b5e20;color:#a5d6a7;}.chip.negative{background:#b71c1c;color:#ffcdd2;}.chip.neutral{background:#37474f;color:#b0bec5;}
.score{display:flex;align-items:center;gap:5px;color:#8ba2c8;font-size:11px;}
.score i{height:5px;border-radius:3px;width:40px;background:#0c1730;}
.hot{font-size:10px;color:#ffd54f;}
.heat{font-size:11px;color:#ffb300;}
.time{margin-left:auto;color:#5b6f94;font-size:11px;}
.title{color:#fff;font-size:15px;display:block;margin-bottom:4px;}
.content{color:#aebadd;font-size:13px;line-height:1.5;margin:0 0 8px;}
.meta{display:flex;gap:12px;font-size:11px;color:#8ba2c8;}
.src{font-weight:600;}
.topic{color:#90caf9;}
.media{color:#5b6f94;}
.none{color:#5b6f94;text-align:center;padding:30px;}
</style>