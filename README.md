#define BLYNK_TEMPLATE_ID "TMPL3Z9CV3dhh"
#define BLYNK_TEMPLATE_NAME "TEMPERATURE MONITORING"
#define BLYNK_AUTH_TOKEN "CD0u4q_lEj9f3n4IYKSdOu6_8IKGnc1K"

#define BLYNK_PRINT Serial

#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <WiFiClient.h>
#include <ThingSpeak.h>

// WiFi
char ssid[] = "Iphone Xs";
char pass[] = "1234567890";

// Sensor
#define ONE_WIRE_BUS 4
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

// Pins
#define RELAY 5
#define BUZZER 18
#define SWITCH 21
#define GREEN_LED 19
#define YELLOW_LED 22
#define RED_LED 23

// States
int fanState = 0;         
bool manualState = false;

// Button debounce
int lastButtonState = HIGH;
unsigned long lastDebounceTime = 0;

// ThingSpeak
WiFiClient client;
unsigned long channelNumber = 3323761;
const char * writeAPIKey = "1HZQ305VKUMAPO1S";
unsigned long lastThingSpeak = 0;

// 🔴 IMPORTANT: Change if needed
bool relayActiveLOW = true;

int lastFanState = -1;

// 📱 Blynk
BLYNK_WRITE(V1) {
  fanState = param.asInt();
}

void setup() {
  Serial.begin(115200);

  WiFi.begin(ssid, pass);
  Blynk.config(BLYNK_AUTH_TOKEN);

  sensors.begin();

  pinMode(RELAY, OUTPUT);
  pinMode(BUZZER, OUTPUT);
  pinMode(SWITCH, INPUT_PULLUP);

  pinMode(GREEN_LED, OUTPUT);
  pinMode(YELLOW_LED, OUTPUT);
  pinMode(RED_LED, OUTPUT);

  ThingSpeak.begin(client);

  // Start fan OFF
  digitalWrite(RELAY, relayActiveLOW ? HIGH : LOW);

  Serial.println("System Started...");
}

void loop() {

  if (WiFi.status() == WL_CONNECTED) {
    Blynk.run();
  }

  // 🌡️ Read temperature
  sensors.requestTemperatures();
  float temp = sensors.getTempCByIndex(0);

  Serial.print("Temp: ");
  Serial.print(temp);
  Serial.print(" °C | ");

  // Send to Blynk
  if (WiFi.status() == WL_CONNECTED) {
    Blynk.virtualWrite(V0, temp);
  }

  // ThingSpeak update
  if (WiFi.status() == WL_CONNECTED && millis() - lastThingSpeak > 15000) {
    ThingSpeak.setField(1, temp);
    ThingSpeak.writeFields(channelNumber, writeAPIKey);
    lastThingSpeak = millis();
  }

  // 🔘 BUTTON (toggle)
  int reading = digitalRead(SWITCH);

  if (reading == LOW && lastButtonState == HIGH && millis() - lastDebounceTime > 200) {
    manualState = !manualState;
    lastDebounceTime = millis();
  }

  lastButtonState = reading;

  // 🔥 AUTOMATION
  int autoFan = 0;

  digitalWrite(GREEN_LED, LOW);
  digitalWrite(YELLOW_LED, LOW);
  digitalWrite(RED_LED, LOW);
  digitalWrite(BUZZER, LOW);

  if (temp > 36) {
    autoFan = 1;
    digitalWrite(BUZZER, HIGH);

    if (millis() % 600 < 100)
      digitalWrite(RED_LED, HIGH);
  }
  else if (temp > 35) {
    autoFan = 1;
    digitalWrite(BUZZER, HIGH);
    digitalWrite(RED_LED, HIGH);
  }
  else if (temp > 34) {
    autoFan = 0;
    digitalWrite(YELLOW_LED, HIGH);
  }
  else {
    autoFan = 0;
    digitalWrite(GREEN_LED, HIGH);
  }

  // 🎯 FINAL DECISION
  int finalFan = 0;

  // 🟢🟡 Force OFF
  if (temp <= 34) {
    finalFan = 0;
  }
  else {
    if (manualState) {
      finalFan = 1;
    }
    else if (fanState == 1) {
      finalFan = 1;
    }
    else {
      finalFan = autoFan;
    }
  }

  // 🔌 Relay control
  if (relayActiveLOW) {
    digitalWrite(RELAY, finalFan ? LOW : HIGH);
  } else {
    digitalWrite(RELAY, finalFan ? HIGH : LOW);
  }

  // 📲 Update Blynk switch
  if (WiFi.status() == WL_CONNECTED && finalFan != lastFanState) {
    Blynk.virtualWrite(V1, finalFan);
    lastFanState = finalFan;
  }

  Serial.print("Fan: ");
  Serial.println(finalFan ? "ON" : "OFF");

  delay(1500);
}
