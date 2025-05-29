// EpEver Solar Charger Monitor
// Author: Andrew Morty | Device: NodeMCU ESP8266 | RS485MAX
// Features: Modbus RS485 polling, MQTT, OTA, DeepSleep, HTTP dashboard

#include <ModbusMaster.h>
#include <ESP8266WiFi.h>
#include <ESP8266HTTPClient.h>
#include <ESP8266WebServer.h>
#include <ESP8266WiFiMulti.h>
#include <SoftwareSerial.h>
#include <ArduinoOTA.h>
#include <PubSubClient.h>
#include <ESP8266HTTPUpdateServer.h>

#define FIRMWARE_VERSION "1.0.0"

// === Pin Definitions ===
#define RS485_RX        3
#define RS485_TX        1
#define RS485_RE        D1  // RS485 Read Enable
#define RS485_DE        D2  // RS485 Driver Enable
#define HB              D4  // Blue LED on microprocessor to flash heartbeat
#define PIN_DEEPSLEEP_1 D5
#define PIN_DEEPSLEEP_2 D6
#define POWERRAIL_PIN   D8

// === WiFi/MQTT ===
#include "secrets.h"  // Contains wifi_ssid, wifi_pass, mqtt credentials, hostname, etc.
ESP8266WiFiMulti wifiMulti;
WiFiClient espClient;
PubSubClient mqttClient(espClient);

bool ha_discovery_sent = false;
String last_mqtt_debug = "";


// === OTA Status Tracking ===
String status_arduinoOTA = "none";

// === RS485/Modbus Setup ===
SoftwareSerial RS485Serial(RS485_RX, RS485_TX);
ModbusMaster node;





// === HTTP Server Setup ===
ESP8266WebServer server(80);



// === Solar Metrics (live telemetry values) ===
float pv_voltage, pv_current, pv_power;         // PV panel metrics
float batt_voltage, batt_current, batt_power;
float load_voltage, load_current, load_power;
float battChargePower, batterySOC, batteryTemp;
String lastTempSource = "none";   // Tracks whether 311B or 3110 was used for temperature


// === Runtime Counters ===
unsigned long millis_start        = 0; // millis() at system boot
unsigned long millis_last_epever = 0;  // last Modbus polling time
unsigned long started_waiting_at = 0;  // Modbus retry timing
unsigned int ct_mqtt = 0;              // MQTT publish counter
unsigned int last_mqtt_status = 0;     // MQTT status: 1 = success, 0 = fail
unsigned long last_mqtt_sent = 0;      // Timestamp of last successful MQTT send
String last_mqtt_payload;              // JSON sent to MQTT last

// === RS485 Polling Success/Failure Stats ===
unsigned long ct_3100_success, ct_3100_fail;  // Battery+PV metrics
unsigned long ct_311A_success, ct_311A_fail;  // Battery SOC
unsigned long ct_3106_success, ct_3106_fail;  // Battery charge power
unsigned long ct_3110_success, ct_3110_fail;  // Fallback temperature register
unsigned long ct_311B_success, ct_311B_fail;  // Battery temperature
uint8_t lastErr_3100 = 0;
uint8_t lastErr_311A = 0;
uint8_t lastErr_3106 = 0;
uint8_t lastErr_3110 = 0;
uint8_t lastErr_311B = 0;
unsigned long errTime_3100, errTime_311A, errTime_3106, errTime_3110, errTime_311B;
unsigned long successTime_3100, successTime_311A, successTime_3106, successTime_3110, successTime_311B;

// === HTML Footer with Version and Timestamp ===
const char* footer_html = 
  "<div style='margin-top:2rem;font-size:0.75rem;color:#777;text-align:center;'>"
  "Morty Labs solar charger firmware v" FIRMWARE_VERSION 
  " &mdash; compiled " __DATE__ " " __TIME__ 
  "</div>";


// ============================================================================
// SYSTEM OVERVIEW
// ----------------------------------------------------------------------------
// This firmware provides real-time solar monitoring for an EpEver charge
// controller via RS485/Modbus using an ESP8266 (NodeMCU) with SoftwareSerial.
// It publishes metrics to MQTT (for Home Assistant or InfluxDB) and serves
// a responsive local web dashboard.
//
// Major subsystems:
// - Modbus polling (EpEver registers)
// - MQTT client with Home Assistant discovery
// - HTTP web dashboard with status indicators
// - OTA firmware updates with live status
// ============================================================================

