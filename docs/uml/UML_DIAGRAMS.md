# 二手書交易平台系統分析與設計規格書（UML 模型）

| 項目 | 內容 |
| :--- | :--- |
| 文件版本 | v1.2 |
| 制定日期 | 2026-11-10 |
| 依據文件 | 軟體需求規格書（SRS）v1.0（需求基線定稿版本） |
| 專案方法 | Waterfall + AI |
| 目前階段 | 系統分析與架構設計（System Analysis & Design） |
| 審查狀態 | 需求基線嚴格對齊審查完成 |

---

## 1. 文件目的與系統範疇

### 1.1 文件目的
本文件依據《二手書交易平台軟體需求規格書（SRS）v1.0》之需求基線，建立正式的統一塑模語言（Unified Modeling Language, UML）系統分析與設計規格。內容涵蓋使用案例模型、領域靜態結構模型、系統狀態機模型及動態循序互動模型，作為後續系統實作（Nuxt.js + Supabase）與驗收測試之架構依據。

### 1.2 系統邊界與核心角色 (Actors)
本系統為專注於台灣個人讀者之 C2C 二手書交易平台。平台不經手真實款項，記錄履約進度並透過模擬物流完成交易閉環。本次 MVP 明確排除站內訊息、收藏、上架通知、**Email 修改功能**以及**獨立檢舉受理功能**。

| 角色 / 介面 | 類型 | 責任與權限範圍（依 SRS 第 2.3 節定稿） | 對應需求 |
| :--- | :--- | :--- | :--- |
| **訪客 / 未驗證會員** | 人類角色 | 瀏覽書籍與刊登清單、依條件搜尋篩選、註冊帳號、登入系統、執行註冊 Email 驗證。未完成驗證前禁止刊登實體書與下單購買。本次不提供註冊後變更 Email 之功能。 | FR-001～FR-003, Q-010 |
| **買家身分 (Buyer Role)** | 交易身分 | 會員於特定訂單中作為購買方之身分。可檢視刊登詳情、對非自身刊登下單建立訂單、待出貨期間主動取消、確認取貨、於期限內確認收貨、提出書況不符爭議並舉證、訂單完成後評分評價。各交易操作僅限自己作為買家的訂單。 | FR-012, FR-014～FR-017, FR-019 |
| **賣家身分 (Seller Role)** | 交易身分 | 會員於特定刊登與訂單中作為販售方之身分。可建立實體書刊登（實拍檢核）、維護自己的刊登狀態、出貨並填寫模擬物流資訊、待出貨期間主動取消、針對爭議案件回應與舉證、訂單完成後評分評價。各交易操作僅限自己作為賣家的訂單與刊登。同一帳號可兼任買賣家（FR-004）。 | FR-005～FR-008, FR-013, FR-018, FR-019 |
| **平台管理員 (Admin)** | 系統角色 | 使用者帳號管理（停用/恢復）、違規刊登強制下架、檢視訂單詳情（不可直接跳過流程變更狀態）、審閱爭議雙方證據並進行裁決、設定交易時限參數、查閱系統稽核紀錄。 | FR-020～FR-023, FR-028, NFR-005, NFR-006 |
| **外部書籍 API** | 外部系統 | 提供依國際標準書號（ISBN）自動查詢書目資料（Google Books API 或 Open Library API）。 | FR-006 |

---

## 2. 系統架構與技術約束

