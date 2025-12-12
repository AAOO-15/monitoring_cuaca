# 🔧 Troubleshooting: Data Tidak Tampil di Dashboard

## Diagnosis Masalah

Meskipun koneksi ke HiveMQ Cloud sudah berhasil ("✅ Connected to HiveMQ Cloud"), data grafik dan tabel tidak tampil. Berikut adalah kemungkinan penyebab dan solusinya:

## 🔍 Langkah Troubleshooting

### 1. Buka Debug Tool

1. Buka file `debug-mqtt.html` di browser
2. Lihat apakah ada pesan masuk dari ESP32
3. Klik tombol "Test Publish Data" untuk memastikan dashboard dapat menerima data

### 2. Periksa ESP32

#### A. Periksa Serial Monitor ESP32

```cpp
// Pastikan ESP32 menampilkan pesan berikut:
- "WiFi connected"
- "Connected to MQTT broker"
- "=== Data Cuaca ===" (setiap pengiriman data)
```

#### B. Periksa Topic dan Format Data

ESP32 harus mengirim data ke topic berikut dengan format yang benar:

```
esp32/suhu           → angka (contoh: 25.5)
esp32/kelembapan     → angka (contoh: 60)
esp32/curah_hujan    → angka (contoh: 0.5)
esp32/kecepatan_angin → angka (contoh: 2.3)
esp32/cuaca          → teks (contoh: "Cerah")
esp32/lux            → angka (contoh: 1200)
```

#### C. Periksa Broker URL di ESP32

Pastikan ESP32 menggunakan broker yang sama:

```cpp
const char* mqtt_server = "751cee3a0dcb4364bbb1e4dbe4a87ca5.s1.eu.hivemq.cloud";
int mqtt_port = 8883; // untuk SSL atau 1883 untuk non-SSL
```

### 3. Periksa Browser Console

1. Buka Developer Tools (F12)
2. Pergi ke tab **Console**
3. Lihat apakah ada pesan berikut:
   - `✅ Connected to HiveMQ Cloud`
   - `🎉 === MESSAGE RECEIVED ===`
   - `📩 MQTT Message Details:`

### 4. Kemungkinan Masalah dan Solusi

#### A. ESP32 Tidak Mengirim Data

**Gejala:** Console browser hanya menampilkan "Connected" tapi tidak ada pesan data
**Solusi:**

1. Reset ESP32
2. Periksa kode ESP32 apakah ada error
3. Pastikan sensor terhubung dengan benar

#### B. Format Data Salah

**Gejala:** Console menampilkan "⚠️ Invalid number from [topic]"
**Solusi:** Pastikan ESP32 mengirim angka murni, bukan teks dengan satuan

#### C. Topic Name Salah

**Gejala:** Console menampilkan "⚠️ Unknown topic"
**Solusi:** Periksa nama topic di ESP32, harus persis sesuai yang diharapkan

#### D. Koneksi ESP32 Terputus

**Gejala:** Dashboard connected tapi tidak ada data baru
**Solusi:**

1. Periksa WiFi ESP32
2. Periksa koneksi MQTT di ESP32
3. Implementasi reconnect di ESP32

### 5. Test Manual

Gunakan MQTT client eksternal untuk test:

```bash
# Install MQTT client
npm install -g mqtt

# Subscribe untuk melihat data
mqtt sub -h 751cee3a0dcb4364bbb1e4dbe4a87ca5.s1.eu.hivemq.cloud -p 8883 -s -u langgamdewa -P Rizkypratama512 -t "esp32/#"

# Publish test data
mqtt pub -h 751cee3a0dcb4364bbb1e4dbe4a87ca5.s1.eu.hivemq.cloud -p 8883 -s -u langgamdewa -P Rizkypratama512 -t "esp32/suhu" -m "25.5"
```

## 🛠️ Perbaikan Kode ESP32

Jika masalah di ESP32, pastikan kode mengikuti struktur ini:

```cpp
#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "your_wifi_name";
const char* password = "your_wifi_password";
const char* mqtt_server = "751cee3a0dcb4364bbb1e4dbe4a87ca5.s1.eu.hivemq.cloud";
const char* mqtt_username = "langgamdewa";
const char* mqtt_password = "Rizkypratama512";

WiFiClientSecure espClient;
PubSubClient client(espClient);

void setup() {
    Serial.begin(115200);

    // WiFi connection
    WiFi.begin(ssid, password);
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        Serial.print(".");
    }
    Serial.println("WiFi connected");

    // MQTT setup
    espClient.setInsecure(); // Untuk testing, gunakan SSL yang proper untuk production
    client.setServer(mqtt_server, 8883);

    // Connect to MQTT
    while (!client.connected()) {
        String clientId = "ESP32Client-" + String(random(0xffff), HEX);
        if (client.connect(clientId.c_str(), mqtt_username, mqtt_password)) {
            Serial.println("Connected to MQTT broker");
        } else {
            Serial.print("MQTT connection failed, rc=");
            Serial.print(client.state());
            Serial.println(" retrying in 5 seconds");
            delay(5000);
        }
    }
}

void loop() {
    if (!client.connected()) {
        // Reconnect logic here
    }

    client.loop();

    // Baca sensor dan kirim data
    float suhu = readTemperature(); // implement your sensor reading
    float kelembapan = readHumidity();
    float curah_hujan = readRainfall();
    float kecepatan_angin = readWindSpeed();
    String cuaca = getWeatherCondition();
    int lux = readLightIntensity();

    // Publish data dengan format yang benar
    client.publish("esp32/suhu", String(suhu).c_str());
    client.publish("esp32/kelembapan", String(kelembapan).c_str());
    client.publish("esp32/curah_hujan", String(curah_hujan).c_str());
    client.publish("esp32/kecepatan_angin", String(kecepatan_angin).c_str());
    client.publish("esp32/cuaca", cuaca.c_str());
    client.publish("esp32/lux", String(lux).c_str());

    Serial.println("=== Data Cuaca ===");
    Serial.printf("Suhu: %.1f°C\n", suhu);
    Serial.printf("Kelembapan: %.1f%%\n", kelembapan);
    Serial.printf("Curah Hujan: %.2f mm\n", curah_hujan);
    Serial.printf("Kecepatan Angin: %.1f m/s\n", kecepatan_angin);
    Serial.printf("Cuaca: %s\n", cuaca.c_str());
    Serial.printf("LUX: %d\n", lux);
    Serial.println("==================");

    delay(5000); // Kirim data setiap 5 detik
}
```

## 📋 Checklist Troubleshooting

- [ ] ESP32 terhubung ke WiFi
- [ ] ESP32 terhubung ke MQTT broker
- [ ] ESP32 mengirim data ke topic yang benar
- [ ] Format data dari ESP32 sesuai (angka tanpa satuan)
- [ ] Browser console menampilkan pesan "MESSAGE RECEIVED"
- [ ] Debug tool menampilkan data masuk
- [ ] Tidak ada error di browser console
- [ ] Nama topic persis sama (case-sensitive)

## 🎯 Quick Fix

Jika semua di atas sudah dicek, coba:

1. **Reset ESP32** dan tunggu 30 detik
2. **Refresh browser** dan tunggu pesan di console
3. **Test dengan debug tool** untuk memastikan data masuk
4. **Periksa network firewall** yang mungkin memblokir WebSocket

Dengan langkah ini, Anda dapat mengidentifikasi apakah masalah ada di ESP32 (tidak mengirim data) atau di dashboard (tidak menerima/memproses data).
