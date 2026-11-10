# 二手書交易平台系統分析與設計規格書（UML 模型）

| 項目 | 內容 |
| :--- | :--- |
| 文件版本 | v1.0 |
| 制定日期 | 2026-11-10 |
| 依據文件 | 軟體需求規格書（SRS）v0.4 |
| 專案方法 | Waterfall + AI |
| 目前階段 | 系統分析與架構設計（System Analysis & Design） |
| 審查狀態 | 待審查（Pending Review） |

---

## 1. 文件目的與系統範疇

### 1.1 文件目的
本文件依據《二手書交易平台軟體需求規格書（SRS）v0.4》之功能需求（FR）、業務規則（BR）與非功能需求（NFR），建立正式的統一塑模語言（Unified Modeling Language, UML）系統分析與設計模型。內容涵蓋使用案例模型、領域靜態結構模型、系統狀態機模型及動態循序互動模型，作為後續系統實作（Nuxt.js + Supabase）與驗收測試之架構依據。

### 1.2 系統邊界與核心角色 (Actors)
本系統為專注於台灣個人讀者之 C2C 二手書交易平台，核心在於建立嚴謹且具公信力之交易與書況查驗流程。平台不經手真實款項，記錄履約進度並透過模擬物流完成交易閉環。

| 角色 / 介面 | 類型 | 責任與權限範圍 | 對應需求 |
| :--- | :--- | :--- | :--- |
| **訪客 / 未驗證會員** | 人類角色 | 瀏覽書籍與刊登清單、依條件搜尋篩選、註冊帳號、登入系統、執行 Email 驗證。未完成驗證前禁止刊登實體書與下單購買。 | FR-001～FR-003, BR-014 |
| **買家 (Buyer)** | 人類角色 | 檢視刊登詳情、下單建立訂單、待出貨期間主動取消、確認取貨、於期限內確認收貨、提出書況不符爭議並舉證、訂單完成後評分評價。 | FR-012, FR-014～FR-017, FR-019 |
| **賣家 (Seller)** | 人類角色 | 建立實體書刊登（實拍檢核）、維護刊登狀態、出貨並填寫模擬物流資訊、待出貨期間主動取消、針對爭議案件回應與舉證、訂單完成後評分評價。 | FR-005～FR-008, FR-013, FR-018, FR-019 |
| **平台管理員 (Admin)** | 人類角色 | 使用者帳號管理（停用/恢復）、違規刊登強制下架、檢視訂單詳情、審閱爭議雙方證據並進行裁決、設定交易時限參數、查閱系統稽核紀錄。 | FR-020～FR-023, FR-028, NFR-005, NFR-006 |
| **外部書籍 API** | 外部系統 | 提供依國際標準書號（ISBN）自動查詢書目資料（Google Books API 或 Open Library API）。 | FR-006 |

---

## 2. 系統架構與技術約束

依據 SRS 之技術約束（NFR-010）與品質需求，本系統之實作規格約束如下：
1. **開發語言與框架**：採用 TypeScript 搭配 Nuxt.js 全端 Web 框架。
2. **後端服務架構**：採用 Supabase 平台，包含 PostgreSQL 關聯式資料庫、身份驗證機制（Supabase Auth）及物件儲存儲存庫（Supabase Storage）。
3. **資訊安全規範**：
   - 密碼儲存採用 bcrypt 雜湊演算法，Cost Factor $\ge$ 10（NFR-004）。
   - 管理後台嚴格實施基於角色的存取控制（RBAC），一般使用者無權限進入（NFR-005）。
   - 後台敏感管理操作（含停用帳號、強制下架、爭議裁決、時限調整）與收件個資查閱皆強制留存操作稽核紀錄（NFR-006）。
   - 圖片上傳僅支援 JPEG、PNG、WebP 格式，單張檔案上限 5 MB（NFR-007）。

---

## 3. 使用案例模型 (Use Case Model)

### 3.1 系統使用案例圖
下圖展示系統邊界內各功能模組與各角色（Actors）之互動關係：

