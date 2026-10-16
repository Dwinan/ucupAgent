# Ucup Agent - AI-Powered IoT Voice Assistant

Ucup Agent adalah sistem asisten pintar berbasis suara (*voice-controlled AI agent*) yang dirancang untuk mengontrol perangkat elektronik dan memantau sensor rumah pintar (*smart home*) secara *real-time*.

Sistem ini mengintegrasikan antarmuka berbasis web (Web Speech API), orkestrasi AI Agentic pada **n8n** (berbasis LangChain), Large Language Model (Google Gemini), dan mikrokontroler IoT (ESP32).

---

## 🏗️ Arsitektur Sistem

```
+-------------------------------------------------------------+
|                 Antarmuka Pengguna (Web UI)                 |
|  - index.html    : Voice Assistant (STT & TTS + Visualizer) |
|  - dashboard.html: Real-time Device Monitor                 |
+------------------------------+------------------------------+
                               |
                   HTTP POST   |   HTTP GET (Polling /status)
                   (Audio/Text)|
                               v
+------------------------------+--------------+
|                n8n Automation               |
|                                             |
|  [Webhook] (/webhook/ucup)                  |
|       |                                     |
|       v                                     |
|  [AI Agent (Ucup)]                          |
|    ├── LLM 1: Google Gemini 3.8 Flash       |
|    ├── Fallback LLM: OpenRouter (free)      |
|    ├── Memory: Buffer Window (10 turns)     |
|    └── Tools:                               |
|          ├── Tool Sensor   (/sensor)        |
|          └── Tool Kontrol  (/kontrol)       |
|       |                                     |
|       v                                     |
|  [Respond to Webhook]                       |
+----------------------+----------------------+
                       |
             HTTP Call via Tools
                       v
+---------------------------------------------+
|           Mikrokontroler IoT (ESP32)        |
|               IP: 192.168.100.15            |
|                                             |
|  - GET /sensor   : Baca suhu, lembab, jarak |
|  - GET /kontrol  : Relay ON/OFF perangkat   |
|  - GET /status   : JSON status alat         |
+---------------------------------------------+
```

---

## 📁 Struktur File Proyek

| File | Deskripsi |
| :--- | :--- |
| `index.html` | Antarmuka interaktif asisten suara Ucup. Menggunakan Speech Recognition untuk menangkap suara, Web Speech Synthesis (TTS) untuk merespons suara, serta indikator visual reaktif. |
| `dashboard.html` | Panel monitoring status perangkat (*Lampu, TV, Kulkas, AC, Mesin Cuci, Radio*) secara langsung dari ESP32 setiap 1 detik. |
| `ucup-workflow.json` | Konfigurasi alur kerja (workflow) n8n yang mendefinisikan Webhook, AI Agent, LLM Gemini, memori sesi, dan HTTP Tools untuk komunikasi IoT. |

---

## ⚙️ Cara Kerja Workflow AI Agent (Ucup)

Alur kerja AI Agent Ucup pada `ucup-workflow.json` beroperasi dengan langkah-langkah berikut:

### 1. Inisialisasi Sesi & Penerimaan Input
- Pengguna berbicara melalui tombol **"Tekan & Mulai Bicara"** di `index.html`.
- Browser mengubah ucapan menjadi teks (*Speech-to-Text*) bahasa Indonesia (`id-ID`).
- Web mengirimkan HTTP POST ke endpoint n8n `http://localhost:5678/webhook/ucup`:
  ```json
  {
    "message": "Ucup, nyalakan lampu dan cek suhu kamar",
    "chatInput": "Ucup, nyalakan lampu dan cek suhu kamar",
    "session_id": "usr_xxxxxx"
  }
  ```

### 2. Node Webhook (`n8n-nodes-base.webhook`)
- Menerima request POST pada path `/ucup`.
- Mengatur `responseMode` ke `responseNode` agar webhook menunggu hingga proses AI selesai sebelum membalas ke browser.

### 3. Node AI Agent (`@n8n/n8n-nodes-langchain.agent`)
Node ini bertindak sebagai otak utama dari asisten Ucup:
- **Persona & System Prompt**:
  > *"Kamu adalah seorang AI Assistant bernama Ucup yang sangat bisa diandalkan. Kamu tidak suka bertele-tele dan hanya berbicara yang diperlukan."*
- **Penanganan Fallback**: Fitur `needsFallback: true` diaktifkan untuk memastikan agen tetap merespons jika model utama mengalami kendala.