// === Helper ===
String get_uptime(unsigned long start) {
  unsigned long secs = (millis() - start) / 1000;
  char buf[12];
  snprintf(buf, sizeof(buf), "%02lu:%02lu:%02lu", secs / 3600, (secs / 60) % 60, secs % 60);
  return String(buf);
}

String get_timestamp(unsigned long ms) {
  time_t t = ms / 1000;
  struct tm* timeinfo = gmtime(&t);
  char buf[24];
  snprintf(buf, sizeof(buf), "%02d:%02d:%02d", timeinfo->tm_hour, timeinfo->tm_min, timeinfo->tm_sec);
  return String(buf);
}

String getModbusErrorDescription(uint8_t code) {
  switch (code) {
    case 0x01: return "Illegal Function";
    case 0x02: return "Illegal Data Address";
    case 0x03: return "Illegal Data Value";
    case 0x04: return "Slave Device Failure";
    case 0x05: return "Acknowledge";
    case 0x06: return "Slave Device Busy";
    case 0x07: return "Negative Acknowledge";
    case 0x08: return "Memory Parity Error";
    case 0x0A: return "Gateway Path Unavailable";
    case 0x0B: return "Target Failed to Respond";
    case 0xE0: return "Invalid Slave ID";
    case 0xE1: return "Invalid Function";
    case 0xE2: return "Response Timed Out";
    case 0xE3: return "Invalid CRC";
    default:   return "Unknown Error";
  }
}

void preTransmission() {
  digitalWrite(RS485_RE, HIGH);
  digitalWrite(RS485_DE, HIGH);
  digitalWrite(HB, LOW);
}

void postTransmission() {
  digitalWrite(RS485_RE, LOW);
  digitalWrite(RS485_DE, LOW);
  digitalWrite(HB, HIGH);
}

void mqtt_reconnect() {
  unsigned long start = millis();
  while (!mqttClient.connected() && millis() - start < 10000) {
    if (mqttClient.connect(hostname, mqtt_user, mqtt_pass)) {
      if (!ha_discovery_sent) {
        send_ha_discovery();
        ha_discovery_sent = true;
      }
    }
    delay(500);
  }
}

void check_wifi_reconnect() {
  if (WiFi.status() != WL_CONNECTED) {
    unsigned long startAttemptTime = millis();
    while (WiFi.status() != WL_CONNECTED && millis() - startAttemptTime < 5000) {
      wifiMulti.run();
      delay(500);
    }
    if (WiFi.status() != WL_CONNECTED) {
      ESP.restart();  // fallback if still not connected
    }
  }
}



int get_modbus_data(uint16_t addr, uint8_t len) {
  started_waiting_at = millis();
  node.clearResponseBuffer();
  uint8_t result;
  do {
    result = node.readInputRegisters(addr, len);
    if (millis() - started_waiting_at > 400) break;
    delay(50);
  } while (result != node.ku8MBSuccess);

  if (result == node.ku8MBSuccess) {
    if (addr == 0x3100) {
      ct_3100_success++;
      successTime_3100 = millis();
    }
    else if (addr == 0x311A) {
      ct_311A_success++;
      successTime_3100 = millis();
    }
    else if (addr == 0x3106) {
      ct_3106_success++;
      successTime_3106 = millis();
    }
    else if (addr == 0x3110) {
      ct_3110_success++;
      successTime_3106 = millis();
    }
    else if (addr == 0x311B) {
      ct_311B_success++;
      successTime_311B = millis();
    }
  } else {
    if (addr == 0x3100) { 
        ct_3100_fail++; 
        lastErr_3100 = result; 
        errTime_3100 = millis();
        mqtt_debug("Modbus FAIL: 0x3100 — " + getModbusErrorDescription(result));
    }
    else if (addr == 0x311A) { 
      ct_311A_fail++; 
      lastErr_311A = result; 
      errTime_311A= millis();
      mqtt_debug("Modbus FAIL: 0x311A — " + getModbusErrorDescription(result));
    }
    else if (addr == 0x3106) { 
      ct_3106_fail++; 
      lastErr_3106 = result; 
      errTime_3106 = millis();
      mqtt_debug("Modbus FAIL: 0x3106 — " + getModbusErrorDescription(result));
    }
    else if (addr == 0x3110) { 
      ct_3110_fail++; 
      lastErr_3110 = result; 
      errTime_3110 = millis();
      mqtt_debug("Modbus FAIL: 0x3110 — " + getModbusErrorDescription(result));
    }
    else if (addr == 0x311B) { 
      ct_311B_fail++; 
      lastErr_311B = result; 
      errTime_311B = millis();
      mqtt_debug("Modbus FAIL: 0x311B — " + getModbusErrorDescription(result));
    }
  }

  return result;
}



