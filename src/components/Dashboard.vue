<template>
  <div>
    <h2>🌤️ Dashboard Cuaca ESP32</h2>

    <div class="data-box">Suhu: {{ suhu }} °C</div>
    <div class="data-box">Kelembapan: {{ kelembapan }} %</div>
    <div class="data-box">Curah Hujan: {{ curahHujan }} mm</div>
    <div class="data-box">Kecepatan Angin: {{ kecepatanAngin }} m/s</div>
    <div class="data-box">Cuaca Hari Ini: {{ cuaca }}</div>

    <div class="chart-grid">
      <div class="chart-box">
        <h3>🌧️ Curah Hujan</h3>
        <canvas ref="rainCanvas"></canvas>
      </div>
      <div class="chart-box">
        <h3>🌡️ Suhu</h3>
        <canvas ref="suhuCanvas"></canvas>
      </div>
      <div class="chart-box">
        <h3>💨 Kecepatan Angin</h3>
        <canvas ref="anginCanvas"></canvas>
      </div>
      <div class="chart-box">
        <h3>💧 Kelembapan</h3>
        <canvas ref="humCanvas"></canvas>
      </div>
    </div>

    <button @click="downloadCSV">📥 Download Data Curah Hujan</button>
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
const cuaca = ref('--')

// Canvas refs
const rainCanvas = ref<HTMLCanvasElement | null>(null)
const suhuCanvas = ref<HTMLCanvasElement | null>(null)
const anginCanvas = ref<HTMLCanvasElement | null>(null)
const humCanvas = ref<HTMLCanvasElement | null>(null)

// Historis data
const rainData = ref<number[]>([])
const rainLabels = ref<string[]>([])
const suhuData = ref<number[]>([])
const suhuLabels = ref<string[]>([])
const anginData = ref<number[]>([])
const anginLabels = ref<string[]>([])
const humData = ref<number[]>([])
const humLabels = ref<string[]>([])

// Chart instances
let rainChart: Chart | null = null
let suhuChart: Chart | null = null
let anginChart: Chart | null = null
let humChart: Chart | null = null

onMounted(async () => {
  await nextTick()

  const client = mqtt.connect('wss://751cee3a0dcb4364bbb1e4dbe4a87ca5.s1.eu.hivemq.cloud:8884/mqtt', {
    username: 'langgamdewa',
    password: 'Rizkypratama512'
  })

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
      if (suhuChart) {
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
      if (humChart) {
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
      if (rainChart) {
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
      if (anginChart) {
        anginChart.data.labels = anginLabels.value.slice()
        anginChart.data.datasets[0].data = anginData.value.slice()
        anginChart.update()
      }
    }

    if (topic === 'esp32/cuaca') {
      cuaca.value = val
    }
  })

  // Chart setup
  if (rainCanvas.value) {
    rainChart = new Chart(rainCanvas.value, {
      type: 'line',
      data: {
        labels: [],
        datasets: [{
          label: 'Curah Hujan (mm)',
          data: [],
          borderColor: 'blue',
          fill: false
        }]
      },
      options: { responsive: true }
    })
  }

  if (suhuCanvas.value) {
    suhuChart = new Chart(suhuCanvas.value, {
      type: 'line',
      data: {
        labels: [],
        datasets: [{
          label: 'Suhu (°C)',
          data: [],
          borderColor: 'red',
          fill: false
        }]
      },
      options: { responsive: true }
    })
  }

  if (humCanvas.value) {
    humChart = new Chart(humCanvas.value, {
      type: 'line',
      data: {
        labels: [],
        datasets: [{
          label: 'Kelembapan (%)',
          data: [],
          borderColor: 'purple',
          fill: false
        }]
      },
      options: { responsive: true }
    })
  }

  if (anginCanvas.value) {
    anginChart = new Chart(anginCanvas.value, {
      type: 'line',
      data: {
        labels: [],
        datasets: [{
          label: 'Kecepatan Angin (m/s)',
          data: [],
          borderColor: 'green',
          fill: false
        }]
      },
      options: { responsive: true }
    })
  }
})

function downloadCSV() {
  let csv = "Waktu,Curah Hujan (mm)\n"
  for (let i = 0; i < rainData.value.length; i++) {
    csv += `${rainLabels.value[i]},${rainData.value[i]}\n`
  }
  const blob = new Blob([csv], { type: 'text/csv' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = 'data_curah_hujan.csv'
  a.click()
  URL.revokeObjectURL(url)
}
</script>

<style scoped>
h2 {
  font-size: 24px;
  margin-bottom: 10px;
}
.data-box {
  margin-bottom: 8px;
  font-size: 16px;
}
.chart-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 20px;
  margin-top: 20px;
}
.chart-box {
  flex: 1 1 300px;
  border: 1px solid #ddd;
  padding: 10px;
  border-radius: 8px;
  background: #f9f9f9;
}
.chart-box h3 {
  margin-bottom: 10px;
  font-size: 18px;
}
canvas {
  max-width: 100%;
}
button {
  margin-top: 20px;
  padding: 8px 16px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}
button:hover {
  background-color: #0056b3;
}
</style>