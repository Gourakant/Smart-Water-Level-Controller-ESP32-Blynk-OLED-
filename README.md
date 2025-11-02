# Smart-Water-Level-Controller-ESP32-Blynk-OLED-
# 💧 Smart Water Level Controller (ESP32 + Blynk + OLED)

A dual-mode IoT water level controller that automatically manages your water pump or allows manual operation via the Blynk app. Designed by **Goura Sen**.

---

## ⚙️ Features

- 🧠 **Dual Mode:** Auto / Blynk Manual (SPDT switch)
- 🌐 **Wi-Fi Active Only in Manual Mode**
- 💡 **OLED Display:** Real-time water level (0–100%)
- 📱 **Blynk Interface:** Remote level monitoring and pump control
- 🔔 **Buzzer Alerts:** On pump start and stop
- 🟢🔴 **LED Indicators:** Pump ON/OFF status
- 🚰 **4-Level Sensors:** Top, High, Mid, Low

---

## 🔌 Hardware Used

| Component | Quantity | Description |
|------------|-----------|-------------|
| ESP32 | 1 | Main controller |
| 0.96" OLED Display | 1 | I2C display |
| Relay Module | 1 | To control pump |
| Buzzer | 1 | Sound alert |
| LED (Red, Green) | 2 | Status indicator |
| Water Level Sensors | 4 | Tank sensors |
| SPDT Toggle Switch | 1 | Mode selector |

---

## 🧩 Wiring Diagram
*(Upload your image to `/assets/circuit_diagram.png` and link it here.)*

---

## 💻 Blynk Configuration

- App: **Blynk IoT (New Version)**
- Widgets:
  - **Button:** Virtual pin `V1` → Pump Control
  - **Gauge or Level Bar:** Virtual pin `V2` → Tank Level (%)
- Authentication Token: Update in `main.ino`

---

## 📜 Code Description

- Initializes Wi-Fi **only when in Blynk (Manual) mode**
- In **Auto mode**, controls the pump using sensor logic:
  - Pump starts when bottom sensor detects low level
  - Pump stops when top sensor detects full tank
- Real-time water level sent to OLED and Blynk
- Debounced readings to prevent false triggers

---

## 🔧 Setup Steps

1. Clone this repository  
   ```bash
   git clone https://github.com/yourusername/Smart-Water-Level-Controller.git