String create_json_payload() {
  String json = "{\n";
  json += " \"pv_volt\": " + (successTime_3100 > errTime_3100 ? String(pv_voltage) : "null") + ",\n";
  json += " \"pv_amps\": " + (successTime_3100 > errTime_3100 ? String(pv_current) : "null") + ",\n";
  json += " \"pv_power\": " + (successTime_3100 > errTime_3100 ? String(pv_power) : "null") + ",\n";
  json += " \"batt_volt\": " + (successTime_3100 > errTime_3100 ? String(batt_voltage) : "null") + ",\n";
  json += " \"batt_amps\": " + (successTime_3100 > errTime_3100 ? String(batt_current) : "null") + ",\n";
  json += " \"batt_power\": " + (successTime_3100 > errTime_3100 ? String(batt_power) : "null") + ",\n";
  json += " \"load_volt\": " + (successTime_3100 > errTime_3100 ? String(load_voltage) : "null") + ",\n";
  json += " \"load_amps\": " + (successTime_3100 > errTime_3100 ? String(load_current) : "null") + ",\n";
  json += " \"load_power\": " + (successTime_3100 > errTime_3100 ? String(load_power) : "null") + ",\n";
  json += " \"batt_charge_power\": " + (successTime_3106 > errTime_3106 ? String(battChargePower) : "null") + ",\n";
  json += " \"batt_temperature\": " + (batteryTemp != 0 && batteryTemp < 100 ? String(batteryTemp) : "null") + ",\n";

  json += " \"batt_soc\": " + (successTime_311A > errTime_311A ? String(batterySOC) : "null") + ",\n";
  json += " \"RSSI\": " + String(WiFi.RSSI()) + ",\n";
  json += " \"MAC\": \"" + WiFi.macAddress() + "\"\n";
  json += " \"firmware_version\": \"" + String(FIRMWARE_VERSION) + "\"\n";

  json += "}";
  last_mqtt_payload = json;
  return json;
}

const char html_template[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html lang='en'>
<head>
  <meta charset='UTF-8'>
  <meta name='viewport' content='width=device-width, initial-scale=1.0'>
  <title>Solar Monitor</title>
  <style>
    body {
      background: #0e0e0e;
      color: #e0e0e0;
      font-family: 'Segoe UI', sans-serif;
      margin: 0;
      padding: 1rem;
    }
    h1, h2 {
      color: #2ecc71;
      margin-top: 2rem;
    }
    .metrics {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
      gap: 1rem;
    }
    .metric {
      background: #1e1e1e;
      padding: 1rem;
      border-radius: 10px;
      text-align: center;
    }
    .metric label {
      display: block;
      font-size: 0.85rem;
      color: #aaa;
    }
    .metric .value {
      font-size: 1.4rem;
      font-weight: bold;
      margin-top: 0.2rem;
    }
    table {
      width: 100%;
      margin-top: 1rem;
      border-collapse: collapse;
    }
    th, td {
      border: 1px solid #444;
      padding: 0.5rem;
      text-align: center;
    }
    th {
      background: #222;
      color: #2ecc71;
    }
    .success-count {
      color: #8f8;
    }
    .error-count.zero {
      color: #888;
    }
    .error-count.nonzero {
      color: #f88;
    }
    .mqtt-section {
      margin-top: 2rem;
    }
    .mqtt-row {
      margin-bottom: 1rem;
    }
    .mqtt-row label {
      font-weight: bold;
      color: #2ecc71;
      display: inline-block;
      min-width: 140px;
    }
    .mqtt-value.connected {
      color: #8f8;
    }
    .mqtt-value.disconnected {
      color: #f88;
    }
    .mqtt-payload {
      background: #1e1e1e;
      padding: 1rem;
      border-radius: 8px;
      white-space: pre-wrap;
      overflow-x: auto;
    }
    .upload-btn {
  display: inline-block;
  padding: 0.5rem 1rem;
  margin-top: 0.5rem;
  background-color: #2ecc71;
  color: #000;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: bold;
  text-decoration: none;
}
.upload-btn:hover {
  background-color: #27ae60;
}

  </style>
</head>
<body>
  <h1>🔋 EpEver Solar Monitor</h1>
  <div class='metrics'>
    {{solar_status}}
  </div>
  <h2>📈 RS485 Communication</h2>
  <div class='metrics'>
    {{rs485_stats}}
  </div>
  <h2>📡 MQTT Status</h2>
  <div class='mqtt-section'>
    {{mqtt_status}}
  </div>
</body>
</html>
)rawliteral";

