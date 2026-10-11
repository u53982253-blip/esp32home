// 4x3 keypad toggle controller for the four relay channels.
// Key 1 toggles relay/bulb 1, key 2 toggles relay/bulb 2, etc.
// A second press of the same key turns that channel back off.

const uint8_t rowPins[4] = {21, 22, 23, 19}; // L1..L4
const uint8_t colPins[3] = {18, 5, 4};       // C1..C3
const uint8_t relayPins[4] = {13, 12, 14, 27}; // IN1..IN4

// Standard 4x3 telephone keypad layout, rows L1 to L4.
const char keyMap[4][3] = {
  {'1', '2', '3'},
  {'4', '5', '6'},
  {'7', '8', '9'},
  {'*', '0', '#'}
};

// Typical 5 V relay modules are active-low: HIGH releases the relay.
// Change this to false only if the particular relay board is active-high.
const bool RELAY_ACTIVE_LOW = true;
bool relayOn[4] = {false, false, false, false};

char scanKeypad() {
  for (uint8_t r = 0; r < 4; r++) {
    // Keep all rows idle high, then pull one row low for scanning.
    for (uint8_t i = 0; i < 4; i++) digitalWrite(rowPins[i], HIGH);
    digitalWrite(rowPins[r], LOW);
    delayMicroseconds(5);

    for (uint8_t c = 0; c < 3; c++) {
      if (digitalRead(colPins[c]) == LOW) {
        return keyMap[r][c];
      }
    }
  }
  return '\0';
}

void setRelay(uint8_t channel, bool on) {
  relayOn[channel] = on;
  bool energizedLevel = RELAY_ACTIVE_LOW ? LOW : HIGH;
  digitalWrite(relayPins[channel], on ? energizedLevel : !energizedLevel);
}

void setup() {
  // Set inactive output levels before enabling outputs, so channels start off.
  const uint8_t offLevel = RELAY_ACTIVE_LOW ? HIGH : LOW;
  for (uint8_t i = 0; i < 4; i++) {
    digitalWrite(relayPins[i], offLevel);
    pinMode(relayPins[i], OUTPUT);
  }

  for (uint8_t r = 0; r < 4; r++) {
    pinMode(rowPins[r], OUTPUT);
    digitalWrite(rowPins[r], HIGH);
  }
  for (uint8_t c = 0; c < 3; c++) {
    pinMode(colPins[c], INPUT_PULLUP);
  }
}

void loop() {
  static char lastHandledKey = '\0';
  static unsigned long keyStartMs = 0;
  char key = scanKeypad();

  if (key == '\0') {
    lastHandledKey = '\0';
    return;
  }

  // Require a stable press, and handle it only once until the key is released.
  if (key != lastHandledKey) {
    lastHandledKey = key;
    keyStartMs = millis();
    return;
  }
  if (millis() - keyStartMs < 30) return;

  if (key >= '1' && key <= '4') {
    uint8_t channel = (uint8_t)(key - '1');
    setRelay(channel, !relayOn[channel]);
  }

  // Wait for release before accepting another press (including a held key).
  while (scanKeypad() != '\0') delay(10);
  lastHandledKey = '\0';
}