```mermaid
flowchart LR
    %% Actors 定義
    subgraph Actors ["系統使用者與外部介面"]
        Guest["訪客 / 未驗證會員"]
        Buyer["買家（Buyer）"]
        Seller["賣家（Seller）"]
        Admin["平台管理員（Admin）"]
        BookAPI["外部書籍 API（Google Books / Open Library）"]
    end

    %% System Boundary 定義
    subgraph SystemBoundary ["二手書交易平台系統邊界（MVP 範圍）"]
        subgraph AccountModule ["帳號與驗證模組"]
            UC01(["FR-001 註冊帳號"])
            UC02(["FR-002 登入帳號"])
            UC03(["FR-003, BR-014 驗證與重驗 Email"])
            UC04(["FR-004 編輯個人檔案與 Email"])
            UC05(["FR-024 申請刪除帳號"])
        end

        subgraph BrowseModule ["瀏覽與搜尋模組"]
            UC06(["FR-009, FR-010 搜尋與分類價格篩選"])
            UC07(["FR-011 檢視書籍與在售刊登列表"])
        end

        subgraph ListingModule ["書籍與刊登管理模組"]
            UC08(["FR-006 ISBN 自動帶入書目"])
            UC09(["FR-007 手動填寫書籍資料"])
            UC10(["FR-005, FR-008 建立刊登與照片檢核"])
            UC11(["FR-005, BR-013 編輯、下架與重新上架"])
        end

        subgraph OrderModule ["交易與物流履約模組"]
            UC12(["FR-012, BR-002 下單購買並鎖定刊登"])
            UC13(["FR-016, BR-013 待出貨主動取消訂單"])
            UC14(["FR-013 填寫物流並標記出貨"])
            UC15(["FR-014, BR-004 標記已取貨"])
            UC16(["FR-015, BR-005 確認收貨（人工或逾期自動）"])
        end

        subgraph AfterSalesModule ["爭議與評價模組"]
            UC17(["FR-017, BR-006 提出書況不符爭議"])
            UC18(["FR-018, BR-007 回應爭議與提交證據"])
            UC19(["FR-019, BR-009 訂單完成後雙向互評"])
        end

        subgraph AdminModule ["後台管理模組"]
            UC20(["FR-020 使用者管理（停用與恢復）"])
            UC21(["FR-021 違規刊登強制下架"])
            UC22(["FR-022, NFR-009 檢視訂單與收件個資"])
            UC23(["FR-018, FR-023 爭議裁決（買家或賣家有理）"])
            UC24(["FR-028, BR-012 調整交易與評價期限"])
            UC25(["NFR-006 檢視後台操作稽核日誌"])
        end
    end

    %% Actor 關聯連線
    Guest --> UC01
    Guest --> UC02
    Guest --> UC03
    Guest --> UC06
    Guest --> UC07

    Buyer --> UC02
    Buyer --> UC03
    Buyer --> UC04
    Buyer --> UC05
    Buyer --> UC06
    Buyer --> UC07
    Buyer --> UC12
    Buyer --> UC13
    Buyer --> UC15
    Buyer --> UC16
    Buyer --> UC17
    Buyer --> UC19

    Seller --> UC02
    Seller --> UC03
    Seller --> UC04
    Seller --> UC05
    Seller --> UC08
    Seller --> UC09
    Seller --> UC10
    Seller --> UC11
    Seller --> UC13
    Seller --> UC14
    Seller --> UC18
    Seller --> UC19

    Admin --> UC20
    Admin --> UC21
    Admin --> UC22
    Admin --> UC23
    Admin --> UC24
    Admin --> UC25

    UC08 -.->|外部資料查詢| BookAPI
    UC12 -.->|include| UC07
    UC17 -.->|extend| UC16
```

### 3.2 使用案例說明表 (Use Case Specifications)