String get_elapsed(unsigned long last, unsigned long count) {
  if (count == 0) return "-";
  unsigned long delta = (millis() - last) / 1000;
  char buf[12];
  snprintf(buf, sizeof(buf), "%02lu:%02lu:%02lu", delta / 3600, (delta / 60) % 60, delta % 60);
  return String(buf);
}

String metric_style(bool success) {
  return success ? "'value'" : "'value' style=\"color:#f88;\"";
}

String METRIC_HTML(const char* label, const char* id, const String& value, bool success) {
  String displayValue = value;
  if (!success) displayValue = "-";
  return "<div class='metric'><label>" + String(label) + "</label><span class=" + metric_style(success) + " id='" + id + "'>" + displayValue + "</span></div>";
}



String RS485_TABLE_ROW(const String& label, unsigned long success, unsigned long successTime, unsigned long fail, unsigned long failTime, uint8_t lastError) {
  String status;

  if (fail > 0) {
    // Format error code as 0xE2
    String hexCode = "0x" + String(lastError, HEX);
    hexCode.toUpperCase();

    // Format status message
    String desc = getModbusErrorDescription(lastError);
    status = "<span style='color:#f88; white-space:nowrap;'>" + hexCode + " – " + desc + "</span>";
  } else {
    status = "<span style='color:#8f8;'>OK</span>";
  }

  return "<tr>"
         "<td style='white-space:nowrap;'>" + label + "</td>"
         "<td class='success-count'>" + String(success) + "</td>"
         "<td>" + get_elapsed(successTime, success) + "</td>"
         "<td class='error-count " + (fail != 0 ? "nonzero" : "zero") + "'>" + String(fail) + "</td>"
         "<td>" + get_elapsed(failTime, fail) + "</td>"
         "<td style='white-space:nowrap;'>" + status + "</td>"
         "</tr>";
}



void mqtt_debug(const String& msg) {
  if (mqttClient.connected()) {
    mqttClient.publish(mqtt_debug_topic, msg.c_str(), false);
  }
  last_mqtt_debug = msg;
}



