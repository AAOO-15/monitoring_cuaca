<template>
  <div class="dashboard">
    <div class="container">
      <header class="top">
        <h2>🌤️ Dashboard Cuaca ESP32</h2>
        <div class="current-weather" :title="'Cuaca saat ini'">{{ cuaca }}</div>
      </header>

      <section class="stats" aria-label="Ringkasan Data">
        <div class="stat-card">
          <div class="stat-label">Suhu</div>
          <div class="stat-value">{{ suhu }}<span class="unit">°C</span></div>
        </div>
        <div class="stat-card">
          <div class="stat-label">Kelembapan</div>
          <div class="stat-value">{{ kelembapan }}<span class="unit">%</span></div>
        </div>
        <div class="stat-card">
          <div class="stat-label">Curah Hujan</div>
          <div class="stat-value">{{ curahHujan }}<span class="unit">mm</span></div>
        </div>
        <div class="stat-card">
          <div class="stat-label">Angin</div>
          <div class="stat-value">{{ kecepatanAngin }}<span class="unit">m/s</span></div>
        </div>
        <div class="stat-card">
          <div class="stat-label">Intensitas Cahaya</div>
          <div class="stat-value">{{ intensitasCahaya }}<span class="unit">LUX</span></div>
        </div>
      </section>
      <section class="charts" aria-label="Grafik Data">
        <div class="chart-card">
          <div class="chart-title">Curah Hujan</div>
          <canvas ref="rainCanvas"></canvas>
        </div>

        <div class="chart-card">
          <div class="chart-title">Suhu</div>
          <canvas ref="suhuCanvas"></canvas>
        </div>
        <div class="chart-card">
          <div class="chart-title">Kecepatan Angin</div>
          <canvas ref="anginCanvas"></canvas>
        </div>
        <div class="chart-card">
          <div class="chart-title">Kelembapan</div>
          <canvas ref="humCanvas"></canvas>
        </div>
        <div class="chart-card">
          <div class="chart-title">Intensitas Cahaya</div>
          <canvas ref="luxCanvas"></canvas>
        </div>

        <div class="table-box">
          <div class="chart-title">Data Curah Hujan</div>
          <div class="table-wrap">
            <table>
              <thead>
                <tr>
                  <th>Waktu</th>
                  <th>Suhu (°C)</th>
                  <th>Kelembapan (%)</th>
                  <th>Kecepatan Angin (m/s)</th>
                  <th>Curah Hujan (mm)</th>
                  <th>Intensitas Cahaya (LUX)</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(label, index) in rainLabels" :key="index">
                  <td>{{ label }}</td>
                  <td>{{ suhuData[index] ?? '_' }}</td>
                  <td>{{ humData[index] ?? '_' }}</td>
                  <td>{{ anginData[index] ?? '_' }}</td>
                  <td>{{ rainData[index] ?? '_' }}</td>
                  <td>{{ luxData[index] ?? '_' }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </section>

      <div class="actions">
        <button class="btn" @click="downloadCSV" title="Unduh CSV">📥 Download CSV</button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, nextTick } from 'vue'
import mqtt from 'mqtt'
import { Chart } from 'chart.js/auto'

// Realtime data
const suhu = ref('--')
const kelembapan = ref('--')
const curahHujan = ref('--')
const kecepatanAngin = ref('--')
const kondisiCuaca = ref('--')
const cuaca = ref('--')
const intensitasCahaya = ref('--')

// Canvas refs
const rainCanvas = ref<HTMLCanvasElement | null>(null)
const suhuCanvas = ref<HTMLCanvasElement | null>(null)
const anginCanvas = ref<HTMLCanvasElement | null>(null)
const humCanvas = ref<HTMLCanvasElement | null>(null)
const luxCanvas = ref<HTMLCanvasElement | null>(null)

// Historis data
const rainData = ref<number[]>([])
const rainLabels = ref<string[]>([])
const suhuData = ref<number[]>([])
const suhuLabels = ref<string[]>([])
const anginData = ref<number[]>([])
const anginLabels = ref<string[]>([])
const humData = ref<number[]>([])
const humLabels = ref<string[]>([])
const luxData = ref<number[]>([])
const luxLabels = ref<string[]>([])

// Chart instances
let rainChart: Chart | null = null
let suhuChart: Chart | null = null
let anginChart: Chart | null = null
let humChart: Chart | null = null
let luxChart: Chart | null = null