| 編號 | 使用案例名稱 | 主要參與者 | 前置條件 | 成果與後置狀態 | 需求來源 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **UC01** | 註冊帳號 | 訪客 | Email 未曾註冊 | 建立帳號實體，寄送驗證信函，處於未驗證狀態。 | FR-001 |
| **UC03** | 驗證 / 重驗 Email | 訪客 / 會員 | 具有未過期之驗證 Token | 帳號驗證狀態標記為 true，開放刊登與下單權限。修改 Email 後重置為未驗證狀態，既有進行中訂單操作不受阻斷。 | FR-003, BR-014 |
| **UC10** | 建立刊登 | 賣家 | 通過信箱驗證、已綁定書目 | 檢核上傳 1～5 張實拍照片，狀態設為「上架中」。 | FR-005, FR-008 |
| **UC12** | 下單購買並鎖定 | 買家 | 通過信箱驗證、非自身刊登 | 鎖定刊登實體（避免重複購買），產生快照資料，建立待出貨訂單並啟動出貨計時器（預設 72 小時）。 | FR-012, BR-002 |
| **UC14** | 填寫物流並標記出貨 | 賣家 | 訂單處於「待出貨」且未逾期 | 填寫自由文字物流方式與模擬單號，訂單轉為「運送中」。 | FR-013 |
| **UC15** | 標記已取貨 | 買家 | 訂單處於「運送中」 | 記錄取貨時間戳記，訂單轉為「待確認收貨」，啟動 72 小時確認計時器。 | FR-014, BR-004 |
| **UC16** | 確認收貨 | 買家 / 系統 | 訂單處於「待確認收貨」且無爭議 | 手動確認或滿 72 小時自動觸發，訂單轉為「訂單完成」，刊登轉為「已售出」，向賣家隱藏收件資料，開啟 336 小時評價期限。 | FR-015, BR-005 |
| **UC17** | 提出書況爭議 | 買家 | 處於「待確認收貨」且未逾期 | 原因限「書況不符」，上傳照片與說明，訂單轉為「爭議處理中」，暫停自動完成。 | FR-017, BR-006 |
| **UC18** | 回應爭議案件 | 賣家 | 訂單處於「爭議處理中」 | 賣家上傳說明、照片與外部對話截圖以供管理員佐證。 | FR-018, BR-007 |
| **UC23** | 裁決爭議 | 管理員 | 案件處於「爭議處理中」 | 裁決「買家有理」（轉已取消）或「賣家有理」（轉訂單完成）。刊登維持不可售。 | FR-018, FR-023, BR-008 |

---

## 4. 領域靜態結構模型 (Domain Class Model)

### 4.1 領域類別圖 (Class Diagram)
領域模型定義平台實體結構、屬性型別、領域操作方法與多重性關聯：

