# Smart-Water-Level-Controller-ESP32-Blynk-OLED-
#define BLYNK_TEMPLATE_ID "TMPL6KNbJzxg3"
#define BLYNK_TEMPLATE_NAME "WaterLevelMonitor"
#define BLYNK_AUTH_TOKEN "feIxfqGiHBcgsDw5jxfTh7OrTtLoLRdb"

#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SH110X.h>

// ===== OLED Display =====
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
#define SCREEN_ADDRESS 0x3C
Adafruit_SH1106G display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// ===== Wi-Fi credentials =====
char ssid[] = "Redmi 14C";
char pass[] = "000000009";

// ===== Pin Configuration =====
#define relayPin 26
#define greenLED 25
#define redLED 27
#define buzzerPin 32

#define S1 34   // Bottom sensor
#define S2 35   // Mid-low
#define S3 33   // Mid-high
#define S4 14   // Top sensor
#define modePin 13  // SPDT toggle: LOW = Auto, HIGH = Blynk

// ===== Blynk Virtual Pins =====
#define VPIN_TANK_STATUS V1
#define VPIN_MOTOR_SWITCH V2
#define VPIN_MOTOR_LED V3

// ===== Global Variables =====
bool motorState = false;
bool autoMode = true;
bool wifiConnected = false;
String tankStatus = "EMPTY";
int waterPercent = 0;
bool manualMode = false;
unsigned long lastDisplayUpdate = 0;
unsigned long lastSensorCheck = 0;
unsigned long lastCountdownTime = 0;
int countdown = 0;
bool countdownActive = false;
bool startAction = false;
bool stopAction = false;

// ===== Display Function =====
void showDisplay(String status, String mode, int percent, int count = 0) {
  display.clearDisplay();
  display.drawRect(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT, SH110X_WHITE);
  display.setTextSize(1);
  display.setTextColor(SH110X_WHITE);
  display.setCursor(15, 0);
  display.println("TANK CONTROLLER");

  display.setCursor(0, 20);
  display.print("Tank: ");
  display.println(status);

  display.setCursor(0, 35);
  display.print("Water: ");
  display.print(percent);
  display.println("%");

  display.setCursor(0, 50);
  display.print("Mode: ");
  display.println(mode);

  if (count > 0) {
    display.setCursor(90, 50);
    display.print("T-");
    display.print(count);
    display.print("s");
  }

  display.display();
}

// ===== Motor Control =====
void motorBeep(bool starting) {
  int toneDelay = starting ? 200 : 100;
  for (int i = 0; i < 3; i++) {
    digitalWrite(buzzerPin, HIGH);
    delay(toneDelay);
    digitalWrite(buzzerPin, LOW);
    delay(toneDelay);
  }
}

void updateMotor(bool state, String reason) {
  motorState = state;
  digitalWrite(relayPin, state);
  digitalWrite(greenLED, state);
  digitalWrite(redLED, !state);
  if (wifiConnected) Blynk.virtualWrite(VPIN_MOTOR_LED, state ? 1 : 0);
  motorBeep(state);
  Serial.println(reason);
}

// ===== Wi-Fi Handling =====
void connectWiFi() {
  if (!wifiConnected) {
    WiFi.begin(ssid, pass);
    Serial.print("Connecting WiFi");
    int retries = 0;
    while (WiFi.status() != WL_CONNECTED && retries < 20) {
      delay(500);
      Serial.print(".");
      retries++;
    }
    if (WiFi.status() == WL_CONNECTED) {
      Serial.println("\nWiFi Connected!");
      Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);
      wifiConnected = true;
    } else {
      Serial.println("\nWiFi Failed.");
      wifiConnected = false;
    }
  }
}

void disconnectWiFi() {
  if (wifiConnected) {
    WiFi.disconnect(true);
    wifiConnected = false;
    Serial.println("WiFi Disconnected.");
  }
}

// ===== Blynk Manual Control =====
BLYNK_WRITE(VPIN_MOTOR_SWITCH) {
  int manualControl = param.asInt();
  if (!autoMode) {
    manualMode = true;
    countdown = 3;
    countdownActive = true;
    startAction = manualControl;
  }
}

