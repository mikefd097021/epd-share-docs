# 電子紙管理系統 API 文件列表

本文檔匯總了電子紙（ESL）管理系統所提供的 API 接口功能說明。

## 目錄

- [Part 1: 登入 (Log in)](#part-1-登入-log-in)
- [Part 2: 門市管理 API (Store API)](#part-2-門市管理-api-store-api)
- [Part 3: 商品數據 API (Data/product API)](#part-3-商品數據-api-dataproduct-api)
- [Part 4: 模板 API (Template API)](#part-4-模板-api-template-api)
- [Part 5: 基站/網關 API (Gateway API)](#part-5-基站網關-api-gateway-api)
- [Part 6: 設備/標籤 API (Device/tag API)](#part-6-設備標籤-api-devicetag-api)
- [Part 7: 管理員帳號 API (Admin Account API)](#part-7-管理員帳號-api-admin-account-api)
- [Part 8: APP 應用 API (APP API)](#part-8-app-應用-api-app-api)

---

## Part 1: 登入 (Log in)

*   **1.1** 系統登入 (Log in)

## Part 2: 門市管理 API (Store API)

本章節包含門市的新增、修改、狀態控制及資訊獲取。

*   **2.1** 新增門市 (Add store)
*   **2.2** 修改門市 (Modify store)
*   **2.3** 開啟/關閉門市 (Close / open store)
*   **2.4** 獲取告警資訊 (Get warning information)
*   **2.5** 獲取門市資訊 (Get store information)
*   **2.6** 獲取操作日誌資訊 (Get operation log information)

## Part 3: 商品數據 API (Data/product API)

本章節涉及商品資料的管理以及螢幕更新（刷屏）控制。
> **注意**：術語 "Brush" 指的是刷新電子紙螢幕畫面。

*   **3.1** 查詢動態欄位 (Query dynamic fields)
*   **3.2** 新增門市商品數據 (Add store data/product)
*   **3.3** 修改商品數據並刷屏 (Modify data/product (brush))
*   **3.4** 批量修改商品數據但不刷屏 (Modify data/product in batch (no brush))
*   **3.5** 刪除商品數據 (Delete data/product)
*   **3.6** 獲取商品數據資訊 (Get data/product information)
*   **3.7** 根據商品數據查詢綁定的標籤及其所屬門市 (Query the binding tag and the store it belongs to by data/product)
*   **3.8** 批量新增/修改商品數據並刷屏（包含 RGB 燈光開關控制） (Add/Modify data/product in batch and brush including turn on and off the RGB light)

## Part 4: 模板 API (Template API)

本章節用於管理電子紙的顯示版型（Template）。

*   **4.1** 查詢門市模板列表 (Query the store template list)
*   **4.2** 未綁定數據的模板預覽 (Template preview for unbound data)
*   **4.3** 已綁定數據的模板預覽 (Template preview for bound data)

## Part 5: 基站/網關 API (Gateway API)

本章節用於管理負責通訊的基站（Gateway/AP）。

*   **5.1** 新增基站 (Add gateway)
*   **5.2** 刪除基站 (Delete gateway)
*   **5.3** 查詢基站列表 (Query gateway list)
*   **5.4** 修改基站資訊 (Modify gateway information)

## Part 6: 設備/標籤 API (Device/tag API)

本章節為核心功能，包含標籤（Tag）的綁定、解綁、喚醒及燈光控制。

*   **6.1** 批量新增設備 (Add devices in batch)
*   **6.2** 根據標籤 MAC 批量喚醒設備 (Wake up devices in batch according to the label mac)
*   **6.3** 批量刪除設備 (Delete devices in batch)
*   **6.4** 獲取設備列表 (Get device list)
*   **6.5** 綁定設備 (Bind devices)
*   **6.6** 刪除標籤與數據的綁定關係 (Delete the binding relationship between tag and data)
*   **6.7** 根據標籤 MAC 點亮 RGB 燈 (Light up RGB light by tag mac)
*   **6.8** 批量刷新標籤/刷屏 (Refresh tags in batch (brush))
*   **6.9** 修改並刷新標籤、數據與模板的綁定關係（刷屏） (Modify & brush the binding relationship of tags, data & templates (brush))
*   **6.10** 將系統數據導入指定門市 (Import the system data to the appointed store)
*   **6.11** 新增設備備註 (Add remarks of the device)
*   **6.12** 根據標籤 MAC 查詢已綁定數據 (Query bound data according to label Mac)
*   **6.13** 查詢設備詳細資訊 (Query device specific information)
*   **6.14** 根據 MAC 地址查詢設備（僅限管理員權限） (Query the device according to the MAC address - only admin account has permission)
*   **6.15** 查詢門市內未綁定設備的 MAC 列表或已綁定設備的關係 (Query the MAC list of unbound devices in the store or the binding relationship of bound devices)
*   **6.16** 查詢與基站通訊的標籤設備 (Query the label devices that communicate with gateway)
*   **6.17** 匯出警示燈與標籤的綁定列表 (Export the binding list of the warning light and the label)
*   **6.18** 設備綁定模板（多商品數據） (Bind devices with template (multi-data))

## Part 7: 管理員帳號 API (Admin Account API)

本章節用於系統層級的帳號與權限管理。

*   **7.1** 建立主帳號 (Create master account)
*   **7.2** 主帳號列表 - 分頁查詢 (List of Primary Accounts - Paginated Query)
*   **7.3** 修改主帳號 (Modification of Master Account)
*   **7.4** 刪除主帳號 (Delete master account)
*   **7.5** 將設備導入白名單 (Import the device to the whitelist)

## Part 8: APP 應用 API (APP API)

*   **8.1** 獲取 APP 藍牙更新數據 (Get APP Bluetooth Update Data)