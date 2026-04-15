#include <WiFi.h>
#include <WebServer.h>
#include <Preferences.h>
#include <BleKeyboard.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_NeoPixel.h>
#include <ArduinoJson.h>
#include <Update.h>

// ------------------------------------------------------------
// OLED
// ------------------------------------------------------------
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// ------------------------------------------------------------
// BLE + Preferences + Web
// ------------------------------------------------------------
Preferences prefs;
BleKeyboard bleKeyboard("ESP32 MacroPad", "Custom", 100);
WebServer server(80);

// ------------------------------------------------------------
// TOUCH KEYS (9x TTP223)
// ------------------------------------------------------------
int keys[9] = {32, 33, 25, 26, 27, 14, 12, 13, 4};
bool lastState[9];

unsigned long lastPressTime[9] = {0};
unsigned long lastTapTime[9]   = {0};
const unsigned long LONG_PRESS_MS  = 600;
const unsigned long DOUBLE_TAP_MS  = 400;

// ------------------------------------------------------------
// ROTARY ENCODER
// ------------------------------------------------------------
#define CLK 18
#define DT 19
#define SW 5
int lastCLK;

// ------------------------------------------------------------
// MEDIA BUTTONS
// ------------------------------------------------------------
#define BTN_PREV 15
#define BTN_PLAY 2
#define BTN_NEXT 23
bool lastPrev = HIGH;
bool lastPlay = HIGH;
bool lastNext = HIGH;

// ------------------------------------------------------------
// PROFILE BUTTON
// ------------------------------------------------------------
#define BTN_PROFILE 17
int currentProfile = 0;
bool lastProfile = HIGH;

// ------------------------------------------------------------
// RGB (WS2812)
// ------------------------------------------------------------
#define LED_PIN 16
#define LED_COUNT 9
Adafruit_NeoPixel strip(LED_COUNT, LED_PIN, NEO_GRB + NEO_KHZ800);

// Per-key RGB colors for each profile
uint32_t keyColors[3][9] = {
  {0x00FFAA,0x00DDAA,0x00BBAA,0x0099AA,0x0077AA,0x0055AA,0x0033AA,0x0011AA,0x00FFCC},
  {0xAA00FF,0x8800DD,0x6600BB,0x440099,0x220077,0x110055,0x3300AA,0x5500CC,0x7700EE},
  {0xFFAA00,0xDD8800,0xBB6600,0x994400,0x773300,0x552200,0xFFCC33,0xFFDD66,0xFFEE99}
};

// RGB effects
enum RgbEffect {
  EFFECT_STATIC = 0,
  EFFECT_BREATHING,
  EFFECT_RAINBOW,
  EFFECT_REACTIVE
};
RgbEffect currentEffect = EFFECT_STATIC;
unsigned long lastRgbFrame = 0;
uint16_t rainbowOffset = 0;

// ------------------------------------------------------------
// BATTERY (optional)
// ------------------------------------------------------------
#define BATTERY_PIN 34
float batteryPercent = 0;
unsigned long lastBatteryRead = 0;
const unsigned long BATTERY_INTERVAL_MS = 5000;

// ------------------------------------------------------------
// SCREENSAVER
// ------------------------------------------------------------
unsigned long lastActivity = 0;
const unsigned long IDLE_TIMEOUT_MS = 120000;
bool screensaverOn = false;

// ------------------------------------------------------------
// MENU SYSTEM
// ------------------------------------------------------------
enum UiState {
  UI_NORMAL = 0,
  UI_MENU_MAIN
};
UiState uiState = UI_NORMAL;
int menuIndex = 0;

const char* mainMenuItems[] = {
  "Profiles",
  "RGB Effects",
  "Settings",
  "About"
};
const int mainMenuCount = 4;

// ------------------------------------------------------------
// PROFILE NAMES
// ------------------------------------------------------------
String profileNames[3] = {"Profile 1","Profile 2","Profile 3"};