onMounted(async () => {
  await nextTick()

  const client = mqtt.connect(
    'wss://751cee3a0dcb4364bbb1e4dbe4a87ca5.s1.eu.hivemq.cloud:8884/mqtt',
    {
      username: 'langgamdewa',
      password: 'Rizkypratama512',
    },
  )

  client.on('connect', () => {
    console.log('✅ Connected to HiveMQ Cloud')
    client.subscribe('esp32/#')
  })

  client.on('error', (err) => {
    console.error('❌ MQTT Error:', err)
  })

  client.on('message', (topic, message) => {
    const val = message.toString()
    const now = new Date().toLocaleTimeString()
    console.log('📩', topic, val)

    const num = parseFloat(val)
    if (isNaN(num) && topic !== 'esp32/cuaca') return

    if (topic === 'esp32/suhu') {
      suhu.value = val
      suhuData.value.push(num)
      suhuLabels.value.push(now)
      if (suhuData.value.length > 288) {
        suhuData.value.shift()
        suhuLabels.value.shift()
      }
      if (suhuChart && suhuChart.data.datasets[0]) {
        suhuChart.data.labels = suhuLabels.value.slice()
        suhuChart.data.datasets[0].data = suhuData.value.slice()
        suhuChart.update()
      }
    }

    if (topic === 'esp32/kelembapan') {
      kelembapan.value = val
      humData.value.push(num)
      humLabels.value.push(now)
      if (humData.value.length > 288) {
        humData.value.shift()
        humLabels.value.shift()
      }
      if (humChart && humChart.data.datasets[0]) {
        humChart.data.labels = humLabels.value.slice()
        humChart.data.datasets[0].data = humData.value.slice()
        humChart.update()
      }
    }
    if (topic === 'esp32/curah_hujan') {
      curahHujan.value = val
      rainData.value.push(num)
      rainLabels.value.push(now)
      if (rainData.value.length > 288) {
        rainData.value.shift()
        rainLabels.value.shift()
      }
      if (rainChart && rainChart.data.datasets[0]) {
        rainChart.data.labels = rainLabels.value.slice()
        rainChart.data.datasets[0].data = rainData.value.slice()
        rainChart.update()
      }
    }

    if (topic === 'esp32/kecepatan_angin') {
      kecepatanAngin.value = val
      anginData.value.push(num)
      anginLabels.value.push(now)
      if (anginData.value.length > 288) {
        anginData.value.shift()
        anginLabels.value.shift()
      }
      if (anginChart && anginChart.data.datasets[0]) {
        anginChart.data.labels = anginLabels.value.slice()
        anginChart.data.datasets[0].data = anginData.value.slice()
        anginChart.update()
      }
    }

    if (topic === 'esp32/cuaca') {
      cuaca.value = val
    }

    if (topic === 'esp32/lux') {
      intensitasCahaya.value = val
      luxData.value.push(num)
      luxLabels.value.push(now)
      if (luxData.value.length > 288) {
      luxData.value.shift()
      luxLabels.value.shift()
      }
      if (luxChart && luxChart.data.datasets[0]) {
      luxChart.data.labels = luxLabels.value.slice()
      luxChart.data.datasets[0].data = luxData.value.slice()
      luxChart.update()
      }
    }
  })

  // Chart setup
  if (rainCanvas.value) {
    rainChart = new Chart(rainCanvas.value, {
      type: 'line',
      data: {
        labels: [],
        datasets: [
          {
            label: 'Curah Hujan (mm)',
            data: [],
            borderColor: 'blue',
            fill: false,
          },
        ],
      },
      options: { responsive: true },
    })
  }

  if (suhuCanvas.value) {
    suhuChart = new Chart(suhuCanvas.value, {
      type: 'line',
      data: {
        labels: [],
        datasets: [
          {
            label: 'Suhu (°C)',
            data: [],
            borderColor: 'red',
            fill: false,
          },
        ],
      },
      options: { responsive: true },
    })
  }

  if (humCanvas.value) {
    humChart = new Chart(humCanvas.value, {
      type: 'line',
      data: {
        labels: [],
        datasets: [
          {
            label: 'Kelembapan (%)',
            data: [],
            borderColor: 'purple',
            fill: false,
          },
        ],
      },
      options: { responsive: true },
    })
  }

  if (anginCanvas.value) {
    anginChart = new Chart(anginCanvas.value, {
      type: 'line',
      data: {
        labels: [],
        datasets: [
          {
            label: 'Kecepatan Angin (m/s)',
            data: [],
            borderColor: 'green',
            fill: false,
          },
        ],
      },
      options: { responsive: true },
    })
  }

  if (luxCanvas.value) {
    luxChart = new Chart(luxCanvas.value, {
      type: 'line',
      data: {
        labels: [],
        datasets: [
          {
            label: 'Intensitas Cahaya (LUX)',
            data: [],
            borderColor: 'orange',
            fill: false,
          },
        ],
      },
      options: { responsive: true },
    })
  }
})