```mermaid
classDiagram
    direction TB

    class UserRole {
        <<enumeration>>
        BUYER
        SELLER
        ADMIN
    }

    class ListingStatus {
        <<enumeration>>
        ACTIVE
        LOCKED
        SOLD
        INACTIVE
    }

    class OrderStatus {
        <<enumeration>>
        PENDING_SHIPMENT
        SHIPPING
        PENDING_CONFIRMATION
        COMPLETED
        CANCELLED
        DISPUTED
    }

    class BookCategory {
        <<enumeration>>
        LITERATURE
        BUSINESS
        SCIENCE
        HUMANITIES
        ART_DESIGN
        COMPUTER_IT
        LANGUAGE
        EXAM
        COMIC_LIGHT_NOVEL
        LIFESTYLE
        CHILDREN
        OTHER
    }

    class DisputeRuling {
        <<enumeration>>
        PENDING
        BUYER_FAVORED
        SELLER_FAVORED
    }

    class User {
        +UUID id
        +String email
        +String passwordHash
        +String nickname
        +UserRole role
        +Boolean isEmailVerified
        +Float averageRating
        +Boolean isActive
        +DateTime registeredAt
        +register(email, password, nickname)
        +login(email, password)
        +verifyEmail(token)
        +updateEmail(newEmail)
        +updateProfile(nickname, password)
        +requestDeletion()
    }

    class Book {
        +UUID id
        +String isbn
        +String title
        +String author
        +String publisher
        +String edition
        +BookCategory category
        +String coverUrl
        +fetchExternalByIsbn(isbn)
    }

    class Listing {
        +UUID id
        +UUID bookId
        +UUID sellerId
        +Integer price
        +String conditionDescription
        +String[] photoUrls
        +ListingStatus status
        +DateTime createdAt
        +DateTime updatedAt
        +createListing(photos, price, condition)
        +editListing(price, condition)
        +delist()
        +relist()
        +lock()
        +markSold()
    }

    class Order {
        +UUID id
        +UUID listingId
        +UUID buyerId
        +UUID sellerId
        +Integer closingPrice
        +String snapshotBookTitle
        +String snapshotCondition
        +OrderStatus status
        +String recipientName
        +String recipientPhone
        +String recipientAddress
        +DateTime createdAt
        +DateTime shipDeadline
        +DateTime confirmDeadline
        +DateTime reviewDeadline
        +createOrder()
        +cancelOrder(reason)
        +markShipped(method, trackingNo)
        +markPickedUp()
        +confirmReceipt()
        +raiseDispute(reason, evidence)
        +autoCompleteOnExpiry()
        +autoCancelOnExpiry()
    }

    class Shipment {
        +UUID id
        +UUID orderId
        +String shippingMethod
        +String trackingNumber
        +DateTime shippedAt
        +DateTime pickedUpAt
        +String status
        +recordShipping(method, trackingNo)
        +recordPickup()
    }

    class Dispute {
        +UUID id
        +UUID orderId
        +UUID initiatorId
        +String reason
        +String buyerDescription
        +String[] buyerEvidencePhotos
        +String sellerResponse
        +String[] sellerEvidencePhotos
        +DisputeRuling ruling
        +UUID ruledByAdminId
        +DateTime ruledAt
        +submitDispute(photos, description)
        +submitSellerResponse(photos, description)
        +adjudicate(ruling, adminId)
    }

    class Review {
        +UUID id
        +UUID orderId
        +UUID reviewerId
        +UUID revieweeId
        +Integer rating
        +String comment
        +DateTime createdAt
        +submitReview(rating, comment)
    }

    class AuditLog {
        +UUID id
        +UUID operatorId
        +String actionType
        +String targetEntity
        +UUID targetId
        +String previousState
        +String newState
        +String note
        +DateTime createdAt
        +recordAudit()
    }

    User "1" -- "0..*" Listing : creates
    User "1" -- "0..*" Order : places
    User "1" -- "0..*" Order : receives
    User "1" -- "0..*" Review : authors
    User "1" -- "0..*" Review : receives
    User "1" -- "0..*" AuditLog : operates

    Book "1" -- "0..*" Listing : classified as

    Listing "1" -- "0..1" Order : transacts

    Order "1" -- "1" Shipment : contains
    Order "1" -- "0..1" Dispute : escalates to
    Order "1" -- "0..2" Review : allows

    Dispute "0..1" -- "1" User : ruled by Admin
```

### 4.2 核心實體約束與資料字典說明
依據 SRS 第 6 節與第 6.1 節已決議之資料規格：

1. **書目與刊登分離機制 (BR-010)**：
   - `Book` 為客觀書目實體，以 ISBN 作為唯一主鍵去重；若無 ISBN 則不自動去重，不同版次視為獨立書目。
   - `Listing` 為特定賣家之實體書刊登，每筆刊登對應單一實體書。
2. **交易快照保存機制 (FR-012, Q-019)**：
   - 下單成立時，`Order` 實體必須同時凍結成交當下之價格（`closingPrice`，新臺幣整數 NT$1～100,000）、書名快照（`snapshotBookTitle`）與書況描述快照（`snapshotCondition`）。後續刊登資料若修改，不可變更已成立訂單之歷史紀錄。
3. **個資保護與存取邊界 (NFR-008, NFR-009, BR-011)**：
   - 買家收件資訊僅於訂單處於「待出貨」、「運送中」、「待確認收貨」時提供該筆訂單賣家檢視。
   - 一旦進入「爭議處理中」、「訂單完成」或「已取消」，系統立即向賣家隱藏收件資訊。

---

## 5. 系統狀態機模型 (State Machine Models)

### 5.1 訂單狀態機模型 (Order State Machine)
訂單生命週期由 6 種嚴格定義之狀態組成，受 NFR-002 一致性約束，不允許跳出流程外之任意轉移。

