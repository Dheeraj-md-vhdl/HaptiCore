// BIOHAPTIK - Distance + Sound Reactive Haptic Feedback
// 2 Motors: Motor 1 (Ultrasonic) + Motor 2 (Microphone)
// DEMO MODE - Optimized for 1-hour testing

#include <math.h>

// ===== PIN CONFIGURATION =====
const int MOTOR_US = 13;       // Motor for Ultrasonic (GPIO13)
const int MOTOR_MIC = 12;      // Motor for Microphone (GPIO12)
const int HC_TRIG = 5;         // Ultrasonic trigger
const int HC_ECHO = 18;        // Ultrasonic echo
const int MIC_PIN = 34;        // Analog microphone

// ===== CALIBRATION =====
const int MIN_DISTANCE = 5;    // cm (closest object)
const int MAX_DISTANCE = 100;  // cm (no object)
const int MIC_THRESHOLD = 1500; // Noise floor
const int MIC_MAX = 4095;      // ADC max

// ===== MODE VARIABLES =====
int mode = 1;                  // 1=Both, 2=US only, 3=Mic only
unsigned long lastRead = 0;
int currentDistance = 0;
int currentMic = 0;

void setup() {
  Serial.begin(115200);
  delay(2000);
  Serial.flush();
  
  Serial.println("\n\n╔════════════════════════════════╗");
  Serial.println("║     BIOHAPTIK v2.0 - DEMO      ║");
  Serial.println("║  Motor 1: Ultrasonic Distance  ║");
  Serial.println("║  Motor 2: Microphone Sound     ║");
  Serial.println("╚════════════════════════════════╝\n");

  // Motor relay pins - FORCE LOW
  pinMode(MOTOR_US, OUTPUT);
  pinMode(MOTOR_MIC, OUTPUT);
  digitalWrite(MOTOR_US, LOW);
  digitalWrite(MOTOR_MIC, LOW);
  delay(500);

  // Ultrasonic
  pinMode(HC_TRIG, OUTPUT);
  pinMode(HC_ECHO, INPUT);
  digitalWrite(HC_TRIG, LOW);

  Serial.println("✓ Motor US (GPIO13) initialized");
  Serial.println("✓ Motor MIC (GPIO12) initialized");
  Serial.println("✓ Ultrasonic sensor ready");
  Serial.println("✓ Microphone ready\n");
  
  Serial.println("Commands: 1-Both  2-US Only  3-Mic Only  4-Test  ?-Menu\n");
  Serial.println("Waiting for input...\n");
}

void printMenu() {
  Serial.println("\n╔════ MENU ════╗");
  Serial.println("1 = BOTH MODES (Ultrasonic + Microphone)");
  Serial.println("2 = ULTRASONIC ONLY (Motor vibrates as objects approach)");
  Serial.println("3 = MICROPHONE ONLY (Motor vibrates with sound)");
  Serial.println("4 = Motor Test (both motors pulse)");
  Serial.println("? = Show Menu");
  Serial.println("╚═══════════════╝\n");
}

// ===== SENSOR READS =====
int readDistance() {
  digitalWrite(HC_TRIG, LOW);
  delayMicroseconds(2);
  digitalWrite(HC_TRIG, HIGH);
  delayMicroseconds(10);
  digitalWrite(HC_TRIG, LOW);

  long duration = pulseIn(HC_ECHO, HIGH, 30000);
  if (duration == 0) return MAX_DISTANCE;
  int dist = duration / 58;
  return constrain(dist, MIN_DISTANCE, MAX_DISTANCE);
}

int readMicrophone() {
  int sum = 0;
  for (int i = 0; i < 4; i++) {
    sum += analogRead(MIC_PIN);
  }
  int avg = sum / 4;
  return constrain(avg - MIC_THRESHOLD, 0, MIC_MAX - MIC_THRESHOLD);
}

// ===== MOTOR CONTROL =====
void motorUSOn() {
  digitalWrite(MOTOR_US, HIGH);
}

void motorUSOff() {
  digitalWrite(MOTOR_US, LOW);
}

void motorMicOn() {
  digitalWrite(MOTOR_MIC, HIGH);
}

void motorMicOff() {
  digitalWrite(MOTOR_MIC, LOW);
}

void allMotorsOff() {
  digitalWrite(MOTOR_US, LOW);
  digitalWrite(MOTOR_MIC, LOW);
}

// ===== STATE CHECK =====
bool isMotorUSOn() {
  return digitalRead(MOTOR_US) == HIGH;
}

