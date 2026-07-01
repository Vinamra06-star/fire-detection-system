# fire-detection-system
#include <SoftwareSerial.h>

// ---------- Pin definitions ----------
const int FLAME_PIN   = 2;   // Digital output of flame sensor
const int MQ2_PIN     = A0;  // Analog output of MQ2 gas sensor
const int BUZZER_PIN  = 3;   // Buzzer
const int GSM_RX_PIN  = 7;   // Arduino RX  <- GSM TX
const int GSM_TX_PIN  = 8;   // Arduino TX  -> GSM RX

SoftwareSerial gsm(GSM_RX_PIN, GSM_TX_PIN);

// ---------- Settings ----------
const int GAS_THRESHOLD = 43;   // Adjust after testing (0–1023)
const unsigned long SMS_INTERVAL = 60000UL; // 1 min between SMS if alarm continues

// Replace with your phone number (with country code)
String phoneNumber = "+919956657919";

bool alarmActive = false;
unsigned long lastSmsTime = 0;

void setup() {
  pinMode(FLAME_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);

  Serial.begin(9600);
  gsm.begin(9600);

  Serial.println("System starting...");
  initGSM();
}

void loop() {
  bool flameDetected = (digitalRead(FLAME_PIN) == LOW); 
  // Many flame sensors give LOW when fire is detected. 
  // If yours is opposite, change to == HIGH.

  int gasValue = analogRead(MQ2_PIN);
  bool gasDetected = (gasValue > GAS_THRESHOLD);

  Serial.print("Gas: ");
  Serial.print(gasValue);
  Serial.print("  Flame: ");
  Serial.println(flameDetected ? "YES" : "NO");

  if (flameDetected || gasDetected) {
    // Turn ON alarm
    digitalWrite(BUZZER_PIN, HIGH);
    if (!alarmActive || (millis() - lastSmsTime > SMS_INTERVAL)) {
      sendAlarmSMS(flameDetected, gasDetected, gasValue);
      alarmActive = true;
      lastSmsTime = millis();
    }
  } else {
    // No danger
    digitalWrite(BUZZER_PIN, LOW);
    alarmActive = false;
  }

  delay(500);  // Small delay for stability
}

// ---------- GSM helper functions ----------

void initGSM() {
  delay(1000);
  sendCommand("AT");          // Basic check
  sendCommand("AT+CMGF=1");   // SMS text mode
  sendCommand("AT+CSCS=\"GSM\""); // Character set
  Serial.println("GSM initialised.");
}

void sendAlarmSMS(bool flame, bool gas, int gasVal) {
  Serial.println("Sending SMS...");

  String msg = "ALERT! ";
  if (flame) msg += "Flame detected. ";
  if (gas)   msg += "Gas detected (MQ2 = " + String(gasVal) + ").";

  gsm.print("AT+CMGS=\"");
  gsm.print(phoneNumber);
  gsm.println("\"");
  delay(500);                 // Wait for '>' prompt

  gsm.println(msg);           // Message body
  gsm.write(26);              // Ctrl+Z to send
  delay(5000);                // Wait for send

  Serial.println("SMS sent: " + msg);
}

void sendCommand(const char* cmd) {
  gsm.println(cmd);
  delay(500);
  while (gsm.available()) {
    Serial.write(gsm.read()); // Show GSM response in Serial Monitor
  }
}