String render_html_status_page() {
  String page = FPSTR(html_template);

  bool last3100Success = successTime_3100 > errTime_3100;
  String solar = "";
  solar += METRIC_HTML("PV Voltage", "pv_voltage", String(pv_voltage) + " V", last3100Success);
  solar += METRIC_HTML("PV Current", "pv_current", String(pv_current) + " A", last3100Success);
  solar += METRIC_HTML("PV Power", "pv_power", String(pv_power) + " W", last3100Success);
  solar += METRIC_HTML("Battery Voltage", "batt_voltage", String(batt_voltage) + " V", last3100Success);
  solar += METRIC_HTML("Battery Current", "batt_current", String(batt_current) + " A", last3100Success);
  solar += METRIC_HTML("Battery Power", "batt_power", String(batt_power) + " W", last3100Success);
  solar += METRIC_HTML("Load Voltage", "load_voltage", String(load_voltage) + " V", last3100Success);
  solar += METRIC_HTML("Load Current", "load_current", String(load_current) + " A", last3100Success);
  solar += METRIC_HTML("Load Power", "load_power", String(load_power) + " W", last3100Success);
  solar += METRIC_HTML("Battery Charge Power", "batt_charge_power", String(battChargePower) + " W", successTime_3106 > errTime_3106);
  solar += METRIC_HTML("Battery Temp", "battery_temp", String(batteryTemp) + " C", successTime_311B > errTime_311B);
  solar += METRIC_HTML("Battery SOC", "battery_soc", String(batterySOC) + " %", successTime_311A > errTime_311A);
  solar += METRIC_HTML("Uptime", "uptime", get_uptime(millis_start), true);
  //solar += METRIC_HTML("Last Poll", "last_poll", get_timestamp(millis_last_success), true);

 
  String rs485 = "<table><tr><th>Register</th><th>Success</th><th>Since</th><th>Fail</th><th>Since</th><th>Last Status</th></tr>";

  rs485 += RS485_TABLE_ROW("Battery+PV (0x3100)", ct_3100_success, successTime_3100, ct_3100_fail, errTime_3100, lastErr_3100);
  rs485 += RS485_TABLE_ROW("Battery SOC (0x311A)", ct_311A_success, successTime_311A, ct_311A_fail, errTime_311A, lastErr_311A);
  rs485 += RS485_TABLE_ROW("Charge Power (0x3106)", ct_3106_success, successTime_3106, ct_3106_fail, errTime_3106, lastErr_3106);
  rs485 += RS485_TABLE_ROW("Fallback Temp (0x3110)", ct_3110_success, successTime_3110, ct_3110_fail, errTime_3110, lastErr_3110);
  rs485 += RS485_TABLE_ROW("Battery Temp (0x311B)", ct_311B_success, successTime_311B, ct_311B_fail, errTime_311B, lastErr_311B);
  rs485 += "</table>";


  String mqtt = "<div class='mqtt-section'>";
  mqtt += "<div class='mqtt-row'><label>Status:</label><span class='mqtt-value ";
  mqtt += mqttClient.connected() ? "connected'>&#9989; Connected" : "disconnected'>&#10060; Disconnected";
  mqtt += "</span></div>";
  mqtt += "<div class='mqtt-row'><label>OTA:</label><span class='mqtt-value'>" + status_arduinoOTA + "</span></div>";
  mqtt += "<div class='mqtt-row'><label>Last Payload:</label><span class='mqtt-value'>" + String(last_mqtt_status ? "Success" : "Fail") + " (" + get_elapsed(last_mqtt_sent, 1) + " ago)</span></div>";
  mqtt += "<div class='mqtt-row'><pre class='mqtt-payload'>" + last_mqtt_payload + "</pre></div>";
  mqtt += "<div class='mqtt-row'><label>Last Debug:</label><span class='mqtt-value'>" + last_mqtt_debug + "</span></div>";

  mqtt += "</div>";
 mqtt += "<div class='mqtt-row'><a href='/firmware' class='upload-btn'>Upload Firmware</a></div>";


  page.replace("{{solar_status}}", solar);
  page.replace("{{rs485_stats}}", rs485);
  page.replace("{{mqtt_status}}", mqtt);  
  page += footer_html;
  

  return page;
}

void send_ha_discovery() {
  String topic_prefix = "homeassistant/sensor/";

  struct {
    const char* id;
    const char* name;
    const char* unit;
    const char* device_class;
    const char* state_class;
  } sensors[] = {
    {"pv_voltage", "PV Voltage", "V", "voltage", "measurement"},
    {"pv_current", "PV Current", "A", "current", "measurement"},
    {"pv_power",   "PV Power",   "W", "power",   "measurement"},
    {"batt_voltage", "Battery Voltage", "V", "voltage", "measurement"},
    {"batt_current", "Battery Current", "A", "current", "measurement"},
    {"batt_power",   "Battery Power",   "W", "power",   "measurement"},
    {"load_voltage", "Load Voltage", "V", "voltage", "measurement"},
    {"load_current", "Load Current", "A", "current", "measurement"},
    {"load_power",   "Load Power",   "W", "power",   "measurement"},
    {"batt_charge_power", "Battery Charge Power", "W", "power", "measurement"},
    {"batt_temperature", "Battery Temperature", "C", "temperature", "measurement"},
    {"batt_soc", "Battery SOC", "%", "battery", "measurement"},
  };

  String mac = WiFi.macAddress();
  String device_info =
    "\"device\":{\"identifiers\":[\"" + mac + "\"],\"name\":\"EpEver Solar Monitor\",\"manufacturer\":\"MortyLabs\",\"model\":\"ESP8266 RS485 EPEVER Monitor\"}";

  for (auto& s : sensors) {
    String config = "{\"name\":\"" + String(s.name) +
                    "\",\"state_topic\":\"" + String(mqtt_topic) +
                    "\",\"unit_of_measurement\":\"" + s.unit +
                    "\",\"value_template\":\"{{ value_json." + s.id + " }}\"," +
                    "\"device_class\":\"" + s.device_class +
                    "\",\"state_class\":\"" + s.state_class + "\"," +
                    device_info + "}";

    String discovery_topic = topic_prefix + s.id + "/config";
    mqttClient.publish(discovery_topic.c_str(), config.c_str(), true);
  }
}


