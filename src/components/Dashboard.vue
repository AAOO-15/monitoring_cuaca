<template>
  <div class="dashboard">
    <div class="container">
      <header class="top">
        <h2>🌤️ Dashboard Cuaca ESP32</h2>
        <div
          class="status-indicator"
          :class="{ online: mqttConnected }"
          :title="mqttConnected ? 'MQTT Connected' : 'MQTT Disconnected'"
        >
          <span class="dot"></span>
          {{ mqttConnected ? 'Online' : 'Offline' }}
        </div>
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
                  <th class="col-waktu">Waktu</th>
                  <th class="col-suhu">Suhu (°C)</th>
                  <th class="col-hum">Kelembapan (%)</th>
                  <th class="col-angin">Kecepatan Angin (m/s)</th>
                  <th class="col-rain">Curah Hujan (mm)</th>
                  <th class="col-lux">Intensitas Cahaya (LUX)</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(label, index) in rainLabels" :key="index">
                  <td class="col-waktu">{{ label }}</td>
                  <td class="col-suhu">{{ suhuData[index] ?? '_' }}</td>
                  <td class="col-hum">{{ humData[index] ?? '_' }}</td>
                  <td class="col-angin">{{ anginData[index] ?? '_' }}</td>
                  <td class="col-rain">{{ rainData[index] ?? '_' }}</td>
                  <td class="col-lux">{{ luxData[index] ?? '_' }}</td>
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

