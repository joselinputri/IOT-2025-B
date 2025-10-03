# 📑 Laporan Resmi
**Proyek:** Monitoring Jarak dengan Sensor Ultrasonik & ESP32, Upload Data ke ThingSpeak  

---

## 👥 Anggota Kelompok
| Nama                     | NRP        |
|---------------------------|------------|
| Shinta Alya Ramadani      | 5027241016 |
| Nisrina Bilqis            | 5027241054 |
| Putri Joselina Silitonga  | 5027241116 |

---

## 1. Pendahuluan
Proyek ini bertujuan untuk membaca jarak menggunakan **sensor ultrasonik HC-SR04**, menyalakan LED sebagai indikator jika objek terlalu dekat, dan mengirimkan data hasil pembacaan jarak ke **ThingSpeak** melalui koneksi WiFi menggunakan board **ESP32**.  

ThingSpeak digunakan sebagai IoT cloud untuk menyimpan data secara real-time dan menampilkannya dalam bentuk grafik.  

---

## 2. Alat dan Bahan
- Board **ESP32**  
- Sensor ultrasonik **HC-SR04**  
- LED + resistor 220Ω  
- Breadboard dan kabel jumper  
- WiFi dengan akses internet  
- Laptop/PC dengan **Arduino IDE**
- Akun ThingSpeak

---

## 3. Persiapan Arduino IDE
1. Install **Arduino IDE** (versi terbaru).  
2. Buka **Arduino IDE → File > Preferences**.  
   - Pada kolom *Additional Board Manager URLs* masukkan:  
     ```
     https://dl.espressif.com/dl/package_esp32_index.json
     ```
3. Buka **Tools > Board > Board Manager** → cari **esp32** → klik *Install*.  
4. Pilih board: **ESP32 Dev Module**.  
5. Pilih port sesuai dengan ESP32 yang terhubung ke laptop.  

---

## 4. Skema Rangkaian
- **HC-SR04**:  
  - VCC → 5V ESP32  
  - GND → GND ESP32  
  - TRIG → GPIO 5  
  - ECHO → GPIO 18  
- **LED**:  
  - Anoda (+) → GPIO 12 (D0) via resistor 220Ω  
  - Katoda (–) → GND  

---

## 5. Konfigurasi ThingSpeak
1. Buat akun di [ThingSpeak](https://thingspeak.com).  
2. Buat **New Channel** dengan konfigurasi:  
   - Field 1 → Distance (cm)  
   - Field 2 → LED Status  
3. Catat:  
   - **Channel ID** → `3095474`  
   - **Write API Key** → `B9RPNZFM3GWW4AMK`  

---

## 6. Program Arduino (Kode Final)

```cpp
#include <WiFi.h>

WiFiClient client;

// Pin
const int trigPin = 5;   // TRIG di pin 5
const int echoPin = 18;  // ECHO di pin 18
const int ledPin  = 12;  // LED di pin 12 (D0)

// Kecepatan suara
#define SOUND_SPEED 0.034

long duration;
float distanceCm;

// ThingSpeak
String thingSpeakAddress = "api.thingspeak.com";
String request_string;

// WiFi
const char* ssid = "RedmiNote13";    // ganti WiFi kamu
const char* password = "";           // password WiFi (kosongkan kalau tidak ada)

// ThingSpeak
String writeAPIKey = "B9RPNZFM3GWW4AMK";  // API key
unsigned long myChannelNumber = 3095474;  // Channel ID

void setup() {
  Serial.begin(115200);
  WiFi.disconnect();
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(300);
    Serial.print(".");
  }

  Serial.println("\nWiFi connected");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());

  pinMode(trigPin, OUTPUT); 
  pinMode(echoPin, INPUT);  
  pinMode(ledPin, OUTPUT);
}

void loop() {
  delay(1000);

  // Baca sensor ultrasonik
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  duration = pulseIn(echoPin, HIGH, 30000); // timeout 30ms
  if (duration == 0) {
    distanceCm = -1; // tidak ada objek
  } else {
    distanceCm = duration * SOUND_SPEED / 2;
  }

  Serial.print("Distance (cm): ");
  Serial.println(distanceCm);

  // Kontrol LED
  int ledStatus = 0;
  if (distanceCm > 0 && distanceCm < 10) {
    digitalWrite(ledPin, HIGH);
    ledStatus = 1;
  } else {
    digitalWrite(ledPin, LOW);
    ledStatus = 0;
  }

  // Kirim ke ThingSpeak
  kirim_thingspeak(distanceCm, ledStatus);

  Serial.println("--------------------");

  delay(20000); // minimal 15 detik antar update
}

void kirim_thingspeak(float discm, int ledStatus) {
  if (client.connect("api.thingspeak.com", 80)) {
    request_string = "/update?api_key=" + writeAPIKey;
    request_string += "&field1=";
    request_string += discm;
    request_string += "&field2=";
    request_string += ledStatus;

    Serial.println(String("GET ") + request_string + " HTTP/1.1\r\n" +
                   "Host: " + thingSpeakAddress + "\r\n" +
                   "Connection: close\r\n\r\n");

    client.print(String("GET ") + request_string + " HTTP/1.1\r\n" +
                 "Host: " + thingSpeakAddress + "\r\n" +
                 "Connection: close\r\n\r\n");

    unsigned long timeout = millis();
    while (client.available() == 0) {
      if (millis() - timeout > 5000) {
        Serial.println(">>> Client Timeout !");
        client.stop();
        return;
      }
    }

    while (client.available()) {
      String line = client.readStringUntil('\r');
      Serial.print(line);
    }

    Serial.println();
    Serial.println("closing connection");
  }
}
```
## 7. Cara Upload Program
1. Hubungkan **ESP32** ke laptop dengan kabel USB.  
2. Buka **Arduino IDE**, lalu salin kode program.  
3. Pilih **Tools > Board > ESP32 Dev Module**.  
4. Pilih **Port** sesuai ESP32 yang terhubung.  
5. Klik tombol **Upload** (ikon panah kanan).  
6. Setelah selesai, buka **Serial Monitor** dengan baud rate **115200** untuk melihat hasil.  

## 8. Hasil Pengujian
#### a. Serial Monitor
ESP32 berhasil membaca jarak dari sensor ultrasonik. Saat objek didekatkan <10 cm, LED menyala dan status LED ditampilkan 1. 
Saat objek menjauh >10 cm, LED mati dengan status 0.

- Contoh output serial monitor:
```
Distance (cm): 7.02
LED Status: 1
closing connection
--------------------
Distance (cm): 15.33
LED Status: 0
closing connection
--------------------
```
#### b. ThingSpeak Dashboard
Data berhasil terkirim ke ThingSpeak dengan interval update setiap 20 detik.

- Field 1 (Sensor Jarak) menampilkan grafik perubahan jarak (dalam inci).  
<img width="475" height="332" alt="image" src="https://github.com/user-attachments/assets/197d873e-3e4a-4b6b-ae80-63d132948cad" />

- Field 2 (LED Status) menampilkan status LED (nilai 1 = ON, 0 = OFF).
<img width="475" height="325" alt="image" src="https://github.com/user-attachments/assets/75b007e0-4cc3-4d52-8d49-cd25cde65db3" />

## 9. Kesimpulan
1. Sensor ultrasonik berhasil membaca jarak objek secara real-time.
2. LED berhasil dikontrol otomatis: menyala jika objek <10 cm, mati jika objek menjauh.
3. Data jarak dan status LED berhasil dikirim dan divisualisasikan di ThingSpeak dalam bentuk grafik.



