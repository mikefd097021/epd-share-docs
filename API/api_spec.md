# ESL Management System — Authentication + ⭐⭐⭐⭐ / ⭐⭐⭐⭐⭐ API

本文件分為：

1. **Token 驗證方式（完整解說 + 使用方式）**
2. **RESTful API（5 星 / 4 星）結構化文件**
3. **JSON Request / Response 範本**

---

# #1 Authentication（Token 驗證）

系統提供三種 Token 驗證方式：

---

## ## 1.1 **Method A — Cookie Token（自家系統 / Dashboard）**

### **Header**

```
Cookie: token=YOUR_SESSION_TOKEN
```

### 特點

* 適用：Web 管理後台、內部系統、即時性高的 App / Web
* 支援 WS + REST
* 安全性高（可搭配 HttpOnly + Secure + SameSite）

### Request 範例

```
GET /api/v1/stores/001/items
Cookie: token=abc123def456
```

---

## ## 1.2 **Method B — Bearer Token（給外部開發者、第三方 App）**

### **Header**

```
Authorization: Bearer YOUR_API_TOKEN
```

### 特點

* 適用：外部 App、Node / Python / Go、IoT Client
* 更通用，不需瀏覽器環境
* 推薦用於 API Key 模式

### Request 範例

```
GET /api/v1/devices/11:22:33:44:55:66
Authorization: Bearer xyz789token
```

---

## ## 1.3 **Method C — URL Token（測試用）**

### **URL**

```
GET https://api.example.com/api/v1/stores/001/items?token=YOUR_TOKEN
```

### 特點

* 最低安全性
* 只建議：

  * 內部測試
  * 暫時性工具
  * 不能帶 Header 的簡易設備

---

## ## 1.4 後端驗證邏輯

```
if cookie.token exists → validate session
else if Authorization: Bearer exists → validate api-key
else if query token exists → validate token
else → reject 401
```

---

## ## 1.5 WebSocket Token 驗證推薦

WebSocket 可以同時支援：

* Cookie
* Authorization Bearer

### WebSocket 連線範例（Cookie）

```
wss://api.example.com/ws
Cookie: token=abc123
```

### WebSocket 連線範例（Bearer）

```
wss://api.example.com/ws
Authorization: Bearer xyz789
```

---

---

# #2 ⭐⭐⭐⭐⭐（5 星）核心 API

---

## ## 3.2 Add Store Data

新增商品資料（建檔）

### **HTTP**

```
POST /api/v1/stores/{storeId}/items
```

### **Request JSON**

```json
{
  "itemId": "item001",
  "name": "雪碧 600ml",
  "price": 28.0,
  "spec": "600ml",
  "brand": "Sprite",
  "promo": false
}
```

### **Response JSON**

```json
{
  "success": true,
  "message": "Item created.",
  "itemId": "item001"
}
```

---

## ## 3.3 Modify Data (Brush)

修改商品資料並刷新電子紙

### **HTTP**

```
PUT /api/v1/stores/{storeId}/items/{itemId}
```

### **Request JSON**

```json
{
  "updatedFields": {
    "price": 25.0,
    "promo": true
  },
  "brush": true
}
```

### **Response JSON**

```json
{
  "success": true,
  "message": "Item updated and ESL refreshed."
}
```

---

## ## 3.8 Batch Add / Modify & Brush

批量新增或修改商品並刷新

### **HTTP**

```
POST /api/v1/stores/{storeId}/items/batch
```

### **Request JSON**

```json
{
  "items": [
    {
      "itemId": "item001",
      "name": "飲料 A",
      "price": 20
    },
    {
      "itemId": "item002",
      "price": 18
    }
  ],
  "forceBrush": true,
  "ledBlinkOption": false
}
```

### **Response JSON**

```json
{
  "success": true,
  "processed": 2,
  "details": [
    { "itemId": "item001", "status": "created" },
    { "itemId": "item002", "status": "updated" }
  ]
}
```

---

# #3 ⭐⭐⭐⭐（4 星）高頻 API

---

## ## 2.4 Get Warning Info

取得告警（低電量、離線）

### **HTTP**

```
GET /api/v1/stores/{storeId}/warnings?type={type}
```

### **Query Parameters**

| 名稱   | 說明                         |
| ---- | -------------------------- |
| type | lowBattery / offline / all |

### **Response JSON**

```json
{
  "storeId": "001",
  "warnings": [
    {
      "mac": "AB:CD:EF:00:11:22",
      "type": "offline",
      "lastSeen": "2025-02-01T10:20:30Z"
    },
    {
      "mac": "AB:CD:EF:00:AA:BB",
      "type": "lowBattery",
      "battery": 10
    }
  ]
}
```

---

## ## 3.5 Delete Data

刪除商品資料（下架）

### **HTTP**

```
DELETE /api/v1/stores/{storeId}/items/{itemId}
```

### **Response JSON**

```json
{
  "success": true,
  "message": "Item deleted."
}
```

---

## ## 6.4 Get Device List

取得門店全部設備（含狀態）

### **HTTP**

```
GET /api/v1/stores/{storeId}/devices?page=1&size=50&status=all
```

### Query 參數

| 名稱     | 說明                                            |
| ------ | --------------------------------------------- |
| page   | 頁數                                            |
| size   | 每頁數量                                          |
| status | online / offline / lowBattery / unbound / all |

### Response JSON

```json
{
  "storeId": "001",
  "page": 1,
  "size": 50,
  "total": 212,
  "devices": [
    {
      "mac": "11:22:33:44:55:66",
      "status": "online",
      "battery": 87,
      "signal": -70,
      "lastUpdate": "2025-02-05T12:11:22Z",
      "boundItemId": "4712345678901"
    },
    {
      "mac": "AA:BB:CC:DD:EE:FF",
      "status": "offline",
      "battery": 52,
      "signal": -100,
      "lastUpdate": "2025-01-30T09:12:00Z",
      "boundItemId": null
    }
  ]
}
```

---

## ## 6.13 Query Device Info

查詢單一電子紙資訊

### **HTTP**

```
GET /api/v1/devices/{mac}
```

### **Response JSON**

```json
{
  "mac": "11:22:33:44:55:66",
  "storeId": "001",
  "model": "ESL-2.9",
  "battery": 85,
  "signal": -68,
  "firmware": "v1.4.3",
  "lastUpdate": "2025-02-05T12:11:22Z",
  "boundItem": {
    "itemId": "item001",
    "name": "雪碧 600ml",
    "price": 28
  }
}
```

---

# #4 📌 五星/四星 API 總表

| 星級    | API 名稱              | Path                                    |
| ----- | ------------------- | --------------------------------------- |
| ⭐⭐⭐⭐⭐ | Add Store Data      | POST /stores/{storeId}/items            |
| ⭐⭐⭐⭐⭐ | Modify Data & Brush | PUT /stores/{storeId}/items/{itemId}    |
| ⭐⭐⭐⭐⭐ | Batch Add/Modify    | POST /stores/{storeId}/items/batch      |
| ⭐⭐⭐⭐  | Get Warning Info    | GET /stores/{storeId}/warnings          |
| ⭐⭐⭐⭐  | Delete Data         | DELETE /stores/{storeId}/items/{itemId} |
| ⭐⭐⭐⭐  | Get Device List     | GET /stores/{storeId}/devices           |
| ⭐⭐⭐⭐  | Query Device Info   | GET /devices/{mac}                      |

---