依據 SRS 之技術約束（NFR-010）與品質需求，本系統之實作規格約束如下：
1. **開發語言與框架**：採用 TypeScript 搭配 Nuxt.js 全端 Web 框架。
2. **後端服務架構**：採用 Supabase 平台，包含 PostgreSQL 關聯式資料庫、身份驗證機制（Supabase Auth）及物件儲存儲存庫（Supabase Storage）。具體版本與服務配置交接至設計階段細化。
3. **資訊安全規範**：
   - 密碼儲存採用 bcrypt 雜湊演算法，Cost Factor $\ge$ 10（NFR-004）。
   - 管理後台嚴格實施基於角色的存取控制（RBAC），一般使用者無權限進入（NFR-005）。
   - 後台敏感管理操作（停用帳號、強制下架、爭議裁決、時限調整）與收件個資查閱皆強制留存操作稽核紀錄，且**永久保留**（NFR-006）。
   - 圖片上傳僅支援 JPEG、PNG、WebP 格式，單張檔案上限 5 MB（NFR-007）。
   - 個資最小化（NFR-008）：下單依選擇「宅配」或「超商取貨」填寫對應收件欄位，不收身分證字號與生日；不串接真實物流或門市查詢。

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
            UC03(["FR-003 註冊 Email 驗證"])
            UC04(["FR-004 編輯個人檔案（暱稱與密碼）"])
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
            UC11(["FR-005, BR-013 編輯、下架與手動重新上架"])
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
| **UC01** | 註冊帳號 | 訪客 | Email 未曾註冊 | 建立帳號實體，發送驗證連結，初始狀態為未驗證。 | FR-001 |
| **UC03** | 註冊 Email 驗證 | 訪客 / 會員 | 具有有效之註冊驗證連結 | 帳號驗證狀態標記為 true，開放刊登與下單權限。Email 於註冊後不可變更。 | FR-003, Q-010 |
| **UC04** | 編輯個人檔案 | 會員本人 | 已成功登入 | 僅允許修改暱稱與密碼；Email 欄位為唯讀不可修改。同一帳號可兼任買家與賣家。 | FR-004, Q-010 |
| **UC05** | 申請刪除帳號 | 會員本人 | 帳號無未結束訂單且無未結爭議 | 符合條件後刪除登入憑證與個人檔案；既有評價保留並改署名為「已刪除使用者」；不連帶刪除既有交易（FR-024, Q-017）。 | FR-024, Q-017 |
| **UC10** | 建立刊登 | 賣家 | 通過信箱驗證、已指定書目 | 檢核 1～5 張實拍照片並宣告實拍，刊登狀態設為「上架中」。 | FR-005, FR-008 |
| **UC11** | 管理刊登 | 賣家 | 刊登之擁有者本人 | 可編輯上架中刊登、手動下架。因訂單取消轉為「已下架」之刊登，賣家確認實體在庫、更新書況並通過原有上架檢核後可手動重新上架。 | FR-005, BR-013 |
| **UC12** | 下單購買並鎖定 | 買家 | 買家已驗證、未停用、非自購、刊登上架中 | 依選擇之宅配或超商取貨填妥 NFR-008 規定之收件資料，鎖定刊登實體（防止重複售出），凍結成交價、書目快照與書況快照，建立待出貨訂單並啟動出貨計時器（預設 72 小時，依適用設定 BR-012）。 | FR-012, BR-002, NFR-008 |
| **UC14** | 填寫物流並出貨 | 賣家 | 訂單處於「待出貨」且出貨期限未屆滿 | 必填自由文字物流方式（1～50 字）與模擬單號（1～100 字元），訂單轉為「運送中」，物流狀態更新為 `SHIPPED`。滿期限即拒絕出貨。 | FR-013, BR-003, Q-006 |
| **UC15** | 標記已取貨 | 買家 | 訂單處於「運送中」 | 記錄取貨時間戳記，物流狀態更新為 `DELIVERED`，訂單轉為「待確認收貨」，啟動確認收貨計時器（預設 72 小時，依適用設定 BR-012）。 | FR-014, BR-004 |
| **UC16** | 確認收貨 | 買家 / 系統 | 訂單處於「待確認收貨」且無爭議 | 確認期限內買家手動確認收貨，或滿期限且未提爭議時由系統於 60 秒內自動完成；訂單轉為「訂單完成」，刊登轉為「已售出」，向賣家隱藏收件資料，開啟評價期限（預設 336 小時，依適用設定 BR-012）。 | FR-015, BR-005 |
| **UC17** | 提出書況爭議 | 買家 | 處於「待確認收貨」且確認期限未屆滿 | 原因限「書況不符」，上傳照片與說明，訂單轉為「爭議處理中」，暫停自動完成排程。滿期限即拒絕提出爭議。 | FR-017, BR-004, BR-006 |
| **UC18** | 回應爭議案件 | 賣家 | 訂單處於「爭議處理中」 | 賣家上傳文字說明、照片與外部對話截圖以供管理員佐證。 | FR-018, BR-007 |
| **UC23** | 裁決爭議 | 管理員 | 案件處於「爭議處理中」 | 審閱雙方證據後裁決「買家有理」（轉已取消，刊登轉已下架維持不可售）或「賣家有理」（轉訂單完成，刊登轉已售出）。 | FR-018, FR-023, BR-008, BR-013 |
| **UC24** | 調整交易期限 | 管理員 | 具備管理員權限 | 以整數小時調整出貨、收貨確認及評價期限（範圍 1～720 小時），僅對調整後新開始計時的階段生效，並留存後台操作稽核紀錄。 | FR-028, BR-012, NFR-006 |

