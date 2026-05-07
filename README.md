
<h1 align="center">📋 Zoyi703s ESP32-S3</h1>

Projekcik dla Zoyi703s na ESP32-s3. Czyli odczyt z UART z zoyi do esp.

LCD na jakim to robie to ST7796. Link do aliexpress ponizej:

https://pl.aliexpress.com/item/1005012140511377.html

Link do zoyi 703s na aliexpress ponizej:

https://pl.aliexpress.com/item/1005006679533411.html

Ulepszona wesja zoyi 703s 

https://pl.aliexpress.com/item/1005012230968253.html

PS1: Jezeli ktos stworzy swojego i chcialby sie podzielic kodem z innymi to piszcie na gg 8772666. Wkleje tutaj wasze wypociny ze zdjeciami i kodem :)

<img width="1126" height="2000" alt="2026-05-06_20 24 21" src="https://github.com/user-attachments/assets/ea969d2a-b950-4510-84ff-97ad5890fdcd" />

Kod dla kazdego kto chce sobie poeksperymentowac ponizej:

```C++
#include <Arduino.h>
#include <WiFi.h>
#include <WebServer.h>
#include <TFT_eSPI.h>
#include <SPI.h>

#define TFT_BL 14

const char* ssid_ap = "ZOYI_703s_PRO";
const char* pass_ap = "12345678";

WebServer server(80); 
TFT_eSPI tft = TFT_eSPI();
TFT_eSprite img = TFT_eSprite(&tft);

String displayValue = "0.0000";
String currentUnit = "V";       
String currentLabel = "DC Voltage"; 
float lastNumericValue = 0.0;
float currentFloat = 0.0;
String serialBuffer = ""; 
const float threshold = 0.0005; 

float valMin = 999999.0;
float valMax = -999999.0;
double valSum = 0.0;
long sampleCount = 0;

void resetStats() { valMin = 999999.0; valMax = -999999.0; valSum = 0.0; sampleCount = 0; }
void updateStats(float val) { if (val < valMin) valMin = val; if (val > valMax) valMax = val; valSum += val; sampleCount++; }
uint16_t getGradientColor(int percent) { percent = constrain(percent, 0, 100); uint8_t r = (31 * percent) / 100; uint8_t g = (63 * (100 - percent)) / 100; return (r << 11) | (g << 5); }

void drawUI() {
    img.fillSprite(TFT_BLACK);
    img.drawRoundRect(5, 5, 470, 310, 15, 0x03E0); 
    img.fillRoundRect(10, 10, 460, 35, 10, 0x0180); 
    img.setTextColor(TFT_CYAN, 0x0180);
    img.drawCentreString("ZOYI 703S - " + currentLabel, 240, 15, 4);
    img.fillRoundRect(20, 55, 440, 110, 15, 0x0841); 
    img.setTextColor(TFT_WHITE);
    img.drawCentreString(displayValue, 243, 73, 7); 
    img.setTextColor(0x07E0); 
    img.drawCentreString(displayValue, 240, 70, 7); 
    img.setTextColor(TFT_GOLD);
    img.drawCentreString(currentUnit, 240, 135, 4);
    int barRange = 5;
    if (abs(currentFloat) > 50.0) barRange = 500; else if (abs(currentFloat) > 5.0) barRange = 50;
    int percent = (int)((abs(currentFloat) * 100.0) / barRange);
    img.drawRect(40, 180, 400, 16, TFT_WHITE);
    img.fillRect(42, 182, map(constrain(percent, 0, 100), 0, 100, 0, 396), 12, getGradientColor(percent));
    img.fillRoundRect(20, 225, 440, 50, 10, 0x0042);
    img.setTextColor(TFT_WHITE, 0x0042);
    img.drawString("MIN: " + String(valMin, 3), 30, 235, 2);
    img.drawCentreString("AVG: " + String((sampleCount==0?0:(float)(valSum/sampleCount)), 3), 240, 235, 4);
    img.drawRightString("MAX: " + String(valMax, 3), 450, 235, 2);
    img.pushSprite(0, 0);
}

void handleRoot() {
    String html = R"rawliteral(
<!DOCTYPE html><html><head><meta charset='UTF-8'>
<style>
    body { background: #111; color: #0f0; text-align: center; font-family: sans-serif; }
    .box { border: 4px solid #333; display: inline-block; padding: 40px; margin: 20px; border-radius: 20px; background: #000; }
    #val { font-size: 100px; font-weight: bold; }
    #unit { font-size: 40px; color: gold; }
    button { padding: 15px 30px; font-size: 18px; cursor: pointer; background: #222; color: #fff; border: 1px solid #555; border-radius: 10px; }
    .status { color: #555; margin-top: 10px; }
</style>
</head><body>
    <h1>ZOYI 703S PRO - AUDIO</h1>
    <div class="box">
        <div id="label">Czekam...</div>
        <div id="val">0.000</div> <span id="unit">---</span>
    </div>
    <br><button onclick="enableTTS()">Aktywuj Głos (TTS)</button>
    <div id="info" class="status">TTS: Wyłączony</div>
    
<script>
    let lastVal = "";
    let stableTime = 0;
    let ttsEnabled = false;
    let lastSpoken = "";

    function enableTTS() { 
        ttsEnabled = true; 
        document.getElementById('info').innerText = "TTS: Aktywny (Pomiar > 1V + Stabilność 1s)";
        document.getElementById('info').style.color = "cyan";
    }

    function update() {
        fetch('/data').then(res => res.json()).then(data => {
            document.getElementById('val').innerText = data.v;
            document.getElementById('unit').innerText = data.u;
            document.getElementById('label').innerText = data.l;
            
            let currentValFloat = parseFloat(data.v);

            // WARUNKI TTS:
            // 1. Wartość musi być identyczna jak poprzednia (stabilizacja)
            // 2. Wartość musi być > 1.000
            // 3. Wartość nie może być tą samą, którą już raz wypowiedzieliśmy (lastSpoken)
            if (data.v === lastVal && currentValFloat > 1.0 && data.v !== lastSpoken) {
                stableTime += 200; // Interwał sprawdzania
                if (stableTime >= 500 && ttsEnabled) {
                    let msg = new SpeechSynthesisUtterance(data.v.replace('.', ',') + " " + data.u);
                    msg.lang = 'pl-PL';
                    window.speechSynthesis.speak(msg);
                    lastSpoken = data.v; // Zapamiętaj, żeby nie powtarzać w kółko
                }
            } else {
                stableTime = 0; // Reset licznika jeśli wartość drgnęła
            }
            lastVal = data.v;
        });
    }
    setInterval(update, 200);
</script>
</body></html>)rawliteral";
    server.send(200, "text/html", html);
}

void handleData() {
    String json = "{\"v\":\"" + displayValue + "\",\"u\":\"" + currentUnit + "\",\"l\":\"" + currentLabel + "\",\"min\":\"" + String(valMin, 3) + "\",\"max\":\"" + String(valMax, 3) + "\"}";
    server.send(200, "application/json", json);
}

void setup() {
    Serial.begin(115200); 
    Serial1.begin(115200, SERIAL_8N1, 18, 17); 
    pinMode(TFT_BL, OUTPUT); digitalWrite(TFT_BL, HIGH);
    SPI.begin(12, -1, 11, 10); 
    tft.init(); tft.setRotation(3);
    img.setColorDepth(8); img.createSprite(480, 320);
    WiFi.softAP(ssid_ap, pass_ap);
    server.on("/", handleRoot);
    server.on("/data", handleData);
    server.begin();
    drawUI(); 
}

void loop() {
    server.handleClient();
    if (Serial1.available()) {
        while (Serial1.available()) {
            char c = (char)Serial1.read();
            serialBuffer += c;
            if (serialBuffer.length() > 128) serialBuffer = serialBuffer.substring(64);
        }
        const char* tags[] = {"Electricity:","mAElectricity:","MOMResistance:","OMResistance:","KOMResistance:","OMbeep:","VDiode:","nFCap:","uFCap:","mFCap:","VVoltage:"};
        for (int i = 0; i < 11; i++) {
            int pos = serialBuffer.indexOf(tags[i]);
            if (pos != -1) {
                String tag = tags[i]; tag.replace(":","");
                int off = strlen(tags[i]);
                String sLabel = "Pomiar", sUnit = "";
                if (tag.indexOf("Voltage")!=-1) { sLabel="Napięcie DC"; sUnit="Wolta"; }
                else if (tag.indexOf("Resistance")!=-1) { sLabel="Rezystancja"; sUnit="Omów"; }
                else if (tag.indexOf("Cap")!=-1) { sLabel="Pojemność"; sUnit="Farady"; }
                else if (tag.indexOf("Electricity")!=-1) { sLabel="Natężenie"; sUnit="Ampery"; }
                int endPos = serialBuffer.indexOf(" ", pos + off);
                if (endPos > (pos + off)) {
                    String val = serialBuffer.substring(pos + off, endPos); val.trim();
                    if (val.length() > 0) {
                        float fv = val.toFloat();
                        if (currentLabel != sLabel) resetStats();
                        if (fabs(fv - lastNumericValue) > threshold || currentLabel != sLabel) {
                            displayValue = val; lastNumericValue = fv; currentFloat = fv;
                            currentLabel = sLabel; currentUnit = sUnit;
                            updateStats(fv); drawUI();
                        }
                    }
                    serialBuffer = serialBuffer.substring(endPos);
                }
                break;
            }
        }
    }
}

```

