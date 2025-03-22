#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <RTClib.h>
#include <SPI.h>
#include <SD.h>

// Definisi LCD I2C
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Definisi motor
#define STEP_PIN 4   // Pin STEP untuk TB6600
#define DIR_PIN 5    // Pin DIR untuk arah putaran
#define ENABLE_PIN 6 // Pin ENABLE (opsional)

// Definisi sensor arus dan tegangan
#define CURRENT_SENSOR_PIN A0  // Pin untuk ACS712-30A
#define VOLTAGE_SENSOR_PIN A1  // Pin untuk voltage divider

// Definisi SD card
#define CS_PIN 10

const int pinVol = A0;  // Pin sensor tegangan
const int pinAmp = A1;  // Pin sensor arus
const int chipSelect = 10; // Pin CS untuk SD Card

// Variabel untuk menyimpan hasil pembacaan
float tegangan = 0.0;
float arus = 0.0;
const int sample = 100; // Jumlah sampel untuk averaging

const int stepsPerCycle = 10;       // 250 langkah per siklus
const int cycleTime = 100;          // Waktu satu siklus (ms)
const unsigned long delayBetweenCycles = 420000; // 7 menit dalam milidetik
const unsigned long dataLogInterval = 420000;  // Interval pencatatan data (7 menit)

unsigned long previousCycleTime = 0;
unsigned long previousMillis = 0;
unsigned long previousLogTime = 0;

// RTC
RTC_DS3231 rtc;

void setup() {
  Serial.begin(115200);
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Initializing...");

  // Inisialisasi RTC
  if (!rtc.begin()) {
    Serial.println("Couldn't find RTC");
    lcd.setCursor(0, 1);
    lcd.print("RTC ERROR");
    while (1);
  }

  if (rtc.lostPower()) {
    Serial.println("RTC lost power, let's set the time!");
    rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
  }

  // Inisialisasi SD card
  if (!SD.begin(CS_PIN)) {
    Serial.println("SD Card initialization failed!");
    return;
  }
  Serial.println("SD Card initialized.");

  // Definisi motor
  pinMode(STEP_PIN, OUTPUT);
  pinMode(DIR_PIN, OUTPUT);
  pinMode(ENABLE_PIN, OUTPUT);
  digitalWrite(ENABLE_PIN, LOW); // Aktifkan driver

  // Definisi sensor
  pinMode(CURRENT_SENSOR_PIN, INPUT);
  pinMode(VOLTAGE_SENSOR_PIN, INPUT);

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("System Ready");
}

// Fungsi membaca arus (ACS712 ELC-30A)
float bacaArus() {
  int sampleArus = 0;
  float ampout = 0;

  for (int i = 0; i < sample; i++) {
    sampleArus = analogRead(pinAmp);
    ampout += sampleArus;
  }

  ampout = ampout / sample;
  float arusADC = ampout - 513; // Offset ADC (karena ACS712 idle di 2.5V)

  float x = (arusADC * 26.378) / 1000.0; // 26.378 mV/A (sesuai datasheet)
  arus = (1.192 * x) - 0.02273; // Persamaan regresi kalibrasi

  if (arus < 0.02273) arus = 0;

  return arus;
}

// Fungsi membaca tegangan (Voltage Divider 0-25V)
float bacaTegangan() {
  int sampleTegangan = 0;
  float vout = 0;

  for (int i = 0; i < sample; i++) {
    sampleTegangan = analogRead(pinVol);
    vout += sampleTegangan;
  }

  vout = vout / sample;
  vout = vout * 5.0 / 1023.0; // Konversi ADC ke Volt (5V referensi, resolusi 10-bit)
  
  float x = vout * 5.0; // Kalibrasi rasio pembagi tegangan
  
  tegangan = (1.028 * x) + 0.1359; // Persamaan regresi kalibrasi

  if (tegangan <= 0.1359) tegangan = 0;

  return tegangan;
}

// Fungsi untuk menggerakkan motor 250 langkah dalam 500 ms
void moveStepper() {
  unsigned long startTime = millis();
  digitalWrite(DIR_PIN, HIGH);
  lcd.setCursor(0, 1);
  lcd.print("Motor Moving...");

  while (millis() - startTime < cycleTime) {
    for (int i = 0; i < stepsPerCycle; i++) {
      digitalWrite(STEP_PIN, HIGH);
      delayMicroseconds(100);
      digitalWrite(STEP_PIN, LOW);
      delayMicroseconds(100);
    }
  }
  lcd.setCursor(0, 1);
  lcd.print("Motor Stopped ");
}