```mermaid
stateDiagram-v2
    direction TB

    [*] --> 待出貨 : 買家下單成功並鎖定刊登（FR-012, BR-002）

    state 待出貨 {
        [*] --> 計時出貨中 : 啟動預設 72h 出貨計時（BR-003）
        計時出貨中 --> 填寫物流 : 賣家準備出貨
    }

    待出貨 --> 運送中 : 賣家填寫物流方式與託運單號（FR-013）
    待出貨 --> 已取消 : 買家或賣家主動取消（FR-016, BR-013）
    待出貨 --> 已取消 : 出貨期限屆滿未出貨（BR-003）

    state 運送中 {
        [*] --> 配送中 : 模擬物流配送中（FR-014）
    }

    運送中 --> 待確認收貨 : 買家手動點選已取貨（FR-014, BR-004）

    state 待確認收貨 {
        [*] --> 計時確認中 : 啟動預設 72h 確認收貨計時（BR-004）
        計時確認中 --> 檢查書況 : 買家核驗實體書籍
    }

    待確認收貨 --> 訂單完成 : 買家手動點選確認收貨（FR-015）
    待確認收貨 --> 訂單完成 : 確認期限屆滿且未提出爭議（BR-005）
    待確認收貨 --> 爭議處理中 : 買家以書況不符提出爭議（FR-017, BR-006）

    state 爭議處理中 {
        [*] --> 雙方舉證 : 賣家上傳說明與截圖證據（FR-018）
        雙方舉證 --> 等待裁決 : 管理員審閱證據
    }

    爭議處理中 --> 已取消 : 管理員裁決買家有理（FR-018, BR-008）
    爭議處理中 --> 訂單完成 : 管理員裁決賣家有理（FR-018, BR-008）

    state 訂單完成 {
        [*] --> 進入互評階段 : 啟動預設 336h 評價計時（FR-019, BR-009）
    }

    訂單完成 --> [*] : 雙方完成互評或滿 336h 評價截止
    已取消 --> [*] : 刊登維持不可售（BR-013）
```

#### 訂單狀態轉移規則表

| 當前狀態 | 觸發事件 / 條件 | 目標狀態 | 系統動作與業務規則 |
| :--- | :--- | :--- | :--- |
| **初始** | 買家點選下單並通過庫存檢核 | **待出貨** | 鎖定刊登實體（BR-002），凍結快照資料，啟動 72 小時出貨計時。 |
| **待出貨** | 賣家於出貨期限內填妥物流標記出貨 | **運送中** | 記錄出貨時間戳記（FR-013）。 |
| **待出貨** | 買家或賣家主動取消（出貨期限內） | **已取消** | 人工取消無須對方同意，刊登維持不可售（FR-016, BR-013）。 |
| **待出貨** | 滿 72 小時未出貨 | **已取消** | 系統排程自動取消，刊登維持不可售（BR-003）。 |
| **運送中** | 買家手動點選「已取貨」 | **待確認收貨** | 記錄取貨時間戳記，啟動 72 小時確認收貨計時（FR-014, BR-004）。 |
| **待確認收貨** | 買家手動點選「確認收貨」 | **訂單完成** | 刊登轉已售出，向賣家隱藏收件資訊，開啟 336 小時評價期（FR-015）。 |
| **待確認收貨** | 滿 72 小時無爭議案件 | **訂單完成** | 系統排程自動完成訂單（BR-005）。 |
| **待確認收貨** | 買家提出爭議（限「書況不符」） | **爭議處理中** | 暫停自動完成排程，向賣家隱藏收件資訊，通知賣家舉證（FR-017, BR-006）。 |
| **爭議處理中** | 管理員裁決「買家有理」 | **已取消** | 刊登維持不可售（BR-013），寫入稽核紀錄（NFR-006）。 |
| **爭議處理中** | 管理員裁決「賣家有理」 | **訂單完成** | 刊登轉已售出，開啟 336 小時評價期，寫入稽核紀錄（BR-008）。 |

---

### 5.2 刊登狀態機模型 (Listing State Machine)
刊登實體狀態分為：**上架中 (Active)、已鎖定 (Locked)、已售出 (Sold)、已下架 (Inactive)**。

```mermaid
stateDiagram-v2
    direction TB

    [*] --> 上架中 : 賣家建立刊登（檢核實拍照片 1~5 張）
    已下架 --> 上架中 : 賣家手動重新上架（FR-005）

    上架中 --> 已鎖定 : 買家下單成功（FR-012, BR-002）
    上架中 --> 已下架 : 賣家手動下架（FR-005）
    上架中 --> 已下架 : 管理員強制下架違規刊登（FR-021）

    已鎖定 --> 已售出 : 訂單狀態轉為訂單完成
    已鎖定 --> 已下架 : 訂單狀態轉為已取消（維持不可售 BR-013）

    已售出 --> [*]
```

