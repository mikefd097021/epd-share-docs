# ⭐⭐⭐ ESL Management System — API（完整修正版）

以下為 **進階應用（Level 3 APIs）**，涵蓋查詢、綁定、批次操作、監控等常用功能。

---

# 📘 目錄（3星 API）

1. [2.5 Get store info](#25-get-store-info)
2. [3.4 Batch modify data](#34-batch-modify-data-no-brush)
3. [3.6 Get data info](#36-get-data-info)
4. [3.7 Query binding tag](#37-query-binding-tag)
5. [4.1 Query template list](#41-query-template-list)
6. [6.5 Bind devices](#65-bind-devices)
7. [6.6 Delete binding](#66-delete-binding)
8. [6.8 Refresh tags batch](#68-refresh-tags-batch)
9. [6.12 Query bound data by MAC](#612-query-bound-data-by-mac)
10. [6.15 Query unbound/bound device list](#615-query-unboundbound-device-list)
11. [6.18 Bind devices with template (multi-data) — deprecated](#618-bind-devices-with-template-multi-data-deprecated)

---

# ## 2.5 Get store info

**GET /api/store/{storeId}**

取得門店基本資訊。

### Response

```json
{
  "storeId": "A001",
  "storeName": "台北一店",
  "status": "open",
  "createdAt": "2024-12-01T10:00:00Z"
}
```

---

# ## 3.4 Batch modify data (no brush)

**PUT /api/store/{storeId}/data/batch**

批量修改門店資料（但不刷新電子紙）。

### Request

```json
{
  "items": [
    {
      "productId": "P001",
      "price": 199,
      "promo": "限時優惠"
    },
    {
      "productId": "P002",
      "price": 89
    }
  ]
}
```

### Response

```json
{
  "updated": 2,
  "status": "success"
}
```

---

# ## 3.6 Get data info

**GET /api/store/{storeId}/data/{productId}**

查詢某筆門店資料內容（非電子紙目前呈現值）。

### Response

```json
{
  "productId": "P001",
  "name": "牛奶 1L",
  "price": 199,
  "promo": "限時優惠",
  "lastUpdate": "2024-12-10T09:00:00Z"
}
```

---

# ## 3.7 Query binding tag

查詢某筆資料目前綁定到哪些電子紙（可能多個）。

**GET /api/store/{storeId}/data/{productId}/binding**

### Response

```json
{
  "productId": "P001",
  "tags": [
    {
      "tagMac": "AC:12:FF:23:90:11",
      "templateId": "T01"
    },
    {
      "tagMac": "AC:12:FF:23:90:12",
      "templateId": "T02"
    }
  ]
}
```

---

# ## 4.1 Query template list（含動態欄位）

**GET /api/store/{storeId}/templates**

### 支援 Query Filter（optional）

| 參數    | 說明    | 範例              |
| ----- | ----- | --------------- |
| size  | 電子紙尺寸 | 2.9inch         |
| color | 顏色模式  | black-white-red |
| name  | 名稱關鍵字 | promo           |

### Response

```json
{
  "storeId": "A001",
  "filters": {
    "size": "2.9inch",
    "color": "black-white-red"
  },
  "templates": [
    {
      "templateId": "T01",
      "name": "一般版型",
      "size": "2.9inch",
      "color": "black-white-red",
      "fields": [
        { "fieldKey": "productName", "fieldType": "text",   "required": true },
        { "fieldKey": "price",       "fieldType": "number", "required": true },
        { "fieldKey": "promo",       "fieldType": "text",   "required": false }
      ]
    }
  ]
}
```

---

# ## 6.5 Bind devices（使用 StoreData 欄位 ID綁定，支援多電子紙一次操作）

**POST /api/store/{storeId}/binding**

> Fields 不存值！
> **只存 StoreData 的欄位 ID（如 SD123-F001）**

### Request（方案 B — 每個 Tag 可指定不同 template 與 fields）

```json
{
  "items": [
    {
      "tagMac": "AC:12:FF:23:90:11",
      "templateId": "T01",
      "fields": {
        "productName": "SD123-F001",
        "price": "SD123-F002",
        "promo": "SD123-F003"
      }
    },
    {
      "tagMac": "AC:12:FF:23:90:12",
      "templateId": "T02",
      "fields": {
        "productName": "SD555-F010",
        "price": "SD555-F011"
      }
    }
  ]
}
```

### Response

```json
{
  "status": "success",
  "bindings": [
    {
      "bindingId": "BIND-20250101-0001",
      "tagMac": "AC:12:FF:23:90:11",
      "templateId": "T01",
      "fields": {
        "productName": "SD123-F001",
        "price": "SD123-F002",
        "promo": "SD123-F003"
      }
    },
    {
      "bindingId": "BIND-20250101-0002",
      "tagMac": "AC:12:FF:23:90:12",
      "templateId": "T02",
      "fields": {
        "productName": "SD555-F010",
        "price": "SD555-F011"
      }
    }
  ]
}
```

---

# ## 6.6 Delete binding

**DELETE /api/store/{storeId}/binding/{tagMac}**

### Response

```json
{
  "status": "success",
  "tagMac": "AC:12:FF:23:90:11",
  "unbound": true
}
```

---

# ## 6.8 Refresh tags batch

**POST /api/store/{storeId}/tags/refresh**

### Request

```json
{
  "tagList": [
    "AC:12:FF:23:90:11",
    "AC:12:FF:23:90:22"
  ]
}
```

### Response

```json
{
  "status": "queued",
  "refreshCount": 2
}
```

---

# ## 6.12 Query bound data by MAC

### **GET /api/store/{storeId}/tag/{mac}/binding**

### Response（包含 store data 欄位）

```json
{
  "tagMac": "AC:12:FF:23:90:11",
  "templateId": "T01",
  "fields": {
    "productName": "SD123-F001",
    "price": "SD123-F002",
    "promo": "SD123-F003"
  },
  "lastBrush": "2024-12-10T09:00:00Z"
}
```

---

# ## 6.15 Query unbound/bound device list

### **GET /api/store/{storeId}/devices/binding**

### Response（包含 store data 資料綁定內容）

```json
{
  "unbound": [
    "AC:12:FF:00:11:22",
    "AC:12:FF:00:11:33"
  ],
  "bound": [
    {
      "tagMac": "AC:12:FF:23:90:11",
      "templateId": "T01",
      "fields": {
        "productName": "SD123-F001",
        "price": "SD123-F002",
        "promo": "SD123-F003"
      },
      "lastBrush": "2024-12-10T09:00:00Z"
    },
    {
      "tagMac": "AC:12:FF:23:90:12",
      "templateId": "T03",
      "fields": {
        "bigPrice": "SD888-F010",
        "discount": "SD888-F011"
      },
      "lastBrush": "2024-12-10T09:05:00Z"
    }
  ]
}
```

---

# ## 6.18 Bind devices with template (multi-data) — deprecated

> **⚠ Deprecated**
> 使用 6.5 的 multi-item 方式即可完全取代此 API。

### **POST /api/store/{storeId}/binding/multi**

### Request 範例（舊版，請勿再使用）

```json
{
  "tagMac": "AC:12:FF:23:90:11",
  "templateId": "T99",
  "dataList": [
    { "productId": "P001", "price": 199 },
    { "productId": "P002", "price": 89 }
  ]
}
```

### Response 範例（舊版，請勿再使用）

```json
{
  "status": "success",
  "tagMac": "AC:12:FF:23:90:11",
  "templateId": "T99",
  "dataItemCount": 2
}
```