#include <SPI.h>
#include <MFRC522.h>
#include <Servo.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// RFID
#define SS_PIN 9
#define RST_PIN 10
MFRC522 mfrc522(SS_PIN, RST_PIN);
byte allowedUID[4] = {0x73, 0xDF, 0x91, 0xE2}; // Ganti sesuai UID kartu Anda

// Servo
Servo servoMasuk;
Servo servoKeluar;
const int SERVO_MASUK_PIN = 3;
const int SERVO_KELUAR_PIN = 5;

// Sensor & tombol
const int SENSOR_IR_MASUK = 2;
const int SENSOR_IR_KELUAR = 4;
const int SWITCH_KELUAR = A0;

// Slot kiri (HIGH = kosong)
const int SLOT_L1 = 6;
const int SLOT_L2 = 7;
const int SLOT_L3 = 8;

// Slot kanan (LOW = kosong)
const int SLOT_R1 = A1;
const int SLOT_R2 = A2;
const int SLOT_R3 = A3;

// LCD
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Timer kirim status slot
unsigned long lastSendTime = 0;
const unsigned long interval = 1000;  // 1 detik

// Variabel status slot sebelumnya
int prevL1 = -1, prevL2 = -1, prevL3 = -1;
int prevR1 = -1, prevR2 = -1, prevR3 = -1;

void setup() {
  Serial.begin(9600);
  SPI.begin();
  mfrc522.PCD_Init();

  lcd.init();
  lcd.backlight();
  tampilkanSiapScan();

  servoMasuk.attach(SERVO_MASUK_PIN);
  servoKeluar.attach(SERVO_KELUAR_PIN);
  servoMasuk.write(0);
  servoKeluar.write(0);

  pinMode(SENSOR_IR_MASUK, INPUT);
  pinMode(SENSOR_IR_KELUAR, INPUT);
  pinMode(SWITCH_KELUAR, INPUT);

  pinMode(SLOT_L1, INPUT);
  pinMode(SLOT_L2, INPUT);
  pinMode(SLOT_L3, INPUT);
  pinMode(SLOT_R1, INPUT);
  pinMode(SLOT_R2, INPUT);
  pinMode(SLOT_R3, INPUT);
}

void loop() {
  if (millis() - lastSendTime >= interval) {
    kirimStatusSlot();
    lastSendTime = millis();
  }

  // MODE KELUAR
  if (digitalRead(SWITCH_KELUAR) == LOW) {
    lcd.clear();
    lcd.setCursor(2, 0); lcd.print("Akses Keluar");
    lcd.setCursor(0, 1); lcd.print("Tab Access Card");

    while (digitalRead(SWITCH_KELUAR) == LOW) {
      if (millis() - lastSendTime >= interval) {
        kirimStatusSlot();
        lastSendTime = millis();
      }

      if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
        if (isAllowedUID(mfrc522.uid.uidByte)) {
          mfrc522.PICC_HaltA();
          tampilkanSelamatJalan();
          prosesKeluar();
          break;
        } else {
          lcd.clear();
          lcd.setCursor(3, 0); lcd.print("Kartu Tidak");
          lcd.setCursor(2, 1); lcd.print("Dikenal");
          delayLCD(1000);
          lcd.setCursor(2, 0); lcd.print("Akses Keluar");
          lcd.setCursor(0, 1); lcd.print("Tab Access Card");
        }
      }
    }

    if (digitalRead(SWITCH_KELUAR) == HIGH) {
      tampilkanSiapScan();
    }
    return;
  }

  // Mode masuk - hanya via RFID
  if (!mfrc522.PICC_IsNewCardPresent() || !mfrc522.PICC_ReadCardSerial()) return;

  if (isAllowedUID(mfrc522.uid.uidByte)) {
    bukaPintuMasuk();
    tampilkanSlotKosong();
    tungguKendaraanMasuk();
    tutupPintuMasuk();
  } else {
    lcd.clear();
    lcd.setCursor(0, 0); lcd.print("Akses Ditolak");
    lcd.setCursor(0, 1); lcd.print("Coba lagi");
    delayLCD(1000);
  }

  mfrc522.PICC_HaltA();
  tampilkanSiapScan();
}

// --- Fungsi Cek UID ---
bool isAllowedUID(byte *uid) {
  for (byte i = 0; i < 4; i++) {
    if (uid[i] != allowedUID[i]) return false;
  }
  return true;
}