// Debug/connection status
const mqttConnected = ref(false)
const lastMessageTime = ref('--')
const messageCount = ref(0)

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
    'wss://751cee3a0dcb4364bbb1e4dbe4a87ca5.s1.eu.hivemq.cloud:8883/mqtt',
    {
      username: 'langgamdewa',
      password: 'Rizkypratama512',
      reconnectPeriod: 5000,
      connectTimeout: 30000,
      keepalive: 60,
      clean: true,
      clientId: 'web_dashboard_' + Math.random().toString(16).substr(2, 8),
    },
  )

  client.on('reconnect', () => {
    console.log('🔄 Reconnecting to MQTT...')
  })

  client.on('offline', () => {
    mqttConnected.value = false
    console.log('📴 MQTT went offline')
  })

  client.on('connect', () => {
    mqttConnected.value = true
    console.log('✅ Connected to HiveMQ Cloud at', new Date().toLocaleTimeString())
    console.log('🌐 Environment:', import.meta.env.MODE)
    console.log('🔗 Broker:', 'wss://751cee3a0dcb4364bbb1e4dbe4a87ca5.s1.eu.hivemq.cloud:8883')

    // Subscribe ke semua topic secara individual (backup jika wildcard gagal)
    const topics = [
      'esp32/suhu',
      'esp32/kelembapan',
      'esp32/curah_hujan',
      'esp32/kecepatan_angin',
      'esp32/cuaca',
      'esp32/lux',
      'esp32/#',
    ]

    let subscribeCount = 0
    topics.forEach((topic) => {
      client.subscribe(topic, (err) => {
        if (err) {
          console.error(`❌ Subscribe error for ${topic}:`, err)
        } else {
          subscribeCount++
          console.log(`✅ Subscribed to ${topic} (${subscribeCount}/${topics.length})`)
        }
      })
    })

    // Debug info setelah subscribe
    setTimeout(() => {
      console.log('\n📊 === DEBUG INFO ===')
      console.log('✅ MQTT Connected:', mqttConnected.value)
      console.log('📝 Subscribed to:', subscribeCount, 'topics')
      console.log('⏳ Waiting for data from ESP32...')
      console.log('\n💡 If no data appears after 30 seconds, check:')
      console.log('   1. ESP32 is powered on')
      console.log('   2. ESP32 Serial Monitor shows "terhubung"')
      console.log('   3. ESP32 shows "=== Data Cuaca ==="')
      console.log(
        '   4. ESP32 using same broker: 751cee3a0dcb4364bbb1e4dbe4a87ca5.s1.eu.hivemq.cloud',
      )
      console.log('===================\n')
    }, 2000)
  })

  client.on('disconnect', () => {
    mqttConnected.value = false
    console.log('❌ Disconnected from MQTT')
  })

  client.on('error', (err) => {
    mqttConnected.value = false
    console.error('❌ MQTT Error:', err)
  })

  client.on('message', (topic, message) => {
    // FIRST LOG - Pastikan ini muncul
    console.log('🎉 === MESSAGE RECEIVED ===')

    if (!topic) {
      console.warn('⚠️ Received message without topic')
      return
    }

    const val = message.toString()
    const now = new Date().toLocaleTimeString()

    // Update status
    lastMessageTime.value = now
    messageCount.value++

    // LOG SEMUA pesan yang masuk - DETAILED
    console.log('📩 MQTT Message Details:')
    console.log('   Topic:', topic)
    console.log('   Value:', val)
    console.log('   Time:', now)
    console.log('   Total Messages:', messageCount.value)
    console.log('========================\n')

    const num = parseFloat(val)

    // Validasi lebih relaxed - log tapi tetap proses
    if (isNaN(num) && topic !== 'esp32/cuaca') {
      console.warn(`⚠️ Invalid number from ${topic}:`, val)
      return
    }

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
      console.log('✅ Cuaca updated:', val)
    }

    if (topic === 'esp32/lux' || topic === 'esp32/intensitas_cahaya') {
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
      console.log('✅ LUX updated:', val)
    }

    // Catch-all: log topic yang tidak dikenali
    const knownTopics = [
      'esp32/suhu',
      'esp32/kelembapan',
      'esp32/curah_hujan',
      'esp32/kecepatan_angin',
      'esp32/cuaca',
      'esp32/lux',
      'esp32/intensitas_cahaya',
    ]

    if (!knownTopics.includes(topic)) {
      console.warn('⚠️ Unknown topic:', topic, '| value:', val)
      console.warn('💡 Did you mean one of these?', knownTopics)
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
  flex-wrap: wrap;
}
h2 {
  margin: 0;
  font-size: 22px;
  color: #0a2e5c;
}
.status-panel {
  display: flex;
  gap: 8px;
  align-items: center;
  font-size: 12px;
}
.status-item {
  background: rgba(255, 255, 255, 0.7);
  border: 1px solid rgba(220, 53, 69, 0.3);
  padding: 4px 8px;
  border-radius: 6px;
  color: #dc3545;
  display: flex;
  align-items: center;
  gap: 6px;
}
.status-item.connected {
  border-color: rgba(40, 167, 69, 0.3);
  color: #28a745;
}
.status-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #dc3545;
  animation: pulse 2s infinite;
}
.status-item.connected .status-dot {
  background: #28a745;
}
@keyframes pulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
}
.status-indicator {
  background: rgba(255, 255, 255, 0.7);
  border: 1px solid rgba(220, 53, 69, 0.3);
  padding: 4px 10px;
  border-radius: 6px;
  color: #dc3545;
  font-size: 13px;
  display: flex;
  align-items: center;
  gap: 6px;
  font-weight: 600;
}
.status-indicator.online {
  border-color: rgba(40, 167, 69, 0.3);
  color: #28a745;
}
.status-indicator .dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #dc3545;
  animation: pulse 2s infinite;
}
.status-indicator.online .dot {
  background: #28a745;
}
@keyframes pulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
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
.chart-card {
  background: #ffffff;
  border: 1px solid rgba(10, 46, 92, 0.08);
  border-radius: 12px;
  padding: 12px;
  box-shadow: 0 6px 20px rgba(10, 46, 92, 0.08);
  height: 150px;
  display: flex;
  flex-direction: column;
}
.chart-card canvas {
  flex: 1;
  min-height: 0;
}
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
  max-height: 350px;
  overflow-x: auto;
  overflow-y: auto;
  border-radius: 8px;
  flex: 1;
}
table {
  width: 100%;
  min-width: 800px;
  border-collapse: collapse;
  table-layout: fixed;
}
.table-box {
  margin-top: 30px;
  padding: 10px;
  border-radius: 8px;
  background: #fff;
  overflow: hidden;
  width: 100%;
  max-height: 450px;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
}

th,
td {
  border: 1px solid #ccc;
  padding: 8px 12px;
  text-align: center;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* Fixed column widths */
.col-waktu {
  width: 100px;
}
.col-suhu {
  width: 90px;
}
.col-hum {
  width: 120px;
}
.col-angin {
  width: 150px;
}
.col-rain {
  width: 130px;
}
.col-lux {
  width: 160px;
  min-width: 160px;
  max-width: 160px;
}

thead {
  background: #f3f8ff;
  color: #0a2e5c;
  position: sticky;
  top: 0;
  z-index: 10;
}
thead th {
  background: #f3f8ff;
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
