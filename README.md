# RBAC後臺專案架構文件

# 專案架構

本文件描述 **RBAC**前端專案的整體架構、資料流與關鍵設計決策（內部的資料及截圖均為mock data）。

---

## 1. 專案概述

本專案為基於 **Next.js（App Router）** 與 **React Admin** 的後台管理系統，主要提供：

- **RBAC（角色權限）**：組織、部門、使用者、角色、選單、權限、操作紀錄、LINE/Slack 綁定、產品與訂單等管理。
- **OCR 功能**：多類型票據／單據的辨識結果管理（發票、水電瓦斯、交通票、郵票、電話費、其他等），含上傳、列表、編輯與報表。

後端 API 以 REST 為主，認證採 JWT（access + refresh），前端透過 OpenAPI 產生的 TypeScript client 與自訂 `dataProvider` 與後端對接。

---

## 2. 認證與授權

### 2.1 認證（authProvider）

- **login**：呼叫 `login(username, password)`（實為 email + password），將回傳的 `access`、`refresh` 寫入 `localStorage`。
- **logout**：清除 access/refresh，並 `queryClient.clear()`。

Token 過期判斷與 refresh 邏輯共用同一套 refresh 佇列，避免並發重複 refresh。

### 2.2 授權（權限）

- **超級使用者**：`identity.is_superuser === true` 時，列表與 API 路徑不綁組織（如直接 `resource`），且 `useHasApiPermission` 一律 true。
- **一般使用者**：**按 API 權限控制按鈕，**`useHasApiPermission(method, pathTemplates)` 依 `useMePermissionData()` 與 path 範本（支援 `{id}`）比對，決定是否顯示編輯／刪除等按鈕。

選單可見性由後端「依角色回傳選單」決定（useMeMenuData），前端僅負責依 `frontend_url`解析連結。

---

## 3. 狀態管理與快取

- **伺服器狀態**：以 **TanStack React Query** 為主。
- **快取策略**：`queryClient` 預設 `staleTime: 60 * 1000`、`retry: 1`；各 feature 的 query 可再覆寫 `staleTime` 。
- **React Admin 自身狀態**：列表 filter、pagination、sort 等存在 React Admin store，與 Resource 綁定；部分列表進階篩選使用 `useStore` 或本地 state。
- **Websocket：**進入專案開啟一條Websocket通道並實作連線檢查及斷線重連機制。

---

## 4. 重點功能模組概覽

### 4.1 RBAC

- **組織**：列表、新增、編輯、詳情（含分頁成員、設定、儲存空間、分析）；組織架構圖為 React Flow + dagre 排版，路徑 `/org-flow/:orgId`。

- **角色**：CRUD；新增、編輯頁含權限區（PermissionArea），與選單的 min_perm/max_perm 對應。
  - 權限矩陣：
    ![image.png](image%202.png)
    角色擁有的選單由所擁有的權限驅動，若該角色滿足選單所設定之最小權限，則顯示該選單，提供最大/最小權限按鈕供使用者一鍵加入。
- **選單**：CRUD；選單列表由 `useMeMenuData` 取得，供 SideBar 渲染；權限對應由 `useMenuPermissionMap` 提供。

### 4.2 OCR

- **辨識結果入口**：列表頁，以 tab 區分票據類型。

- **各類型**：各自有 config（欄位定義、列表欄、表單）、列表與編輯頁；部分列表即時更新依賴 WebSocket。
- **上傳與辨識**：UploadArea、OcrRecognizingPopover、全域 listener（useGlobalOcrListener）處理上傳與辨識中狀態並用useStore記錄辨識狀態，使得user可以在其他頁面查看辨識狀態。

- **報表**：CustomRoute `/analysis/ocr/report`（OcrDashboard）。
  ![image.png](image%206.png)
  根據使用者權限可選擇儀表板資料範圍(系統、組織、部門、個人)。

---

## 5. 核心技術決策

### 5.1：WebSocket 即時狀態同步與資源最佳化

**原始痛點**
系統的 OCR 票據辨識屬於非同步的耗時任務。原先設計若採用前端 HTTP Polling（每 10-20 秒輪詢一次），在多位使用者同時上傳大量單據的情境下，會對後端伺服器產生巨大的無效 Request 併發壓力（Thundering Herd Problem），且使用者無法在第一時間獲得「辨識完成」的 UI 反饋。

**架構解法**
捨棄輪詢，全面改用 WebSocket 建立雙向通訊，並在前端實作了客製化的連線管理層：

1. **Singleton 與 Pub/Sub 模式：** 將 WebSocket Client 封裝為單一實體。無論畫面上有多少個 React Components 需要監聽 OCR 狀態，底層永遠只維持「一條」連線，並透過自訂的 `subscribe` 機制將訊息派發給需要的元件，避免多重連線浪費。
2. **連線保活機制：** 實作定時健康檢查 (`scheduleHealthCheck`) 與異常斷線自動重連邏輯，確保長時間掛網的背景分頁不會成為死連線。
3. **生命週期管理：** 提供明確的 `disconnect` 介面，在使用者登出或 Token 失效時主動銷毀連線與計時器，杜絕 Memory Leak。

**商業效益**
將狀態更新延遲從原本預期的 10-20 秒降至 **毫秒級**，達成使用者「零延遲」的無縫體驗；同時將伺服器處理狀態查詢的負載 **降至極低**，徹底解決了未來用戶量擴展時的效能隱患。

---

### 5.2：動態 RBAC 權限流轉與 Zod 邊界防禦

**原始痛點**
開發時面臨兩大挑戰：

1. **權限矩陣極度複雜：** 系統不僅有「超級管理員」與「一般使用者」，更牽涉到多個組織、部門與自訂角色。若前端使用 Hardcoding判斷角色來控制畫面，系統將無法維護。
2. **UI 元件與 API 規格的資料割裂 (Dirty Data)：** 前端複雜的 UI 元件（如多選 Select）經常吐出物件 (Object) 或模糊型別，但後端 API 嚴格要求純數字 ID 陣列。若缺乏防禦，極易導致 API 報錯 (500 Internal Server Error) 或寫入髒資料。

**架構解法**
實作了完全由「資料驅動 (Data-Driven)」的 RBAC 架構與嚴格的資料邊界：

1. **API 級別的授權：** 捨棄傳統的 `role === 'admin'` 判斷。開發 `useHasApiPermission(method, path)` Hook，讓前端按鈕的顯示與否，直接與後端 API 權限規格綁定。當角色權限動態變更時，前端 UI 可達成「零改動」自動適應。
2. **Zod Schema 攔截與變形 (Validation & Transformation)：** 在表單送出層導入 Zod 進行嚴格校驗。不僅做基礎格式驗證，並利用`transform` function 作為「資料清洗層」。將前端 UI 產生的模糊型別（如包含 label 的 Object），在發送 Request 前強制轉型為後端合法的 Payload，徹底隔絕髒資料。
3. **動態路由與選單生成：** 側邊選單完全依賴後端回傳的權限樹 (`useMeMenuData`) 進行動態渲染，確保越權使用者連選單入口都無法看見。

**商業效益**
將前端的「商業邏輯」與「UI 呈現」完美解耦。系統成功支撐了高達上百種權限節點的自由組合配置，且大幅降低了因為前後端資料型別不一致導致的 Bug 率，使前端具備極高的防禦性與維護性。

---
