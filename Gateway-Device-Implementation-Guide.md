# Gateway/Device 實作指南

## 概述

本文件為 Gateway/Device 實作人員提供與 Server 溝通的完整協議規範和架構指南。

## 🔄 操作流程概覽

```mermaid
flowchart TD
    A[用戶在 Server Web 新增 Gateway]
    B[點擊鑰匙圖標按鈕]
    C[複製 Gateway Config 資訊]
    D[將 Config 資訊貼到 Gateway 設備]
    E[Gateway 使用 Config 建立 WebSocket 連接]
    F{驗證 JWT Token}
    G[收到 welcome 消息]
    H[連接被拒絕]
    I[發送 gatewayInfo 消息]
    J{Server 驗證 MAC 地址}
    K[收到 gatewayInfoAck success]
    L[收到 gatewayInfoAck fatal 錯誤]
    M[開始發送 deviceStatus]
    N[正常運行狀態]
    O[連接被強制中斷]
    P[檢查 Config 和網關狀態]
    Q[修正問題後重新嘗試]

    A --> B
    B --> C
    C --> D
    D --> E
    E --> F

    F -->|成功| G
    F -->|失敗| H

    G --> I
    I --> J

    J -->|匹配| K
    J -->|不匹配| L

    K --> M
    M --> N

    L --> O
    H --> P
    O --> P
    P --> Q

    style A fill:#e1f5fe
    style G fill:#e8f5e8
    style K fill:#e8f5e8
    style N fill:#e8f5e8
    style H fill:#ffebee
    style L fill:#ffebee
    style O fill:#ffebee
```

## 目錄