---

## 4. 領域靜態結構模型 (Domain Class Model)

### 4.1 領域類別圖 (Class Diagram)
領域模型設計採用職責與身分分離模式：系統角色（`SystemRole`）區分一般會員與管理員；買家與賣家為會員在交易中扮演之交易身分，落實同一帳號可兼任買賣家（FR-004）：

```mermaid
classDiagram
    direction TB

    class SystemRole {
        <<enumeration>>
        MEMBER
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

    class DeliveryType {
        <<enumeration>>
        HOME_DELIVERY
        CONVENIENCE_STORE
    }

    class ShipmentStatus {
        <<enumeration>>
        SHIPPED
        DELIVERED
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
        +SystemRole role
        +Boolean isEmailVerified
        +Float averageRating
        +Boolean isActive
        +DateTime registeredAt
        +register(email, password, nickname)
        +login(email, password)
        +verifyEmail(token)
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
        +String snapshotBookAuthor
        +String snapshotBookEdition
        +String snapshotCondition
        +OrderStatus status
        +DeliveryType deliveryType
        +String recipientName
        +String recipientPhone
        +String recipientAddress
        +String storeName
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
        +ShipmentStatus status
        +DateTime shippedAt
        +DateTime pickedUpAt
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

    User "1" -- "0..*" Listing : creates as seller
    User "1" -- "0..*" Order : places as buyer
    User "1" -- "0..*" Order : receives as seller
    User "1" -- "0..*" Review : authors
    User "1" -- "0..*" Review : receives
    User "1" -- "0..*" AuditLog : operates

    Book "1" -- "0..*" Listing : classified as

    Listing "1" -- "0..*" Order : recorded in

    Order "1" -- "1" Shipment : contains
    Order "1" -- "0..1" Dispute : escalates to
    Order "1" -- "0..2" Review : allows

    Dispute "0..*" -- "0..1" User : adjudicated by Admin
```

### 4.2 核心實體約束與資料字典說明
依據 SRS 第 6 節、第 6.1 節及第 7 節已確認之資料規格：

1. **使用者角色與交易身分 (FR-004, NFR-005)**：
   - 系統層級透過 `SystemRole`（MEMBER / ADMIN）區分後台管理權限。買家（Buyer）與賣家（Seller）為會員參與個別交易之交易身分（由 `Order.buyerId`、`Order.sellerId` 與 `Listing.sellerId` 關聯體現），同一會員帳號可自由兼任買家與賣家，但不可購買自己所刊登之書籍。
2. **書目與刊登分離機制 (BR-010)**：
   - `Book` 為客觀書目實體，以 ISBN 作為唯一主鍵比對去重；若無 ISBN 則不自動去重，不同版次視為獨立書目。
   - `Listing` 為特定賣家之實體書刊登，每筆刊登對應單一實體書。
3. **刊登與訂單基數關係 (FR-005, BR-013)**：
   - `Listing` 對 `Order` 之關聯基數為 `1` 對 `0..*`。因為訂單取消後，原訂單仍保留於系統歷史中供核對；賣家可手動重新上架該刊登並產生新訂單。系統業務規則強制約束：**同一筆刊登在任何時間點最多僅允許一筆有效進行中（未取消且未結案）的訂單**，防止重複售出（BR-002）。