function downloadCSV() {
  let csv =
    'Waktu,Suhu (°C),Kelembapan (%),Kecepatan Angin (m/s),Curah Hujan (mm),Intensitas Cahaya (LUX)\n'
  for (let i = 0; i < rainLabels.value.length; i++) {
    const waktu = rainLabels.value[i] ?? '-'
    const suhuVal = suhuData.value[i] ?? '-'
    const humVal = humData.value[i] ?? '-'
    const anginVal = anginData.value[i] ?? '-'
    const hujanVal = rainData.value[i] ?? '-'
    const luxVal = luxData.value[i] ?? '-'
    csv += `${waktu},${suhuVal},${humVal},${anginVal},${hujanVal},${luxVal}\n`
  }

  const blob = new Blob([csv], { type: 'text/csv' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = 'data_monitoring_cuaca.csv'
  a.click()
  URL.revokeObjectURL(url)
}
</script>

<style scoped>
/* Layout and background */
.dashboard {
  min-height: 100vh;
  background: linear-gradient(180deg, #eaf3ff 0%, #d7e7ff 60%, #c6dcff 100%);
  padding: 24px 16px;
  box-sizing: border-box;
}
.container {
  max-width: 1100px;
  margin: 0 auto;
}
.top {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 16px;
}
h2 {
  margin: 0;
  font-size: 22px;
  color: #0a2e5c;
}
.current-weather {
  background: rgba(255, 255, 255, 0.8);
  border: 1px solid rgba(10, 46, 92, 0.15);
  padding: 6px 10px;
  border-radius: 8px;
  color: #0a2e5c;
  font-weight: 600;
}

/* Stats */
.stats {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 12px;
  margin-bottom: 20px;
}
.stat-card {
  background: #ffffff;
  border: 1px solid rgba(10, 46, 92, 0.08);
  border-radius: 12px;
  padding: 12px;
  box-shadow: 0 6px 20px rgba(10, 46, 92, 0.08);
}
.stat-label {
  font-size: 12px;
  color: #4a6a90;
}
.stat-value {
  margin-top: 4px;
  font-size: 24px;
  font-weight: 700;
  color: #0a2e5c;
}
.unit {
  margin-left: 4px;
  font-size: 14px;
  color: #4a6a90;
  font-weight: 600;
}

/* Charts */
.charts {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 16px;
}
.chart-card,
.table-box {
  background: #ffffff;
  border: 1px solid rgba(10, 46, 92, 0.08);
  border-radius: 12px;
  padding: 12px;
  box-shadow: 0 6px 20px rgba(10, 46, 92, 0.08);
}
.chart-title {
  font-size: 14px;
  font-weight: 700;
  color: #0a2e5c;
  margin-bottom: 8px;
}
canvas {
  max-width: 100%;
}

/* Table */

.container .table-box {
  width: 54vw;
  max-width: 55vw;
}
.table-wrap {
  width: 100%;
  overflow-x: auto;
  border-radius: 10px;
  box-shadow: 0 4px 12px rgba(10, 46, 92, 0.1);
}
table {
  width: 800px;
  border-collapse: collapse;
  table-layout: auto;
}
.table-box {
  margin-top: 30px;
  padding: 10px;
  border-radius: 8px;
  background: #fff;
  overflow-x: auto;
  width: 100%;
  box-sizing: border-box;
  display: inline-block;
}

th,
td {
  border: 1px solid #ccc;
  padding: 8px 12px;
  text-align: center;
  white-space: nowrap;
}
td {
  white-space: normal;
  word-break: break-word;
}
thead {
  background: #f3f8ff;
  color: #0a2e5c;
}

/* Actions */
.actions {
  margin-top: 16px;
  display: flex;
  justify-content: flex-end;
}
.btn {
  padding: 8px 14px;
  background: #0a66ff;
  color: #fff;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  box-shadow: 0 6px 16px rgba(10, 102, 255, 0.25);
}
.btn:hover {
  background: #0856d6;
}

@media (max-width: 600px) {
  .actions {
    justify-content: stretch;
  }
  .btn {
    width: 100%;
  }
}
</style>