// ------------------------------------------------------------
// HELPERS
// ------------------------------------------------------------
void updateBattery() {
  int raw = analogRead(BATTERY_PIN);
  float voltage = (raw / 4095.0f) * 3.3f;
  float cellV = voltage * (4.2f / 3.3f);

  float pct = (cellV - 3.3f) / (4.2f - 3.3f);
  if (pct < 0) pct = 0;
  if (pct > 1) pct = 1;
  batteryPercent = pct * 100.0f;
}

void setProfileColor() {
  for (int i = 0; i < LED_COUNT; i++)
    strip.setPixelColor(i, keyColors[currentProfile][i]);
  strip.show();
}

void flashKey(int index, uint32_t color, int ms = 80) {
  uint32_t old = strip.getPixelColor(index);
  strip.setPixelColor(index, color);
  strip.show();
  delay(ms);
  strip.setPixelColor(index, keyColors[currentProfile][index]);
  strip.show();
}

void profileFlash() {
  for (int i = 0; i < 3; i++) {
    strip.fill(0xFFFFFF);
    strip.show();
    delay(60);
    setProfileColor();
    delay(60);
  }
}

// ------------------------------------------------------------
// OLED PROFILE BAR
// ------------------------------------------------------------
void drawProfileIndicator() {
  display.fillRect(0, 0, 128, 16, SSD1306_BLACK);

  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 4);
  display.print("P");
  display.print(currentProfile + 1);
  display.print(": ");
  display.print(profileNames[currentProfile]);

  int bx = 96, by = 2;
  display.drawRect(bx, by, 20, 10, SSD1306_WHITE);
  display.fillRect(bx + 20, by + 3, 2, 4, SSD1306_WHITE);

  int fillWidth = (int)((batteryPercent / 100.0f) * 18);
  if (fillWidth < 0) fillWidth = 0;
  if (fillWidth > 18) fillWidth = 18;
  display.fillRect(bx + 1, by + 1, fillWidth, 8, SSD1306_WHITE);

  display.display();
}

void showText(const String &msg) {
  display.clearDisplay();
  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 20);
  display.println(msg);
  display.display();
  drawProfileIndicator();
}

// ------------------------------------------------------------
// MACRO EXECUTION (simple parser)
// Format: "Ctrl+C|Delay100|Text:Hello"
// ------------------------------------------------------------
void sendKeyCombo(const String &token) {
  String t = token;
  t.trim();
  if (t.equalsIgnoreCase("Ctrl+C")) {
    bleKeyboard.press(KEY_LEFT_CTRL);
    bleKeyboard.press('c');
    bleKeyboard.releaseAll();
  } else if (t.equalsIgnoreCase("Ctrl+V")) {
    bleKeyboard.press(KEY_LEFT_CTRL);
    bleKeyboard.press('v');
    bleKeyboard.releaseAll();
  } else if (t.equalsIgnoreCase("Ctrl+S")) {
    bleKeyboard.press(KEY_LEFT_CTRL);
    bleKeyboard.press('s');
    bleKeyboard.releaseAll();
  } else if (t.equalsIgnoreCase("VolumeUp")) {
    bleKeyboard.write(KEY_MEDIA_VOLUME_UP);
  } else if (t.equalsIgnoreCase("VolumeDown")) {
    bleKeyboard.write(KEY_MEDIA_VOLUME_DOWN);
  } else if (t.equalsIgnoreCase("PlayPause")) {
    bleKeyboard.write(KEY_MEDIA_PLAY_PAUSE);
  } else if (t.equalsIgnoreCase("Next")) {
    bleKeyboard.write(KEY_MEDIA_NEXT_TRACK);
  } else if (t.equalsIgnoreCase("Prev")) {
    bleKeyboard.write(KEY_MEDIA_PREVIOUS_TRACK);
  } else if (t.equalsIgnoreCase("Mute")) {
    bleKeyboard.write(KEY_MEDIA_MUTE);
  } else if (t.startsWith("Text:")) {
    bleKeyboard.print(t.substring(5));
  } else {
    bleKeyboard.print(t);
  }
}