4. **完整交易快照保存機制 (FR-012, Q-019)**：
   - 下單成立時，`Order` 實體除凍結成交價格（`closingPrice`，NT$1～100,000 整數）與書況描述快照（`snapshotCondition`，上限 2,000 字）外，**必須完整凍結下單當下之書目資料（包含書名 `snapshotBookTitle`、作者 `snapshotBookAuthor`、版次 `snapshotBookEdition` 等）**，避免日後書目資料庫被維護或修改時，歷史訂單產生資訊不一致。
5. **收件資料與物流規格 (FR-013, FR-014, NFR-008, Q-006)**：
   - 下單依選擇之 `deliveryType` 分流填寫必要收件欄位（NFR-008）：宅配必填收件人姓名、電話與地址；超商取貨必填收件人姓名、電話、超商名稱與門市名稱。不收集身分證字號與生日，不串接外部真實物流。
   - 賣家出貨必填自由文字物流方式（去除前後空白後為 1～50 字）及模擬託運單號（1～100 字元英數與連字號）。
   - 物流模擬狀態（`ShipmentStatus`）僅有 `SHIPPED`（運送中）與 `DELIVERED`（已取貨），分別對應賣家出貨與買家取貨。
6. **裁決管理員關聯基數 (FR-018, FR-023)**：
   - 管理員（`User`）與爭議（`Dispute`）之裁決關聯基數為 `0..1` 對 `0..*`。爭議在未被裁決前無裁決人（`0..1`），而具備管理員權限之使用者可審閱並裁決零到多起爭議案件（`0..*`）。
7. **個資保護與資料保留規則 (FR-024, NFR-006, NFR-008, NFR-009, BR-011, Q-017)**：
   - 買家收件資訊僅在「待出貨」、「運送中」、「待確認收貨」時提供給賣家；一旦進入「爭議處理中」、「訂單完成」或「已取消」，立即向賣家隱藏。
   - 使用者申請刪除帳號時，須於無進行中訂單且無未結爭議時方可執行；刪帳時刪除登入憑證與個人檔案，既有評價保留並改署名為「已刪除使用者」，不連帶刪除既有交易（保留訂單紀錄）。
   - 本次 MVP 排除定期自動刪除資料功能；系統中明確規範**永久保留**的資料為管理員操作稽核日誌（NFR-006）。

---

## 5. 系統狀態機模型 (State Machine Models)

### 5.1 訂單狀態機模型 (Order State Machine)
訂單生命週期由 6 種嚴格定義之狀態組成，受 NFR-002 一致性約束。所有時限均採用經過時間計算，時區統一為 Asia/Taipei；各階段時限以階段開始時適用的設定為準（FR-028, BR-012），**剛好滿期限即到期並拒絕逾期操作**；系統自動轉移於到期後 60 秒內完成，但延遲期間仍嚴格拒絕逾期操作。

```mermaid
stateDiagram-v2
    direction TB

    [*] --> 待出貨 : 買家下單成功並鎖定刊登（FR-012, BR-002）

    state 待出貨 {
        [*] --> 計時出貨中 : 啟動出貨計時（預設 72h，依適用設定 BR-012）
        計時出貨中 --> 填寫物流 : 賣家準備出貨
    }

    待出貨 --> 運送中 : 出貨期限前賣家填寫物流並標記出貨（FR-013）
    待出貨 --> 已取消 : 出貨期限前買家或賣家主動取消（FR-016, BR-013）
    待出貨 --> 已取消 : 出貨期限屆滿自動取消（滿期限即拒絕出貨，60秒內自動取消 BR-003）

    state 運送中 {
        [*] --> 配送中 : 模擬物流配送中（Shipment 狀態為 SHIPPED FR-014）
    }

    運送中 --> 待確認收貨 : 買家手動點選已取貨（Shipment 狀態為 DELIVERED FR-014, BR-004）

    state 待確認收貨 {
        [*] --> 計時確認中 : 啟動確認收貨計時（預設 72h，依適用設定 BR-012）
        計時確認中 --> 檢查書況 : 買家核驗實體書籍
    }

    待確認收貨 --> 訂單完成 : 確認期限前買家手動點選確認收貨（FR-015）
    待確認收貨 --> 訂單完成 : 確認期限屆滿自動完成（滿期限即拒絕確認與爭議，60秒內自動完成 BR-005）
    待確認收貨 --> 爭議處理中 : 確認期限前買家以書況不符提出爭議（FR-017, BR-006）

    state 爭議處理中 {
        [*] --> 雙方舉證 : 賣家上傳說明與截圖證據（FR-018）
        雙方舉證 --> 等待裁決 : 管理員審閱證據
    }

    爭議處理中 --> 已取消 : 管理員裁決買家有理（FR-018, BR-008）
    爭議處理中 --> 訂單完成 : 管理員裁決賣家有理（FR-018, BR-008）

    state 訂單完成 {
        [*] --> 進入互評階段 : 刊登轉已售出，開啟評價期限（預設 336h，依適用設定，滿期限即拒絕評價 BR-009）
    }

    訂單完成 --> [*] : 雙方完成互評或滿評價期限截止
    已取消 --> [*] : 刊登轉已下架並維持不可售（BR-013）
```