WEBSERWER:

Nazwe i haslo mozna sobie zmienic na swoja. Pod taka nazwa bedzie wykrywany przez siec wifi.

```C++
const char* ssid_ap = "ZOYI_703s_PRO";
const char* pass_ap = "12345678";

```

Fotka z webserwwera na PC/LAPKU:

<img width="885" height="560" alt="Screenshot 2026-05-06 at 20-50-28 " src="https://github.com/user-attachments/assets/66920aad-6202-49ee-b6b8-a361c6122ad8" />


GLOS TTS. 

Jezeli chodzi o czytanie pomiaru na glos to jest to dopiero pierwsza wersja. Nie jest idealnie i jest do poprawy. Ale jakos tam dziala.



<h1 align="center">📋 INCOMING: CSV</h1>

<img width="703" height="521" alt="Screenshot 2026-05-06 at 21-35-24 " src="https://github.com/user-attachments/assets/fa58f5b9-c33e-4056-b0ea-7c9e36838056" />


<img width="1126" height="2000" alt="2026-05-06_21 33 17" src="https://github.com/user-attachments/assets/15d97e63-cdeb-4846-ad01-f7cef7b53ea8" />

Wrzucam kod aby nie przepadl bo szkoda by bylo :)

```C++
#include <Arduino.h>
#include <WiFi.h>
#include <WebServer.h>
#include <TFT_eSPI.h>
#include <SPI.h>

#define TFT_BL 14

// --- KONFIGURACJA ---
const char* ssid_ap = "ZOYI_703s_PRO";
const char* pass_ap = "12345678";

WebServer server(80); 
TFT_eSPI tft = TFT_eSPI();
TFT_eSprite img = TFT_eSprite(&tft);

// --- DANE POMIAROWE ---
String displayValue = "0.0000";
String currentUnit = "V";       
String currentLabel = "DC Voltage"; 
float lastNumericValue = 0.0;
float currentFloat = 0.0;
String serialBuffer = ""; 
const float threshold = 0.0005; 

// --- STATYSTYKI ---
float valMin = 999999.0;
float valMax = -999999.0;
double valSum = 0.0;
long sampleCount = 0;

void resetStats() {
    valMin = 999999.0; valMax = -999999.0; valSum = 0.0; sampleCount = 0;
}

void updateStats(float val) {
    if (val < valMin) valMin = val;
    if (val > valMax) valMax = val;
    valSum += val;
    sampleCount++;
}

// Kolor od zielonego (0%) do czerwonego (100%)
uint16_t getGradientColor(int percent) {
    percent = constrain(percent, 0, 100);
    uint8_t r = (31 * percent) / 100;
    uint8_t g = (63 * (100 - percent)) / 100;
    return (r << 11) | (g << 5);
}

// --- GUI TFT ---
void drawUI() {
    img.fillSprite(TFT_BLACK);
    
    // Ramka
    img.drawRoundRect(5, 5, 470, 310, 15, 0x03E0); 
    img.fillRoundRect(10, 10, 460, 35, 10, 0x0180); 
    img.setTextColor(TFT_CYAN, 0x0180);
    img.drawCentreString("ZOYI 703S - " + currentLabel, 240, 15, 4);

    // Wyświetlacz główny
    img.fillRoundRect(20, 55, 440, 110, 15, 0x0841); 
    img.setTextColor(TFT_WHITE);
    img.drawCentreString(displayValue, 243, 73, 7); 
    img.setTextColor(0x07E0); 
    img.drawCentreString(displayValue, 240, 70, 7); 
    img.setTextColor(TFT_GOLD);
    img.drawCentreString(currentUnit, 240, 135, 4);

    // Pasek z gradientem
    int barRange = 5;
    if (abs(currentFloat) > 50.0) barRange = 500;
    else if (abs(currentFloat) > 5.0) barRange = 50;
    int percent = (int)((abs(currentFloat) * 100.0) / barRange);
    
    img.drawRect(40, 180, 400, 16, TFT_WHITE);
    img.fillRect(42, 182, map(constrain(percent, 0, 100), 0, 100, 0, 396), 12, getGradientColor(percent));

    // Statystyki dolne
    img.fillRoundRect(20, 225, 440, 50, 10, 0x0042);
    img.setTextColor(TFT_WHITE, 0x0042);
    img.drawString("MIN: " + String(valMin, 3), 30, 235, 2);
    img.drawCentreString("AVG: " + String((sampleCount==0?0:(float)(valSum/sampleCount)), 3), 240, 235, 4);
    img.drawRightString("MAX: " + String(valMax, 3), 450, 235, 2);

    img.pushSprite(0, 0);
}

// --- SERWER WWW ---
void handleRoot() {
    String html = R"rawliteral(
<!DOCTYPE html><html><head><meta charset='UTF-8'>
<style>
    body { background: #111; color: #0f0; text-align: center; font-family: sans-serif; padding: 20px; }
    .box { border: 4px solid #333; display: inline-block; padding: 40px; margin: 20px; border-radius: 20px; background: #000; min-width: 320px; }
    #val { font-size: 85px; font-weight: bold; }
    #unit { font-size: 35px; color: gold; }
    .controls { margin: 20px 0; display: flex; justify-content: center; gap: 10px; flex-wrap: wrap; }
    button { padding: 12px 24px; font-size: 16px; cursor: pointer; background: #222; color: #fff; border: 1px solid #555; border-radius: 10px; }
    button:hover { background: #444; border-color: cyan; }
    #log-info { color: #888; font-size: 0.9em; }
</style>
</head><body>
    <h1 id="label">ZOYI 703S PRO</h1>
    <div class="box">
        <div id="val">0.000</div> <span id="unit">---</span>
    </div>
    <div class="controls">
        <button onclick="enableTTS()">🔊 Głos (TTS)</button>
        <button onclick="downloadCSV()" style="border-color: #0f0;">💾 Pobierz CSV</button>
        <button onclick="clearLogs()" style="border-color: #f00;">🗑️ Czyść</button>
    </div>
    <div id="log-info">Zebrano próbek: 0</div>
<script>
    let lastVal = "", stableTime = 0, ttsEnabled = false, lastSpoken = "", logs = [];
    function enableTTS() { ttsEnabled = true; alert("TTS aktywny (>1V, 1s stabilny)"); }
    function clearLogs() { if(confirm("Usunąć dane?")) { logs = []; updateLog(); } }
    function updateLog() { document.getElementById('log-info').innerText = "Zebrano próbek: " + logs.length; }
    
    function downloadCSV() {
        if(!logs.length) return alert("Brak danych!");
        let csv = "Czas;Etykieta;Wartosc;Jednostka\n" + logs.map(e => `${e.t};${e.l};${e.v};${e.u}`).join("\n");
        let blob = new Blob([csv], {type: 'text/csv;charset=utf-8;'});
        let link = document.createElement("a");
        link.href = URL.createObjectURL(blob);
        link.download = "zoyi_data.csv";
        link.click();
    }

    function update() {
        fetch('/data').then(r => r.json()).then(d => {
            document.getElementById('val').innerText = d.v;
            document.getElementById('unit').innerText = d.u;
            document.getElementById('label').innerText = "ZOYI 703S - " + d.l;
            
            let fv = parseFloat(d.v);
            // Logowanie CSV
            if(d.v !== lastVal && d.v !== "0.0000") {
                logs.push({t: new Date().toLocaleTimeString(), l: d.l, v: d.v.replace('.',','), u: d.u});
                updateLog();
            }
            // TTS
            if(d.v === lastVal && fv > 1.0 && d.v !== lastSpoken) {
                stableTime += 200;
                if(stableTime >= 1000 && ttsEnabled) {
                    let m = new SpeechSynthesisUtterance(d.v.replace('.',',') + " " + d.u);
                    m.lang = 'pl-PL'; window.speechSynthesis.speak(m);
                    lastSpoken = d.v;
                }
            } else { stableTime = 0; }
            lastVal = d.v;
        });
    }
    setInterval(update, 200);
</script></body></html>)rawliteral";
    server.send(200, "text/html", html);
}

void handleData() {
    String json = "{\"v\":\"" + displayValue + "\",\"u\":\"" + currentUnit + "\",\"l\":\"" + currentLabel + "\",\"min\":\"" + String(valMin,3) + "\",\"max\":\"" + String(valMax,3) + "\"}";
    server.send(200, "application/json", json);
}

// --- SETUP & LOOP ---
void setup() {
    Serial.begin(115200); 
    Serial1.begin(115200, SERIAL_8N1, 18, 17); 
    pinMode(TFT_BL, OUTPUT); digitalWrite(TFT_BL, HIGH);
    SPI.begin(12, -1, 11, 10); 
    tft.init();
    
    tft.setRotation(3); // FLIP EKRANU O 180 STOPNI
    
    img.setColorDepth(8); img.createSprite(480, 320);
    WiFi.softAP(ssid_ap, pass_ap);
    server.on("/", handleRoot);
    server.on("/data", handleData);
    server.begin();
    drawUI(); 
}

void loop() {
    server.handleClient();
    if (Serial1.available()) {
        while (Serial1.available()) {
            char c = (char)Serial1.read();
            serialBuffer += c;
            if (serialBuffer.length() > 128) serialBuffer = serialBuffer.substring(64);
        }
        const char* tags[] = {"Electricity:","mAElectricity:","MOMResistance:","OMResistance:","KOMResistance:","OMbeep:","VDiode:","nFCap:","uFCap:","mFCap:","VVoltage:"};
        for (int i = 0; i < 11; i++) {
            int pos = serialBuffer.indexOf(tags[i]);
            if (pos != -1) {
                String tag = tags[i]; tag.replace(":","");
                int off = strlen(tags[i]);
                String sL = "Pomiar", sU = "";
                if (tag.indexOf("Voltage")!=-1) { sL="Napiecie DC"; sU="V"; }
                else if (tag.indexOf("Resistance")!=-1) { sL="Rezystancja"; sU="Ohm"; }
                else if (tag.indexOf("Cap")!=-1) { sL="Pojemnosc"; sU="F"; }
                else if (tag.indexOf("Electricity")!=-1) { sL="Prad"; sU="A"; }

                int ep = serialBuffer.indexOf(" ", pos + off);
                if (ep > (pos + off)) {
                    String v = serialBuffer.substring(pos + off, ep); v.trim();
                    if (v.length() > 0) {
                        float fv = v.toFloat();
                        if (currentLabel != sL) resetStats();
                        if (fabs(fv - lastNumericValue) > threshold || currentLabel != sL) {
                            displayValue = v; lastNumericValue = fv; currentFloat = fv;
                            currentLabel = sL; currentUnit = sU;
                            updateStats(fv); drawUI();
                        }
                    }
                    serialBuffer = serialBuffer.substring(ep);
                }
                break;
            }
        }
    }
}

```

