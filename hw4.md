# 作業 4：UML 類別圖、循序圖與活動圖

> 第 11 組｜高科大校園失物招領系統
>
> 回到 [README](README.md)｜使用案例請見 [hw3.md](hw3.md)

本作業的圖表依系統實際的資料模型（`prisma/schema.prisma`）與程式邏輯（`src/lib/`）繪製。循序圖與活動圖針對 hw3 的三個使用案例：

| 使用案例 | 循序圖 | 活動圖 |
| --- | --- | --- |
| UC-03 登記拾獲物品 | [2.1](#21-uc-03-登記拾獲物品) | [3.1](#31-uc-03-登記拾獲物品) |
| UC-04 提出認領申請 | [2.2](#22-uc-04-提出認領申請) | [3.2](#32-uc-04-提出認領申請) |
| UC-07 審核認領申請 | [2.3](#23-uc-07-審核認領申請) | [3.3](#33-uc-07-審核認領申請) |

---

## 1. UML 類別圖

```mermaid
classDiagram
    direction TB

    class User {
        +String id
        +String email
        +String passwordHash
        +String name
        +String phone
        +Role role
        +DateTime createdAt
        +register()
        +authenticate()
    }

    class Item {
        +String id
        +String name
        +String description
        +String locationDetail
        +DateTime foundAt
        +String storageLocation
        +String hiddenFeatures
        +Boolean isContactPublic
        +ItemStatus status
        +Boolean isRemoved
        +String removedReason
        +DateTime removedAt
        +create()
        +getDisplayStatus() DisplayStatus
        +remove(reason)
        +restore()
    }

    class ItemPhoto {
        +String id
        +String path
        +Int sortOrder
    }

    class Category {
        +String id
        +String name
        +Int sortOrder
        +Boolean isActive
        +create()
        +rename()
        +setActive()
    }

    class Location {
        +String id
        +String name
        +Int sortOrder
        +Boolean isActive
        +create()
        +rename()
        +setActive()
    }

    class Claim {
        +String id
        +String description
        +String lostDetails
        +ClaimStatus status
        +ClaimAutoReject autoRejectedBy
        +String reviewNote
        +DateTime reviewedAt
        +DateTime withdrawnAt
        +DateTime pickedUpAt
        +submit()
        +withdraw()
        +approve()
        +reject(reason)
        +confirmPickup()
    }

    class Notification {
        +String id
        +NotificationType type
        +String message
        +DateTime readAt
        +DateTime createdAt
        +markRead()
    }

    class Role {
        <<enumeration>>
        STUDENT
        ADMIN
    }

    class ItemStatus {
        <<enumeration>>
        UNCLAIMED
        UNDER_REVIEW
        RETURNED
    }

    class ClaimStatus {
        <<enumeration>>
        PENDING
        APPROVED
        REJECTED
        WITHDRAWN
    }

    class ClaimAutoReject {
        <<enumeration>>
        OTHER_APPROVED
        ITEM_REMOVED
    }

    class NotificationType {
        <<enumeration>>
        CLAIM_APPROVED
        ITEM_CLAIMED
        CLAIM_REJECTED
        ITEM_REMOVED
    }

    User "1" -- "0..*" Item : 登記（拾獲者）
    User "0..1" -- "0..*" Item : 下架（管理員）
    Category "1" -- "0..*" Item : 分類
    Location "1" -- "0..*" Item : 拾獲地點
    Item "1" *-- "0..3" ItemPhoto : 照片
    Item "1" -- "0..*" Claim : 被申請
    User "1" -- "0..*" Claim : 申請（失主）
    User "0..1" -- "0..*" Claim : 審核（管理員）
    User "1" *-- "0..*" Notification : 收到
    Item "0..1" -- "0..*" Notification : 相關物品
    Claim "0..1" -- "0..*" Notification : 相關申請

    User ..> Role
    Item ..> ItemStatus
    Claim ..> ClaimStatus
    Claim ..> ClaimAutoReject
    Notification ..> NotificationType
```

### 類別說明

| 類別 | 說明 | 主要操作實作位置 |
| --- | --- | --- |
| User | 使用者帳號，角色分為學生與管理員；信箱一律存小寫，密碼存 bcrypt 雜湊 | `src/lib/auth-service.ts`（`registerUser`、`authenticateUser`） |
| Item | 拾獲物品。`hiddenFeatures`（隱藏特徵）只有管理員看得到；`storageLocation`（保管位置）審核通過後才給認領人看 | `src/lib/item-service.ts`（`createItem`）、`src/lib/item-status.ts`（`getDisplayStatus`）、`src/lib/admin-service.ts`（`removeItem`、`restoreItem`） |
| ItemPhoto | 物品照片，一件物品最多 3 張；只存路徑，檔案放在 Vercel Blob | `src/lib/storage.ts` |
| Category／Location | 物品類別與拾獲地點，由管理員維護；不提供刪除，以「停用」取代 | `src/lib/taxonomy-service.ts` |
| Claim | 認領申請。一件物品可以有多筆申請；`autoRejectedBy` 記錄系統自動駁回的原因 | `src/lib/claim-service.ts`（`submitClaim`、`withdrawClaim`）、`src/lib/admin-service.ts`（`approveClaim`、`rejectClaim`、`confirmPickup`） |
| Notification | 站內通知；`readAt` 為空表示未讀 | `src/lib/notification-service.ts` |

### 設計重點

- **「逾期未領」不存進資料庫**：資料庫只存三種狀態（`ItemStatus`），`getDisplayStatus()` 在查詢時判斷「待認領且拾獲超過 30 天」顯示為逾期未領，不需要排程程式。
- **「下架」不是狀態**：用 `isRemoved` 等欄位另外記錄，這樣復原時可以回到原本的狀態。
- **組合關係（實心菱形）**：照片屬於物品、通知屬於使用者，主體刪除時一併刪除。

---

## 2. 循序圖

參與者說明：**Server Action** 是 Next.js 在伺服器上執行的表單處理函式；**Service** 是 `src/lib/` 裡的商業邏輯；資料庫為 Turso，照片儲存為 Vercel Blob。

### 2.1 UC-03 登記拾獲物品

```mermaid
sequenceDiagram
    actor S as 學生（拾獲者）
    participant UI as 登記表單（瀏覽器）
    participant SA as createItemAction
    participant IS as item-service
    participant ST as 照片儲存（Blob）
    participant DB as 資料庫（Turso）

    S->>UI: 填寫物品資訊並選擇照片
    UI->>UI: 照片縮小至長邊 1600px、1MB 以內
    S->>UI: 按下送出
    UI->>SA: 送出表單資料與照片
    SA->>SA: 確認已登入，取得拾獲者身分
    SA->>IS: createItem(資料, 照片)
    IS->>IS: 檢查必填欄位、拾獲時間、照片格式與張數
    IS->>DB: 查詢類別與地點是否存在且啟用
    DB-->>IS: 類別、地點
    alt 檢查未通過
        IS-->>SA: 各欄位錯誤訊息
        SA-->>UI: 在欄位旁顯示錯誤，保留已填內容
    else 檢查通過
        loop 每一張照片
            IS->>IS: 轉正並移除 EXIF（含 GPS）
            IS->>ST: 以隨機檔名上傳
            ST-->>IS: 上傳完成
        end
        IS->>DB: 建立 Item（狀態 UNCLAIMED）與 ItemPhoto
        DB-->>IS: 物品編號
        IS-->>SA: 登記成功
        SA-->>UI: 導向「我登記的物品」
        UI-->>S: 顯示登記成功
    end
```

### 2.2 UC-04 提出認領申請

```mermaid
sequenceDiagram
    actor S as 學生（失主）
    participant UI as 申請頁（瀏覽器）
    participant SA as submitClaimAction
    participant CS as claim-service
    participant DB as 資料庫（Turso）

    S->>UI: 填寫物品特徵描述與遺失時間地點
    S->>UI: 按下送出
    UI->>SA: 送出申請
    SA->>SA: 確認已登入，取得申請人身分
    SA->>CS: submitClaim(物品, 申請人, 描述)
    CS->>CS: 檢查描述至少 10 個字
    alt 描述太短
        CS-->>SA: 欄位錯誤
        SA-->>UI: 顯示錯誤，保留已填內容
    else 描述合格
        CS->>DB: 開始 transaction，鎖定物品
        CS->>DB: 讀取物品狀態與此申請人的申請紀錄
        DB-->>CS: 物品、申請紀錄
        CS->>CS: checkClaimEligibility 檢查申請資格
        alt 不可申請
            Note over CS: 已領回、已逾期、已下架、自己登記的物品、已有待審申請、曾被駁回
            CS->>DB: 取消 transaction
            CS-->>SA: 拒絕原因
            SA-->>UI: 顯示無法申請的原因
        else 可以申請
            CS->>DB: 建立 Claim（狀態 PENDING）
            opt 物品原本為待認領
                CS->>DB: 物品改為 UNDER_REVIEW（審核中）
            end
            CS->>DB: 提交 transaction
            CS-->>SA: 申請成功
            SA-->>UI: 導向「我的認領申請」
            UI-->>S: 顯示申請狀態「待審核」
        end
    end
```

### 2.3 UC-07 審核認領申請

```mermaid
sequenceDiagram
    actor A as 管理員（學務處）
    participant UI as 認領審核頁（瀏覽器）
    participant SA as Server Action
    participant AS as admin-service
    participant NS as notification-service
    participant DB as 資料庫（Turso）

    A->>UI: 開啟認領審核頁
    UI->>DB: 讀取待審申請、隱藏特徵、照片、保管位置
    DB-->>UI: 依物品分組的待審申請
    A->>A: 比對申請人描述與隱藏特徵
    alt 描述相符，核准
        A->>UI: 按「核准」並確認
        UI->>SA: approveClaimAction(申請編號)
        SA->>SA: requireAdmin 確認管理員身分
        SA->>AS: approveClaim
        AS->>DB: 開始 transaction，鎖定申請與物品
        AS->>DB: 申請改 APPROVED，物品改 RETURNED（已領回）
        AS->>DB: 同物品其他待審申請自動駁回
        AS->>NS: 產生通知事件
        NS->>DB: 寫入通知（申請人、拾獲者、被自動駁回者）
        AS->>DB: 提交 transaction
        AS-->>SA: 成功
        SA-->>UI: 顯示「已核准」
    else 描述不符，駁回
        A->>UI: 填寫駁回原因並按「駁回」
        UI->>SA: rejectClaimAction(申請編號, 原因)
        SA->>SA: requireAdmin 確認管理員身分
        SA->>AS: rejectClaim
        AS->>DB: 開始 transaction，鎖定申請與物品
        AS->>DB: 申請改 REJECTED，記錄駁回原因
        opt 物品已沒有其他待審申請
            AS->>DB: 物品改回 UNCLAIMED（待認領）
        end
        AS->>NS: 產生駁回通知事件
        NS->>DB: 寫入通知（申請人）
        AS->>DB: 提交 transaction
        AS-->>SA: 成功
        SA-->>UI: 顯示「已駁回」
    end
```

---

## 3. 活動圖

圖例：藍色為使用者的動作，綠色為系統的處理，菱形為判斷。

### 3.1 UC-03 登記拾獲物品

```mermaid
flowchart TD
    Start(("開始")) --> A1["點選「登記拾獲物品」"]
    A1 --> D1{"已登入？"}
    D1 -- "否" --> A0["導向登入頁並登入"]
    A0 --> S1
    D1 -- "是" --> S1["顯示登記表單<br/>讀取類別與地點選項"]
    S1 --> A2["填寫名稱、類別、地點、拾獲時間、保管位置<br/>描述、隱藏特徵、是否公開聯絡方式"]
    A2 --> D2{"要附照片？"}
    D2 -- "是" --> A3["選擇照片（最多 3 張）"]
    A3 --> S2["瀏覽器縮小、壓縮照片"]
    S2 --> A4["按下送出"]
    D2 -- "否" --> A4
    A4 --> S3["伺服器檢查欄位與照片"]
    S3 --> D3{"檢查通過？"}
    D3 -- "否" --> S4["顯示錯誤訊息，保留已填內容"]
    S4 --> A2
    D3 -- "是" --> S5["照片轉正、移除 EXIF<br/>以隨機檔名上傳"]
    S5 --> S6["建立物品，狀態為「待認領」"]
    S6 --> S7["導向「我登記的物品」"]
    S7 --> End(("結束"))

    classDef user fill:#E6F1FB,stroke:#185FA5,color:#0C447C
    classDef sys fill:#EAF3DE,stroke:#3B6D11,color:#27500A
    class A0,A1,A2,A3,A4 user
    class S1,S2,S3,S4,S5,S6,S7 sys
```

### 3.2 UC-04 提出認領申請

```mermaid
flowchart TD
    Start(("開始")) --> A1["搜尋並開啟物品詳情頁"]
    A1 --> D0{"物品為待認領<br/>或審核中？"}
    D0 -- "否" --> S0["不顯示申請按鈕"]
    S0 --> End(("結束"))
    D0 -- "是" --> A2["點選「申請認領」"]
    A2 --> D1{"已登入？"}
    D1 -- "否" --> A0["導向登入頁並登入"]
    A0 --> S1
    D1 -- "是" --> S1["檢查申請資格"]
    S1 --> D2{"可以申請？"}
    D2 -- "否" --> S2["顯示無法申請的原因<br/>（自己的物品、已申請、曾被駁回等）"]
    S2 --> End
    D2 -- "是" --> S3["顯示物品公開資訊與申請表單"]
    S3 --> A3["填寫特徵描述與遺失時間地點"]
    A3 --> A4["按下送出"]
    A4 --> D3{"描述至少 10 個字？"}
    D3 -- "否" --> S4["顯示錯誤，保留已填內容"]
    S4 --> A3
    D3 -- "是" --> S5["鎖定物品並重新檢查資格"]
    S5 --> D4{"仍可申請？"}
    D4 -- "否" --> S2
    D4 -- "是" --> S6["建立申請，狀態為「待審核」"]
    S6 --> D5{"物品原本為待認領？"}
    D5 -- "是" --> S7["物品改為「審核中」"]
    S7 --> S8["導向「我的認領申請」"]
    D5 -- "否" --> S8
    S8 --> End

    classDef user fill:#E6F1FB,stroke:#185FA5,color:#0C447C
    classDef sys fill:#EAF3DE,stroke:#3B6D11,color:#27500A
    class A0,A1,A2,A3,A4 user
    class S0,S1,S2,S3,S4,S5,S6,S7,S8 sys
```

### 3.3 UC-07 審核認領申請

```mermaid
flowchart TD
    Start(("開始")) --> A1["管理員開啟「認領審核」"]
    A1 --> S1["列出待審申請（依物品分組）<br/>並排顯示申請描述與隱藏特徵"]
    S1 --> A2["比對申請人描述與隱藏特徵"]
    A2 --> D1{"描述相符？"}
    D1 -- "是" --> A3["按「核准」並確認"]
    A3 --> S2["申請改為「已通過」<br/>物品改為「已領回」"]
    S2 --> S3["同物品其他待審申請自動駁回"]
    S3 --> S4["通知申請人與拾獲者"]
    S4 --> End(("結束"))
    D1 -- "否" --> A4["填寫駁回原因，按「駁回」"]
    A4 --> D2{"已填寫原因？"}
    D2 -- "否" --> S5["提示原因必填"]
    S5 --> A4
    D2 -- "是" --> S6["申請改為「已駁回」"]
    S6 --> D3{"物品還有其他待審申請？"}
    D3 -- "是" --> S7["物品維持「審核中」"]
    D3 -- "否" --> S8["物品改回「待認領」"]
    S7 --> S9["通知申請人（含駁回原因）"]
    S8 --> S9
    S9 --> End

    classDef user fill:#E6F1FB,stroke:#185FA5,color:#0C447C
    classDef sys fill:#EAF3DE,stroke:#3B6D11,color:#27500A
    class A1,A2,A3,A4 user
    class S1,S2,S3,S4,S5,S6,S7,S8,S9 sys
```