#### 訂單狀態轉移規則表

| 當前狀態 | 觸發事件 / 條件 | 目標狀態 | 系統動作與業務規則 |
| :--- | :--- | :--- | :--- |
| **初始** | 買家點選下單並通過身分與庫存檢核 | **待出貨** | 鎖定刊登實體（BR-002），凍結成交價、書況及完整書目快照，依當時適用設定啟動出貨計時（預設 72 小時）。 |
| **待出貨** | 出貨期限截止前，賣家填妥物流標記出貨 | **運送中** | 記錄出貨時間戳記，物流狀態更新為 `SHIPPED`（FR-013）。滿期限即拒絕出貨操作。 |
| **待出貨** | 出貨期限截止前，買家或賣家主動取消 | **已取消** | 人工取消無須對方同意，刊登轉為「已下架」維持不可售，向賣家隱藏收件資料（FR-016, BR-011, BR-013）。 |
| **待出貨** | 滿出貨期限仍未出貨 | **已取消** | 滿期限即不再接受出貨，系統於到期後 60 秒內自動取消，刊登轉為「已下架」維持不可售，向賣家隱藏收件資料（BR-003, BR-011, BR-013）。 |
| **運送中** | 買家手動點選「已取貨」 | **待確認收貨** | 記錄取貨時間戳記，物流狀態更新為 `DELIVERED`，依當時適用設定啟動確認收貨計時（預設 72 小時）（FR-014, BR-004）。 |
| **待確認收貨** | 確認期限截止前，買家點選「確認收貨」 | **訂單完成** | 刊登轉為「已售出」，向賣家隱藏收件資訊，依當時適用設定開啟評價期限（預設 336 小時）（FR-015, BR-009, BR-011）。 |
| **待確認收貨** | 滿確認期限且未提出爭議 | **訂單完成** | 滿期限即不再接受收貨或爭議操作，系統於到期後 60 秒內自動確認收貨，刊登轉已售出，向賣家隱藏收件資訊，開啟評價期限（BR-005, BR-009, BR-011）。 |
| **待確認收貨** | 確認期限截止前，買家提出書況爭議 | **爭議處理中** | 暫停自動完成排程，向賣家隱藏收件資訊，通知賣家舉證（FR-017, BR-006, BR-011）。滿期限即拒絕提出爭議。 |
| **爭議處理中** | 管理員裁決「買家有理」 | **已取消** | 刊登轉為「已下架」維持不可售（BR-013），永久留存後台操作稽核紀錄（NFR-006）。 |
| **爭議處理中** | 管理員裁決「賣家有理」 | **訂單完成** | 刊登轉為「已售出」，開啟評價期限，永久留存後台操作稽核紀錄（BR-008, BR-009, NFR-006）。 |

---

### 5.2 刊登狀態機模型 (Listing State Machine)
刊登實體狀態分為：**上架中 (Active)、已鎖定 (Locked)、已售出 (Sold)、已下架 (Inactive)**。

