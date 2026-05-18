#include "HX711.h"
#include <AccelStepper.h>

// ================= CẤU HÌNH LOADCELL =================
const int LOADCELL_DOUT_PIN = 4;
const int LOADCELL_SCK_PIN = 5;
HX711 scale;
float heSoHieuChuan = 87899.8465985;
unsigned long thoiGianTruoc = 0;
const long khoangThoiGianDoc = 100;

// ================= CẤU HÌNH ĐỘNG CƠ =================
#define DIR_PIN 8
#define STEP_PIN 9
AccelStepper motor(1, STEP_PIN, DIR_PIN);

// ================= CẤU HÌNH THƯỚC QUANG =================
volatile long soXungThuocQuang = 0;
const float DO_PHAN_GIAI = 0.005;


// ================= TRẠNG THÁI =================
float gioiHanMm = -1.0;
String inputBuffer = "";

enum TrangThai { CHO, DANG_KEO, DANG_TRA_VE };
TrangThai trangThai = CHO;

void demXungThuocQuang() {
  if (digitalRead(2) == digitalRead(3)) soXungThuocQuang--;
  else soXungThuocQuang++;
}

void setup() {
  Serial.begin(115200);

  scale.begin(LOADCELL_DOUT_PIN, LOADCELL_SCK_PIN);
  scale.set_scale(heSoHieuChuan);
  scale.tare();

  pinMode(2, INPUT_PULLUP);
  pinMode(3, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(2), demXungThuocQuang, CHANGE);

  // QUAN TRỌNG: Tắt acceleration hoàn toàn
  motor.setMaxSpeed(500);
  motor.setAcceleration(0);   // <--- Không giảm tốc, đảo chiều ngay lập tức

  Serial.println("=== HE THONG SAN SANG ===");
  Serial.println("Nhap so mm -> chay toi roi tu dong tra ve 0");
  Serial.println("0 / t / T  -> Reset");
}

void loop() {
  float doGianDai_mm = abs(soXungThuocQuang) * DO_PHAN_GIAI;

  // ── NHẬN LỆNH SERIAL ──
  while (Serial.available() > 0) {
    char c = Serial.read();
    if (c == '\n' || c == '\r') {
      inputBuffer.trim();
      if (inputBuffer.length() > 0) {

        if (inputBuffer == "0" || inputBuffer == "t" || inputBuffer == "T") {
          motor.setSpeed(0);
          soXungThuocQuang = 0;
          scale.tare();
          trangThai = CHO;
          gioiHanMm = -1.0;
          Serial.println(">> Da reset.");

        } else {
          float giaTriNhap = inputBuffer.toFloat();
          if (giaTriNhap > 0) {
            gioiHanMm = giaTriNhap;
            trangThai = DANG_KEO;
            motor.setSpeed(500);      // Chiều dương = kéo ra
            Serial.print(">> Dang chay toi ");
            Serial.print(gioiHanMm, 2);
            Serial.println(" mm...");
          } else {
            Serial.println(">> Lenh khong hop le.");
          }
        }
      }
      inputBuffer = "";
    } else {
      inputBuffer += c;
    }
  }

  // ── ĐIỀU KHIỂN ĐỘNG CƠ ──
  switch (trangThai) {

    case DANG_KEO:
      if (doGianDai_mm >= gioiHanMm) {
        motor.setSpeed(-500);         // ĐẢO CHIỀU NGAY, không giảm tốc
        trangThai = DANG_TRA_VE;
        Serial.println(">> DAT GIOI HAN -> DANG TRA VE 0...");
      }
      motor.runSpeed();
      break;

    case DANG_TRA_VE:
      if (doGianDai_mm <= 0.05) {
        motor.setSpeed(0);            // DỪNG khi về gần 0
        soXungThuocQuang = 0;
        trangThai = CHO;
        Serial.println(">> DA VE 0. KET THUC.");
      }
      motor.runSpeed();
      break;

    case CHO:
      break;
  }

  // ── GỬI DỮ LIỆU LÊN LABVIEW ──
  unsigned long thoiGianHienTai = millis();
  if (thoiGianHienTai - thoiGianTruoc >= khoangThoiGianDoc) {
    thoiGianTruoc = thoiGianHienTai;
    if (scale.is_ready()) {
      float khoiLuong = scale.get_units(1);
      Serial.print(khoiLuong, 2);
      Serial.print(",");
      Serial.println(doGianDai_mm, 3);
    }
  }
}
