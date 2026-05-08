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
// ── SETUP ──────────────────────────────────────────────────────
void setup() {
    pinMode(ENCODER_A, INPUT_PULLUP);
    pinMode(ENCODER_B, INPUT_PULLUP);
    pinMode(RED_LED,   OUTPUT);
    pinMode(GREEN_LED, OUTPUT);
    pinMode(BUZZER,    OUTPUT);
    pinMode(SOLENOID,  OUTPUT);

    ledAyarla(true, false);
    digitalWrite(SOLENOID, LOW);

    attachInterrupt(digitalPinToInterrupt(ENCODER_A), encoderISR, CHANGE);

    Serial.begin(9600);
    Serial.println("=====================================");
    Serial.println("        SAFE CRACKER BASLADI        ");
    Serial.println("=====================================");
    Serial.println("Hane 1 | Hedef: 20");
    Serial.println("-------------------------------------");
}

// ── LOOP ───────────────────────────────────────────────────────
void loop() {

    int enc  = enkOku();
    int fark = abs(enc - SIFRE[mevcutHane]);

    // ──────────────────────────────────────────────────────────
    // BEKLE: Encoder cevrilmeyi bekliyor
    // ──────────────────────────────────────────────────────────
    if (durum == BEKLE) {

        ledAyarla(true, false);

        Serial.print("Hane:");    Serial.print(mevcutHane + 1);
        Serial.print(" Enc:");    Serial.print(enc);
        Serial.print(" Hedef:");  Serial.print(SIFRE[mevcutHane]);
        Serial.print(" Fark:");   Serial.println(fark);

        if (fark <= ESIK) {
            durum         = ONAYLIYOR;
            onayBaslangic = millis();
            Serial.println(">>> DOGRU BOLGE! 1.5 sn bekle...");
        }
    }

    // ──────────────────────────────────────────────────────────
    // ONAYLIYOR: Dogru bolgede, 1.5 sn sayiyor
    // ──────────────────────────────────────────────────────────
    else if (durum == ONAYLIYOR) {

        unsigned long gecen = millis() - onayBaslangic;

        // Yesil LED yanip soner
        ledAyarla(false, (millis() / 200) % 2);

        Serial.print("Onaylaniyor: ");
        Serial.print(gecen / 100);
        Serial.print("/15  Fark: ");
        Serial.println(fark);

        // Bolgeden ciktiysa geri don
        if (fark > ESIK + 3) {
            durum = BEKLE;
            ledAyarla(true, false);
            Serial.println(">>> Kaymadi! Tekrar dene.");
            return;
        }

        // 1.5 sn doldu → onayla
        if (gecen >= ONAY_SURESI) {
            durum = ONAYLANDI;
        }
    }

    // ──────────────────────────────────────────────────────────
    // ONAYLANDI: Hane dogru, gecise hazirla
    // ──────────────────────────────────────────────────────────
    else if (durum == ONAYLANDI) {

        Serial.print(">>> Hane "); Serial.print(mevcutHane + 1);
        Serial.println(" ONAYLANDI!");

        if (mevcutHane < 3) {

            animBasari();
            mevcutHane++;
            enkSifirla();
            ledAyarla(true, false);
            durum = BEKLE;

            Serial.println("-------------------------------------");
            Serial.print("Hane "); Serial.print(mevcutHane + 1);
            Serial.print(" | Hedef: "); Serial.println(SIFRE[mevcutHane]);
            Serial.println("-------------------------------------");

        } else {
            durum = KAZANDI;
        }
    }

    // ──────────────────────────────────────────────────────────
    // KAZANDI: Tum haneler dogru, kasa aciliyor
    // ──────────────────────────────────────────────────────────
    else if (durum == KAZANDI) {

        Serial.println("=====================================");
        Serial.println("         KASA ACILIYOR!!!           ");
        Serial.println("=====================================");

        animKazan();

        digitalWrite(SOLENOID, HIGH);
        delay(SOLENOID_SURE);
        digitalWrite(SOLENOID, LOW);

        // Sifirla
        mevcutHane = 0;
        enkSifirla();
        ledAyarla(true, false);
        durum = BEKLE;

        Serial.println("=== Oyun sifirlandi ===");
        Serial.println("Hane 1 | Hedef: 20");
        Serial.println("-------------------------------------");
    }

    delay(100);
}