```mermaid
stateDiagram-v2
    direction TB

    [*] --> 上架中 : 賣家建立刊登（檢核實拍照片 1~5 張）
    已下架 --> 上架中 : 賣家確認在庫並更新書況後手動重新上架（FR-005, BR-013）

    上架中 --> 已鎖定 : 買家下單成功（FR-012, BR-002）
    上架中 --> 已下架 : 賣家主動下架（FR-005）
    上架中 --> 已下架 : 管理員強制下架違規刊登（FR-021）

    已鎖定 --> 已售出 : 訂單完成（買家確認 / 逾期自動完成 / 裁決賣家有理）
    已鎖定 --> 已下架 : 訂單取消（人工取消 / 逾期未出貨 / 裁決買家有理）

    已售出 --> [*]
```

#### 刊登狀態約束說明
1. **防止重複售出 (BR-002)**：下單成功即刻由「上架中」轉入「已鎖定」，禁止其他使用者下單或賣家編輯。
2. **取消不自動恢復可售 (BR-013)**：不論人工取消、逾期未出貨自動取消或爭議裁決買家有理，刊登皆明確轉為「已下架」，絕對不自動轉回上架中。
3. **重新上架條件 (FR-005, BR-013, Q-011)**：賣家確認仍持有該實體書、更新書況說明並通過實拍照片檢核後，方可手動轉回「上架中」；重新上架後產生的交易為全新訂單，不影響已取消舊訂單之歷史紀錄。

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
    Buyer->>Sys: 瀏覽並下單 (選擇宅配或超商取貨，填寫收件人姓名、電話、地址或門市 NFR-008)
    Sys->>Sys: 檢核買家身分 (已登入、已驗證、非停用、非自購 FR-012)
    Sys->>Sys: 驗證庫存並執行並行鎖定 (防止重複售出 BR-002)
    Sys->>Sys: 凍結成交價、書目快照 (書名、作者、版次等) 與書況快照 (FR-012, Q-019)
    Sys->>Sys: 建立待出貨訂單 (啟動出貨計時，預設 72h，採階段開始時設定 BR-012)
    Sys->>Sys: 刊登狀態轉為「已鎖定」
    Sys-->>Buyer: 下單成功通知
    Sys-->>Seller: 待出貨通知 (提供買家收件資訊 BR-011)

    alt 異常分支：待出貨期間取消 (FR-016)
        Buyer->>Sys: 主動取消訂單 (或賣家取消 / 出貨期限屆滿系統自動取消，滿期限即拒絕出貨 BR-003)
        Sys->>Sys: 訂單轉為「已取消」，刊登轉為「已下架 (維持不可售 BR-013)」
        Sys->>Sys: 向賣家隱藏買家收件資訊 (BR-011, Q-017)
        Sys-->>Buyer: 訂單已取消通知
        Sys-->>Seller: 訂單已取消通知
    else 正常履約：賣家出貨 (FR-013)
        Note over Buyer, Seller: 【階段二：物流運送與買家取貨】
        Seller->>Sys: 出貨期限前填寫物流方式與模擬單號，標記已出貨 (滿期限即拒絕出貨 BR-003)
        Sys->>Sys: 訂單轉為「運送中」，物流狀態更新為「SHIPPED」，記錄出貨時間
        Sys-->>Buyer: 已出貨通知 (顯示物流方式與模擬單號)
        
        Buyer->>Sys: 實體取貨後點選「已取貨」 (FR-014)
        Sys->>Sys: 訂單轉為「待確認收貨」，物流狀態更新為「DELIVERED」，記錄取貨時間
        Sys->>Sys: 啟動確認收貨計時 (預設 72h，採階段開始時設定 BR-012)
        Sys-->>Seller: 買家已取貨通知

        alt 正常收貨：確認完成 (FR-015)
            Buyer->>Sys: 確認期限前點選「確認收貨」 (或確認期限屆滿無爭議系統自動完成，滿期限即拒絕收貨與爭議 BR-005)
            Sys->>Sys: 訂單轉為「訂單完成」，刊登轉為「已售出」
            Sys->>Sys: 向賣家隱藏買家收件資訊 (BR-011, Q-017)
            Sys->>Sys: 開啟評價期限 (預設 336h，採階段開始時設定，滿期限即拒絕評價 BR-009)
            Sys-->>Buyer: 訂單完成通知 (邀請互評)
            Sys-->>Seller: 訂單完成通知 (邀請互評)
            Buyer->>Sys: 評價期限前提交評價 (1~5 星整數，文字選填上限 500 字，滿期限即拒絕)
            Seller->>Sys: 評價期限前提交評價 (1~5 星整數，文字選填上限 500 字，滿期限即拒絕)
            Sys->>Sys: 更新雙方公開平均評價 (四捨五入至小數一位)

        else 異常爭議：提出書況不符爭議 (FR-017, FR-018)
            Note over Buyer, Admin: 【階段三：爭議處理與管理員裁決】
            Buyer->>Sys: 確認期限前提出爭議 (限書況不符，附照片說明與文字，滿期限即拒絕 BR-004, BR-006)
            Sys->>Sys: 訂單轉為「爭議處理中」，暫停自動完成計時
            Sys->>Sys: 向賣家隱藏收件資訊 (BR-011)
            Sys-->>Seller: 爭議案件通知
            Seller->>Sys: 提交回應說明、照片與外部對話截圖 (BR-007)
            
            Admin->>Sys: 後台審閱爭議詳情與雙方證據 (留存查閱紀錄 NFR-006)
            alt 裁決分支 1：買家有理
                Admin->>Sys: 裁決「買家有理」 (BR-008)
                Sys->>Sys: 訂單轉為「已取消」，刊登轉為「已下架 (維持不可售 BR-013)」
                Sys->>Sys: 記錄管理員操作稽核日誌 (永久保存 NFR-006)
                Sys-->>Buyer: 裁決通知 (買家有理，訂單取消)
                Sys-->>Seller: 裁決通知
            else 裁決分支 2：賣家有理
                Admin->>Sys: 裁決「賣家有理」 (BR-008)
                Sys->>Sys: 訂單轉為「訂單完成」，刊登轉為「已售出」
                Sys->>Sys: 開啟評價期限 (預設 336h，採階段開始時設定 BR-009)
                Sys->>Sys: 記錄管理員操作稽核日誌 (永久保存 NFR-006)
                Sys-->>Buyer: 裁決通知 (賣家有理，訂單完成)
                Sys-->>Seller: 裁決通知 (邀請評價)
            end
        end
    end