### 4. Language Models (LLM)
- **Model Utama**: `Google Gemini Chat Model` (`models/gemini-3.8-flash`) via `googlePalmApi`. Bertugas menganalisis maksud pengguna (*intent recognition*), menentukan pemanggilan tool (*tool routing*), dan menyusun respons ringkas.
- **Model Cadangan (Fallback)**: `OpenRouter Chat Model` (`openrouter/free`) via `openRouterApi`. Mengambil alih jika kuota atau koneksi ke Google Gemini terganggu.

### 5. Pengelolaan Konteks Percakapan (`Simple Memory`)
- Node `@n8n/n8n-nodes-langchain.memoryBufferWindow`.
- Kunci Sesi: `{{ $json.body.session_id }}` (unik untuk setiap sesi peramban).
- Kapasitas: Menyimpan 10 riwayat percakapan terakhir (`contextWindowLength: 10`) sehingga pengguna dapat melakukan percakapan lanjutan yang berkesinambungan.

### 6. Pemanggilan Tool (*Function Calling / Tools*)
AI Agent dilengkapi dengan dua alat eksternal berbasis HTTP:

#### a. `HTTP Request Sensor`
- **Fungsi**: Dipanggil otomatis jika pengguna menanyakan data sensor, suhu ruangan, kelembapan, atau jarak benda.
- **Endpoint**: `GET http://192.168.100.15/sensor`
- **Hasil**: Data sensor dikembalikan ke model AI untuk diformulasikan ke dalam jawaban singkat untuk pengguna.

#### b. `HTTP Request Perangkat`
- **Fungsi**: Dipanggil saat pengguna memberi perintah menyalakan atau mematikan perangkat elektronik.
- **Endpoint**: `GET http://192.168.100.15/kontrol?device={device}&state={state}`
- **Parameter Ekstraksi AI**:
  - `device`: Dibatasi secara ketat hanya pada salah satu nilai: `"lampu"`, `"tv"`, `"kulkas"`, `"ac"`, `"mesin_cuci"`, atau `"radio"`.
  - `state`: HANYA bernilai `"on"` atau `"off"`.

### 7. Respons ke Klien (`Respond to Webhook`)
- Node `n8n-nodes-base.respondToWebhook` mengemas output AI ke format JSON:
  ```json
  {
    "reply": "Lampu utama berhasil dinyalakan. Suhu ruangan saat ini 27°C."
  }
  ```
- Browser di `index.html` menerima balasan, menampilkan teks di antarmuka chat, dan memutar audio balasan menggunakan sintesis suara browser (*Text-to-Speech*).

---

## 🎛️ Spesifikasi Perangkat & Endpoint ESP32

Sistem mengasumsikan mikrokontroler ESP32 berada pada jaringan lokal dengan IP default `192.168.100.15`.

### 1. Endpoint Sensor
- **URL**: `GET /sensor`
- **Respons (Contoh)**:
  ```json
  {
    "suhu": 28.5,
    "kelembapan": 65,
    "jarak": 120
  }
  ```

### 2. Endpoint Kontrol
- **URL**: `GET /kontrol?device=<nama_alat>&state=<on|off>`
- **Daftar Perangkat yang Didukung**:
  - `lampu`
  - `tv`
  - `kulkas`
  - `ac`
  - `mesin_cuci`
  - `radio`

### 3. Endpoint Status (Digunakan `dashboard.html`)
- **URL**: `GET /status`
- **Respons (Contoh)**:
  ```json
  {
    "lampu": "on",
    "tv": "off",
    "kulkas": "on",
    "ac": "off",
    "mesin_cuci": "off",
    "radio": "off"
  }
  ```

---

## 🚀 Panduan Menjalankan Sistem

1. **Persiapan n8n**:
   - Pastikan aplikasi n8n terpasang dan berjalan (default: `http://localhost:5678`).
   - Impor file `ucup-workflow.json` ke dalam n8n.
   - Siapkan kredensial untuk Google Gemini (`googlePalmApi`) dan OpenRouter (`openRouterApi`).
   - Aktifkan (*Activate / Toggle ON*) workflow tersebut.

2. **Persiapan Hardware (ESP32)**:
   - Hubungkan ESP32 ke jaringan Wi-Fi lokal.
   - Pastikan alamat IP ESP32 disesuaikan jika berbeda dengan `192.168.100.15` (edit di `ucup-workflow.json` dan `dashboard.html`).

3. **Menjalankan Web UI**:
   - Jalankan web server lokal (seperti Laragon, Apache, atau Live Server).
   - Akses `index.html` untuk berinteraksi langsung melalui suara dengan AI Agent Ucup.
   - Buka `dashboard.html` pada layar sekunder untuk memantau status fisik perangkat rumah secara *live*.