1. [操作流程詳解](#操作流程詳解)
2. [系統架構概覽](#系統架構概覽)
3. [WebSocket 連接流程](#websocket-連接流程)
4. [消息協議規範](#消息協議規範)
5. [失敗狀況與錯誤回應](#失敗狀況與錯誤回應)
6. [Gateway 內部架構](#gateway-內部架構)
7. [Device 管理架構](#device-管理架構)
8. [錯誤處理機制](#錯誤處理機制)
9. [安全考量](#安全考量)
10. [實作範例](#實作範例)

## 操作流程詳解

### 1. Server 端準備
1. **新增 Gateway**: 用戶在 Server Web 管理界面新增 Gateway 記錄
2. **獲取 Config**: 點擊鑰匙圖標按鈕，複製包含 JWT Token 的 Gateway Config 資訊
3. **Config 格式**:
   ```json
   {
     "url": "ws://server-ip:port/ws",
     "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
     "protocol": "json"
   }
   ```

### 2. Gateway 端配置
1. **接收 Config**: 將複製的 Config 資訊貼到 Gateway 設備
2. **解析 Config**: Gateway 解析 JSON 格式的配置信息
3. **建立連接**: 使用 Config 中的 URL 和 Token 建立 WebSocket 連接

### 3. 連接驗證流程
1. **JWT 驗證**: Server 驗證 Token 的有效性、類型、MAC 地址等
2. **歡迎消息**: 驗證成功後 Server 發送 welcome 消息
3. **網關註冊**: Gateway 收到 welcome 後立即發送 gatewayInfo 消息
4. **開始通信**: 註冊成功後開始定期發送 deviceStatus 消息

## 系統架構概覽

```mermaid
graph TB
    subgraph "Server 端"
        S[WebSocket Server]
        DB[(MongoDB)]
        API[REST API]
    end

    subgraph "Gateway 端"
        G[Gateway 主程序]
        WS[WebSocket Client]
        BT[藍牙管理器]
        DM[設備管理器]
    end

    subgraph "Device 端"
        D1[EPD Device 1]
        D2[EPD Device 2]
        DN[EPD Device N]
    end

    S <--> WS
    S <--> DB
    S <--> API
    WS <--> G
    G <--> BT
    BT <--> DM
    DM <--> D1
    DM <--> D2
    DM <--> DN
```

## WebSocket 連接流程

### 1. 連接建立流程圖

```mermaid
sequenceDiagram
    participant G as Gateway
    participant S as Server
    participant DB as Database

    Note over G,S: 1. 連接建立階段
    G->>S: WebSocket 連接請求 (帶 JWT Token)
    S->>S: 驗證 Token (MAC 地址、Gateway ID、Store ID)
    S->>DB: 檢查 Gateway 記錄
    S->>G: 連接成功

    Note over G,S: 2. 初始化階段
    S->>G: welcome 消息
    G->>S: ping 消息
    S->>G: pong 回應
    G->>S: gatewayInfo 消息
    S->>G: gatewayInfoAck 回應

    Note over G,S: 3. 運行階段
    loop 每 5 秒
        G->>S: deviceStatus 消息
        S->>G: deviceStatusAck 回應
    end

    loop 每 25 秒
        G->>S: ping 心跳
        S->>G: pong 回應
    end

    loop 每 30 秒
        G->>S: gatewayInfo 更新
        S->>G: gatewayInfoAck 回應
    end

    Note over G,S: 4. 圖像更新
    S->>G: update_preview 消息
    G->>G: 更新本地 imageCode
    G->>S: deviceStatus (包含新 imageCode)
```

### 2. 連接參數

**WebSocket URL 格式：**
```
ws://server-ip:port/ws?token=JWT_TOKEN
```

**JWT Token 內容：**
```json
{
  "type": "gateway",
  "gatewayId": "gateway_object_id",
  "storeId": "store_object_id",
  "macAddress": "AA:BB:CC:DD:EE:FF",
  "iat": 1640995200,
  "exp": 1640998800
}
```

## 消息協議規範

### 1. 發送消息格式 (Gateway → Server)

#### 1.1 ping 消息 (心跳)
```json
{
  "type": "ping",
  "timestamp": 1640995200000
}
```

**發送頻率：** 每 25 秒
**用途：** 保持連接活躍，防止超時斷線

#### 1.2 gatewayInfo 消息 (網關信息)
```json
{
  "type": "gatewayInfo",
  "info": {
    "macAddress": "AA:BB:CC:DD:EE:FF",
    "model": "Gateway Module 003",
    "hardwareVersion": "1.0.0.1",
    "wifiFirmwareVersion": "1.0.0",
    "btFirmwareVersion": "2.0.0",
    "ipAddress": "192.168.1.100",

    // 新增：分片傳輸能力支援
    "chunkingSupport": {
      "enabled": true,                 // 是否支援分片傳輸
      "maxChunkSize": 200,            // 每個分片的最大大小（4 bytes - 512KB）
      "maxSingleMessageSize": 10240,  // 單次 JSON 訊息的最大大小限制（bytes）
      "embeddedIndex": true,          // 是否支援嵌入式 Index 模式
      "jsonHeader": true,             // 是否支援 JSON Header 模式（向後兼容）
      "supportedFormat": "rawdata"    // 偏好的 rawdata 格式：rawdata, runlendata
    }
  }
}
```

**發送時機：**
- 收到 welcome 消息後立即發送
- 每 30 秒定期更新
- 網關信息變更時

**網關信息字段說明：**
- `macAddress`: 網關 MAC 地址 (必填，必須與 JWT Token 中的完全一致)
- `model`: 網關模組 (如 "Gateway Module 003", "GW-Pro-001")
- `hardwareVersion`: 硬體版本 (x.x.x.x格式，如 "1.0.0.1")
- `wifiFirmwareVersion`: WiFi 固件版本 (如 "1.0.0", "2.1.3")
- `btFirmwareVersion`: 藍牙固件版本 (如 "2.0.0", "1.5.2")
- `ipAddress`: 當前 IP 地址 (如 "192.168.1.100")

**重要注意事項：**
- `macAddress` 必須與 JWT Token 中的 MAC 地址完全一致
- 如果不匹配，Server 會強制中斷連線並記錄安全事件
- `chunkingSupport` 決定 Server 是否對該 Gateway 啟用分片傳輸
- `maxChunkSize` 應根據硬體記憶體限制設定，支援範圍 4 bytes - 512KB
- `maxSingleMessageSize` 用於控制單次 JSON 訊息的大小限制，影響分片決策
- `supportedFormat` 告知 Server 偏好的 rawdata 格式，Server 會優先使用此格式

**分片傳輸決策邏輯：**
Server 會根據以下兩階段邏輯決定是否使用分片傳輸：

1. **第一階段：rawdata 大小檢查**
   - 如果 `rawdata 大小 > maxChunkSize` 且 `enabled: true` → 使用分片傳輸
   - 如果 `rawdata 大小 > maxChunkSize` 但 `enabled: false` → 拋出錯誤
   - 如果 `rawdata 大小 <= maxChunkSize` → 進入第二階段檢查

2. **第二階段：JSON 訊息大小檢查**
   - 構建完整的 `update_preview` JSON 訊息（包含 rawdata、imageData 等所有欄位）
   - 如果 `JSON 訊息大小 > maxSingleMessageSize` 且 `enabled: true` → 使用分片傳輸
   - 如果 `JSON 訊息大小 <= maxSingleMessageSize` → 使用直接傳輸

**參數設定建議：**
- `maxChunkSize`: 根據硬體記憶體限制設定（如 200-1024 bytes）
- `maxSingleMessageSize`: 根據網路環境和處理能力設定（如 2048-10240 bytes）
- 通常 `maxSingleMessageSize` 應大於 `maxChunkSize` 以避免過度分片

#### 1.3 deviceStatus 消息 (設備狀態)
```json
{
  "type": "deviceStatus",
  "devices": [
    {
      "macAddress": "11:22:33:44:55:66",
      "model": "EPD-2.9-BW",
      "hardwareVersion": "1.0.0.1",
      "firmwareVersion": "1.2.3",
      "status": "online",
      "data": {
        "size": "2.9\"",
        "battery": 85,
        "rssi": -65,
        "colorType": "BW",
        "imageCode": "12345678"
      }
    }
  ]
}
```

**發送頻率：** 每 5 秒
**設備狀態字段說明：**
- `macAddress`: 設備 MAC 地址 (必填)
- `model`: 設備模組 (如 "EPD-2.9-BW", "EPD-4.2-BWR")
- `hardwareVersion`: 硬體版本 (x.x.x.x格式，如 "1.0.0.1")
- `firmwareVersion`: 韌體版本 (如 "1.2.3", "2.0.1")
- `status`: 設備狀態 ("online"/"offline")
- `data.size`: 螢幕尺寸 (如 "2.9\"", "4.2\"")
- `data.battery`: 電池電量 (0-100)
- `data.rssi`: 信號強度 (負數)
- `data.colorType`: 顏色類型 ("BW"/"BWR"/"BWRY")
- `data.imageCode`: 當前顯示的圖像代碼 (可選)

**重要注意事項：**
- **不包含 `dataId`**: 這是由前端或API控制的欄位，不是由裝置自己回報的欄位
- **imageCode 處理邏輯：**
  - 設備回報不應主動包含 `imageCode`
  - 只有在本地存儲有 `imageCode` 時才包含此字段
  - 當收到 Server 的圖像更新時，更新本地 `imageCode`
  - 下次狀態回報時包含更新後的 `imageCode`

#### 1.4 chunk_start_ack 消息 (分片開始確認)
```json
{
  "type": "chunk_start_ack",
  "chunkId": "chunk_12345",
  "status": "ready",
  "error": null,
  "timestamp": 1640995200000
}
```

**發送時機：** 收到 `image_chunk_start` 消息後立即發送
**用途：** 確認已準備好接收分片數據

**參數說明：**
- `status`: 操作狀態
  - `"ready"`: 已準備好接收分片數據
  - `"error"`: 準備過程中發生錯誤
- `error`: 錯誤信息（當 status 為 "error" 時提供具體錯誤描述，否則為 null）

#### 1.5 chunk_ack 消息 (分片確認)
```json
{
  "type": "chunk_ack",
  "chunkId": "chunk_12345",
  "chunkIndex": 5,
  "status": "received",
  "error": null,
  "timestamp": 1640995200000
}
```

**發送時機：** 收到每個分片數據後立即發送
**用途：** 確認已成功接收該分片，Server 可以發送下一個分片

**參數說明：**
- `status`: 分片接收狀態
  - `"received"`: 分片已成功接收並存儲
  - `"duplicate"`: 檢測到重複分片（已接收過）
  - `"error"`: 分片接收或處理過程中發生錯誤
- `error`: 錯誤信息（當 status 為 "error" 時提供具體錯誤描述，否則為 null）

#### 1.6 chunk_complete_ack 消息 (分片完成確認)
```json
{
  "type": "chunk_complete_ack",
  "chunkId": "chunk_12345",
  "status": "success",
  "receivedSize": 9484,
  "error": null,
  "timestamp": 1640995200000
}
```

**發送時機：** 收到 `image_chunk_complete` 消息後立即發送
**用途：** 確認所有分片已接收完成並成功重組

**參數說明：**
- `status`: 完成狀態
  - `"success"`: 所有分片已成功接收並重組完成
  - `"error"`: 分片重組過程中發生錯誤
- `receivedSize`: 實際接收到的數據總大小（bytes）
- `error`: 錯誤信息（當 status 為 "error" 時提供具體錯誤描述，否則為 null）

### 2. 接收消息格式 (Server → Gateway)

#### 2.1 welcome 消息 (歡迎)
```json
{
  "type": "welcome",
  "message": "WebSocket 連接成功",
  "timestamp": 1640995200000,
  "gatewayInfo": {
    "gatewayId": "gateway_object_id",
    "storeId": "store_object_id",
    "macAddress": "AA:BB:CC:DD:EE:FF"
  },
  "serverInfo": {
    "time": "2021-12-31T16:00:00.000Z",
    "version": "1.0.0"
  }
}
```

**處理邏輯：**
- 收到此消息表示連接成功建立
- 應立即發送 `gatewayInfo` 消息回應

#### 2.2 pong 消息 (心跳回應)
```json
{
  "type": "pong",
  "timestamp": 1640995200000,
  "serverTime": 1640995200100
}
```

**處理邏輯：**
- 確認心跳消息已被 Server 接收
- 可用於計算網絡延遲

#### 2.3 gatewayInfoAck 消息 (網關信息確認)
```json
{
  "type": "gatewayInfoAck",
  "timestamp": 1640995200000,
  "success": true,
  "message": "網關信息更新成功"
}
```

**錯誤情況：**
```json
{
  "type": "gatewayInfoAck",
  "timestamp": 1640995200000,
  "success": false,
  "message": "MAC地址不匹配，連線已中斷",
  "fatal": true
}
```

**處理邏輯：**
- `success: true`: 網關信息更新成功
- `success: false, fatal: true`: 嚴重錯誤，連線將被強制中斷
- `success: false, fatal: false`: 一般錯誤，可繼續運行

#### 2.4 deviceStatusAck 消息 (設備狀態確認)
```json
{
  "type": "deviceStatusAck",
  "timestamp": 1640995200000,
  "success": true,
  "message": "設備狀態更新成功",
  "detailStatus": [
    {
      "macAddress": "11:22:33:44:55:66",
      "status": "success",
      "deviceId": "device_object_id"
    }
  ]
}
```

#### 2.5 update_preview 消息 (圖像更新 - 直接傳輸)
```json
{
  "type": "update_preview",
  "deviceMac": "11:22:33:44:55:66",
  "imageData": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...",
  "imageCode": "87654321",
  "rawdata": [255, 255, 0, 128, 64, ...],  // EPD 原始數據陣列 (Uint8Array)
  "dataType": "rawdata",  // 數據格式類型：rawdata, runlendata 等
  "timestamp": 1640995200000  // 統一使用數字格式
}
```

**處理邏輯：**
1. 接收圖像數據並保存到設備
2. 更新本地存儲的 `imageCode`
3. 根據 `dataType` 處理 `rawdata` 中的數據
4. 使用處理後的 EPD 二進制數據直接發送到設備顯示
5. 下次 `deviceStatus` 回報時包含新的 `imageCode`

**rawdata 字段說明：**
- 包含轉換後的 EPD 格式二進制數據
- 數據格式根據設備的 colorType 而定 (BW/BWR/BWRY)
- 包含 ImageInfo 結構 (12 bytes) + 像素數據
- 可直接發送到 EPD 設備進行顯示，無需額外轉換

**dataType 字段說明：**
- `"rawdata"`: 未壓縮的原始 EPD 數據
- `"runlendata"`: 使用 Run-Length Encoding 壓縮的數據
- 其他格式可能在未來添加

#### 2.6 image_chunk_start 消息 (分片傳輸開始)
```json
{
  "type": "image_chunk_start",
  "chunkId": "chunk_12345",
  "deviceMac": "11:22:33:44:55:66",
  "imageCode": "87654321",
  "totalChunks": 48,
  "totalSize": 9484,
  "chunkSize": 200,
  "indexSize": 4,
  "dataType": "runlendata",  // 數據格式類型：rawdata, runlendata 等
  "mode": "embedded_index",
  "timestamp": 1640995200000  // 統一使用數字格式
}
```

**處理邏輯：**
1. 準備接收分片數據，初始化接收緩衝區
2. 記錄分片參數（總分片數、分片大小等）
3. 記錄 `dataType` 以便後續正確處理數據
4. 立即發送 `chunk_start_ack` 確認
5. 等待接收二進制分片數據

**字段說明：**
- `chunkId`: 唯一識別符，用於關聯所有分片
- `totalChunks`: 總分片數量
- `totalSize`: 原始數據總大小（不包含 index）
- `chunkSize`: 每個分片的實際數據大小
- `indexSize`: 每個分片前綴的 index 大小（固定 4 bytes）
- `dataType`: 數據格式類型，決定如何處理接收到的數據
- `mode`: "embedded_index" 表示使用嵌入式 Index 模式

#### 2.7 二進制分片數據
**格式：**
```
[4 bytes: chunkIndex][N bytes: 實際數據]
```

**處理邏輯：**
1. 解析前 4 bytes 獲取 chunkIndex (little-endian)
2. 提取後續的實際數據
3. 按 chunkIndex 順序存儲到接收緩衝區
4. 立即發送 `chunk_ack` 確認
5. 檢查是否已接收所有分片
6. 完成後重組完整數據並更新 imageCode

**重要注意事項：**
- chunkIndex 是 32位無符號整數，little-endian 格式
- 必須按順序重組分片，不能遺漏任何分片
- 每個分片接收後必須立即發送 ACK
- 超時未收到分片時，Server 會自動重傳

#### 2.8 image_chunk_complete 消息 (分片傳輸完成)
```json
{
  "type": "image_chunk_complete",
  "chunkId": "chunk_12345",
  "deviceMac": "11:22:33:44:55:66",
  "imageCode": "87654321",
  "totalChecksum": "a1b2",
  "timestamp": 1640995200000  // 統一使用數字格式
}
```

**處理邏輯：**
1. 驗證所有分片是否已正確接收
2. 檢查數據完整性（可選：使用 totalChecksum）
3. 完成圖像數據重組
4. 更新本地 imageCode
5. 發送 `chunk_complete_ack` 確認

**字段說明：**
- `totalChecksum`: 完整數據的校驗碼（可選，用於驗證數據完整性）

#### 2.9 error 消息 (錯誤)
```json
{
  "type": "error",
  "message": "處理消息時發生錯誤",
  "error": "具體錯誤信息",
  "timestamp": 1640995200000
}
```

### 3. 韌體更新消息格式 (Server → Gateway)

#### 3.1 firmware_chunk_start 消息 (韌體分片傳輸開始)
```json
{
  "type": "firmware_chunk_start",
  "chunkId": "firmware_chunk_12345",
  "deviceMac": "11:22:33:44:55:66",
  "firmwareId": "fw_abc123",
  "totalChunks": 128,
  "totalSize": 524288,
  "chunkSize": 4096,
  "indexSize": 4,
  "dataType": "firmware",
  "mode": "embedded_index",
  "firmwareInfo": {
    "version": "2.1.0",
    "deviceType": "gateway",
    "model": "ESP32-S3",
    "functionType": "wifi",
    "checksum": "a1b2c3d4e5f6",
    "minHwVersion": "1.0.0",
    "maxHwVersion": "3.0.0"
  },
  "timestamp": 1640995200000  // 統一使用數字格式
}
```

**發送時機：** 開始韌體分片傳輸時
**用途：** 通知 Gateway 準備接收韌體分片數據

**參數說明：**
- `chunkId`: 唯一的分片傳輸識別符
- `deviceMac`: 目標設備的 MAC 地址（可能是 Gateway 自身或下游設備）
- `firmwareId`: 韌體檔案的唯一識別符
- `totalChunks`: 總分片數量
- `totalSize`: 韌體檔案總大小（bytes）
- `chunkSize`: 每個分片的實際韌體數據大小（不包含 index）
- `indexSize`: 每個分片前綴的 index 大小（固定 4 bytes）
- `dataType`: 數據類型，固定為 "firmware"
- `mode`: 傳輸模式，固定為 "embedded_index"
- `firmwareInfo`: 韌體詳細資訊
  - `version`: 韌體版本
  - `deviceType`: 設備類型（gateway/epd）
  - `model`: 設備模組
  - `functionType`: 功能類型（wifi/ble）
  - `checksum`: 韌體校驗和
  - `minHwVersion`: 最小硬體版本
  - `maxHwVersion`: 最大硬體版本

#### 3.2 firmware_update 消息 (韌體直接更新)
```json
{
  "type": "firmware_update",
  "deviceMac": "11:22:33:44:55:66",
  "firmwareId": "fw_abc123",
  "firmwareData": [0x1F, 0x8B, 0x08, ...],
  "dataType": "firmware",
  "firmwareInfo": {
    "version": "2.1.0",
    "deviceType": "gateway",
    "model": "ESP32-S3",
    "functionType": "wifi",
    "checksum": "a1b2c3d4e5f6"
  },
  "timestamp": 1640995200000  // 統一使用數字格式
}
```

**發送時機：** 韌體檔案較小時使用直接傳輸
**用途：** 直接傳送完整韌體數據

**參數說明：**
- `firmwareData`: 韌體數據數組（完整的韌體檔案內容）
- 其他參數與分片傳輸相同

#### 3.3 firmware_chunk_complete 消息 (韌體分片傳輸完成)
```json
{
  "type": "firmware_chunk_complete",
  "chunkId": "firmware_chunk_12345",
  "deviceMac": "11:22:33:44:55:66",
  "firmwareId": "fw_abc123",
  "totalChecksum": "a1b2c3d4e5f6",
  "timestamp": 1640995200000  // 統一使用數字格式
}
```

**發送時機：** 所有韌體分片傳輸完成後
**用途：** 通知 Gateway 所有分片已傳輸完成，可以開始韌體更新

**參數說明：**
- `totalChecksum`: 完整韌體檔案的校驗和，用於驗證數據完整性

### 4. 韌體更新回應消息格式 (Gateway → Server)

#### 4.1 firmware_chunk_start_ack 消息 (韌體分片開始確認)
```json
{
  "type": "firmware_chunk_start_ack",
  "chunkId": "firmware_chunk_12345",
  "status": "ready",
  "message": null,
  "timestamp": 1640995200000  // 統一使用數字格式
}
```

**發送時機：** 收到 `firmware_chunk_start` 消息後立即發送
**用途：** 確認已準備好接收韌體分片數據

#### 4.2 firmware_chunk_ack 消息 (韌體分片確認)
```json
{
  "type": "firmware_chunk_ack",
  "chunkId": "firmware_chunk_12345",
  "chunkIndex": 42,
  "status": "received",
  "message": null,
  "timestamp": 1640995200000  // 統一使用數字格式
}
```

**發送時機：** 每收到一個韌體分片後立即發送
**用途：** 確認已成功接收該分片

#### 4.3 firmware_chunk_complete_ack 消息 (韌體分片完成確認)
```json
{
  "type": "firmware_chunk_complete_ack",
  "chunkId": "firmware_chunk_12345",
  "status": "success",
  "message": null,
  "finalChecksum": "a1b2c3d4e5f6",
  "timestamp": 1640995200000  // 統一使用數字格式
}
```

**發送時機：** 收到 `firmware_chunk_complete` 消息並驗證完成後發送
**用途：** 確認所有分片已接收完成並通過校驗

#### 4.4 firmware_update_ack 消息 (韌體直接更新確認)
```json
{
  "type": "firmware_update_ack",
  "deviceMac": "11:22:33:44:55:66",
  "firmwareId": "fw_abc123",
  "status": "success",
  "message": null,
  "timestamp": 1640995200000  // 統一使用數字格式
}
```

**發送時機：** 收到 `firmware_update` 消息並處理完成後發送
**用途：** 確認韌體更新請求已接收並開始處理

**注意：** Gateway 不需要發送 `firmware_update_progress` 和 `firmware_update_complete` 消息。這些消息是由 Server 發送給前端客戶端的，用於顯示更新進度和結果。Gateway 需要：
1. 發送 `firmware_update_ack` 或 `firmware_chunk_complete_ack` 確認韌體接收完成
2. 在韌體更新完成後，通過下次的 `gatewayInfo` 消息回報新的韌體版本
3. 在韌體更新過程中發生錯誤時，發送 `firmware_update_error` 消息

#### 4.5 firmware_update_error 消息 (韌體更新錯誤)
```json
{
  "type": "firmware_update_error",
  "deviceMac": "11:22:33:44:55:66",
  "firmwareId": "fw_abc123",
  "errorCode": "CHECKSUM_MISMATCH",
  "errorMessage": "韌體校驗失敗",
  "stage": "installing",
  "timestamp": 1640995200000  // 統一使用數字格式
}
```

**發送時機：** 韌體更新過程中發生錯誤時發送
**用途：** 報告韌體更新錯誤給 Server，Server 會轉發給前端客戶端

**參數說明：**
- `deviceMac`: 目標設備的 MAC 地址
- `firmwareId`: 韌體 ID
- `errorCode`: 錯誤代碼（見下方錯誤代碼列表）
- `errorMessage`: 詳細錯誤訊息
- `stage`: 錯誤發生的階段
  - `"downloading"`: 下載階段
  - `"verifying"`: 驗證階段
  - `"installing"`: 安裝階段
  - `"rebooting"`: 重啟階段

**常見錯誤代碼：**
- `CHECKSUM_MISMATCH`: 校驗和不匹配
- `INSUFFICIENT_MEMORY`: 記憶體不足
- `INCOMPATIBLE_VERSION`: 版本不相容
- `INVALID_FIRMWARE`: 無效的韌體檔案
- `UPDATE_FAILED`: 安裝失敗
- `TIMEOUT_ERROR`: 超時錯誤

**韌體更新錯誤處理：**
如果韌體更新過程中發生錯誤，Gateway 有兩種回報方式：

**方式一：ACK 消息中的錯誤回報**
1. 在相應的 ACK 消息中設置 `status: "error"` 和具體的 `error` 信息
2. 適用於接收階段的錯誤（如格式錯誤、記憶體不足等）

**方式二：專用錯誤消息回報**
1. 發送 `firmware_update_error` 消息報告具體錯誤
2. 適用於安裝階段的錯誤（如校驗失敗、安裝失敗等）
3. 提供更詳細的錯誤信息和錯誤代碼

**錯誤處理流程：**
1. **接收階段錯誤**：在 ACK 中回報 → 繼續使用原韌體
2. **安裝階段錯誤**：發送 `firmware_update_error` → 繼續使用原韌體
3. **最終確認**：通過下次 `gatewayInfo` 中的韌體版本確認結果

## 數據格式處理

### 1. dataType 格式說明

Server 會根據數據特性和 Gateway 能力選擇最適合的數據格式：

#### 1.1 rawdata (原始格式)
- **用途**: 未壓縮的原始 EPD 數據
- **結構**: ImageInfo (12 bytes) + 像素數據
- **處理**: 直接使用，無需額外處理

#### 1.2 runlendata (RLE 壓縮格式)
- **用途**: 使用 Run-Length Encoding 壓縮的數據
- **結構**: ImageInfo (12 bytes) + 壓縮後的像素數據
- **處理**: 需要解壓縮後才能使用

### 2. RLE 編碼格式詳解

#### 2.1 編碼規則
Run-Length Encoding 使用以下格式：

1. **重複序列** (runLength >= 2):
   - 格式: `[runLength, value]`
   - runLength 範圍: 2-127 (0x02-0x7F)
   - bit7 = 0

2. **非重複序列** (runLength = 1 或無重複):
   - 格式: `[0x80|length, data...]`
   - length 範圍: 1-127 (0x01-0x7F)
   - bit7 = 1

**重要說明**:
- bit7 是最高位元 (MSB)
- 壓縮的只有 EPD 像素數據，不包含 ImageInfo 結構 (12 bytes) 頭部
- 不包含 chunk 的 index 資訊

#### 2.2 解壓縮實現範例

```python
def decompress_rle_data(compressed_data):
    """解壓縮 RLE 數據"""
    decompressed = []
    i = 0

    while i < len(compressed_data):
        header = compressed_data[i]
        i += 1

        if (header & 0x80) == 0:
            # 重複序列：bit7 = 0
            run_length = header
            if i >= len(compressed_data):
                raise Exception('Incomplete RLE data: missing value byte')
            value = compressed_data[i]
            i += 1

            # 重複 run_length 次
            for _ in range(run_length):
                decompressed.append(value)
        else:
            # 非重複序列：bit7 = 1
            length = header & 0x7F
            if i + length > len(compressed_data):
                raise Exception('Incomplete RLE data: insufficient data bytes')

            # 複製 length 個字節
            for j in range(length):
                decompressed.append(compressed_data[i + j])
            i += length

    return bytes(decompressed)
```

#### 2.3 數據處理流程

```python
def process_received_data(rawdata, data_type):
    """處理接收到的數據"""
    if data_type == "rawdata":
        # 原始數據，直接使用
        return rawdata
    elif data_type == "runlendata":
        # RLE 壓縮數據，需要解壓縮
        # 分離 ImageInfo 和壓縮的像素數據
        image_info = rawdata[:12]  # 前 12 bytes 是 ImageInfo
        compressed_pixels = rawdata[12:]  # 後續是壓縮的像素數據

        # 解壓縮像素數據
        decompressed_pixels = decompress_rle_data(compressed_pixels)

        # 重新組合完整數據
        complete_data = image_info + decompressed_pixels
        return complete_data
    else:
        raise Exception(f"Unsupported data type: {data_type}")
```

### 3. 分片數據處理

當使用分片傳輸時，需要特別注意：

1. **分片重組**: 先按 chunkIndex 順序重組完整的 rawdata
2. **格式處理**: 重組完成後，根據 `dataType` 處理數據
3. **ImageInfo 保護**: 確保 ImageInfo 結構完整性
4. **壓縮範圍**: RLE 壓縮只應用於像素數據部分

```python
def handle_complete_chunk_data(self, device_mac, image_code, complete_data, data_type):
    """處理完整的分片數據"""
    print(f"分片傳輸完成: 設備 {device_mac}, 數據大小 {len(complete_data)} bytes, 格式 {data_type}")

    # 根據 dataType 處理數據
    processed_data = process_received_data(complete_data, data_type)

    # 更新本地 imageCode
    self.image_codes[device_mac] = image_code

    # 發送到設備
    await self.send_epd_data_to_device(device_mac, processed_data)
```

## 失敗狀況與錯誤回應

### 1. 連接階段失敗

#### 1.1 Token 驗證失敗
**狀況**: JWT Token 無效、過期或格式錯誤
```
WebSocket 連接被拒絕
HTTP 狀態碼: 401 Unauthorized
原因: "Authentication failed"
```

**可能原因**:
- Token 已過期
- Token 格式不正確
- Token 簽名無效
- 缺少 Token 參數

#### 1.2 Token 類型錯誤
**狀況**: Token 不是 gateway 類型
```
WebSocket 連接被拒絕
HTTP 狀態碼: 401 Unauthorized
原因: "Invalid token type"
```

#### 1.3 Gateway ID 不匹配
**狀況**: URL 路徑中的 Gateway ID 與 Token 中的不匹配
```
WebSocket 連接被拒絕
HTTP 狀態碼: 403 Forbidden
原因: "Gateway ID mismatch"
```

#### 1.4 Store ID 不匹配
**狀況**: URL 路徑中的 Store ID 與 Token 中的不匹配
```
WebSocket 連接被拒絕
HTTP 狀態碼: 403 Forbidden
原因: "Store ID mismatch"
```

#### 1.5 MAC 地址缺失
**狀況**: Token 中缺少 MAC 地址信息
```
WebSocket 連接被拒絕
HTTP 狀態碼: 403 Forbidden
原因: "MAC address missing in token"
```

### 2. 運行階段失敗

#### 2.1 MAC 地址不匹配 (致命錯誤)
**狀況**: gatewayInfo 中的 MAC 地址與 Token 中的不匹配
```json
{
  "type": "gatewayInfoAck",
  "timestamp": 1640995200000,
  "success": false,
  "message": "MAC地址不匹配，連線已中斷",
  "fatal": true
}
```
**後果**: 連接被強制中斷，記錄安全事件

#### 2.2 網關被刪除
**狀況**: 網關在連接期間被從系統中刪除
```json
{
  "type": "gatewayInfoAck",
  "timestamp": 1640995200000,
  "success": false,
  "message": "找不到指定的網關",
  "fatal": false
}
```

#### 2.3 網關信息無效
**狀況**: gatewayInfo 消息格式錯誤或缺少必要字段
```json
{
  "type": "gatewayInfoAck",
  "timestamp": 1640995200000,
  "success": false,
  "message": "網關信息無效",
  "fatal": false
}
```

#### 2.4 設備狀態處理失敗
**狀況**: deviceStatus 消息處理時發生錯誤
```json
{
  "type": "deviceStatusAck",
  "timestamp": 1640995200000,
  "success": false,
  "message": "處理設備狀態失敗: 設備列表為空或無效"
}
```

**常見錯誤原因**:
- 設備列表為空
- 設備 MAC 地址格式錯誤
- 設備數據格式不正確
- 數據庫操作失敗

#### 2.5 未知消息類型
**狀況**: 發送了 Server 不支持的消息類型
```json
{
  "type": "error",
  "message": "未知的消息類型",
  "originalType": "unknown_message_type",
  "timestamp": 1640995200000
}
```

#### 2.6 JSON 解析錯誤
**狀況**: 發送的消息不是有效的 JSON 格式
```json
{
  "type": "error",
  "message": "處理消息時發生錯誤",
  "error": "Unexpected token in JSON",
  "timestamp": 1640995200000
}
```

### 3. 心跳檢測失敗

#### 3.1 心跳超時
**狀況**: Gateway 長時間未發送 ping 消息
```
連接被服務器主動關閉
關閉代碼: 1000 (正常關閉)
```
**觸發條件**: 超過 30 秒未收到 ping 消息

#### 3.2 長時間無活動
**狀況**: Gateway 長時間無任何消息活動
```
連接被服務器主動關閉
關閉代碼: 1000 (正常關閉)
```
**觸發條件**: 超過 60 秒無任何消息活動

### 4. 服務器內部錯誤

#### 4.1 數據庫連接失敗
**狀況**: Server 無法連接到數據庫
```json
{
  "type": "error",
  "message": "伺服器內部錯誤",
  "timestamp": 1640995200000
}
```

#### 4.2 權限檢查失敗
**狀況**: 用戶權限不足或權限檢查出錯
```json
{
  "type": "error",
  "message": "沒有權限執行此操作",
  "timestamp": 1640995200000
}
```

### 5. 錯誤處理建議

#### 5.1 致命錯誤處理
```python
if message.get("fatal") == True:
    # 致命錯誤，停止重連嘗試
    logger.error(f"致命錯誤: {message.get('message')}")
    self.stop_reconnection()
    return False
```

#### 5.2 一般錯誤處理
```python
if message.get("success") == False:
    # 一般錯誤，記錄並繼續運行
    logger.warning(f"操作失敗: {message.get('message')}")
    # 可以嘗試重新發送或調整策略
    return True
```

#### 5.3 連接錯誤處理
```python
def handle_connection_error(self, error_code, reason):
    if error_code == 401:
        # 認證失敗，檢查 Token
        logger.error("認證失敗，請檢查 Token 是否有效")
        return False
    elif error_code == 403:
        # 權限不足，檢查配置
        logger.error("權限不足，請檢查 Gateway 配置")
        return False
    else:
        # 其他錯誤，可以嘗試重連
        logger.warning(f"連接錯誤: {error_code} - {reason}")
        return True
```

## Gateway 內部架構

### 1. Gateway 架構流程圖

```mermaid
graph TB
    subgraph "Gateway 主程序"
        Main[主程序]
        Config[配置管理]
        Logger[日誌系統]
    end

    subgraph "網絡通信層"
        WSClient[WebSocket 客戶端]
        WSHandler[消息處理器]
        Heartbeat[心跳管理]
    end

    subgraph "設備管理層"
        DeviceManager[設備管理器]
        DeviceScanner[設備掃描器]
        DeviceCache[設備緩存]
    end

    subgraph "藍牙通信層"
        BTManager[藍牙管理器]
        BTScanner[藍牙掃描]
        BTConnection[連接管理]
    end

    subgraph "圖像處理層"
        ImageProcessor[圖像處理器]
        ImageCache[圖像緩存]
        ImageSender[圖像發送器]
    end

    Main --> Config
    Main --> Logger
    Main --> WSClient
    WSClient --> WSHandler
    WSClient --> Heartbeat
    WSHandler --> DeviceManager
    DeviceManager --> DeviceScanner
    DeviceManager --> DeviceCache
    DeviceManager --> BTManager
    BTManager --> BTScanner
    BTManager --> BTConnection
    WSHandler --> ImageProcessor
    ImageProcessor --> ImageCache
    ImageProcessor --> ImageSender
    ImageSender --> BTManager
```

### 2. Gateway 核心組件

#### 2.1 WebSocket 客戶端管理
```python
class WebSocketClient:
    def __init__(self, url, token):
        self.url = url
        self.token = token
        self.ws = None
        self.is_connected = False
        self.last_ping = 0

    async def connect(self):
        """建立 WebSocket 連接"""

    async def send_message(self, message):
        """發送消息到 Server"""

    async def handle_message(self, message):
        """處理來自 Server 的消息"""

    async def start_heartbeat(self):
        """啟動心跳機制"""
```

#### 2.2 設備管理器
```python
class DeviceManager:
    def __init__(self):
        self.devices = {}  # MAC -> Device 映射
        self.image_codes = {}  # MAC -> imageCode 映射

    def add_device(self, device):
        """添加設備"""

    def update_device_status(self, mac_address, status_data):
        """更新設備狀態"""

    def get_device_status_report(self):
        """生成設備狀態報告"""

    def update_image_code(self, mac_address, image_code):
        """更新設備的 imageCode"""
```

#### 2.3 圖像處理器
```python
class ImageProcessor:
    def __init__(self, device_manager):
        self.device_manager = device_manager

    async def handle_image_update(self, device_mac, image_data, image_code, rawdata=None):
        """處理圖像更新消息"""
        # 1. 保存圖像數據 (用於顯示或備份)
        # 2. 更新本地 imageCode
        # 3. 使用 rawdata 直接發送到 EPD 設備 (優先)
        # 4. 如果沒有 rawdata，則使用 image_data 進行轉換後發送

        if rawdata:
            # 直接使用 EPD 格式數據
            await self.send_epd_data_to_device(device_mac, rawdata)
        else:
            # 需要轉換 image_data 為 EPD 格式
            epd_data = await self.convert_image_to_epd(device_mac, image_data)
            await self.send_epd_data_to_device(device_mac, epd_data)

    async def send_epd_data_to_device(self, device_mac, epd_data):
        """發送 EPD 數據到設備"""
        # 通過藍牙或其他通信方式發送到實際設備
        pass

    async def convert_image_to_epd(self, device_mac, image_data):
        """將圖像數據轉換為 EPD 格式 (備用方案)"""
        # 如果 Server 沒有提供 rawdata，Gateway 需要自行轉換
        pass
```

## Device 管理架構

### 1. Device 狀態管理流程圖

```mermaid
stateDiagram-v2
    [*] --> Scanning: Gateway 啟動
    Scanning --> Discovered: 發現設備
    Discovered --> Connecting: 嘗試連接
    Connecting --> Connected: 連接成功
    Connecting --> Disconnected: 連接失敗
    Connected --> Online: 設備正常
    Online --> Updating: 收到圖像更新
    Updating --> Online: 更新完成
    Online --> Disconnected: 連接丟失
    Disconnected --> Connecting: 重新連接
    Disconnected --> [*]: 設備移除
```

### 2. Device 數據結構

```python
class Device:
    def __init__(self, mac_address):
        self.mac_address = mac_address
        self.status = "offline"
        self.data_id = None
        self.data = {
            "size": None,
            "battery": 0,
            "rssi": -100,
            "colorType": "BW"
        }
        self.last_seen = None
        self.image_code = None

    def update_status(self, **kwargs):
        """更新設備狀態"""

    def to_dict(self):
        """轉換為字典格式用於上報"""
        result = {
            "macAddress": self.mac_address,
            "status": self.status,
            "data": self.data.copy()
        }

        if self.data_id:
            result["dataId"] = self.data_id

        if self.image_code:
            result["data"]["imageCode"] = self.image_code

        return result
```

## 錯誤處理機制

### 1. 連接錯誤處理

```python
async def handle_connection_error(self, error):
    """處理連接錯誤"""
    if "MAC地址不匹配" in str(error):
        # 嚴重安全錯誤，停止重連
        self.logger.error("安全錯誤：MAC 地址不匹配，停止連接")
        return False
    else:
        # 一般網絡錯誤，嘗試重連
        await asyncio.sleep(5)
        return True
```

### 2. 消息處理錯誤

```python
async def handle_message_error(self, message, error):
    """處理消息錯誤"""
    self.logger.error(f"處理消息失敗: {error}")

    # 發送錯誤報告到 Server (可選)
    error_message = {
        "type": "error_report",
        "original_message": message,
        "error": str(error),
        "timestamp": time.time()
    }
    await self.send_message(error_message)
```

## 安全考量

### 1. MAC 地址驗證
- Gateway 發送的 MAC 地址必須與 JWT Token 中的完全一致
- 不匹配時 Server 會立即中斷連線
- 實作時應確保使用正確的 MAC 地址

### 2. Token 管理
- JWT Token 有過期時間，需要定期更新
- Token 洩露時應立即更換
- 存儲 Token 時應加密保護

### 3. 通信安全
- 建議使用 WSS (WebSocket Secure) 加密通信
- 定期檢查連接狀態，防止中間人攻擊
- 對敏感數據進行額外加密

## 實作範例

### 1. 基本 Gateway 實作框架

```python
import asyncio
import websockets
import json
import time
from typing import Dict, List

class EPDGateway:
    def __init__(self, server_url: str, token: str, mac_address: str):
        self.server_url = server_url
        self.token = token
        self.mac_address = mac_address
        self.ws = None
        self.devices: Dict[str, Device] = {}
        self.image_codes: Dict[str, str] = {}
        self.running = False

    async def start(self):
        """啟動 Gateway"""
        self.running = True

        # 建立 WebSocket 連接
        url = f"{self.server_url}?token={self.token}"

        try:
            async with websockets.connect(url) as websocket:
                self.ws = websocket
                print(f"已連接到 Server: {self.server_url}")

                # 啟動各種任務
                await asyncio.gather(
                    self.handle_messages(),
                    self.send_heartbeat(),
                    self.send_device_status(),
                    self.send_gateway_info(),
                    self.scan_devices()
                )
        except Exception as e:
            print(f"連接錯誤: {e}")

    async def handle_messages(self):
        """處理來自 Server 的消息"""
        async for message in self.ws:
            try:
                # 檢查是否為二進制數據（分片）
                if isinstance(message, bytes):
                    await self.handle_chunk_data(message)
                else:
                    # JSON 消息
                    data = json.loads(message)
                    await self.process_message(data)
            except Exception as e:
                print(f"處理消息錯誤: {e}")

    async def process_message(self, data):
        """處理具體消息"""
        msg_type = data.get("type")

        if msg_type == "welcome":
            print("收到歡迎消息，連接成功")
            await self.send_gateway_info_immediate()

        elif msg_type == "pong":
            print("收到心跳回應")

        elif msg_type == "update_preview":
            await self.handle_image_update(data)

        elif msg_type == "image_chunk_start":
            await self.handle_chunk_start(data)

        elif msg_type == "gatewayInfoAck":
            if not data.get("success") and data.get("fatal"):
                print("嚴重錯誤：連線將被中斷")
                self.running = False

    async def send_heartbeat(self):
        """發送心跳"""
        while self.running:
            message = {
                "type": "ping",
                "timestamp": int(time.time() * 1000)
            }
            await self.ws.send(json.dumps(message))
            await asyncio.sleep(25)

    async def send_device_status(self):
        """發送設備狀態"""
        while self.running:
            devices_data = []
            for device in self.devices.values():
                device_dict = device.to_dict()
                # 添加本地存儲的 imageCode
                if device.mac_address in self.image_codes:
                    device_dict["data"]["imageCode"] = self.image_codes[device.mac_address]
                devices_data.append(device_dict)

            message = {
                "type": "deviceStatus",
                "devices": devices_data
            }
            await self.ws.send(json.dumps(message))
            await asyncio.sleep(5)

    async def handle_image_update(self, data):
        """處理圖像更新（直接傳輸）"""
        device_mac = data.get("deviceMac")
        image_code = data.get("imageCode")
        image_data = data.get("imageData")
        rawdata = data.get("rawdata")  # EPD 原始數據
        data_type = data.get("dataType", "rawdata")  # 獲取數據格式類型

        if device_mac and image_code:
            # 更新本地 imageCode
            self.image_codes[device_mac] = image_code
            print(f"已更新設備 {device_mac} 的 imageCode: {image_code}, 格式: {data_type}")

            # 優先使用 rawdata 發送到設備
            if rawdata:
                print(f"使用 EPD 原始數據發送到設備 {device_mac}")
                # 根據 dataType 處理數據
                processed_data = self.process_received_data(rawdata, data_type)
                await self.send_epd_data_to_device(device_mac, processed_data)
            else:
                print(f"使用圖像數據發送到設備 {device_mac}")
                await self.send_image_to_device(device_mac, image_data)

    async def handle_chunk_start(self, data):
        """處理分片傳輸開始"""
        chunk_id = data.get("chunkId")
        device_mac = data.get("deviceMac")
        image_code = data.get("imageCode")
        total_chunks = data.get("totalChunks")
        total_size = data.get("totalSize")
        chunk_size = data.get("chunkSize")
        data_type = data.get("dataType", "rawdata")  # 獲取數據格式類型

        print(f"開始接收分片: {chunk_id}, 設備: {device_mac}, 總分片: {total_chunks}, 格式: {data_type}")

        # 初始化分片接收器
        self.chunk_receiver = ChunkReceiver(
            chunk_id=chunk_id,
            device_mac=device_mac,
            image_code=image_code,
            total_chunks=total_chunks,
            total_size=total_size,
            chunk_size=chunk_size,
            data_type=data_type  # 傳遞數據格式類型
        )

        # 發送開始確認
        ack_message = {
            "type": "chunk_start_ack",
            "chunkId": chunk_id,
            "timestamp": int(time.time() * 1000)
        }
        await self.ws.send(json.dumps(ack_message))

    async def handle_chunk_data(self, binary_data):
        """處理分片數據"""
        if not self.chunk_receiver:
            print("收到分片數據但沒有活躍的接收器")
            return

        # 解析嵌入式 Index
        if len(binary_data) < 4:
            print("分片數據太短，無法解析 index")
            return

        # 提取 chunkIndex (little-endian)
        chunk_index = int.from_bytes(binary_data[:4], byteorder='little')
        actual_data = binary_data[4:]

        print(f"收到分片 {chunk_index}: {len(actual_data)} bytes")

        # 存儲分片數據
        success = self.chunk_receiver.add_chunk(chunk_index, actual_data)

        if success:
            # 發送分片確認
            ack_message = {
                "type": "chunk_ack",
                "chunkId": self.chunk_receiver.chunk_id,
                "chunkIndex": chunk_index,
                "timestamp": int(time.time() * 1000)
            }
            await self.ws.send(json.dumps(ack_message))

            # 檢查是否完成
            if self.chunk_receiver.is_complete():
                complete_data = self.chunk_receiver.get_complete_data()
                await self.handle_complete_chunk_data(
                    self.chunk_receiver.device_mac,
                    self.chunk_receiver.image_code,
                    complete_data,
                    self.chunk_receiver.data_type  # 傳遞數據格式類型
                )
                self.chunk_receiver = None

    async def handle_complete_chunk_data(self, device_mac, image_code, complete_data, data_type):
        """處理完整的分片數據"""
        print(f"分片傳輸完成: 設備 {device_mac}, 數據大小 {len(complete_data)} bytes, 格式: {data_type}")

        # 根據 dataType 處理數據
        processed_data = self.process_received_data(complete_data, data_type)

        # 更新本地 imageCode
        self.image_codes[device_mac] = image_code

        # 發送到設備
        await self.send_epd_data_to_device(device_mac, processed_data)

    def process_received_data(self, rawdata, data_type):
        """處理接收到的數據"""
        if data_type == "rawdata":
            # 原始數據，直接使用
            return rawdata
        elif data_type == "runlendata":
            # RLE 壓縮數據，需要解壓縮
            rawdata_bytes = bytes(rawdata) if isinstance(rawdata, list) else rawdata

            # 分離 ImageInfo 和壓縮的像素數據
            image_info = rawdata_bytes[:12]  # 前 12 bytes 是 ImageInfo
            compressed_pixels = rawdata_bytes[12:]  # 後續是壓縮的像素數據

            # 解壓縮像素數據
            decompressed_pixels = self.decompress_rle_data(compressed_pixels)

            # 重新組合完整數據
            complete_data = image_info + decompressed_pixels
            return complete_data
        else:
            raise Exception(f"Unsupported data type: {data_type}")

    def decompress_rle_data(self, compressed_data):
        """解壓縮 RLE 數據"""
        decompressed = []
        i = 0

        while i < len(compressed_data):
            header = compressed_data[i]
            i += 1

            if (header & 0x80) == 0:
                # 重複序列：bit7 = 0
                run_length = header
                if i >= len(compressed_data):
                    raise Exception('Incomplete RLE data: missing value byte')
                value = compressed_data[i]
                i += 1

                # 重複 run_length 次
                for _ in range(run_length):
                    decompressed.append(value)
            else:
                # 非重複序列：bit7 = 1
                length = header & 0x7F
                if i + length > len(compressed_data):
                    raise Exception('Incomplete RLE data: insufficient data bytes')

                # 複製 length 個字節
                for j in range(length):
                    decompressed.append(compressed_data[i + j])
                i += length

        return bytes(decompressed)

    async def send_epd_data_to_device(self, device_mac, rawdata):
        """發送 EPD 原始數據到設備"""
        # 將 rawdata 陣列轉換為 bytes
        epd_bytes = bytes(rawdata) if isinstance(rawdata, list) else rawdata

        # 實際實作中需要通過藍牙等方式發送到 EPD 設備
        print(f"發送 {len(epd_bytes)} bytes EPD 數據到設備 {device_mac}")
        # await self.bluetooth_manager.send_to_device(device_mac, epd_bytes)

# 使用範例
async def main():
    gateway = EPDGateway(
        server_url="ws://localhost:3000/ws",
        token="your_jwt_token_here",
        mac_address="AA:BB:CC:DD:EE:FF"
    )
    await gateway.start()

class ChunkReceiver:
    """分片接收器"""
    def __init__(self, chunk_id, device_mac, image_code, total_chunks, total_size, chunk_size, data_type="rawdata"):
        self.chunk_id = chunk_id
        self.device_mac = device_mac
        self.image_code = image_code
        self.total_chunks = total_chunks
        self.total_size = total_size
        self.chunk_size = chunk_size
        self.data_type = data_type  # 數據格式類型
        self.received_chunks = {}  # chunkIndex -> data
        self.received_count = 0

    def add_chunk(self, chunk_index, data):
        """添加分片數據"""
        if chunk_index in self.received_chunks:
            print(f"重複的分片 {chunk_index}，忽略")
            return False

        if chunk_index >= self.total_chunks:
            print(f"無效的分片索引 {chunk_index}，總分片數 {self.total_chunks}")
            return False

        self.received_chunks[chunk_index] = data
        self.received_count += 1
        print(f"已接收分片 {chunk_index}/{self.total_chunks} ({self.received_count}/{self.total_chunks})")
        return True

    def is_complete(self):
        """檢查是否已接收所有分片"""
        return self.received_count == self.total_chunks

    def get_complete_data(self):
        """重組完整數據"""
        if not self.is_complete():
            raise Exception("分片接收未完成")

        # 按順序重組數據
        complete_data = bytearray()
        for i in range(self.total_chunks):
            if i not in self.received_chunks:
                raise Exception(f"缺少分片 {i}")
            complete_data.extend(self.received_chunks[i])

        return bytes(complete_data)

    def get_progress(self):
        """獲取接收進度"""
        return {
            "received": self.received_count,
            "total": self.total_chunks,
            "percentage": (self.received_count / self.total_chunks) * 100
        }

if __name__ == "__main__":
    asyncio.run(main())
```

### 2. 設備狀態監控範例

```python
class DeviceMonitor:
    def __init__(self, gateway):
        self.gateway = gateway

    async def scan_bluetooth_devices(self):
        """掃描藍牙設備"""
        while self.gateway.running:
            # 模擬設備掃描
            discovered_devices = await self.bluetooth_scan()

            for device_info in discovered_devices:
                mac = device_info["mac"]
                if mac not in self.gateway.devices:
                    # 新發現的設備
                    device = Device(mac)
                    device.update_status(**device_info)
                    self.gateway.devices[mac] = device
                    print(f"發現新設備: {mac}")
                else:
                    # 更新現有設備狀態
                    self.gateway.devices[mac].update_status(**device_info)

            await asyncio.sleep(10)
```

## 總結

本文檔提供了完整的 Gateway/Device 實作指南，包括：

1. **通信協議**：詳細的 WebSocket 消息格式和處理邏輯
2. **分片傳輸**：嵌入式 Index 分片傳輸機制的完整實作
3. **架構設計**：Gateway 和 Device 的內部架構建議
4. **錯誤處理**：完善的錯誤處理和恢復機制
5. **安全考量**：重要的安全注意事項
6. **實作範例**：可直接使用的代碼框架，包含分片傳輸支援

實作人員可以基於此文檔開發符合系統要求的 Gateway 和 Device 程序。

## 🆕 版本 2.2.0 新功能

### 分片傳輸支援
- **嵌入式 Index 模式**：每個分片前 4 bytes 包含 chunkIndex
- **Gateway 能力上報**：在 gatewayInfo 中上報 chunkingSupport 能力
- **自動判斷機制**：Server 根據 Gateway 能力自動選擇傳輸方式
- **硬體限制支援**：支援 4 bytes - 512KB 的分片大小範圍
- **性能警告系統**：當分片數量過多時發出警告
- **可靠傳輸**：ACK 機制確保每個分片都被正確接收

### 數據格式處理更新
- **dataType 欄位統一**：使用 `dataType` 取代 `rawdataFormat` 欄位
- **RLE 壓縮支援**：完整的 Run-Length Encoding 實作
- **壓縮範圍明確**：只壓縮像素數據，不包含 ImageInfo 結構和 chunk index
- **解壓縮實作**：提供完整的解壓縮算法和範例代碼
- **錯誤處理增強**：針對數據格式處理的錯誤處理機制

### 韌體傳輸支援
- **韌體分片傳輸**：支援大型韌體檔案的分片傳輸
- **韌體直接傳輸**：小型韌體檔案的直接傳輸
- **智能傳輸判斷**：根據韌體大小和Gateway能力自動選擇傳輸方式
- **韌體相容性驗證**：自動驗證韌體版本和硬體相容性
- **進度追蹤**：實時韌體更新進度報告
- **錯誤處理**：完整的韌體更新錯誤處理機制

### 時序圖文檔增強
- **📊 詳細時序圖**：新增圖片傳輸和韌體傳輸的完整時序圖
- **🖼️ 圖片傳輸流程**：直接傳輸和分片傳輸的詳細時序圖
- **🔧 韌體傳輸流程**：直接傳輸和分片傳輸的詳細時序圖
- **📋 實作指導**：為裝置人員提供清晰的開發指導

## 附錄 A：完整消息流程時序圖

```mermaid
sequenceDiagram
    participant G as Gateway
    participant S as Server
    participant D as Device
    participant DB as Database

    Note over G,DB: 完整的消息交互流程

    rect rgb(240, 248, 255)
        Note over G,S: 階段 1: 連接建立
        G->>S: WebSocket 連接 + JWT Token
        S->>S: 驗證 Token (MAC, Gateway ID, Store ID)
        S->>DB: 檢查 Gateway 記錄
        S->>G: 連接成功
        S->>G: welcome 消息
        S->>DB: 更新 Gateway 狀態為 online
    end

    rect rgb(240, 255, 240)
        Note over G,S: 階段 2: 初始化
        G->>S: ping 消息
        S->>G: pong 回應
        G->>S: gatewayInfo 消息
        S->>DB: 更新 Gateway 信息
        S->>G: gatewayInfoAck (success: true)
    end

    rect rgb(255, 248, 240)
        Note over G,DB: 階段 3: 設備發現與狀態同步
        G->>D: 藍牙掃描
        D->>G: 設備信息回應
        G->>G: 更新本地設備列表

        loop 每 5 秒
            G->>S: deviceStatus 消息 (所有設備)
            S->>DB: 更新設備狀態
            S->>G: deviceStatusAck 回應
        end
    end

    rect rgb(255, 240, 255)
        Note over S,D: 階段 4A: 直接圖像更新流程
        S->>S: 用戶觸發圖像更新 (小圖像)
        S->>DB: 計算 imageCode 並保存
        S->>G: update_preview 消息
        G->>G: 更新本地 imageCode
        G->>D: 發送圖像數據 (藍牙)
        D->>G: 更新確認
        G->>S: deviceStatus (包含新 imageCode)
        S->>DB: 更新設備 imageUpdateStatus
    end

    rect rgb(255, 255, 240)
        Note over S,D: 階段 4B: 分片圖像更新流程
        S->>S: 用戶觸發圖像更新 (大圖像)
        S->>S: 判斷需要分片傳輸
        S->>DB: 計算 imageCode 並保存
        S->>G: image_chunk_start 消息
        G->>S: chunk_start_ack 確認

        loop 每個分片
            S->>G: 二進制分片數據 (嵌入式 Index)
            G->>S: chunk_ack 確認
        end

        G->>G: 重組完整數據並更新 imageCode
        G->>D: 發送完整圖像數據 (藍牙)
        D->>G: 更新確認
        G->>S: deviceStatus (包含新 imageCode)
        S->>DB: 更新設備 imageUpdateStatus
    end

    rect rgb(248, 248, 255)
        Note over G,S: 階段 5: 持續監控
        loop 每 25 秒
            G->>S: ping 心跳
            S->>G: pong 回應
        end

        loop 每 30 秒
            G->>S: gatewayInfo 更新
            S->>G: gatewayInfoAck 回應
        end
    end
```

## 附錄 B：圖片傳輸詳細時序圖

### B.1 直接圖片傳輸時序圖

```mermaid
sequenceDiagram
    participant U as 用戶
    participant S as Server
    participant G as Gateway
    participant D as EPD設備

    Note over U,D: 直接圖片傳輸流程 (小圖片或Gateway不支援分片)

    U->>S: 觸發圖片更新請求
    S->>S: 生成 EPD rawdata 和 imageCode
    S->>S: 檢查 Gateway 分片能力
    S->>S: 判斷: rawdata 大小 <= maxChunkSize
    S->>S: 判斷: JSON 訊息大小 <= maxSingleMessageSize

    Note over S: 決定使用直接傳輸

    S->>G: update_preview 消息
    Note right of S: 包含: deviceMac, imageCode,<br/>rawdata, dataType, imageData

    G->>G: 接收並解析 update_preview
    G->>G: 根據 dataType 處理 rawdata
    alt dataType == "runlendata"
        G->>G: 分離 ImageInfo (12 bytes)
        G->>G: 解壓縮像素數據 (RLE)
        G->>G: 重組完整 EPD 數據
    else dataType == "rawdata"
        G->>G: 直接使用 rawdata
    end

    G->>G: 更新本地 imageCode
    G->>D: 透過藍牙發送 EPD 數據
    D->>D: 更新顯示內容
    D->>G: 更新完成確認

    Note over G: 下次 deviceStatus 將包含新 imageCode

    G->>S: deviceStatus (包含新 imageCode)
    S->>S: 更新設備狀態
```

### B.2 分片圖片傳輸時序圖

```mermaid
sequenceDiagram
    participant U as 用戶
    participant S as Server
    participant G as Gateway
    participant D as EPD設備

    Note over U,D: 分片圖片傳輸流程 (大圖片且Gateway支援分片)

    U->>S: 觸發圖片更新請求
    S->>S: 生成 EPD rawdata 和 imageCode
    S->>S: 檢查 Gateway 分片能力
    alt rawdata 大小 > maxChunkSize
        Note over S: 第一階段: rawdata 超過分片大小限制
        S->>S: 決定使用分片傳輸
    else JSON 訊息大小 > maxSingleMessageSize
        Note over S: 第二階段: JSON 訊息過大
        S->>S: 決定使用分片傳輸
    end

    S->>S: 根據 dataType 處理 rawdata
    S->>S: 計算分片參數 (totalChunks, chunkSize)
    S->>S: 生成唯一 chunkId

    S->>G: image_chunk_start 消息
    Note right of S: 包含: chunkId, deviceMac, imageCode,<br/>totalChunks, totalSize, chunkSize,<br/>dataType, mode: "embedded_index"

    G->>G: 初始化 ChunkReceiver
    G->>G: 準備接收分片數據
    G->>S: chunk_start_ack (status: "ready")

    loop 每個分片 (i = 0 to totalChunks-1)
        S->>S: 準備分片 i 數據
        S->>S: 創建嵌入式 Index: [4 bytes: chunkIndex][N bytes: 數據]
        S->>G: 二進制分片數據

        G->>G: 解析前 4 bytes 獲取 chunkIndex
        G->>G: 提取實際數據並存儲
        G->>G: 檢查重複分片
        G->>S: chunk_ack (chunkIndex, status: "received")

        Note over S: 等待 ACK 後發送下一個分片
    end

    S->>G: image_chunk_complete 消息
    Note right of S: 包含: chunkId, deviceMac,<br/>imageCode, totalChecksum

    G->>G: 驗證所有分片已接收
    G->>G: 按順序重組完整數據
    G->>G: 驗證數據完整性 (可選)
    G->>G: 根據 dataType 處理重組數據
    alt dataType == "runlendata"
        G->>G: 分離 ImageInfo (12 bytes)
        G->>G: 解壓縮像素數據 (RLE)
        G->>G: 重組完整 EPD 數據
    else dataType == "rawdata"
        G->>G: 直接使用重組數據
    end

    G->>G: 更新本地 imageCode
    G->>S: chunk_complete_ack (status: "success", receivedSize)

    G->>D: 透過藍牙發送完整 EPD 數據
    D->>D: 更新顯示內容
    D->>G: 更新完成確認

    Note over G: 下次 deviceStatus 將包含新 imageCode

    G->>S: deviceStatus (包含新 imageCode)
    S->>S: 更新設備狀態
```

## 附錄 C：韌體傳輸詳細時序圖

### C.1 直接韌體傳輸時序圖

```mermaid
sequenceDiagram
    participant U as 用戶
    participant S as Server
    participant G as Gateway
    participant T as 目標設備

    Note over U,T: 直接韌體傳輸流程 (小韌體檔案)

    U->>S: 觸發韌體更新請求
    S->>S: 讀取韌體檔案數據
    S->>S: 驗證韌體相容性
    S->>S: 檢查 Gateway 分片能力
    S->>S: 判斷: 韌體大小 <= maxChunkSize
    S->>S: 判斷: JSON 訊息大小 <= maxSingleMessageSize

    Note over S: 決定使用直接傳輸

    S->>G: firmware_update 消息
    Note right of S: 包含: deviceMac, firmwareId,<br/>firmwareData, dataType: "firmware",<br/>firmwareInfo (version, model, checksum)

    G->>G: 接收並解析 firmware_update
    G->>G: 驗證韌體相容性
    G->>G: 檢查硬體版本匹配
    G->>G: 驗證校驗和
    G->>S: firmware_update_ack (status: "success")

    G->>G: 開始韌體更新流程

    alt 目標是 Gateway 自身
        G->>G: 安裝 WiFi/藍牙韌體
        G->>G: 重啟並應用新韌體
    else 目標是下游設備
        G->>T: 透過藍牙發送韌體數據
        T->>T: 安裝韌體
        T->>G: 安裝完成確認
    end

    alt 韌體更新成功
        Note over G: 韌體更新結果通過下次 gatewayInfo 回報
        G->>S: gatewayInfo (新韌體版本)
        S->>S: 檢查韌體版本變化確認更新成功
    else 韌體更新失敗
        G->>S: firmware_update_error (錯誤代碼和訊息)
        S->>S: 轉發錯誤給前端客戶端
        G->>S: gatewayInfo (原韌體版本)
        S->>S: 確認韌體版本未變化，更新失敗
    end
```

### C.2 分片韌體傳輸時序圖

```mermaid
sequenceDiagram
    participant U as 用戶
    participant S as Server
    participant G as Gateway
    participant T as 目標設備

    Note over U,T: 分片韌體傳輸流程 (大韌體檔案)

    U->>S: 觸發韌體更新請求
    S->>S: 讀取韌體檔案數據
    S->>S: 驗證韌體相容性
    S->>S: 檢查 Gateway 分片能力
    alt 韌體大小 > maxChunkSize
        Note over S: 第一階段: 韌體超過分片大小限制
        S->>S: 決定使用分片傳輸
    else JSON 訊息大小 > maxSingleMessageSize
        Note over S: 第二階段: JSON 訊息過大
        S->>S: 決定使用分片傳輸
    end

    S->>S: 計算分片參數 (totalChunks, chunkSize)
    S->>S: 生成唯一 chunkId

    S->>G: firmware_chunk_start 消息
    Note right of S: 包含: chunkId, deviceMac, firmwareId,<br/>totalChunks, totalSize, chunkSize,<br/>dataType: "firmware", mode: "embedded_index",<br/>firmwareInfo (version, model, checksum)

    G->>G: 初始化韌體 ChunkReceiver
    G->>G: 驗證韌體相容性
    G->>G: 準備接收韌體分片數據
    G->>S: firmware_chunk_start_ack (status: "ready")

    loop 每個分片 (i = 0 to totalChunks-1)
        S->>S: 準備韌體分片 i 數據
        S->>S: 創建嵌入式 Index: [4 bytes: chunkIndex][N bytes: 韌體數據]
        S->>G: 二進制韌體分片數據
        S->>S: 廣播下載進度

        G->>G: 解析前 4 bytes 獲取 chunkIndex
        G->>G: 提取實際韌體數據並存儲
        G->>G: 檢查重複分片
        G->>S: firmware_chunk_ack (chunkIndex, status: "received")

        Note over S: 等待 ACK 後發送下一個分片
    end

    S->>G: firmware_chunk_complete 消息
    Note right of S: 包含: chunkId, deviceMac,<br/>firmwareId, totalChecksum
    S->>S: 廣播驗證階段

    G->>G: 驗證所有韌體分片已接收
    G->>G: 按順序重組完整韌體數據
    G->>G: 計算並驗證總校驗和
    G->>S: firmware_chunk_complete_ack (status: "success", finalChecksum)

    G->>G: 開始韌體更新流程

    alt 目標是 Gateway 自身
        G->>G: 安裝 WiFi/藍牙韌體
        G->>G: 重啟並應用新韌體
    else 目標是下游設備
        G->>T: 透過藍牙發送完整韌體數據
        T->>T: 安裝韌體
        T->>G: 安裝完成確認
    end

    alt 韌體更新成功
        Note over G: 韌體更新結果通過下次 gatewayInfo 回報
        G->>S: gatewayInfo (新韌體版本)
        S->>S: 檢查韌體版本變化確認更新成功
    else 韌體更新失敗
        G->>S: firmware_update_error (錯誤代碼和訊息)
        S->>S: 轉發錯誤給前端客戶端
        G->>S: gatewayInfo (原韌體版本)
        S->>S: 確認韌體版本未變化，更新失敗
    end
```

## 附錄 D：錯誤處理流程圖

```mermaid
flowchart TD
    A[收到消息] --> B{解析 JSON}
    B -->|成功| C{檢查消息類型}
    B -->|失敗| D[記錄解析錯誤]

    C -->|welcome| E[處理歡迎消息]
    C -->|pong| F[處理心跳回應]
    C -->|gatewayInfoAck| G{檢查 success 字段}
    C -->|deviceStatusAck| H[處理設備狀態確認]
    C -->|update_preview| I[處理圖像更新]
    C -->|error| J[處理錯誤消息]
    C -->|未知類型| K[記錄未知消息類型]

    G -->|success: true| L[網關信息更新成功]
    G -->|success: false, fatal: true| M[嚴重錯誤，中斷連線]
    G -->|success: false, fatal: false| N[一般錯誤，繼續運行]

    D --> O[發送錯誤報告]
    K --> O
    M --> P[停止所有任務]

    E --> Q[發送 gatewayInfo]
    F --> R[更新心跳狀態]
    H --> S[更新設備狀態]
    I --> T[更新本地 imageCode]
    J --> U[記錄錯誤信息]
    L --> V[繼續正常運行]
    N --> V

    O --> W[記錄到日誌]
    P --> X[清理資源並退出]
    Q --> V
    R --> V
    S --> V
    T --> V
    U --> V
    V --> Y[等待下一個消息]
    W --> Y
```

## 附錄 E：設備生命週期管理

### 1. 設備狀態轉換圖

```mermaid
stateDiagram-v2
    [*] --> Unknown: 設備首次被發現
    Unknown --> Discovered: 成功識別設備
    Discovered --> Pairing: 開始配對
    Pairing --> Paired: 配對成功
    Pairing --> Failed: 配對失敗
    Failed --> Discovered: 重新嘗試
    Paired --> Connected: 建立連接
    Connected --> Online: 設備正常工作
    Online --> Updating: 接收圖像更新
    Updating --> Online: 更新完成
    Updating --> UpdateFailed: 更新失敗
    UpdateFailed --> Online: 重新嘗試成功
    Online --> Disconnected: 連接丟失
    Disconnected --> Connected: 重新連接
    Disconnected --> Offline: 長時間無響應
    Offline --> [*]: 設備移除
    Connected --> [*]: 手動移除設備
```

### 2. 設備管理最佳實踐

```python
class DeviceLifecycleManager:
    def __init__(self):
        self.devices = {}
        self.device_timeouts = {}
        self.max_offline_time = 300  # 5 分鐘

    async def manage_device_lifecycle(self):
        """管理設備生命週期"""
        while True:
            current_time = time.time()

            for mac, device in list(self.devices.items()):
                last_seen = device.last_seen or 0
                offline_duration = current_time - last_seen

                if device.status == "online" and offline_duration > 60:
                    # 設備超過 1 分鐘無響應，標記為離線
                    device.status = "offline"
                    print(f"設備 {mac} 標記為離線")

                elif device.status == "offline" and offline_duration > self.max_offline_time:
                    # 設備超過最大離線時間，移除設備
                    del self.devices[mac]
                    print(f"設備 {mac} 已被移除")

            await asyncio.sleep(30)  # 每 30 秒檢查一次
```

## 附錄 F：性能優化建議

### 1. 消息批處理

```python
class MessageBatcher:
    def __init__(self, max_batch_size=10, max_wait_time=1.0):
        self.max_batch_size = max_batch_size
        self.max_wait_time = max_wait_time
        self.pending_messages = []
        self.last_send_time = time.time()

    async def add_message(self, message):
        """添加消息到批處理隊列"""
        self.pending_messages.append(message)

        # 檢查是否需要立即發送
        if (len(self.pending_messages) >= self.max_batch_size or
            time.time() - self.last_send_time >= self.max_wait_time):
            await self.flush_messages()

    async def flush_messages(self):
        """發送所有待處理的消息"""
        if self.pending_messages:
            batch_message = {
                "type": "batch",
                "messages": self.pending_messages,
                "timestamp": int(time.time() * 1000)
            }
            await self.send_to_server(batch_message)
            self.pending_messages.clear()
            self.last_send_time = time.time()
```

### 2. 連接池管理

```python
class ConnectionPool:
    def __init__(self, max_connections=5):
        self.max_connections = max_connections
        self.active_connections = {}
        self.connection_queue = asyncio.Queue()

    async def get_connection(self, device_mac):
        """獲取設備連接"""
        if device_mac in self.active_connections:
            return self.active_connections[device_mac]

        if len(self.active_connections) >= self.max_connections:
            # 等待連接釋放
            await self.connection_queue.get()

        # 建立新連接
        connection = await self.create_connection(device_mac)
        self.active_connections[device_mac] = connection
        return connection

    async def release_connection(self, device_mac):
        """釋放設備連接"""
        if device_mac in self.active_connections:
            await self.active_connections[device_mac].close()
            del self.active_connections[device_mac]
            self.connection_queue.put_nowait(True)
```

## 附錄 G：測試和調試

### 1. 單元測試範例

```python
import unittest
from unittest.mock import AsyncMock, MagicMock
import asyncio

class TestGatewayWebSocket(unittest.TestCase):
    def setUp(self):
        self.gateway = EPDGateway(
            server_url="ws://test-server",
            token="test-token",
            mac_address="AA:BB:CC:DD:EE:FF"
        )

    async def test_handle_welcome_message(self):
        """測試歡迎消息處理"""
        welcome_message = {
            "type": "welcome",
            "message": "WebSocket 連接成功",
            "timestamp": 1640995200000
        }

        # 模擬 WebSocket 發送
        self.gateway.ws = AsyncMock()

        await self.gateway.process_message(welcome_message)

        # 驗證是否發送了 gatewayInfo 消息
        self.gateway.ws.send.assert_called_once()

    async def test_device_status_report(self):
        """測試設備狀態報告"""
        # 添加測試設備
        device = Device("11:22:33:44:55:66")
        device.status = "online"
        device.data["battery"] = 85
        self.gateway.devices[device.mac_address] = device

        # 生成狀態報告
        report = self.gateway.generate_device_status_report()

        self.assertEqual(len(report["devices"]), 1)
        self.assertEqual(report["devices"][0]["macAddress"], "11:22:33:44:55:66")
        self.assertEqual(report["devices"][0]["status"], "online")

    def test_image_code_update(self):
        """測試 imageCode 更新"""
        device_mac = "11:22:33:44:55:66"
        image_code = "12345678"

        self.gateway.update_image_code(device_mac, image_code)

        self.assertEqual(self.gateway.image_codes[device_mac], image_code)

# 運行測試
if __name__ == "__main__":
    unittest.main()
```

### 2. 調試工具

```python
class DebugLogger:
    def __init__(self, log_level="INFO"):
        self.log_level = log_level
        self.message_history = []

    def log_message(self, direction, message_type, content):
        """記錄消息"""
        timestamp = time.time()
        log_entry = {
            "timestamp": timestamp,
            "direction": direction,  # "sent" or "received"
            "type": message_type,
            "content": content
        }
        self.message_history.append(log_entry)

        if self.log_level == "DEBUG":
            print(f"[{direction.upper()}] {message_type}: {content}")

    def export_logs(self, filename):
        """導出日誌到文件"""
        with open(filename, 'w') as f:
            json.dump(self.message_history, f, indent=2)

    def analyze_performance(self):
        """分析性能指標"""
        ping_times = []
        for i, entry in enumerate(self.message_history):
            if entry["type"] == "ping":
                # 查找對應的 pong 回應
                for j in range(i+1, len(self.message_history)):
                    if self.message_history[j]["type"] == "pong":
                        response_time = self.message_history[j]["timestamp"] - entry["timestamp"]
                        ping_times.append(response_time)
                        break

        if ping_times:
            avg_ping = sum(ping_times) / len(ping_times)
            print(f"平均響應時間: {avg_ping:.3f} 秒")
            print(f"最大響應時間: {max(ping_times):.3f} 秒")
            print(f"最小響應時間: {min(ping_times):.3f} 秒")
```

## 附錄 F：部署檢查清單

### 1. 部署前檢查

- [ ] **網絡配置**
  - [ ] 確認 Server IP 和端口可達
  - [ ] 檢查防火牆設置
  - [ ] 測試 WebSocket 連接

- [ ] **認證配置**
  - [ ] 確認 JWT Token 有效
  - [ ] 驗證 MAC 地址匹配
  - [ ] 檢查權限設置

- [ ] **設備配置**
  - [ ] 確認藍牙功能正常
  - [ ] 測試設備掃描
  - [ ] 驗證設備通信

- [ ] **日誌配置**
  - [ ] 設置日誌級別
  - [ ] 配置日誌輪轉
  - [ ] 確認日誌存儲空間

### 2. 運行時監控

```python
class HealthMonitor:
    def __init__(self, gateway):
        self.gateway = gateway
        self.health_metrics = {
            "connection_status": "unknown",
            "last_heartbeat": 0,
            "device_count": 0,
            "message_count": 0,
            "error_count": 0
        }

    async def monitor_health(self):
        """監控系統健康狀態"""
        while True:
            # 更新健康指標
            self.health_metrics.update({
                "connection_status": "connected" if self.gateway.ws else "disconnected",
                "device_count": len(self.gateway.devices),
                "uptime": time.time() - self.gateway.start_time
            })

            # 檢查異常情況
            if self.health_metrics["error_count"] > 10:
                print("警告：錯誤數量過多")

            if time.time() - self.health_metrics["last_heartbeat"] > 60:
                print("警告：心跳超時")

            await asyncio.sleep(30)

    def get_health_report(self):
        """獲取健康報告"""
        return self.health_metrics.copy()
```

這份完整的實作指南為 Gateway/Device 開發人員提供了：

1. **詳細的通信協議規範**
2. **完整的架構設計指導**
3. **實用的代碼範例**
4. **全面的錯誤處理機制**
5. **性能優化建議**
6. **測試和調試工具**
7. **部署檢查清單**
8. **📊 詳細時序圖**：圖片傳輸和韌體傳輸的完整流程圖

開發人員可以直接基於這些內容進行實際的 Gateway 和 Device 程序開發。

---

**最後更新**: 2025年7月
**版本**: 2.5.0 - 錯誤處理增強版
**主要更新**:
- **📊 函數呼叫時序圖**: 新增圖片傳輸和韌體傳輸的詳細時序圖
- **🖼️ 圖片傳輸流程**: 直接傳輸和分片傳輸的完整時序圖
- **🔧 韌體傳輸流程**: 直接傳輸和分片傳輸的完整時序圖
- **兩階段分片決策邏輯**: 詳細的分片決策流程說明
- **maxSingleMessageSize 參數**: JSON 訊息大小限制參數
- **智能分片切換**: 當 rawdata 小但 JSON 訊息大時自動切換到分片傳輸
- **參數設定指南**: 提供 maxChunkSize 和 maxSingleMessageSize 的設定建議
- **supportedFormat 參數**: rawdata 格式偏好設定
- **韌體傳輸支援**: 完整的韌體更新流程和時序圖
- **🔧 韌體更新流程修正**: 明確 Gateway 不需要發送 `firmware_update_progress` 和 `firmware_update_complete`
- **📊 韌體結果確認**: 韌體更新結果通過 `gatewayInfo` 中的韌體版本變化確認
- **🎯 實作指導優化**: 為裝置人員提供更準確的開發指導
- **🚨 錯誤處理增強**: 新增 `firmware_update_error` 消息支援，Gateway 可主動報告韌體更新錯誤
- **⏱️ ACK Timeout 機制**: 完善的 timeout 處理，確保傳輸問題能正確告知用戶
- **🧪 錯誤模擬功能**: test-ws-client-interactive.js 新增 `firmware-error` 命令支援錯誤測試
- **📋 錯誤處理流程**: 詳細的錯誤處理流程說明，包含 ACK 錯誤和專用錯誤消息兩種方式