```

---

## 7. 需求可追溯性矩陣 (Traceability Matrix)

本矩陣將 SRS v1.0 定稿需求規格、業務規則映射至對應之 UML 模型元素，確保架構設計之完整性與可驗證性：

| 需求 / 規則編號 | 規格主題概要 | 對應 UML 模型元素 | 實作關鍵檢核點 |
| :--- | :--- | :--- | :--- |
| **FR-001～FR-003** | 帳號註冊、登入與註冊 Email 驗證 | 使用案例 UC01～UC03、類別 `User` | Email 唯一性、密碼 bcrypt 雜湊、未驗證權限隔離、註冊後 Email 唯讀不可修改。 |
| **FR-004** | 個人檔案維護與買賣家兼任 | 使用案例 UC04、類別 `User.updateProfile` | 僅允許修改暱稱與密碼，Email 不可修改。同一帳號可兼任買家與賣家交易身分。 |
| **FR-005, FR-008** | 刊登管理與實拍照片檢核 | 使用案例 UC10～UC11、類別 `Listing`、刊登狀態機 | 照片檢核 1～5 張、JPEG/PNG/WebP、賣家宣告實拍、取消後手動重新上架前置檢核。 |
| **FR-006, FR-007** | ISBN 自動帶入與手動建檔 | 使用案例 UC08～UC09、類別 `Book` | 整合 Google Books API，以 ISBN 為鍵進行書目去重。 |
| **FR-012, BR-002** | 下單並立即鎖定刊登 | 使用案例 UC12、類別 `Order`、訂單與刊登狀態機、循序圖階段一 | 買家身分驗證檢核、非自購檢核、防競爭重複下單、成交價與書況及完整書目快照凍結。 |
| **FR-013, FR-014** | 物流出貨與買家模擬取貨 | 使用案例 UC14～UC15、類別 `Shipment`、循序圖階段二 | 自由文字物流（1～50 字）、模擬單號（1～100 字元）、記錄出貨/取貨時間、物流狀態轉移 `SHIPPED` / `DELIVERED`。滿出貨期限即拒絕出貨操作。 |
| **FR-015, BR-005** | 確認收貨與自動結案 | 使用案例 UC16、訂單狀態機、循序圖正常收貨分支 | 人工確認或滿期限 60 秒內自動結案、刊登轉已售出、向賣家隱藏收件資料。滿期限即拒絕確認操作。 |
| **FR-016, BR-013** | 待出貨主動與逾期取消 | 使用案例 UC13、訂單狀態機、刊登狀態機、循序圖異常分支 | 滿出貨期限 60 秒內自動取消、所有取消均將刊登轉為「已下架」維持不可售、向賣家隱藏收件資料。 |
| **FR-017, FR-018** | 提出爭議與管理員裁決 | 使用案例 UC17～UC18, UC23、類別 `Dispute`、循序圖爭議分支 | 限書況不符、提交照片與對話截圖、裁決結果僅買家有理（轉已取消，刊登轉已下架）與賣家有理（轉訂單完成，刊登轉已售出）。滿確認期限即拒絕提出爭議。 |
| **FR-019, BR-009** | 雙向評分與文字評價 | 使用案例 UC19、類別 `Review`、循序圖階段三結案 | 1～5 星整數、上限 500 字、僅訂單完成可評、依階段開始時設定計時、滿期限即拒絕評價。 |
| **FR-020～FR-023** | 後台資料管理與權限 | 使用案例 UC20～UC23、類別 `AuditLog` | 管理員 RBAC 權限、不可越權跳轉訂單狀態、操作稽核紀錄永久保留（NFR-006）。 |
| **FR-024** | 刪除帳號與資料保留 | 使用案例 UC05、類別 `User.requestDeletion` | 無未結訂單與爭議方可刪帳；符合條件後刪除登入憑證與個人檔案，既有評價改署名為「已刪除使用者」，不連帶刪除既有交易。 |
| **FR-028, BR-012** | 調整交易與評價期限 | 使用案例 UC24、類別 `AuditLog` | 以整數小時輸入（1～720 小時）、時區 Asia/Taipei、僅對未開始階段生效、記錄修改前後值。 |

---

## 8. 版本修訂歷程

| 版本 | 修訂日期 | 變更說明 | 審查狀態 |
| :--- | :--- | :--- | :--- |
| **v1.0** | 2026-11-10 | 依據 `SRS.md` v0.4 規格初稿建立系統分析與設計規格書，發布 5 大核心 UML 模型、實體資料字典與追溯矩陣。修正 Mermaid 標籤括號語法解析問題。 | 待審查（Pending Review） |
| **v1.1** | 2026-11-10 | 全面對齊 `SRS.md` v1.0 需求基線定稿版本，更新註冊驗證、物流狀態及重新上架前置檢核。 | 需求基線對齊定稿（Baselined） |
| **v1.2** | 2026-11-10 | **依需求基線審查意見進行嚴格校準修訂**：<br/>1. 類別圖：修正刊登與訂單基數為 `1` 對 `0..*`（允許歷史訂單，業務約束同時僅一筆有效訂單）；將使用者角色改為 `SystemRole`（MEMBER / ADMIN），買賣家改由交易身分體現以支援同一帳號兼任；補齊完整書目快照屬性；修正爭議與裁決人基數為 `0..*` 對 `0..1`。<br/>2. 訂單狀態機圖：明確時限為預設值，實際依階段開始時適用設定，嚴格標明截止前接受、滿期限即拒絕。<br/>3. 交易循序圖：補上待出貨取消分支「向賣家隱藏買家收件資訊」（BR-011），並同步更新時限採用規則與滿期限拒絕標註。<br/>4. 規格書全文：修正刪帳資料保留敘述嚴格對齊 FR-024 / Q-017（僅 NFR-006 稽核為明確永久保存）；移除未經確認之收件資料字數假設，嚴格對齊 NFR-008。 | 需求基線嚴格對齊審查完成 |
