<script setup>
import { ref, onMounted } from 'vue'

const API = import.meta.env.VITE_API_URL || 'http://localhost:8000'
const tab = ref('predict')
const online = ref(true)

const form = ref({
  tenure: 12, MonthlyCharges: 70,
  Contract: 'Month-to-month', InternetService: 'Fiber optic',
  PaymentMethod: 'Electronic check', SeniorCitizen: 0,
  TechSupport: 'No', OnlineSecurity: 'No',
})
const result = ref(null)
const error = ref('')

const question = ref('Strategi retensi untuk pelanggan month-to-month berisiko tinggi?')
const answer = ref(null)
const loading = ref(false)

const metrics = ref(null)
const history = ref(null)

async function call(path, options) {
  const r = await fetch(`${API}${path}`, options)
  if (!r.ok) {
    const t = await r.json().catch(() => ({}))
    throw new Error(t.detail || `HTTP ${r.status}`)
  }
  return r.json()
}

async function predict() {
  error.value = ''
  try {
    const body = { ...form.value, TotalCharges: form.value.tenure * form.value.MonthlyCharges }
    result.value = await call('/predict', {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    })
  } catch (e) { error.value = e.message }
}

async function ask() {
  loading.value = true; error.value = ''
  try {
    answer.value = await call('/ask', {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        question: question.value,
        churn_probability: result.value?.churn_probability ?? null,
      }),
    })
  } catch (e) { error.value = e.message } finally { loading.value = false }
}

async function loadMetrics() {
  error.value = ''
  try { metrics.value = await call('/metrics') } catch (e) { error.value = e.message }
}
async function loadHistory() {
  error.value = ''
  try { history.value = await call('/history') } catch (e) { error.value = e.message }
}
function open(t) {
  tab.value = t
  if (t === 'metrics') loadMetrics()
  if (t === 'history') loadHistory()
}
const pct = (v) => (v * 100).toFixed(1) + '%'

onMounted(async () => {
  try { await call('/health') } catch { online.value = false }
})
</script>