void handleFirmwareUpload() {
  HTTPUpload& upload = server.upload();
  if (upload.status == UPLOAD_FILE_START) {
    Serial.setDebugOutput(true);
    WiFiUDP::stopAll();
    if (!Update.begin((ESP.getFreeSketchSpace() - 0x1000) & 0xFFFFF000)) {
      Update.printError(Serial);
    }
  } else if (upload.status == UPLOAD_FILE_WRITE) {
    if (Update.write(upload.buf, upload.currentSize) != upload.currentSize) {
      Update.printError(Serial);
    }
  } else if (upload.status == UPLOAD_FILE_END) {
    if (Update.end(true)) {
      Serial.printf("Update Success: %u bytes\n", upload.totalSize);
    } else {
      Update.printError(Serial);
    }
    Serial.setDebugOutput(false);
  }
  yield();
}

void serveFirmwareForm() {
  server.send(200, "text/html",
    "<html><head><title>Firmware Upload</title>"
    "<style>body{font-family:sans-serif;background:#121212;color:#eee;padding:2em;}"
    "form{background:#1e1e1e;padding:2em;border-radius:10px;}"
    "input[type=file]{margin-bottom:1em;}"
    "progress{width:100%;}</style>"
    "<script>"
    "function startUpload(){"
    "  var form = document.querySelector('form');"
    "  var progress = document.getElementById('prog');"
    "  var status = document.getElementById('status');"
    "  var xhr = new XMLHttpRequest();"
    "  xhr.upload.addEventListener('progress', function(e) {"
    "    if (e.lengthComputable) {"
    "      var percent = (e.loaded / e.total) * 100;"
    "      progress.value = percent;"
    "    }"
    "  });"
    "  xhr.onreadystatechange = function() {"
    "    if (xhr.readyState == 4 && xhr.status == 200) {"
    "      status.innerHTML = 'Upload complete. Rebooting...';"
    "    }"
    "  };"
    "  xhr.open('POST', '/firmware', true);"
    "  xhr.send(new FormData(form));"
    "  status.innerHTML = 'Uploading...';"
    "  return false;"
    "}"
    "</script></head><body>"
    "<h2>MortyLabs OTA Firmware Upload</h2>"
    "<form onsubmit='return startUpload();'>"
    "<input type='file' name='update' required><br>"
    "<input type='submit' value='Upload'><br><br>"
    "<progress id='prog' value='0' max='100'></progress>"
    "</form><div id='status'></div>"
    "</body></html>");
}


void loop() {
  if (WiFi.status() != WL_CONNECTED) {
    check_wifi_reconnect();
  }

  server.handleClient();
  ArduinoOTA.handle();

  if (millis() - millis_last_epever > 60000) {
    digitalWrite(HB, LOW);
    millis_last_epever = millis();

    if (get_modbus_data(0x3100, 16) == node.ku8MBSuccess) {
      pv_voltage = node.getResponseBuffer(0x00) / 100.0f;     // 0x00 = 0
      pv_current = node.getResponseBuffer(0x01) / 100.0f;     // 0x01 = 1 etc
      pv_power   = ((node.getResponseBuffer(0x03) << 8) | node.getResponseBuffer(0x02)) / 100.0f;
      batt_voltage = node.getResponseBuffer(0x04) / 100.0f;
      batt_current = node.getResponseBuffer(0x05) / 100.0f;
      batt_power   = ((node.getResponseBuffer(0x07) << 8) | node.getResponseBuffer(0x06)) / 100.0f;
      load_voltage = node.getResponseBuffer(0x0C) / 100.0f;   // 0x0C = 12
      load_current = node.getResponseBuffer(0x0D) / 100.0f;   // 0x0D = 13
      load_power   = ((node.getResponseBuffer(0x0F) << 8) | node.getResponseBuffer(0x0E)) / 100.0f;   // 0x0F = 15, 0x0E = 14 
    } else {
      mqtt_debug("Failed to read 0x3100");
    }
delay(100);

    if (get_modbus_data(0x3106, 2) == node.ku8MBSuccess) {
      battChargePower = ((uint32_t)node.getResponseBuffer(1) << 16 | node.getResponseBuffer(0)) / 100.0f;
    } else {
      mqtt_debug("Failed to read 0x3106");
    }

    if (get_modbus_data(0x311B, 1) == node.ku8MBSuccess) {
      batteryTemp = node.getResponseBuffer(0x00) / 100.0f;
      lastTempSource = "311B";

    } else {
      mqtt_debug("Failed to read 0x311B");
    }
    
    if (batteryTemp == 0 || batteryTemp >= 100) {
      delay(100);
      if (get_modbus_data(0x3110, 3) == node.ku8MBSuccess) {
        batteryTemp = node.getResponseBuffer(0) / 100.0f;
        lastTempSource = "3110";
      } else {
        mqtt_debug("Failed to read 0x3110");
        }
    }
    delay(100);

    if (get_modbus_data(0x311A, 2) == node.ku8MBSuccess) {
      batterySOC = node.getResponseBuffer(0);    
   } else {
      mqtt_debug("Failed to read 0x311A");
    }
    delay(100);

    if (!mqttClient.connected()) mqtt_reconnect();
    last_mqtt_status = mqttClient.publish(mqtt_topic, create_json_payload().c_str(), true);
    if (last_mqtt_status) {
        last_mqtt_sent = millis();
        ct_mqtt++;
    }
    
    delay(100);
    digitalWrite(HB, HIGH);
  }

  if (digitalRead(PIN_DEEPSLEEP_2) == HIGH) {
    ESP.deepSleep(60e6);
  }
}