---

## 6. 動態互動模型：循序圖 (Sequence Model)

### 6.1 核心交易履約與爭議處理循序圖
循序圖描繪買家、賣家、平台系統與管理員在完整交易生命週期中之訊息傳遞與業務驗證流程，涵蓋異常取消分支、正常履約完成分支以及爭議裁決分支：

```mermaid
sequenceDiagram
    autonumber
    actor Buyer as 買家 (Buyer)
    actor Seller as 賣家 (Seller)
    participant Sys as 交易平台 (Nuxt + Supabase)
    actor Admin as 管理員 (Admin)

    Note over Buyer, Seller: 【階段一：建立訂單與刊登鎖定】
    Buyer->>Sys: 瀏覽並下單 (送出收件人姓名、電話、地址)
    Sys->>Sys: 驗證庫存與鎖定 (防止重複售出 BR-002)
    Sys->>Sys: 保存成交價、書目與書況快照 (Q-019)
    Sys->>Sys: 建立待出貨訂單 (啟動 72h 計時器 BR-003)
    Sys->>Sys: 刊登狀態轉為「已鎖定」
    Sys-->>Buyer: 下單成功通知
    Sys-->>Seller: 待出貨通知 (提供買家收件資訊 BR-011)

    alt 異常分支：待出貨期間取消 (FR-016)
        Buyer->>Sys: 主動取消訂單 (或賣家取消 / 滿 72h 逾期取消)
        Sys->>Sys: 訂單轉為「已取消」，刊登轉為「已下架 (不可售 BR-013)」
        Sys-->>Buyer: 訂單已取消通知
        Sys-->>Seller: 訂單已取消通知
    else 正常履約：賣家出貨 (FR-013)
        Note over Buyer, Seller: 【階段二：物流運送與買家取貨】
        Seller->>Sys: 填寫自由文字物流方式與模擬單號，標記已出貨
        Sys->>Sys: 訂單轉為「運送中」，記錄出貨時間
        Sys-->>Buyer: 已出貨通知 (顯示物流方式與模擬單號)
        
        Buyer->>Sys: 實體取貨後點選「已取貨」 (FR-014)
        Sys->>Sys: 訂單轉為「待確認收貨」，記錄取貨時間
        Sys->>Sys: 啟動 72h 確認收貨計時器 (BR-004)
        Sys-->>Seller: 買家已取貨通知

        alt 正常收貨：確認完成 (FR-015)
            Buyer->>Sys: 點選「確認收貨」 (或滿 72h 逾期無爭議自動完成 BR-005)
            Sys->>Sys: 訂單轉為「訂單完成」，刊登轉為「已售出」
            Sys->>Sys: 向賣家隱藏買家收件資訊 (BR-011, Q-017)
            Sys->>Sys: 啟動 336h 評價期限 (BR-009)
            Sys-->>Buyer: 訂單完成通知 (邀請互評)
            Sys-->>Seller: 訂單完成通知 (邀請互評)
            Buyer->>Sys: 提交評價 (1~5 星，文字選填上限 500 字)
            Seller->>Sys: 提交評價 (1~5 星，文字選填上限 500 字)
            Sys->>Sys: 更新雙方公開平均評價 (四捨五入至小數一位)

        else 異常爭議：提出書況不符爭議 (FR-017, FR-018)
            Note over Buyer, Admin: 【階段三：爭議處理與管理員裁決】
            Buyer->>Sys: 提出爭議 (限書況不符，附照片說明與文字)
            Sys->>Sys: 訂單轉為「爭議處理中」，暫停自動完成計時
            Sys->>Sys: 向賣家隱藏收件資訊 (BR-011)
            Sys-->>Seller: 爭議案件通知
            Seller->>Sys: 提交回應說明、照片與外部對話截圖 (BR-007)
            
            Admin->>Sys: 後台審閱爭議詳情與雙方證據 (留存查閱紀錄 NFR-006)
            alt 裁決分支 1：買家有理
                Admin->>Sys: 裁決「買家有理」 (BR-008)
                Sys->>Sys: 訂單轉為「已取消」，刊登維持不可售 (BR-013)
                Sys->>Sys: 記錄管理員操作稽核日誌 (NFR-006)
                Sys-->>Buyer: 裁決通知 (買家有理，訂單取消)
                Sys-->>Seller: 裁決通知
            else 裁決分支 2：賣家有理
                Admin->>Sys: 裁決「賣家有理」 (BR-008)
                Sys->>Sys: 訂單轉為「訂單完成」，刊登轉為「已售出」
                Sys->>Sys: 啟動 336h 評價期限 (BR-009)
                Sys->>Sys: 記錄管理員操作稽核日誌 (NFR-006)
                Sys-->>Buyer: 裁決通知 (賣家有理，訂單完成)
                Sys-->>Seller: 裁決通知 (邀請評價)
            end
        end
    end
```