// ===== Sensor Reading =====
void readSensors() {
  static int stableCount = 0;
  bool b1 = digitalRead(S1);
  bool b2 = digitalRead(S2);
  bool b3 = digitalRead(S3);
  bool b4 = digitalRead(S4);

  // Simple debounce: only update if readings stable for 3 cycles
  static bool prevB1, prevB2, prevB3, prevB4;
  if (b1 == prevB1 && b2 == prevB2 && b3 == prevB3 && b4 == prevB4)
    stableCount++;
  else
    stableCount = 0;

  prevB1 = b1; prevB2 = b2; prevB3 = b3; prevB4 = b4;

  if (stableCount >= 3) {
    if (!b1 && !b2 && !b3 && !b4) { tankStatus = "EMPTY"; waterPercent = 0; }
    else if (b1 && !b2 && !b3 && !b4) { tankStatus = "LOW"; waterPercent = 25; }
    else if (b1 && b2 && !b3 && !b4) { tankStatus = "HALF"; waterPercent = 50; }
    else if (b1 && b2 && b3 && !b4) { tankStatus = "ALMOST FULL"; waterPercent = 75; }
    else if (b1 && b2 && b3 && b4) { tankStatus = "FULL"; waterPercent = 100; }
  }
}

// ===== Setup =====
void setup() {
  Serial.begin(115200);
  pinMode(relayPin, OUTPUT);
  pinMode(greenLED, OUTPUT);
  pinMode(redLED, OUTPUT);
  pinMode(buzzerPin, OUTPUT);
  pinMode(S1, INPUT);
  pinMode(S2, INPUT);
  pinMode(S3, INPUT);
  pinMode(S4, INPUT);
  pinMode(modePin, INPUT_PULLUP);

  digitalWrite(relayPin, LOW);
  digitalWrite(greenLED, LOW);
  digitalWrite(redLED, HIGH);

  display.begin(SCREEN_ADDRESS, true);
  display.clearDisplay();
  display.display();

  autoMode = (digitalRead(modePin) == LOW);
  if (autoMode)
    showDisplay("Ready", "Auto Mode", 0);
  else {
    showDisplay("WiFi Connecting", "Blynk Mode", 0);
    connectWiFi();
  }
}

// ===== Main Loop =====
void loop() {
  autoMode = (digitalRead(modePin) == LOW);
  if (autoMode) {
    if (wifiConnected) disconnectWiFi();
  } else {
    if (!wifiConnected) connectWiFi();
  }

  if (wifiConnected) Blynk.run();

  unsigned long now = millis();

  // Sensor read every 500ms
  if (now - lastSensorCheck > 500) {
    readSensors();
    lastSensorCheck = now;
  }

  // Countdown handling (non-blocking)
  if (countdownActive && now - lastCountdownTime >= 1000) {
    showDisplay(tankStatus, startAction ? "Starting" : "Stopping", waterPercent, countdown);
    countdown--;
    lastCountdownTime = now;
    if (countdown <= 0) {
      countdownActive = false;
      updateMotor(startAction, startAction ? "Motor ON via Blynk" : "Motor OFF via Blynk");
      manualMode = false;
    }
  }

  // Auto Mode Logic with hysteresis
  static bool tankWasEmpty = false;
  static bool tankWasFull = false;

  if (autoMode && !manualMode && !countdownActive) {
    if (tankStatus == "EMPTY" && !motorState && !tankWasEmpty) {
      tankWasEmpty = true;
      tankWasFull = false;
      countdown = 3;
      countdownActive = true;
      startAction = true;
    }
    else if (tankStatus == "FULL" && motorState && !tankWasFull) {
      tankWasFull = true;
      tankWasEmpty = false;
      countdown = 3;
      countdownActive = true;
      startAction = false;
    }
  }

  // Display updates every 1s if no countdown
  if (!countdownActive && now - lastDisplayUpdate > 1000) {
    String modeText = autoMode
      ? (motorState ? "Auto ON" : "Auto OFF")
      : (motorState ? "Blynk ON" : "Blynk OFF");

    showDisplay(tankStatus, modeText, waterPercent);
    if (wifiConnected) Blynk.virtualWrite(VPIN_TANK_STATUS, tankStatus);
    lastDisplayUpdate = now;
  }
}

