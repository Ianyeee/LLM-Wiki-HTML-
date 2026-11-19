# 04. 物聯網資料格式 JSON (JSON Data Formatting in IoT)

> **知識庫編號**：`WIKI-04`  
> **所屬領域**：資料序列化與封包格式 (Data Serialization & Message Payload)  
> **維護人員**：王庭翊 (Hardware Lab Techo)  
> **更新日期**：2026-11-19  

---

## 1. 核心概念 (Core Concepts)

**JSON（JavaScript Object Notation，JavaScript 物件標記法）** 是一種以純文字為基礎的輕量級資料交換格式。它採用由鍵值對（Key-Value Pairs）組成的物件結構，以及由中括號包覆的陣列結構。

在早期的網路通訊中，XML 曾經是主流，但其冗長繁瑣的標籤定義（如 `<temperature>25.6</temperature>`）大量浪費頻寬；而純二進位格式（Binary Struct）雖然極致節省空間，卻存在「跨處理器大小端位元組序（Big-Endian vs. Little-Endian）相容問題」且工程師無法用肉眼直接除錯。

JSON 正是在**「輕量傳輸效率」**、**「跨平台與跨語言相容性」**、以及**「開發與除錯便利性」**這三大矛盾維度之間，取得了工業界公認的最佳甜蜜點。

---

## 2. 為什麼物聯網系統首選 JSON？

```
                   [ 跨語言跨平台支援 ]
             (C, Python, JS, Go, Rust 均原生支援)
                           ▲
                           │
                  ★ JSON 最佳甜蜜點 ★
                  /                 \
                 ▼                   ▼
    [ 輕量無贅述的傳輸效率 ]     [ 人類可讀與除錯友善性 ]
      (純文字無 XML 標籤冗餘)      (直觀 Key-Value，Log 一眼看懂)
```

1. **跨程式語言無縫互通 (Universal Compatibility)**：
   在我們的系統中，ESP32 韌體使用 **C/C++ (ArduinoJson)** 打包封包；Raspberry Pi 伺服器使用 **Python (json 模組)** 解析儲存；而使用者的網頁儀表板則使用 **JavaScript (JSON.parse)** 即時渲染圖表。三個不同生態系的語言無需轉換協定，共享同一個資料標準。
2. **自我描述性高 (Self-describing)**：
   欄位名稱直接寫在封包中，例如 `"pm25": 18.4`，後續若新增欄位（如 `"co2": 450`），舊有的接收程式碼只要忽略未定義的 Key 即可運作，具備優異的向下擴充彈性。
3. **支援階層與巢狀結構 (Nested Structure)**：
   能夠將感測數值、硬體狀態與診斷旗標完美分層包裝在同一個 Payload 之中。

---

## 3. 本專案真實感測器 Payload 封包結構

以下為 ESP32 在發送至 MQTT Broker 時所產生的標準 JSON 封包格式：

```json
{
  "device_id": "ESP32_LAB_01",
  "timestamp": 1732018800,
  "data": {
    "pm25": 24.5,
    "pm10": 38.2,
    "temperature": 26.8,
    "humidity": 58.2
  },
  "status": {
    "wifi_rssi": -62,
    "fan_duty": 45,
    "alert_level": "NORMAL"
  }
}
```

### 欄位解析與型態說明：
- `device_id` *(String)*：感測節點的全球唯一識別碼，利於多節點管理。
- `timestamp` *(Integer)*：UNIX 時間戳記（秒），確保時間軸排序一致。
- `data.pm25` *(Float)*：SPS30 感測器計算後的質量濃度數值 ($\mu g/m^3$)。
- `status.wifi_rssi` *(Integer)*：Wi-Fi 接收訊號強度指標（dBm），供遠端網路健檢。
- `status.alert_level` *(String)*：由微控制器初步判斷的警示等級（`NORMAL`, `WARNING`, `DANGER`）。

---

## 4. 嵌入式 C 與前端 JavaScript 處理對照

```c
// ESP32 端：使用 ArduinoJson 函式庫序列化 (Serialize)
StaticJsonDocument<256> doc;
doc["device_id"] = "ESP32_LAB_01";
doc["data"]["pm25"] = readSPS30();
char output[256];
serializeJson(doc, output);
mqttClient.publish("lab/env", output);
```

```javascript
// 前端網頁端：使用原生 JavaScript 反序列化 (Deserialize)
client.on('message', (topic, message) => {
  const payload = JSON.parse(message.toString());
  console.log(`收到節點: ${payload.device_id}, PM2.5 數值: ${payload.data.pm25}`);
  updateDashboardChart(payload.data.pm25);
});
```

---

## 5. 新手生活化比喻 (Novice Analogy)

> 💡 **國際標準化的「宅配包裹托運單」**：  
> - **Raw Data（一串雜亂的二進位字串）**：就像寄件人把一堆零件胡亂塞在箱子裡，上面什麼都沒寫，收件人打開來根本不知道哪個螺絲是配哪個螺栓。  
> - **JSON 格式**：就像在包裹外面貼上一張格式規範統一的「黑貓 / 順豐托運單」。上面清楚印著【寄件人（Key）：ESP32】、【物品名稱（Key）：PM2.5】、【測量數值（Value）：24.5】。不管快遞員講中文、日文還是英文，一眼看過托運單就知道包裹裝什麼、該怎麼配送！  

