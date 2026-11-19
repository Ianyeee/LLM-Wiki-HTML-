# 05. 資料視覺化與儀表板實作 (Data Visualization & Real-time Dashboards)

> **知識庫編號**：`WIKI-05`  
> **所屬領域**：前端視覺化與人機介面 (Frontend Visualization & HMI)  
> **維護人員**：王庭翊 (Hardware Lab Techo)  
> **更新日期**：2026-11-19  

---

## 1. 核心概念 (Core Concepts)

再精密的物理感測器與再快速的通訊協定，如果最終產出的只是一行行在黑底視窗中跳動的冷冰冰文字代碼，一般使用者或機房維運人員依然難以在第一時間察覺異常趨勢。

**資料視覺化（Data Visualization）** 是物聯網架構中屬於「最後一哩路（Last Mile）」的關鍵環節。它的核心價值在於：將時序性的數值矩陣，透過圖形化介面（趨勢折線圖、量表刻度指針、警示色標）轉化為人類大腦能夠**瞬間感知、直觀理解並做出決策**的視覺資訊。

---

## 2. 數據到畫面的三大轉換步驟 (Visualization Pipeline)

當帶有感測數值的 JSON 封包抵達終端時，必須經歷以下 3 個連續處理階段：

```
+-------------------------------------------------------------------------------+
|                       前端即時視覺化處理流程管道                              |
|                                                                               |
|  [步驟 1: 傳輸與接收]       [步驟 2: 資料解析與過濾]        [步驟 3: 動態渲染圖表]    |
|   MQTT over WebSockets ──▶  JSON.parse(message)   ──▶ Chart.js Canvas Update  |
|   (持續監聽 Topic 訊息)       (提取數值、時間戳與狀態)      (折線推移、顏色警戒更新)  |
+-------------------------------------------------------------------------------+
```

### 步驟 1：傳輸與即時接收 (Transport & Ingestion)
傳統 HTTP 輪詢（Polling）會造成大量不必要的重複連線。在現代物聯網網頁中，普遍採用 **MQTT over WebSockets**。瀏覽器在頁面載入時建立一次持久性的雙向通道，每當 ESP32 發布新資料，MQTT Broker 便立刻推播至瀏覽器，達到零延遲更新。

### 步驟 2：解析格式與邊界過濾 (Parsing & Sanitization)
前端在收到文字訊息後，利用 `JSON.parse()` 將字串還原為 JavaScript 物件。此階段需進行**資料驗證**（例如檢查數值是否為 `NaN`、PM2.5 是否落在合理物理區間 $0 \sim 1000\,\mu g/m^3$），避免髒資料導致前端圖表崩潰。

### 步驟 3：動態繪製與圖表推移 (Dynamic Chart Rendering)
將清洗後的數據推送至時序陣列。若資料筆數超過上限（如最近 30 筆），則透過 FIFO（先進先出）佇列彈出最舊資料，並呼叫繪圖引擎進行重繪（Chart Update）。

---

## 3. 常見視覺化工具與技術選型比較

| 工具 / 框架 | 部署位置 | 運作特點 | 適用情境 |
| :--- | :--- | :--- | :--- |
| **Chart.js** | 瀏覽器前端 (Web) | 基於 HTML5 Canvas，體積輕巧、動畫平滑、客製化極高 | 個人專案網頁、嵌入式客製化儀表板 (首選) |
| **Node-RED Dashboard** | 本地邊緣端 (Raspberry Pi) | 低程式碼 (Low-Code) 流程圖節點拖拉，內建多種 UI 元件 | 實驗室原型快速驗證、工業現場本機監控看板 |
| **Grafana** | 雲端 / 伺服器 (Server) | 企業級監控平台，需搭配 InfluxDB / Prometheus 時序資料庫 | 跨校區大規模感測節點監控、長時間歷史資料分析 |

---

## 4. 關鍵前端實作範例 (HTML5 Canvas + Chart.js)

```javascript
// 建立即時 PM2.5 平滑折線圖
const ctx = document.getElementById('pm25Chart').getContext('2d');
const pm25Chart = new Chart(ctx, {
  type: 'line',
  data: {
    labels: [], // 時間標籤
    datasets: [{
      label: 'PM2.5 濃度 (μg/m³)',
      data: [],
      borderColor: '#1F7A76', // 使用專案 PCB 松石綠
      backgroundColor: 'rgba(31, 122, 118, 0.1)',
      tension: 0.35,           // 平滑貝茲曲線
      fill: true
    }]
  },
  options: {
    responsive: true,
    scales: {
      y: { min: 0, max: 100, grid: { color: 'rgba(30, 41, 59, 0.08)' } }
    }
  }
});

// 當 MQTT 收到新封包時更新圖表
function onSensorMessage(newPm25, timeString) {
  if (pm25Chart.data.labels.length >= 20) {
    pm25Chart.data.labels.shift();
    pm25Chart.data.datasets[0].data.shift();
  }
  pm25Chart.data.labels.push(timeString);
  pm25Chart.data.datasets[0].data.push(newPm25);
  pm25Chart.update();
}
```

---

## 5. 新手生活化比喻 (Novice Analogy)

> 💡 **就像超跑的「全數位多功能儀表板」**：  
> 當你開車在高速公路上奔馳時，引擎每秒燃燒多少滴汽油、渦輪增壓閥的電壓是幾毫伏特，這些複雜的內部電子訊號都不需要你分心理解。  
> 儀表板把底層錯綜複雜的訊號，直接化為「時速表上的數字」以及「胎壓異常時亮起的紅色警示燈」。你只要輕瞄一眼，立刻知道目前的車況安不安全！  

