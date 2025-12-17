# Special Device API（特殊版 API）

此 API 適用於圖片推送與更新狀態查詢流程。共有三個 API：

1. 取得裝置圖片解析度
2. 推送圖片至裝置
3. 查詢圖片更新狀態

---

# 目錄

1. [1. Get device resolution](#1-get-device-resolution)
2. [2. Push image to device](#2-push-image-to-device)
3. [3. Query image update status](#3-query-image-update-status)

---

# ## 1. Get device resolution

透過裝置 MAC 查詢該裝置支援的圖片解析度。

**GET /api/special/device/{mac}/resolution**

### URL Path Params

| 參數    | 說明        |
| ----- | --------- |
| `mac` | 裝置 MAC 地址 |

### Response

```json
{
  "mac": "AC:12:FF:23:90:11",
  "resolution": {
    "width": 296,
    "height": 128
  }
}
```

### 說明

* 回傳解析度以 `width` x `height` 表示
* 僅回傳解析度資訊，其餘欄位不返回

---

# ## 2. Push image to device

透過裝置 MAC 推送圖片。**圖片必須符合解析度限制**，若解析度錯誤將無法推送。

**POST /api/special/device/{mac}/image**

### URL Path Params

| 參數    | 說明        |
| ----- | --------- |
| `mac` | 裝置 MAC 地址 |

### Request

* 直接上傳 BMP 圖片檔（multipart/form-data 或 raw body），不需另外提供解析度資訊

### Response

```json
{
  "status": "success",
  "imagePushId": "IMG-PUSH-20251217-0001"
}
```

### 說明

* `imagePushId`：本次圖片推送操作的識別碼
* 若解析度不符，會回傳 `status: "failed"` 並說明錯誤原因

---

# ## 3. Query image update status

使用裝置 MAC 與 `imagePushId` 查詢圖片是否已更新到裝置。

**GET /api/special/device/{mac}/image/status?imagePushId={imagePushId}**

### URL Path Params

| 參數    | 說明        |
| ----- | --------- |
| `mac` | 裝置 MAC 地址 |

### Query Parameters

| 參數            | 說明             |
| ------------- | -------------- |
| `imagePushId` | 上一步推送圖片所取得的識別碼 |

### Response

```json
{
  "mac": "AC:12:FF:23:90:11",
  "imagePushId": "IMG-PUSH-20251217-0001",
  "status": "updated"   // "updated" 或 "pending"
}
```

### 說明

* `status` 可為：

  * `"updated"`：圖片已更新完成
  * `"pending"`：圖片尚未完成更新

