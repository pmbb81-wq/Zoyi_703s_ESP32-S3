
<h1 align="center">📋 Zoyi703s ESP32-S3</h1>

Projekcik dla Zoyi703s na ESP32-s3. Czyli odczyt z UART z zoyi do esp.

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





