---

## 7. 需求可追溯性矩陣 (Traceability Matrix)

本矩陣將 SRS 需求規格、業務規則映射至對應之 UML 模型元素，確保架構設計之完整性與可驗證性：

| 需求 / 規則編號 | 規格主題概要 | 對應 UML 模型元素 | 實作關鍵檢核點 |
| :--- | :--- | :--- | :--- |
| **FR-001～FR-003** | 帳號註冊、登入與 Email 驗證 | 使用案例 UC01～UC03、類別 `User` | Email 唯一性、密碼 bcrypt 雜湊、未驗證權限隔離。 |
| **FR-005, FR-008** | 刊登管理與實拍照片檢核 | 使用案例 UC10～UC11、類別 `Listing`、刊登狀態機 | 照片檢核 1～5 張、JPEG/PNG/WebP、賣家聲明實拍。 |
| **FR-006, FR-007** | ISBN 自動帶入與手動建檔 | 使用案例 UC08～UC09、類別 `Book` | 整合 Google Books API，以 ISBN 為鍵進行書目去重。 |
| **FR-012, BR-002** | 下單並立即鎖定刊登 | 使用案例 UC12、類別 `Order`、訂單與刊登狀態機、循序圖階段一 | 防競爭重複下單、成交價與書況快照凍結。 |
| **FR-013, FR-014** | 物流出貨與買家模擬取貨 | 使用案例 UC14～UC15、類別 `Shipment`、循序圖階段二 | 記錄出貨與取貨時間戳記、觸發 72 小時收貨確認計時。 |
| **FR-015, BR-005** | 確認收貨與自動結案 | 使用案例 UC16、訂單狀態機、循序圖正常收貨分支 | 人工確認或滿 72 小時自動結案、隱藏賣家收件資料。 |
| **FR-016, BR-013** | 待出貨主動與逾期取消 | 使用案例 UC13、訂單狀態機、刊登狀態機、循序圖異常分支 | 72 小時逾期自動取消、取消後刊登維持不可售。 |
| **FR-017, FR-018** | 提出爭議與管理員裁決 | 使用案例 UC17～UC18, UC23、類別 `Dispute`、循序圖爭議分支 | 限書況不符、提交照片與對話截圖、裁決買家/賣家有理。 |
| **FR-019, BR-009** | 雙向評分與文字評價 | 使用案例 UC19、類別 `Review`、循序圖階段三結案 | 1～5 星整數、上限 500 字、僅訂單完成可評、336 小時時效。 |
| **FR-020～FR-023** | 後台資料管理與權限 | 使用案例 UC20～UC23、類別 `AuditLog` | 管理員 RBAC 權限、操作稽核永久留痕。 |
| **BR-014** | 修改 Email 重新驗證 | 使用案例 UC03、類別 `User.updateEmail` | 驗證狀態重置為未驗證，禁止新刊登與下單，保留履約中訂單操作。 |

---

## 8. 版本修訂歷程

| 版本 | 修訂日期 | 變更說明 | 審查狀態 |
| :--- | :--- | :--- | :--- |
| **v1.0** | 2026-11-10 | 依據 `SRS.md` v0.4 規格基線正式建立軟體分析與設計規格書，發布完整 5 大核心 UML 模型、實體資料字典與追溯矩陣。修正 Mermaid 標籤括號語法解析問題。 | 待審查（Pending Review） |