2026.05.07

<img width="1853" height="576" alt="Screenshot 2026-05-07 at 14-53-08 " src="https://github.com/user-attachments/assets/e0e5b94c-2fc4-4a22-8a65-1da40f70fa86" />

KOD:

```C++
#include <Arduino.h>
#include <WiFi.h>
#include <WebServer.h>
#include <TFT_eSPI.h>
#include <SPI.h>

#define TFT_BL 14

// --- KONFIGURACJA ---
const char* ssid_ap = "ZOYI_703s_PRO";
const char* pass_ap = "12345678";

WebServer server(80); 
TFT_eSPI tft = TFT_eSPI();
TFT_eSprite img = TFT_eSprite(&tft);

// --- DANE POMIAROWE ---
String displayValue = "0.0000";
String currentUnit = "V";       
String currentLabel = "DC Voltage"; 
float lastNumericValue = 0.0;
float currentFloat = 0.0;
String serialBuffer = ""; 
const float threshold = 0.0005; 

// --- STATYSTYKI ---
float valMin = 999999.0;
float valMax = -999999.0;
double valSum = 0.0;
long sampleCount = 0;

void resetStats() {
    valMin = 999999.0; valMax = -999999.0; valSum = 0.0; sampleCount = 0;
}

void updateStats(float val) {
    if (val < valMin) valMin = val;
    if (val > valMax) valMax = val;
    valSum += val;
    sampleCount++;
}

uint16_t getGradientColor(int percent) {
    percent = constrain(percent, 0, 100);
    uint8_t r = (31 * percent) / 100;
    uint8_t g = (63 * (100 - percent)) / 100;
    return (r << 11) | (g << 5);
}

// --- GRAFIKA TFT ---
void drawUI() {
    img.fillSprite(TFT_BLACK);
    img.drawRoundRect(5, 5, 470, 310, 15, 0x03E0); 
    img.fillRoundRect(10, 10, 460, 35, 10, 0x0180); 
    img.setTextColor(TFT_CYAN, 0x0180);
    img.drawCentreString("ZOYI 703S - " + currentLabel, 240, 15, 4);

    img.fillRoundRect(20, 55, 440, 110, 15, 0x0841); 
    img.setTextColor(TFT_WHITE);
    img.drawCentreString(displayValue, 243, 73, 7); 
    img.setTextColor(0x07E0); 
    img.drawCentreString(displayValue, 240, 70, 7); 
    img.setTextColor(TFT_GOLD);
    img.drawCentreString(currentUnit, 240, 135, 4);

    int barRange = 5;
    if (abs(currentFloat) > 50.0) barRange = 500;
    else if (abs(currentFloat) > 5.0) barRange = 50;
    int percent = (int)((abs(currentFloat) * 100.0) / barRange);
    
    img.drawRect(40, 180, 400, 16, TFT_WHITE);
    img.fillRect(42, 182, map(constrain(percent, 0, 100), 0, 100, 0, 396), 12, getGradientColor(percent));

    img.setTextColor(TFT_SILVER);
    for(int i=0; i<=4; i++) {
        int x = 40 + (i * 100);
        img.drawFastVLine(x, 196, 5, TFT_SILVER);
        img.drawCentreString(String((barRange * i) / 4), x, 202, 1);
    }

    img.fillRoundRect(20, 225, 440, 50, 10, 0x0042);
    img.setTextColor(TFT_WHITE, 0x0042);
    img.drawString("MIN: " + String(valMin, 3), 30, 235, 2);
    img.drawCentreString("AVG: " + String((sampleCount==0?0:(float)(valSum/sampleCount)), 3), 240, 235, 4);
    img.drawRightString("MAX: " + String(valMax, 3), 450, 235, 2);

    img.pushSprite(0, 0);
}

// --- STRONA WWW ---
void handleRoot() {
    String html = R"rawliteral(
<!DOCTYPE html><html><head><meta charset='UTF-8'>
<meta name="viewport" content="width=device-width, initial-scale=1">
<style>
    body { background: #111; color: #0f0; text-align: center; font-family: sans-serif; padding: 10px; transition: background 0.3s; }
    /* Styl dla alarmu - migające tło */
    body.alarm-active { background: #600 !important; } 
    .box { border: 4px solid #333; display: inline-block; padding: 25px; margin: 10px; border-radius: 20px; background: #000; min-width: 320px; }
    #val { font-size: 70px; font-weight: bold; margin: 10px 0; }
    #unit { font-size: 30px; color: gold; display: block; }
    .progress-container { width: 100%; background: #222; border-radius: 10px; height: 25px; margin: 20px 0; border: 1px solid #555; overflow: hidden; position: relative; }
    #progress-bar { width: 0%; height: 100%; background: #0f0; transition: width 0.2s; }
    .controls { margin: 20px 0; display: flex; justify-content: center; gap: 10px; flex-wrap: wrap; }
    button { padding: 15px 20px; font-size: 16px; cursor: pointer; background: #222; color: #fff; border: 1px solid #555; border-radius: 10px; flex: 1; min-width: 140px; }
    #tts-btn.active, #beep-btn.active { background: #005500; border-color: #0f0; box-shadow: 0 0 10px #0f0; }
    #log-info { color: #888; font-size: 1em; margin-top: 10px; }
    .alarm-settings { margin: 10px; color: #f44; font-weight: bold; }
    input { background: #000; color: #fff; border: 1px solid #555; padding: 5px; width: 60px; border-radius: 5px; text-align: center; }
</style>
</head><body>
    <h1 id="label">ZOYI 703S PRO</h1>
    <div class="alarm-settings">
        ALARM POWYŻEJ: <input type="number" id="alarm-limit" value="5.0" step="0.1"> V
    </div>
    <div class="box">
        <div id="val">0.0000</div>
        <span id="unit">---</span>
        <div class="progress-container">
            <div id="progress-bar"></div>
        </div>
    </div>
    <div class="controls">
        <button id="beep-btn" onclick="toggleBeep()">🔔 Beep: OFF</button>
        <button id="tts-btn" onclick="toggleTTS()">🔊 Głos (TTS): OFF</button>
        <button onclick="downloadCSV()" style="border-color: #0f0;">💾 Pobierz CSV</button>
        <button onclick="clearLogs()" style="border-color: #f00;">🗑️ Czyść Dane</button>
    </div>
    <div id="log-info">Zebrano próbek: 0</div>
<script>
    let lastVal = "", stableTime = 0, ttsEnabled = false, beepEnabled = false, lastSpoken = "", logs = [];
    const audioCtx = new (window.AudioContext || window.webkitAudioContext)();

    function playBeep(freq = 880, duration = 0.2) {
        if (!beepEnabled) return;
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.connect(gain);
        gain.connect(audioCtx.destination);
        osc.frequency.value = freq;
        osc.start();
        gain.gain.exponentialRampToValueAtTime(0.0001, audioCtx.currentTime + duration);
        osc.stop(audioCtx.currentTime + duration);
    }

    function toggleBeep() {
        beepEnabled = !beepEnabled;
        const btn = document.getElementById('beep-btn');
        btn.innerText = beepEnabled ? "🔔 Beep: ON" : "🔔 Beep: OFF";
        btn.classList.toggle('active', beepEnabled);
        if(beepEnabled && audioCtx.state === 'suspended') audioCtx.resume();
    }

    function toggleTTS() {
        ttsEnabled = !ttsEnabled;
        const btn = document.getElementById('tts-btn');
        btn.innerText = ttsEnabled ? "🔊 Głos (TTS): ON" : "🔊 Głos (TTS): OFF";
        btn.classList.toggle('active', ttsEnabled);
        if(ttsEnabled) speak("Głos włączony"); else window.speechSynthesis.cancel();
    }

    function speak(text) {
        let m = new SpeechSynthesisUtterance(text);
        m.lang = 'pl-PL'; window.speechSynthesis.speak(m);
    }

    function clearLogs() {
        if(confirm("Wyczyścić dane?")) { logs = []; updateLogCount(); }
    }

    function updateLogCount() {
        document.getElementById('log-info').innerText = "Zebrano próbek: " + logs.length;
    }

    function downloadCSV() {
        if(!logs.length) return alert("Brak danych!");
        let csv = "Czas;Etykieta;Wartosc;Jednostka\n" + logs.map(e => `${e.t};${e.l};${e.v};${e.u}`).join("\n");
        let blob = new Blob([csv], {type: 'text/csv;charset=utf-8;'});
        let link = document.createElement("a");
        link.href = URL.createObjectURL(blob);
        link.download = "pomiary_zoyi.csv"; link.click();
    }

    function update() {
        fetch('/data').then(r => r.json()).then(d => {
            document.getElementById('val').innerText = d.v;
            document.getElementById('unit').innerText = d.u;
            document.getElementById('label').innerText = "ZOYI 703S - " + d.l;
            
            let fv = parseFloat(d.v);
            let limit = parseFloat(document.getElementById('alarm-limit').value);
            
            // LOGIKA ALARMU
            if (d.u === "V" && Math.abs(fv) > limit) {
                document.body.classList.add('alarm-active');
                playBeep(1000, 0.1); // Krótki sygnał dźwiękowy
            } else {
                document.body.classList.remove('alarm-active');
            }
            
            let absFv = Math.abs(fv);
            let range = absFv > 50 ? 500 : (absFv > 5 ? 50 : 5);
            let p = Math.min((absFv * 100) / range, 100);
            
            let bar = document.getElementById('progress-bar');
            bar.style.width = p + "%";
            let r = Math.floor((255 * p) / 100);
            let g = Math.floor((255 * (100 - p)) / 100);
            bar.style.background = `rgb(${r},${g},0)`;

            if(d.v !== lastVal && d.v !== "0.0000") {
                logs.push({t: new Date().toLocaleTimeString(), l: d.l, v: d.v.replace('.',','), u: d.u});
                updateLogCount();
            }

            if(ttsEnabled && d.v === lastVal && absFv > 1.0 && d.v !== lastSpoken) {
                stableTime += 200;
                if(stableTime >= 1000) {
                    speak(d.v.replace('.',',') + " " + d.u);
                    lastSpoken = d.v;
                }
            } else if (d.v !== lastVal) { stableTime = 0; }
            lastVal = d.v;
        });
    }
    setInterval(update, 200);
</script></body></html>)rawliteral";
    server.send(200, "text/html", html);
}

void handleData() {
    String json = "{\"v\":\"" + displayValue + "\",\"u\":\"" + currentUnit + "\",\"l\":\"" + currentLabel + "\"}";
    server.send(200, "application/json", json);
}

void setup() {
    Serial.begin(115200); 
    Serial1.begin(115200, SERIAL_8N1, 18, 17); 
    pinMode(TFT_BL, OUTPUT); digitalWrite(TFT_BL, HIGH);
    SPI.begin(12, -1, 11, 10); 
    tft.init();
    tft.setRotation(3); 
    img.setColorDepth(8); img.createSprite(480, 320);
    WiFi.softAP(ssid_ap, pass_ap);
    server.on("/", handleRoot);
    server.on("/data", handleData);
    server.begin();
    drawUI(); 
}

void loop() {
    server.handleClient();
    if (Serial1.available()) {
        while (Serial1.available()) {
            char c = (char)Serial1.read();
            serialBuffer += c;
            if (serialBuffer.length() > 128) serialBuffer = serialBuffer.substring(64);
        }
        const char* tags[] = {"Electricity:","mAElectricity:","MOMResistance:","OMResistance:","KOMResistance:","OMbeep:","VDiode:","nFCap:","uFCap:","mFCap:","VVoltage:"};
        for (int i = 0; i < 11; i++) {
            int pos = serialBuffer.indexOf(tags[i]);
            if (pos != -1) {
                String tag = tags[i]; tag.replace(":","");
                int off = strlen(tags[i]);
                String sL = "Pomiar", sU = "";
                if (tag.indexOf("Voltage")!=-1) { sL="Napiecie DC"; sU="V"; }
                else if (tag.indexOf("Resistance")!=-1) { sL="Rezystancja"; sU="Ohm"; }
                else if (tag.indexOf("Cap")!=-1) { sL="Pojemnosc"; sU="F"; }
                else if (tag.indexOf("Electricity")!=-1) { sL="Prad"; sU="A"; }

                int ep = serialBuffer.indexOf(" ", pos + off);
                if (ep > (pos + off)) {
                    String v = serialBuffer.substring(pos + off, ep); v.trim();
                    if (v.length() > 0) {
                        float fv = v.toFloat();
                        if (currentLabel != sL) resetStats();
                        if (fabs(fv - lastNumericValue) > threshold || currentLabel != sL) {
                            displayValue = v; lastNumericValue = fv; currentFloat = fv;
                            currentLabel = sL; currentUnit = sU;
                            updateStats(fv); drawUI();
                        }
                    }
                    serialBuffer = serialBuffer.substring(ep);
                }
                break;
            }
        }
    }
}

```






































