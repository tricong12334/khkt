#define BLYNK_TEMPLATE_ID "TMPL6cLfNXX2y"
#define BLYNK_TEMPLATE_NAME "Nhà"
#define BLYNK_AUTH_TOKEN "NWh7SuvChvnKBUc2lv_7KlbtJfJEmUtr"

#include <WiFi.h>
#include <HTTPClient.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_MLX90614.h>
#include <MPU6050.h>
#include <MAX30100_PulseOximeter.h>
#include <Ticker.h>
#include <BlynkSimpleEsp32.h>

// OLED Config
#define OLED_WIDTH 128
#define OLED_HEIGHT 32
#define OLED_RESET -1
Adafruit_SSD1306 display(OLED_WIDTH, OLED_HEIGHT, &Wire, OLED_RESET);

// GY-906 Sensor
Adafruit_MLX90614 mlx = Adafruit_MLX90614();

// MPU6050 Sensor
MPU6050 mpu;

// MAX30100 Sensor
PulseOximeter pox;
Ticker timer;

// SIM Module
HardwareSerial simSerial(1); // UART1 RX=16, TX=17
#define SIM_PHONE_NUMBER "0355390945"

// Wi-Fi Credentials
char ssid[] = "Galaxy";
char pass[] = "11111111";
String GOOGLE_SCRIPT_ID = "AKfycbxWwrBXp6gZAFTTAGzehiUNoJ_OdImT357L37Fu7h0bUSUgVob4fRDIE2_vrJbWhodG";

// Global Variables
unsigned long lastDisplayUpdate = 0;
unsigned long lastBlynkUpdate = 0;
float temperature = 0.0;
float heartRate = 0.0;
float spO2 = 0.0;
bool fallDetected = false;

void setup() {
  Serial.begin(115200);
  Wire.begin();

  // Initialize WiFi
  WiFi.begin(ssid, pass);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }
  Serial.println("Connected to WiFi");

  // Initialize OLED
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("OLED initialization failed!");
    while (1);
  }
  display.clearDisplay();
  display.setTextColor(WHITE);
  display.setTextSize(1);

  // Initialize SIM module
  simSerial.begin(115200, SERIAL_8N1, 16, 17);

  // Initialize Sensors
  if (!mlx.begin()) {
    Serial.println("MLX90614 initialization failed!");
    while (1);
  }
  mpu.initialize();
  if (!mpu.testConnection()) {
    Serial.println("MPU6050 connection failed!");
    while (1);
  }
  if (!pox.begin()) {
    Serial.println("MAX30100 initialization failed!");
    while (1);
  }
  pox.setIRLedCurrent(MAX30100_LED_CURR_7_6MA);
  pox.setOnBeatDetectedCallback(onBeatDetected);

  // Initialize Blynk
  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);

  // Start Timer for MAX30100 Updates
  timer.attach_ms(100, update);

  // Display Welcome Message
  displayMessage("Welcome to", "khkt");
}

void loop() {
  Blynk.run();

  // Update temperature
  temperature = mlx.readObjectTempC();

  // Update heart rate and SpO2
  heartRate = pox.getHeartRate();
  spO2 = pox.getSpO2();

  // Detect Fall
  fallDetected = detectFall();
  if (fallDetected) {
    sendSMS("Warning: Fall detected!");
  }

  // Detect High Temperature
  if (temperature > 37.5) {
    sendSMS("Warning: High temperature!");
  }

  // Update OLED every second
  if (millis() - lastDisplayUpdate >= 1000) {
    displayData(temperature, heartRate, spO2, fallDetected);
    lastDisplayUpdate = millis();
  }

  // Update Google Sheets and Blynk every 5 seconds
  if (millis() - lastBlynkUpdate >= 5000) {
    sendToGoogleSheets(temperature, heartRate, spO2, fallDetected);
    Blynk.virtualWrite(V0, temperature);
    Blynk.virtualWrite(V1, heartRate);
    Blynk.virtualWrite(V2, spO2);
    Blynk.virtualWrite(V3, fallDetected ? 1 : 0);
    lastBlynkUpdate = millis();
  }
}

void update() {
  pox.update();
}

void onBeatDetected() {
  Serial.println("Beat detected!");
}

bool detectFall() {
  int16_t ax, ay, az;
  mpu.getAcceleration(&ax, &ay, &az);
  float acceleration = sqrt(ax * ax + ay * ay + az * az) / 16384.0;
  return acceleration < 0.7; // Fall threshold
}

void sendSMS(String message) {
  simSerial.println("AT+CMGF=1");
  delay(100);
  simSerial.println("AT+CMGS=\"" SIM_PHONE_NUMBER "\"");
  delay(100);
  simSerial.print(message);
  simSerial.write(26);
  delay(1000);
}

void sendToGoogleSheets(float temp, float hr, float spo2, bool fall) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    String url = "https://script.google.com/macros/s/" + GOOGLE_SCRIPT_ID +
                 "/exec?temp=" + String(temp) +
                 "&hr=" + String(hr) +
                 "&spo2=" + String(spo2) +
                 "&fall=" + String(fall ? "1" : "0");

    http.begin(url);
    int httpResponseCode = http.GET();
    if (httpResponseCode > 0) {
      String response = http.getString();
      Serial.println("Google Sheets Response: " + response);
    } else {
      Serial.println("Error sending data: " + String(httpResponseCode));
    }
    http.end();
  } else {
    Serial.println("WiFi not connected");
  }
}

void displayMessage(String line1, String line2) {
  display.clearDisplay();
  display.setCursor(20, 10);
  display.println(line1);
  display.setCursor(0, 30);
  display.println(line2);
  display.display();
}

void displayData(float temp, float hr, float spO2, bool fall) {
  display.clearDisplay();
  display.setCursor(0, 0);
  display.print("Temp: ");
  display.print(temp, 1);
  display.println(" C");
  display.setCursor(0, 10);
  display.print("HR: ");
  display.print(hr, 0);
  display.println(" bpm");
  display.setCursor(0, 20);
  display.print("SpO2: ");
  display.print(spO2, 0);
  display.println(" %");
  display.setCursor(0, 20);
  display.print("           Fall: ");
  display.println(fall ? "Yes" : "No");
  display.display();
}