void setup() {
  millis_start = millis();
  millis_last_epever = millis() - 100000;

//  pinMode(POWERRAIL_PIN, OUTPUT); digitalWrite(POWERRAIL_PIN, HIGH);
  pinMode(HB, OUTPUT);
  pinMode(RS485_RE, OUTPUT);
  pinMode(RS485_DE, OUTPUT);

  pinMode(PIN_DEEPSLEEP_1, OUTPUT); digitalWrite(PIN_DEEPSLEEP_1, LOW);
  pinMode(PIN_DEEPSLEEP_2, OUTPUT); digitalWrite(PIN_DEEPSLEEP_2, LOW);
  pinMode(RS485_RE, OUTPUT); pinMode(RS485_DE, OUTPUT);

  //Serial.begin(115200);
  RS485Serial.begin(9600);
  node.begin(1, RS485Serial);
  node.preTransmission(preTransmission);
  node.postTransmission(postTransmission);

  wifiMulti.addAP(wifi_ssid, wifi_pass);
  WiFi.hostname(hostname);
  WiFi.persistent(false);
  WiFi.mode(WIFI_STA);
  while (wifiMulti.run() != WL_CONNECTED) {
    digitalWrite(HB, !digitalRead(HB)); // Toggle LED
    delay(200);
  }
  digitalWrite(HB, HIGH); // Turn off LED once connected


  mqttClient.setBufferSize(512);
  
  mqttClient.setServer(mqtt_server, mqtt_port);
  if (!mqttClient.connected()) mqtt_reconnect();

  ArduinoOTA.setHostname(hostname);
  ArduinoOTA.onStart([]() { status_arduinoOTA = "OTA Start"; mqtt_debug("OTA: Update started"); });
  ArduinoOTA.onEnd([]() { status_arduinoOTA = "OTA Complete"; mqtt_debug("OTA: Update complete"); });
  ArduinoOTA.onError([](ota_error_t error) {
    status_arduinoOTA = "OTA Error: " + String(error);
    mqtt_debug("OTA ERROR: Code " + String(error));
  });
ArduinoOTA.onProgress([](unsigned int progress, unsigned int total) { status_arduinoOTA = "OTA Progress: " + String((progress * 100) / total) + "%";});
  
  ArduinoOTA.setPassword(ota_pass);
  ArduinoOTA.begin();

  server.on("/", []() {
    server.send(200, "text/html",  render_html_status_page());
  });

  server.on("/json", []() {
    server.send(200, "application/json", create_json_payload());
  });

  server.on("/reboot", []() {
    server.send(200, "text/plain", "Rebooting...");
    delay(1000);
    ESP.restart();
  });

  server.on("/firmware", HTTP_GET, serveFirmwareForm);
server.on("/firmware", HTTP_POST, []() {
  server.sendHeader("Connection", "close");
  server.send(200, "text/html", "<meta http-equiv='refresh' content='10;url=/'><h2>Update successful. Rebooting...</h2>");
  delay(1000);
  ESP.restart();
}, handleFirmwareUpload);

  server.begin();

  mqtt_debug("Boot: EpEver Solar Monitor online — firmware " + String(FIRMWARE_VERSION));

}