// --- Tampilkan LCD ---
void tampilkanSiapScan() {
  lcd.clear();
  lcd.setCursor(1, 0); lcd.print("SMART PARKING");
  lcd.setCursor(0, 1); lcd.print("Tab Access Card");
}

void tampilkanSlotKosong() {
  bool L1 = digitalRead(SLOT_L1) == HIGH;
  bool L2 = digitalRead(SLOT_L2) == HIGH;
  bool L3 = digitalRead(SLOT_L3) == HIGH;
  bool R1 = digitalRead(SLOT_R1) == LOW;
  bool R2 = digitalRead(SLOT_R2) == LOW;
  bool R3 = digitalRead(SLOT_R3) == LOW;

  lcd.clear();
  lcd.setCursor(2, 0);
  lcd.print("L:");
  lcd.print(R1 ? "1" : "X");
  lcd.print(R2 ? "2" : "X");
  lcd.print(R3 ? "3" : "X");
  lcd.print(" R:");
  lcd.print(L1 ? "1" : "X");
  lcd.print(L2 ? "2" : "X");
  lcd.print(L3 ? "3" : "X");

  lcd.setCursor(1, 1); lcd.print("Silakan Masuk");
  delayLCD(1000);
}

void tampilkanSelamatJalan() {
  lcd.clear();
  lcd.setCursor(1, 0); lcd.print("Selamat Jalan");
  lcd.setCursor(3, 1); lcd.print("Hati-hati");
  delayLCD(1000);
}

// --- Kontrol Pintu ---
void bukaPintuMasuk() {
  servoMasuk.write(90);
}
void tutupPintuMasuk() {
  servoMasuk.write(0);
}
void tungguKendaraanMasuk() {
  while (digitalRead(SENSOR_IR_MASUK) == HIGH) {
    delay(10);
    kirimStatusSlot();
  }
  lcd.setCursor(0, 1); lcd.print("Kendaraan Masuk");
  while (digitalRead(SENSOR_IR_MASUK) == LOW) {
    delay(10);
    kirimStatusSlot();
  }
}

void prosesKeluar() {
  bukaPintuKeluar();
  tungguKendaraanKeluar();
  tutupPintuKeluar();
  tampilkanSiapScan();
}
void bukaPintuKeluar() {
  servoKeluar.write(90);
}
void tutupPintuKeluar() {
  servoKeluar.write(0);
}
void tungguKendaraanKeluar() {
  while (digitalRead(SENSOR_IR_KELUAR) == HIGH) {
    delay(10);
    kirimStatusSlot();
  }
  lcd.setCursor(0, 1); lcd.print("Kendaraan Keluar");
  while (digitalRead(SENSOR_IR_KELUAR) == LOW) {
    delay(10);
    kirimStatusSlot();
  }
}

// --- Delay non-blokir untuk LCD + Serial ---
void delayLCD(unsigned long durasi) {
  unsigned long start = millis();
  while (millis() - start < durasi) {
    if (millis() - lastSendTime >= interval) {
      kirimStatusSlot();
      lastSendTime = millis();
    }
  }
}

// --- Kirim Status Slot ke Serial dalam Format JSON ---
void kirimStatusSlot() {
  int L1 = digitalRead(SLOT_L1) == LOW ? 1 : 0;
  int L2 = digitalRead(SLOT_L2) == LOW ? 1 : 0;
  int L3 = digitalRead(SLOT_L3) == LOW ? 1 : 0;
  int R1 = digitalRead(SLOT_R1) != LOW;
  int R2 = digitalRead(SLOT_R2) != LOW;
  int R3 = digitalRead(SLOT_R3) != LOW;

  // Kirim hanya jika berubah atau dipanggil periodik
  if (L1 != prevL1 || L2 != prevL2 || L3 != prevL3 ||
      R1 != prevR1 || R2 != prevR2 || R3 != prevR3) {

    Serial.print("{\"L1\":"); Serial.print(L1);
    Serial.print(",\"L2\":"); Serial.print(L2);
    Serial.print(",\"L3\":"); Serial.print(L3);
    Serial.print(",\"R1\":"); Serial.print(R1);
    Serial.print(",\"R2\":"); Serial.print(R2);
    Serial.print(",\"R3\":"); Serial.print(R3);
    Serial.println("}");

    prevL1 = L1; prevL2 = L2; prevL3 = L3;
    prevR1 = R1; prevR2 = R2; prevR3 = R3;
  }
}