void sendMacro(String macro) {
  macro.trim();
  if (!macro.length()) return;

  int start = 0;
  while (start < macro.length()) {
    int sep = macro.indexOf('|', start);
    String token;
    if (sep == -1) {
      token = macro.substring(start);
      start = macro.length();
    } else {
      token = macro.substring(start, sep);
      start = sep + 1;
    }
    token.trim();
    if (token.startsWith("Delay")) {
      int d = token.substring(5).toInt();
      if (d < 0) d = 0;
      if (d > 2000) d = 2000;
      delay(d);
    } else {
      sendKeyCombo(token);
    }
  }
}

// ------------------------------------------------------------
// ACTION SENDER
// ------------------------------------------------------------
void sendAction(String action) {
  sendMacro(action);
}

// ------------------------------------------------------------
// PREF KEY NAMES
// ------------------------------------------------------------
String keyPrefName(int keyIndex, const char* type, int profileOverride = -1) {
  int p = (profileOverride == -1 ? currentProfile : profileOverride);
  return "p" + String(p) + "_k" + String(keyIndex + 1) + "_" + String(type);
}

// ------------------------------------------------------------
// WEB UI HTML
// ------------------------------------------------------------
String htmlPage() {
  String page = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>ESP32 MacroPad</title>
<style>
  body { background:#111;color:#eee;font-family:Arial,sans-serif;margin:20px; }
  h1,h2 { color:#0ff; }
  .card { background:#222;padding:15px;margin-bottom:20px;border-radius:8px;border:1px solid #333; }
  select,button,input,textarea { padding:6px;margin:4px 0;background:#333;color:#eee;border:1px solid #555;border-radius:4px; }
  button { cursor:pointer; }
  .keybox { margin-bottom:10px;padding:10px;background:#1a1a1a;border-radius:6px;border:1px solid #333; }
  .row { display:flex;gap:10px;flex-wrap:wrap; }
  .col { flex:1 1 200px; }
</style>
</head>
<body>

<h1>ESP32 MacroPad</h1>

<div class="card">
  <h2>Device Status</h2>
  <p><b>Profile:</b> <span id="profile">...</span></p>
  <p><b>Profile Name:</b> <span id="profileName">...</span></p>
  <p><b>RGB Effect:</b> <span id="effect">...</span></p>
  <p><b>Battery:</b> <span id="battery">...</span>%</p>
</div>

<div class="card">
  <h2>Profile Management</h2>
  <label>Active Profile:
    <select id="profileSelect" onchange="changeProfile()">
      <option value="0">Profile 1</option>
      <option value="1">Profile 2</option>
      <option value="2">Profile 3</option>
    </select>
  </label><br>
  <label>Profile Name:
    <input id="profileNameInput">
  </label>
  <button onclick="saveProfileName()">Save Name</button>
</div>

<div class="card">
  <h2>RGB Effect & Per-Key Colors</h2>
  <div class="row">
    <div class="col">
      <label>Effect:
        <select id="rgbSelect" onchange="setRgb()">
          <option value="0">Static</option>
          <option value="1">Breathing</option>
          <option value="2">Rainbow</option>
          <option value="3">Reactive</option>
        </select>
      </label>
    </div>
    <div class="col">
      <h3>Per-Key Colors</h3>
      <div id="colorGrid"></div>
      <button onclick="saveColors()">Save Colors</button>
    </div>
  </div>
</div>

<div class="card">
  <h2>Key Configuration (Macros)</h2>
  <p>Use <code>|</code> to separate macro steps (e.g. <code>Ctrl+C|Delay100|Ctrl+V</code>).</p>
  <form id="keyForm" method="POST" action="/save">
    <div id="keys"></div>
    <button type="submit">Save All Actions</button>
  </form>
</div>

<div class="card">
  <h2>Profiles Import / Export</h2>
  <button onclick="exportProfiles()">Export Profiles</button><br>
  <textarea id="importText" rows="6" cols="60" placeholder="Paste exported JSON here"></textarea><br>
  <button onclick="importProfiles()">Import Profiles</button>
</div>

<div class="card">
  <h2>Firmware Update (OTA)</h2>
  <form method="POST" action="/update" enctype="multipart/form-data">
    <input type="file" name="firmware">
    <button type="submit">Upload & Flash</button>
  </form>
</div>

<script>
let currentProfile = 0;
let keyCount = 9;

function loadStatus() {
  fetch('/status')
    .then(r => r.json())
    .then(data => {
      currentProfile = data.profile;
      document.getElementById('profile').innerText = data.profile + 1;
      document.getElementById('effect').innerText = data.effect;
      document.getElementById('battery').innerText = data.battery.toFixed(0);

      document.getElementById('profileSelect').value = data.profile;
      document.getElementById('profileName').innerText = data.profileNames[data.profile] || '';
      document.getElementById('profileNameInput').value = data.profileNames[data.profile] || '';

      let keysDiv = document.getElementById('keys');
      keysDiv.innerHTML = "";
      data.keys.forEach(k => {
        keysDiv.innerHTML += `
          <div class="keybox">
            <h3>Key ${k.index + 1}</h3>
            Tap: <input name="p${data.profile}_k${k.index+1}_tap" value="${k.tap}"><br>
            Long: <input name="p${data.profile}_k${k.index+1}_long" value="${k.long}"><br>
            Double: <input name="p${data.profile}_k${k.index+1}_dbl" value="${k.dbl}"><br>
          </div>
        `;
      });

      let colorGrid = document.getElementById('colorGrid');
      colorGrid.innerHTML = "";
      data.keyColors[data.profile].forEach((c, idx) => {
        colorGrid.innerHTML += `
          Key ${idx+1}: <input type="color" id="color_${idx}" value="${c}"><br>
        `;
      });
    });
}

function setRgb() {
  let e = document.getElementById('rgbSelect').value;
  fetch('/rgb?effect=' + e).then(() => loadStatus());
}

function saveColors() {
  let params = [];
  for (let i = 0; i < keyCount; i++) {
    let v = document.getElementById('color_' + i).value;
    params.push(v);
  }
  fetch('/rgb', {
    method: 'POST',
    headers: {'Content-Type':'application/json'},
    body: JSON.stringify({profile: currentProfile, colors: params})
  }).then(() => loadStatus());
}

function changeProfile() {
  let p = document.getElementById('profileSelect').value;
  fetch('/rgb?effect=-1&profile=' + p).then(() => loadStatus());
}

function saveProfileName() {
  let name = document.getElementById('profileNameInput').value;
  fetch('/setProfileName', {
    method: 'POST',
    headers: {'Content-Type':'application/x-www-form-urlencoded'},
    body: 'profile=' + encodeURIComponent(currentProfile) + '&name=' + encodeURIComponent(name)
  }).then(() => loadStatus());
}

function exportProfiles() {
  fetch('/exportProfiles')
    .then(r => r.text())
    .then(t => {
      document.getElementById('importText').value = t;
    });
}

function importProfiles() {
  let txt = document.getElementById('importText').value;
  fetch('/importProfiles', {
    method: 'POST',
    headers: {'Content-Type':'application/json'},
    body: txt
  }).then(() => loadStatus());
}

loadStatus();
setInterval(loadStatus, 5000);
</script>

</body>
</html>
)rawliteral";
  return page;
}

// ------------------------------------------------------------
// /status JSON
// ------------------------------------------------------------
void handleStatus() {
  StaticJsonDocument<4096> doc;
  doc["profile"] = currentProfile;
  doc["effect"] = (int)currentEffect;
  doc["battery"] = batteryPercent;

  JsonArray names = doc.createNestedArray("profileNames");
  for (int p = 0; p < 3; p++) names.add(profileNames[p]);

  JsonArray keysArr = doc.createNestedArray("keys");
  for (int i = 0; i < 9; i++) {
    JsonObject k = keysArr.createNestedObject();
    k["index"] = i;
    k["tap"]   = prefs.getString(keyPrefName(i,"tap").c_str(), "");
    k["long"]  = prefs.getString(keyPrefName(i,"long").c_str(), "");
    k["dbl"]   = prefs.getString(keyPrefName(i,"dbl").c_str(), "");
  }

  JsonArray colors = doc.createNestedArray("keyColors");
  for (int p = 0; p < 3; p++) {
    JsonArray cArr = colors.createNestedArray();
    for (int k = 0; k < 9; k++) {
      char buf[8];
      sprintf(buf, "#%06X", keyColors[p][k] & 0xFFFFFF);
      cArr.add(String(buf));
    }
  }

  String out;
  serializeJson(doc, out);
  server.send(200, "application/json", out);
}

// ------------------------------------------------------------
// /rgb GET (effect + optional profile switch)
// ------------------------------------------------------------
void handleRgbGet() {
  if (server.hasArg("effect")) {
    int e = server.arg("effect").toInt();
    if (e >= 0 && e <= 3) currentEffect = (RgbEffect)e;
  }
  if (server.hasArg("profile")) {
    int p = server.arg("profile").toInt();
    if (p >= 0 && p < 3) {
      currentProfile = p;
      setProfileColor();
    }
  }
  server.send(200, "text/plain", "OK");
}

// ------------------------------------------------------------
// /rgb POST (per-key colors)
// ------------------------------------------------------------
void handleRgbPost() {
  StaticJsonDocument<1024> doc;
  DeserializationError err = deserializeJson(doc, server.arg((size_t)0));
  if (err) {
    server.send(400, "text/plain", "JSON error");
    return;
  }
  int p = doc["profile"] | 0;
  if (p < 0 || p > 2) p = 0;
  JsonArray cols = doc["colors"];
  for (int k = 0; k < 9 && k < cols.size(); k++) {
    String hex = (const char*)cols[k];
    long val = strtol(hex.c_str()+1, NULL, 16);
    keyColors[p][k] = (uint32_t)val;
  }
  setProfileColor();
  server.send(200, "text/plain", "OK");
}

// ------------------------------------------------------------
// /save (key actions)
// ------------------------------------------------------------
void handleSave() {
  for (int p = 0; p < 3; p++) {
    for (int i = 0; i < 9; i++) {
      String tapName  = keyPrefName(i, "tap", p);
      String longName = keyPrefName(i, "long", p);
      String dblName  = keyPrefName(i, "dbl", p);

      if (server.hasArg(tapName))  prefs.putString(tapName.c_str(),  server.arg(tapName));
      if (server.hasArg(longName)) prefs.putString(longName.c_str(), server.arg(longName));
      if (server.hasArg(dblName))  prefs.putString(dblName.c_str(),  server.arg(dblName));
    }
  }
  server.send(200, "text/html",
              "<html><body>Saved! <a href='/'>Back</a></body></html>");
}

// ------------------------------------------------------------
// Profile names
// ------------------------------------------------------------
void handleSetProfileName() {
  int p = server.arg("profile").toInt();
  String name = server.arg("name");
  if (p >= 0 && p < 3) {
    profileNames[p] = name;
    prefs.putString(("pName" + String(p)).c_str(), name);
  }
  server.send(200, "text/plain", "OK");
}

// ------------------------------------------------------------
// Export / Import profiles
// ------------------------------------------------------------
void handleExportProfiles() {
  StaticJsonDocument<4096> doc;
  JsonArray names = doc.createNestedArray("names");
  for (int p = 0; p < 3; p++) names.add(profileNames[p]);

  JsonArray profiles = doc.createNestedArray("profiles");
  for (int p = 0; p < 3; p++) {
    JsonArray keysArr = profiles.createNestedArray();
    for (int k = 0; k < 9; k++) {
      JsonObject o = keysArr.createNestedObject();
      o["tap"]  = prefs.getString(keyPrefName(k,"tap",p).c_str(), "");
      o["long"] = prefs.getString(keyPrefName(k,"long",p).c_str(), "");
      o["dbl"]  = prefs.getString(keyPrefName(k,"dbl",p).c_str(), "");
    }
  }

  JsonArray colors = doc.createNestedArray("colors");
  for (int p = 0; p < 3; p++) {
    JsonArray cArr = colors.createNestedArray();
    for (int k = 0; k < 9; k++) {
      char buf[8];
      sprintf(buf, "#%06X", keyColors[p][k] & 0xFFFFFF);
      cArr.add(String(buf));
    }
  }

  String out;
  serializeJson(doc, out);
  server.send(200, "application/json", out);
}

void handleImportProfiles() {
  StaticJsonDocument<4096> doc;
  DeserializationError err = deserializeJson(doc, server.arg((size_t)0));
  if (err) {
    server.send(400, "text/plain", "JSON error");
    return;
  }

  JsonArray names = doc["names"];
  for (int p = 0; p < 3 && p < names.size(); p++) {
    profileNames[p] = (const char*)names[p];
    prefs.putString(("pName" + String(p)).c_str(), profileNames[p]);
  }

  JsonArray profiles = doc["profiles"];
  for (int p = 0; p < 3 && p < profiles.size(); p++) {
    JsonArray keysArr = profiles[p];
    for (int k = 0; k < 9 && k < keysArr.size(); k++) {
      JsonObject o = keysArr[k];
      prefs.putString(keyPrefName(k,"tap",p).c_str(),  (const char*)o["tap"]);
      prefs.putString(keyPrefName(k,"long",p).c_str(), (const char*)o["long"]);
      prefs.putString(keyPrefName(k,"dbl",p).c_str(),  (const char*)o["dbl"]);
    }
  }

  JsonArray colors = doc["colors"];
  for (int p = 0; p < 3 && p < colors.size(); p++) {
    JsonArray cArr = colors[p];
    for (int k = 0; k < 9 && k < cArr.size(); k++) {
      String hex = (const char*)cArr[k];
      long val = strtol(hex.c_str()+1, NULL, 16);
      keyColors[p][k] = (uint32_t)val;
    }
  }

  setProfileColor();
  server.send(200, "text/plain", "OK");
}

// ------------------------------------------------------------
// OTA Update
// ------------------------------------------------------------
void handleUpdateUpload() {
  HTTPUpload& upload = server.upload();
  if (upload.status == UPLOAD_FILE_START) {
    Update.begin();
  } else if (upload.status == UPLOAD_FILE_WRITE) {
    Update.write(upload.buf, upload.currentSize);
  } else if (upload.status == UPLOAD_FILE_END) {
    Update.end(true);
  }
}

void handleUpdate() {
  server.sendHeader("Connection", "close");
  server.send(200, "text/plain", "Update complete. Rebooting...");
  delay(1000);
  ESP.restart();
}

// ------------------------------------------------------------
// RGB EFFECTS
// ------------------------------------------------------------
void effectStatic() {
  setProfileColor();
}

void effectBreathing() {
  static float phase = 0;
  float speed = 0.02f;
  phase += speed;
  float b = (sin(phase) + 1.0f) / 2.0f;

  for (int i = 0; i < LED_COUNT; i++) {
    uint32_t c = keyColors[currentProfile][i];
    uint8_t r = ((c >> 16) & 0xFF) * b;
    uint8_t g = ((c >> 8) & 0xFF) * b;
    uint8_t bC = (c & 0xFF) * b;
    strip.setPixelColor(i, strip.Color(r,g,bC));
  }
  strip.show();
}

void effectRainbow() {
  for (int i = 0; i < LED_COUNT; i++) {
    int hue = (i * 256 / LED_COUNT + rainbowOffset) & 255;
    strip.setPixelColor(i, strip.gamma32(strip.ColorHSV(hue * 256)));
  }
  strip.show();
  rainbowOffset += 2;
}

void effectReactiveIdle() {
  // keep whatever is currently set; flashes happen on key press
}

void renderRgb() {
  unsigned long now = millis();
  if (now - lastRgbFrame < 20) return;
  lastRgbFrame = now;

  switch (currentEffect) {
    case EFFECT_STATIC:    effectStatic(); break;
    case EFFECT_BREATHING: effectBreathing(); break;
    case EFFECT_RAINBOW:   effectRainbow(); break;
    case EFFECT_REACTIVE:  effectReactiveIdle(); break;
  }
}

// ------------------------------------------------------------
// MENU
// ------------------------------------------------------------
void drawMainMenu() {
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0,0);
  display.println("Main Menu");

  for (int i = 0; i < mainMenuCount; i++) {
    if (i == menuIndex) display.print("> ");
    else display.print("  ");
    display.println(mainMenuItems[i]);
  }
  display.display();
}

void enterMenu(UiState s) {
  uiState = s;
  menuIndex = 0;
  if (s == UI_MENU_MAIN) drawMainMenu();
}

// ------------------------------------------------------------
// SETUP
// ------------------------------------------------------------
void setup() {
  Serial.begin(115200);

  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  display.clearDisplay();
  display.display();

  prefs.begin("keypad", false);

  for (int i = 0; i < 9; i++) {
    pinMode(keys[i], INPUT);
    lastState[i] = false;
  }

  pinMode(CLK, INPUT);
  pinMode(DT, INPUT);
  pinMode(SW, INPUT_PULLUP);
  lastCLK = digitalRead(CLK);

  pinMode(BTN_PREV, INPUT_PULLUP);
  pinMode(BTN_PLAY, INPUT_PULLUP);
  pinMode(BTN_NEXT, INPUT_PULLUP);

  pinMode(BTN_PROFILE, INPUT_PULLUP);

  strip.begin();
  strip.show();
  setProfileColor();

  // load profile names
  for (int p = 0; p < 3; p++) {
    String n = prefs.getString(("pName" + String(p)).c_str(), "");
    if (n.length()) profileNames[p] = n;
  }

  bleKeyboard.begin();

  WiFi.softAP("KeypadSetup", "12345678");
  server.on("/", [](){ server.send(200, "text/html", htmlPage()); });
  server.on("/save", HTTP_POST, handleSave);
  server.on("/status", handleStatus);
  server.on("/rgb", HTTP_GET, handleRgbGet);
  server.on("/rgb", HTTP_POST, handleRgbPost);
  server.on("/setProfileName", HTTP_POST, handleSetProfileName);
  server.on("/exportProfiles", handleExportProfiles);
  server.on("/importProfiles", HTTP_POST, handleImportProfiles);
  server.on("/update", HTTP_POST, handleUpdate, handleUpdateUpload);
  server.begin();

  showText("Setup Ready");
  lastActivity = millis();
}

// ------------------------------------------------------------
// MAIN LOOP
// ------------------------------------------------------------
void loop() {
  server.handleClient();
  unsigned long now = millis();

  if (now - lastBatteryRead > BATTERY_INTERVAL_MS) {
    lastBatteryRead = now;
    updateBattery();
    drawProfileIndicator();
  }

  renderRgb();

  if (screensaverOn) {
    if (digitalRead(BTN_PREV)==LOW || digitalRead(BTN_PLAY)==LOW ||
        digitalRead(BTN_NEXT)==LOW || digitalRead(SW)==LOW ||
        digitalRead(BTN_PROFILE)==LOW) {
      screensaverOn = false;
      setProfileColor();
      showText("Wake");
      lastActivity = now;
    }
  }

  bool profState = digitalRead(BTN_PROFILE);
  if (profState == LOW && lastProfile == HIGH) {
    currentProfile = (currentProfile + 1) % 3;
    setProfileColor();
    profileFlash();
    showText("Profile " + String(currentProfile + 1));
    lastActivity = now;
    delay(250);
  }
  lastProfile = profState;

  if (digitalRead(BTN_PLAY) == LOW && uiState == UI_NORMAL) {
    delay(300);
    if (digitalRead(BTN_PLAY) == LOW) {
      enterMenu(UI_MENU_MAIN);
      lastActivity = now;
    }
  }

  if (uiState != UI_NORMAL) {
    int currentCLK = digitalRead(CLK);
    if (currentCLK != lastCLK) {
      if (digitalRead(DT) != currentCLK) menuIndex++;
      else menuIndex--;
      if (menuIndex < 0) menuIndex = mainMenuCount - 1;
      if (menuIndex >= mainMenuCount) menuIndex = 0;
      drawMainMenu();
    }
    lastCLK = currentCLK;

    if (digitalRead(SW) == LOW) {
      delay(200);
      if (uiState == UI_MENU_MAIN) {
        if (menuIndex == 0) {
          currentProfile = (currentProfile + 1) % 3;
          setProfileColor();
          profileFlash();
          showText("Profile " + String(currentProfile + 1));
        } else if (menuIndex == 1) {
          currentEffect = (RgbEffect)((currentEffect + 1) % 4);
          showText("Effect " + String((int)currentEffect));
        } else if (menuIndex == 2) {
          showText("Settings");
        } else if (menuIndex == 3) {
          showText("MacroPad v1");
        }
      }
      lastActivity = now;
    }

    if (digitalRead(BTN_PREV) == LOW) {
      uiState = UI_NORMAL;
      showText("Exit Menu");
      lastActivity = now;
      delay(200);
    }

    return;
  }

  if (!bleKeyboard.isConnected()) return;

  for (int i = 0; i < 9; i++) {
    bool pressed = (digitalRead(keys[i]) == HIGH);

    if (pressed && !lastState[i]) {
      lastPressTime[i] = now;
    }

    if (!pressed && lastState[i]) {
      unsigned long dur = now - lastPressTime[i];

      String tapAction  = prefs.getString(keyPrefName(i,"tap").c_str(), "");
      String longAction = prefs.getString(keyPrefName(i,"long").c_str(), "");
      String dblAction  = prefs.getString(keyPrefName(i,"dbl").c_str(), "");

      if (dur >= LONG_PRESS_MS && longAction.length()) {
        showText("L" + String(i + 1));
        flashKey(i, 0xFF0000);
        sendAction(longAction);
      } else {
        if ((now - lastTapTime[i]) <= DOUBLE_TAP_MS && dblAction.length()) {
          showText("D" + String(i + 1));
          flashKey(i, 0x0000FF);
          sendAction(dblAction);
          lastTapTime[i] = 0;
        } else {
          if (tapAction.length()) {
            showText("K" + String(i + 1));
            flashKey(i, 0x00FF00);
            sendAction(tapAction);
          }
          lastTapTime[i] = now;
        }
      }

      if (currentEffect == EFFECT_REACTIVE) {
        flashKey(i, 0xFFFFFF, 40);
      }

      lastActivity = now;
    }

    lastState[i] = pressed;
  }

  int currentCLK = digitalRead(CLK);
  if (currentCLK != lastCLK) {
    if (digitalRead(DT) != currentCLK) {
      bleKeyboard.write(KEY_MEDIA_VOLUME_UP);
      showText("Vol +");
    } else {
      bleKeyboard.write(KEY_MEDIA_VOLUME_DOWN);
      showText("Vol -");
    }
    lastActivity = now;
  }
  lastCLK = currentCLK;

  if (digitalRead(SW) == LOW) {
    bleKeyboard.write(KEY_MEDIA_PLAY_PAUSE);
    showText("Play/Pause");
    lastActivity = now;
    delay(300);
  }

  bool prevState = digitalRead(BTN_PREV);
  bool playState = digitalRead(BTN_PLAY);
  bool nextState = digitalRead(BTN_NEXT);

  if (prevState == LOW && lastPrev == HIGH) {
    bleKeyboard.write(KEY_MEDIA_PREVIOUS_TRACK);
    showText("Prev");
    lastActivity = now;
    delay(200);
  }
  lastPrev = prevState;

  if (playState == LOW && lastPlay == HIGH) {
    bleKeyboard.write(KEY_MEDIA_PLAY_PAUSE);
    showText("Play/Pause");
    lastActivity = now;
    delay(200);
  }
  lastPlay = playState;

  if (nextState == LOW && lastNext == HIGH) {
    bleKeyboard.write(KEY_MEDIA_NEXT_TRACK);
    showText("Next");
    lastActivity = now;
    delay(200);
  }
  lastNext = nextState;

  if (!screensaverOn && (now - lastActivity > IDLE_TIMEOUT_MS)) {
    screensaverOn = true;

    for (int i = 0; i < LED_COUNT; i++) {
      uint32_t c = keyColors[currentProfile][i];
      uint8_t r = ((c >> 16) & 0xFF) / 8;
      uint8_t g = ((c >> 8) & 0xFF) / 8;
      uint8_t b = (c & 0xFF) / 8;
      strip.setPixelColor(i, strip.Color(r,g,b));
    }
    strip.show();

    display.clearDisplay();
    display.setTextSize(2);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(40, 20);
    display.print("Zzz");
    display.display();
  }
}
