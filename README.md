# Safe-Cracker

/*
==============================================================
   SAFE CRACKER - 4 Haneli Dijital Kasa
   Arduino Uno | C++
   Mustafa Berkay Aydin  - 220202007
   Yusuf Kazimoglu       - 220202014
==============================================================
   PIN BAGLANTILARI:
   D2  → Encoder SAG bacak (A - Interrupt)
   D3  → Encoder SOL bacak (B - Yon)
   GND → Encoder ORTA bacak
   D4  → 220Ω → Kirmizi LED (+)
   D5  → 220Ω → Yesil LED  (+)
   D6  → 100Ω → Buzzer     (+)
   D7  → 1kΩ  → TIP122 Baz → Solenoid
==============================================================*/

// ── Pin tanimlari ──────────────────────────────────────────────
#define ENCODER_A   2
#define ENCODER_B   3
#define RED_LED     4
#define GREEN_LED   5
#define BUZZER      6
#define SOLENOID    7

// ── Sifre ve ayarlar ───────────────────────────────────────────
const int SIFRE[4]    = {20, 30, 40, 50};
#define ESIK            3      // Dogru bolgesi: +- 3 pulse
#define ONAY_SURESI     1500   // Onay icin bekleme suresi (ms)
#define SOLENOID_SURE   3000   // Solenoid acik kalma suresi (ms)

// ── State machine ──────────────────────────────────────────────
enum Durum { BEKLE, ONAYLIYOR, ONAYLANDI, KAZANDI };
Durum durum = BEKLE;

// ── Degiskenler ────────────────────────────────────────────────
volatile int encoderRaw    = 0;
int          mevcutHane    = 0;
unsigned long onayBaslangic = 0;

// ── ISR: Encoder her donusunde cagrilir ────────────────────────
void encoderISR() {
    if (digitalRead(ENCODER_B) == HIGH) encoderRaw++;
    else                                encoderRaw--;
}

// ── Yardimci fonksiyonlar ──────────────────────────────────────
int enkOku() {
    noInterrupts();
    int v = encoderRaw;
    interrupts();
    return v;
}

void enkSifirla() {
    noInterrupts();
    encoderRaw = 0;
    interrupts();
}

void ledAyarla(bool kirmizi, bool yesil) {
    digitalWrite(RED_LED,   kirmizi);
    digitalWrite(GREEN_LED, yesil);
}

// ── Ses fonksiyonlari ──────────────────────────────────────────
void sesBasari() {
    tone(BUZZER, 1000, 100); delay(130);
    tone(BUZZER, 2000, 200); delay(250);
    noTone(BUZZER);
}

void sesKazan() {
    for (int f = 800; f <= 3600; f += 400) {
        tone(BUZZER, f, 80);
        delay(100);
    }
    noTone(BUZZER);
}

// ── Animasyonlar ───────────────────────────────────────────────
void animBasari() {
    sesBasari();
    for (int i = 0; i < 4; i++) {
        ledAyarla(false, true);  delay(180);
        ledAyarla(false, false); delay(120);
    }
}

void animKazan() {
    sesKazan();
    for (int i = 0; i < 10; i++) {
        ledAyarla(false, true);  delay(120);
        ledAyarla(false, false); delay(80);
    }
}