<template>
  <main class="wrap">
    <h1>ChurnGuard AI <small>(Vue)</small></h1>
    <p v-if="!online" class="err">
      Backend belum aktif. Jalankan: uvicorn app.main:app --reload --port 8000
    </p>

    <nav>
      <button :class="{ on: tab === 'predict' }" @click="open('predict')">Prediksi</button>
      <button :class="{ on: tab === 'ask' }" @click="open('ask')">Tanya Playbook</button>
      <button :class="{ on: tab === 'metrics' }" @click="open('metrics')">Performa Model</button>
      <button :class="{ on: tab === 'history' }" @click="open('history')">Riwayat</button>
    </nav>
    <p v-if="error" class="err">{{ error }}</p>

    <section v-if="tab === 'predict'">
      <label>Tenure (bulan) <input type="number" min="0" max="72" v-model.number="form.tenure" /></label>
      <label>Monthly charges <input type="number" step="0.1" v-model.number="form.MonthlyCharges" /></label>
      <label>Contract
        <select v-model="form.Contract">
          <option>Month-to-month</option><option>One year</option><option>Two year</option>
        </select>
      </label>
      <label>Internet
        <select v-model="form.InternetService">
          <option>Fiber optic</option><option>DSL</option><option>No</option>
        </select>
      </label>
      <label>Payment
        <select v-model="form.PaymentMethod">
          <option>Electronic check</option><option>Mailed check</option>
          <option>Bank transfer (automatic)</option><option>Credit card (automatic)</option>
        </select>
      </label>
      <button class="primary" @click="predict">Prediksi</button>
      <div v-if="result" class="card">
        Probabilitas churn: <b>{{ pct(result.churn_probability) }}</b>
        <span :class="['badge', result.risk_level]">{{ result.risk_level }}</span>
      </div>
    </section>

    <section v-if="tab === 'ask'">
      <input v-model="question" class="wide" />
      <button class="primary" @click="ask" :disabled="loading">{{ loading ? 'Memproses...' : 'Tanya' }}</button>
      <div v-if="answer" class="card">
        <p style="white-space: pre-wrap">{{ answer.answer }}</p>
        <small>Sumber: {{ answer.sources.join(', ') }} | mode: {{ answer.mode }}</small>
      </div>
    </section>

    <section v-if="tab === 'metrics' && metrics">
      <div class="card">
        ROC-AUC <b>{{ metrics.roc_auc.toFixed(3) }}</b> |
        Recall churn <b>{{ metrics.recall_churn.toFixed(3) }}</b> |
        Precision <b>{{ metrics.precision_churn.toFixed(3) }}</b> |
        F1 <b>{{ metrics.f1_churn.toFixed(3) }}</b>
      </div>
      <h3>Dengan vs tanpa SMOTE</h3>
      <table>
        <tr><th></th><th>ROC-AUC</th><th>Recall</th><th>Precision</th><th>F1</th></tr>
        <tr v-for="(m, k) in metrics.comparison" :key="k">
          <td>{{ k }}</td><td>{{ m.roc_auc.toFixed(3) }}</td><td>{{ m.recall_churn.toFixed(3) }}</td>
          <td>{{ m.precision_churn.toFixed(3) }}</td><td>{{ m.f1_churn.toFixed(3) }}</td>
        </tr>
      </table>
      <h3>Top fitur</h3>
      <div v-for="f in metrics.top_features" :key="f.feature" class="bar">
        <span>{{ f.feature }}</span>
        <i :style="{ width: f.importance * 600 + 'px' }"></i>
      </div>
    </section>

    <section v-if="tab === 'history' && history">
      <div class="card">
        Total: <b>{{ history.summary.total }}</b> |
        Rata-rata: <b>{{ history.summary.avg_probability ? pct(history.summary.avg_probability) : '-' }}</b>
      </div>
      <table>
        <tr><th>Waktu</th><th>Prob</th><th>Risiko</th><th>Kontrak</th></tr>
        <tr v-for="i in history.items" :key="i.ts + i.probability">
          <td>{{ i.ts }}</td><td>{{ pct(i.probability) }}</td><td>{{ i.risk_level }}</td><td>{{ i.payload.Contract }}</td>
        </tr>
      </table>
    </section>
  </main>
</template>

<style>
body { margin: 0; background: #f6f7f9; }
.wrap { max-width: 760px; margin: 2rem auto; padding: 0 1rem; font-family: system-ui, sans-serif; color: #222; }
h1 small { font-size: 0.5em; color: #888; }
nav { display: flex; gap: 6px; margin-bottom: 1rem; }
nav button { padding: 8px 14px; border: 1px solid #ccd; background: #fff; border-radius: 6px; cursor: pointer; }
nav button.on { background: #2b50aa; color: #fff; border-color: #2b50aa; }
label { display: flex; justify-content: space-between; align-items: center; margin: 8px 0; }
input, select { padding: 6px; min-width: 220px; }
input.wide { width: 100%; box-sizing: border-box; margin-bottom: 8px; }
button.primary { margin-top: 10px; padding: 9px 18px; background: #2b50aa; color: #fff; border: 0; border-radius: 6px; cursor: pointer; }
.card { background: #fff; border: 1px solid #dde; border-radius: 8px; padding: 14px; margin: 14px 0; }
.err { color: #b00020; }
.badge { margin-left: 10px; padding: 2px 10px; border-radius: 10px; color: #fff; }
.badge.Tinggi { background: #c62828; } .badge.Sedang { background: #ef8f00; } .badge.Rendah { background: #2e7d32; }
table { width: 100%; border-collapse: collapse; background: #fff; }
td, th { border: 1px solid #dde; padding: 6px 8px; text-align: left; }
.bar { display: flex; align-items: center; gap: 8px; margin: 3px 0; font-size: 13px; }
.bar span { width: 230px; }
.bar i { display: block; height: 12px; background: #2b50aa; border-radius: 3px; }
</style>