void moveBackStepper() {
  unsigned long currentMillis = millis();
  digitalWrite(ENABLE_PIN, LOW);
  digitalWrite(DIR_PIN, LOW);
  lcd.setCursor(0, 1);
  lcd.print("Motor Bergerak");

  if (currentMillis - previousMillis >= 800) {
    previousMillis = currentMillis;
    for (int i = 0; i < 5; i++) {
      digitalWrite(STEP_PIN, HIGH);
      delayMicroseconds(100);
      digitalWrite(STEP_PIN, LOW);
      delayMicroseconds(100);
    }
  }
}

void motor() {
  unsigned long currentTime = millis();

  // Jika sudah waktunya bergerak (setiap 7 menit)
  if (currentTime - previousCycleTime >= delayBetweenCycles) {
    previousCycleTime = currentTime;
    moveStepper();  // Jalankan motor 250 langkah dalam 500 ms
  }
}



// Fungsi untuk menyimpan data ke SD card
void logData(float ampere, float volt) {
  DateTime now = rtc.now();
  File dataFile = SD.open("datalog.txt", FILE_WRITE);
  if (dataFile) {
    dataFile.print(now.hour());
    dataFile.print(":");
    dataFile.print(now.minute());
    dataFile.print(":");
    dataFile.print(now.second());
    dataFile.print(" , ");
    dataFile.print(ampere, 2);
    dataFile.print(" A , ");
    dataFile.print(volt, 2);
    dataFile.println(" V");
    dataFile.close();
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Data Tersimpan");
  } else {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("SD Write Error!");
  }
}

void displayTime(DateTime now) {
  lcd.setCursor(0, 0);
  lcd.print(now.hour()); lcd.print(":");
  lcd.print(now.minute()); lcd.print(":");
  lcd.print(now.second());
}

void loop() {
  DateTime now = rtc.now();
  displayTime(now);
  
  Serial.print(now.hour(), DEC);
  Serial.print(':');
  Serial.print(now.minute(), DEC);
  Serial.print(':');
  Serial.print(now.second(), DEC);
  Serial.println();

  float ampere = bacaArus();
  float volt = bacaTegangan();

  lcd.setCursor(0, 1);
  lcd.print(ampere, 2);
  lcd.print("A ");
  lcd.print(volt, 2);
  lcd.print("V");

  int startHour = 9, startMinute = 00;  // Waktu mulai motor berjalan
  int stopHour = 15, stopMinute = 00;    // Waktu berhenti motor

  int lockHour = 15, lockMinute = 01;  // Waktu motor dikunci
  int unlockHour = 16, unlockMinute = 50; // Waktu motor tidak terkunci lagi

  if (isTimeInRange(now.hour(), now.minute(), startHour, startMinute, stopHour, stopMinute)) {
    unsigned long currentLogTime = millis();
    if (currentLogTime - previousLogTime >= dataLogInterval) {
    previousLogTime = currentLogTime;
    logData(ampere, volt);
  }
    motor(); // Jalankan motor jika dalam rentang waktu
  }

  else if (isTimeInRange(now.hour(), now.minute(), lockHour, lockMinute, unlockHour, unlockMinute)) {
    lockMotor(); // Kunci motor di posisi terakhir
  } 
  else if (now.hour() == 16  && now.minute() == 51) {
    moveBackStepper();
  } 
  
  else if (now.hour() == 17 && now.minute() ==  10) {
    moveBackStepper();
  } 

  delay(1000);
}

// Fungsi untuk mengecek apakah waktu sekarang dalam rentang aktif
bool isTimeInRange(int currentHour, int currentMinute, int startHour, int startMinute, int stopHour, int stopMinute) {
  if ((currentHour > startHour || (currentHour == startHour && currentMinute >= startMinute)) &&
      (currentHour < stopHour || (currentHour == stopHour && currentMinute < stopMinute))) {
    return true; // Waktu sekarang berada dalam rentang
  }
  return false; // Waktu sekarang di luar rentang
}

// Fungsi untuk mengunci motor di posisi terakhir
void lockMotor() {
  Serial.println("Motor Locked");
  lcd.setCursor(0, 1);
  lcd.print("Motor Locked");
  digitalWrite(ENABLE_PIN, LOW);
}