bool isMotorMicOn() {
  return digitalRead(MOTOR_MIC) == HIGH;
}

// ===== HAPTIC FEEDBACK LOGIC =====
void bothModes() {
  currentDistance = readDistance();
  currentMic = readMicrophone();

  // Ultrasonic Motor: closer = vibrate more
  if (currentDistance < 20) {
    motorUSOn();  // Very close
  } else if (currentDistance < 50) {
    motorUSOn();  // Medium distance - still vibrate
  } else {
    motorUSOff();   // Far away - no vibration
  }

  // Microphone Motor: louder = vibrate
  if (currentMic > 500) {
    motorMicOn();  // Loud sound
  } else {
    motorMicOff(); // Quiet
  }

  if (millis() - lastRead > 500) {
    Serial.print("DIST: ");
    Serial.print(currentDistance);
    Serial.print("cm ");
    Serial.print(isMotorUSOn() ? "●" : "○");
    Serial.print(" | MIC: ");
    Serial.print(currentMic);
    Serial.print(" ");
    Serial.println(isMotorMicOn() ? "●" : "○");
    lastRead = millis();
  }
}

void ultrasonicOnly() {
  currentDistance = readDistance();

  // Gradient vibration: closer = stronger response
  if (currentDistance < 15) {
    motorUSOn();  // Very close - strong vibration
  } else if (currentDistance < 40) {
    motorUSOn();  // Medium - still vibrating
  } else {
    motorUSOff();   // Far - off
  }

  motorMicOff();  // Keep mic motor off

  if (millis() - lastRead > 500) {
    Serial.print("Ultrasonic: ");
    Serial.print(currentDistance);
    Serial.print(" cm → Motor: ");
    Serial.println(isMotorUSOn() ? "ON ●" : "OFF ○");
    lastRead = millis();
  }
}

void microphoneOnly() {
  currentMic = readMicrophone();

  // Sound-reactive: louder = vibrate
  if (currentMic > 800) {
    motorMicOn();  // Very loud
  } else if (currentMic > 300) {
    motorMicOn();  // Medium sound
  } else {
    motorMicOff(); // Quiet
  }

  motorUSOff();  // Keep US motor off

  if (millis() - lastRead > 500) {
    Serial.print("Microphone: ");
    Serial.print(currentMic);
    Serial.print(" → Motor: ");
    Serial.println(isMotorMicOn() ? "ON ●" : "OFF ○");
    lastRead = millis();
  }
}

// ===== MOTOR TEST =====
void motorTest() {
  Serial.println("\n╔═══ MOTOR TEST ═══╗");
  Serial.println("Motor US (GPIO13):");
  for (int i = 0; i < 5; i++) {
    motorUSOn();
    Serial.print("●");
    delay(200);
    motorUSOff();
    Serial.print("○");
    delay(200);
  }
  Serial.println();

  delay(500);

  Serial.println("Motor MIC (GPIO12):");
  for (int i = 0; i < 5; i++) {
    motorMicOn();
    Serial.print("●");
    delay(200);
    motorMicOff();
    Serial.print("○");
    delay(200);
  }
  Serial.println("\nBoth motors ON:");
  motorUSOn();
  motorMicOn();
  delay(1000);
  allMotorsOff();
  
  Serial.println("Test complete. ✓\n");
}

// ===== MAIN LOOP =====
void loop() {
  // Check for serial commands
  while (Serial.available()) {
    char cmd = Serial.read();
    
    // Skip whitespace
    if (cmd == '\n' || cmd == '\r' || cmd == ' ') {
      continue;
    }

    Serial.print("\n→ Command: ");
    Serial.println(cmd);

    switch (cmd) {
      case '1':
        mode = 1;
        allMotorsOff();
        Serial.println("BOTH MODES: Ultrasonic + Microphone active\n");
        break;
      case '2':
        mode = 2;
        allMotorsOff();
        Serial.println("ULTRASONIC ONLY: Move hand closer/farther\n");
        break;
      case '3':
        mode = 3;
        allMotorsOff();
        Serial.println("MICROPHONE ONLY: Make sounds\n");
        break;
      case '4':
        motorTest();
        break;
      case '?':
        printMenu();
        break;
      default:
        Serial.println("Unknown. Press ? for menu.\n");
        break;
    }
  }

  // Main feedback loop
  switch (mode) {
    case 1: bothModes(); break;
    case 2: ultrasonicOnly(); break;
    case 3: microphoneOnly(); break;
  }

  delay(50);
}
