# MIDTERM 2 — Online Banking (NMCNPM)

## 1) Bối cảnh & Phạm vi

Hệ thống ngân hàng trực tuyến phục vụ hai loại người dùng cuối:

- **Người dùng A:** Tóm tắt tài khoản, **Chuyển tiền**  
- **Người dùng B:** Tóm tắt tài khoản, **Thanh toán hóa đơn** (điện/nước/Internet…)

Các dịch vụ/hệ thống tích hợp: **Auth Service**, **Core Banking**, **Bill Aggregator/Provider**, **Payment Gateway**, **Notification Service**, **Support/Operator**.

---

## 2) Actors

- Customer (Web/Mobile)
- Auth Service (đăng nhập/OTP)
- Core Banking (tài khoản, số dư, bút toán)
- Bill Aggregator / Provider (Payoo/EVN/…)
- Payment Gateway (thẻ nội địa/quốc tế)
- Notification Service (Email/SMS)
- Support/Operator (tra soát)

---

## 3) Functional Requirements (FR)

- **FR1**: Đăng nhập/đăng xuất an toàn (username + password + 2FA/OTP)  
- **FR2**: Xem tóm tắt tài khoản (số dư, 5–20 giao dịch gần nhất)  
- **FR3**: Chuyển tiền nội bộ/liên ngân hàng (áp dụng cho người dùng A)  
- **FR4**: Quản lý biller (đăng ký/sửa/xóa) — *optional cho B*  
- **FR5**: Tra cứu hóa đơn (theo loại dịch vụ/nhà cung cấp/khu vực/mã KH)  
- **FR6**: Xem chi tiết hóa đơn (kỳ, số tiền, hạn, phí)  
- **FR7**: Thanh toán hóa đơn (tài khoản/thẻ); tạo giao dịch & chứng từ  
- **FR8**: Xem lịch sử thanh toán & trạng thái (Success / Processing / Failed)  
- **FR9**: Cấu hình cảnh báo (SMS/Email) khi có hóa đơn mới/thanh toán  
- **FR10**: Tải biên lai điện tử (PDF/HTML) có mã tra soát

---

## 4) Use Case Diagram (Mermaid)


```plantuml
@startuml
' ========== Use Case (UML) — Top-to-Bottom with Actors ==========
top to bottom direction
skinparam packageStyle rectangle
skinparam usecase {
  BackgroundColor #ffffff
  BorderColor #333333
}
' dùng stickman mặc định (không bật actorStyle awesome)

' ------------------ ACTORS ------------------
' Nhóm bên trái
together {
  actor CustomerA as "Khách hàng A"
  actor CustomerB as "Khách hàng B"
}

' Nhóm bên phải (các hệ thống ngoài)
together {
  actor Auth       as "Auth Service"
  actor CoreBank   as "Core Banking"
  actor BillerProv as "Bill Aggregator/Provider"
  actor PaymentGW  as "Payment Gateway"
  actor Notify     as "Notification Service"
  actor Support    as "Support/Operator"
}

' ------------------ SYSTEM BOUNDARY ------------------
rectangle "Ngân hàng trực tuyến" as System {
  usecase UC_Login           as "UC-Login"
  usecase UC_ViewAcct        as "UC-ViewAccountSummary"
  usecase UC_Transfer        as "UC-Transfer"
  usecase UC_ManageBiller    as "UC-ManageBiller"
  usecase UC_QueryBills      as "UC-QueryBills"
  usecase UC_ViewBillDetail  as "UC-ViewBillDetail"
  usecase UC_PayBill         as "UC-PayBill"
  usecase UC_ViewPaymentHist as "UC-ViewPaymentHistory"
  usecase UC_ConfigAlert     as "UC-ConfigureAlerts"
  usecase UC_DownloadReceipt as "UC-DownloadReceipt"
}

' ------------------ LINKS: CUSTOMER SIDE ------------------
CustomerA --> UC_Login
CustomerB --> UC_Login
CustomerA --> UC_ViewAcct
CustomerB --> UC_ViewAcct
CustomerA --> UC_Transfer
CustomerB --> UC_ManageBiller
CustomerB --> UC_QueryBills
CustomerB --> UC_ViewBillDetail
CustomerB --> UC_PayBill
CustomerB --> UC_ViewPaymentHist
CustomerA --> UC_ViewPaymentHist
CustomerA --> UC_ConfigAlert
CustomerB --> UC_ConfigAlert
CustomerB --> UC_DownloadReceipt

' ------------------ LINKS: EXTERNAL SYSTEMS ------------------
UC_Login           --> Auth
UC_ViewAcct        --> CoreBank
UC_Transfer        --> CoreBank
UC_QueryBills      --> BillerProv
UC_ViewBillDetail  --> BillerProv
UC_PayBill         --> CoreBank
UC_PayBill         --> PaymentGW
UC_PayBill         --> Notify
UC_ViewPaymentHist --> CoreBank
UC_DownloadReceipt --> Notify

' ------------------ RELATIONSHIPS BETWEEN USE CASES ------------------
UC_PayBill .> UC_DownloadReceipt : <<include>>
UC_ViewBillDetail .> UC_QueryBills : <<extend>>

' kênh hỗ trợ/tra soát
Support --- Notify

@enduml

```
---

## 5) Sequence Diagrams

### 5.1 Login (FR1)

```mermaid
sequenceDiagram
autonumber
actor User as Customer
participant Web as Web/Mobile App
participant Auth as Auth Service
participant Noti as Notification Service

User->>Web: Nhập username/password
Web->>Auth: Verify credentials
Auth-->>Web: OK + yêu cầu OTP
Web->>User: Hiển thị form OTP
User->>Web: Nhập OTP
Web->>Auth: Validate OTP (TOTP/SMS)
Auth-->>Web: Token phiên/claims
Web-->>User: Đăng nhập thành công
note over Web,Noti: (Optional) Gửi cảnh báo đăng nhập
Web->>Noti: Send login alert
```

---

### 5.2 Query Bill (FR5 → FR6)

```mermaid
sequenceDiagram
autonumber
actor UserB as Customer B
participant App as Web/Mobile App
participant Biller as Bill Aggregator/Provider

UserB->>App: Chọn loại dịch vụ + nhập mã KH
App->>Biller: QueryBills(service_type, provider, area, customer_code)
Biller-->>App: Danh sách hóa đơn (id, cycle, amount, due_date, status)
App-->>UserB: Hiển thị danh sách
UserB->>App: Chọn một hóa đơn để xem chi tiết
App->>Biller: GetBillDetail(bill_id)
Biller-->>App: {cycle, amount, due_date, fee, status}
App-->>UserB: Hiển thị chi tiết hóa đơn (FR6)
```

---

### 5.3 Pay Bill (FR7 + biên lai FR10 + thông báo FR9)

```mermaid
sequenceDiagram
autonumber
actor UserB as Customer B
participant App as Web/Mobile App
participant Core as Core Banking
participant GW as Payment Gateway
participant Biller as Bill Aggregator/Provider
participant Noti as Notification Service

UserB->>App: Chọn hóa đơn + nguồn tiền (tài khoản/thẻ)
App->>Core: Hold funds / Create Payment(order_id, amount)
Core-->>App: Hold OK + provider_ref
App->>GW: Charge (nếu thanh toán bằng thẻ)
GW-->>App: Charge result (success/fail)
App->>Biller: MarkBillPaid(bill_id, provider_ref)
Biller-->>App: Update status (PAID)
App->>Core: Post transaction (debit + txns)
Core-->>App: Posted OK
App->>Noti: Gửi thông báo + tạo biên lai (PDF/HTML)
App-->>UserB: Kết quả thanh toán + link tải biên lai (FR10)
```

---

### 5.4 View Payment History (FR8)

```mermaid
sequenceDiagram
autonumber
actor User as Customer
participant App as Web/Mobile App
participant Core as Core Banking

User->>App: Xem lịch sử thanh toán
App->>Core: Query payments (date range, paging)
Core-->>App: Danh sách payment (status, amount, method, ref)
App-->>User: Hiển thị + lọc/trang
```

---

### 5.5 Transfer Money (FR3 — Người dùng A)

```mermaid
sequenceDiagram
autonumber
actor UserA as Customer A
participant App as Web/Mobile App
participant Core as Core Banking
participant Noti as Notification Service

UserA->>App: Nhập thông tin chuyển tiền
App->>Core: Validate (balance, limits, beneficiary)
Core-->>App: OK
App->>Core: Execute transfer (create debit/credit)
Core-->>App: Success + txn_ids + balance_after
App->>Noti: Gửi thông báo (SMS/Email)
App-->>UserA: Kết quả + số tham chiếu giao dịch
```

---

### 5.6 Error Rollback (thất bại một bước trong Pay Bill/Transfer)

```mermaid
sequenceDiagram
autonumber
participant App as Web/Mobile App
participant Core as Core Banking
participant GW as Payment Gateway
participant Biller as Bill Aggregator/Provider
participant Noti as Notification Service

App->>Core: Hold/Pre-post
Core-->>App: OK (hold_id)
App->>GW: Charge
GW-->>App: Success
App->>Biller: MarkBillPaid
Biller-->>App: FAILED (timeout/error)

note over App: Bắt đầu rollback (saga/compensation)
App->>GW: Refund/void charge
GW-->>App: Refunded
App->>Core: Release hold / Reversal
Core-->>App: Released
App->>Noti: Thông báo giao dịch thất bại + mã tra soát
```

---

## 6) ER Diagram (ERD)

```mermaid
erDiagram
  USER {
    int user_id PK
    string name
    string email
    string phone
    string role
    string status
    datetime created_at
  }

  ACCOUNT {
    int account_id PK
    int user_id FK
    string acct_no
    string acct_type
    decimal balance
    string currency
    string status
    datetime created_at
  }

  BILLER {
    int biller_id PK
    string service_type
    string provider_name
    string area
    string status
  }

  USER_BILLER {
    int user_id FK
    int biller_id FK
    string customer_code
    string alias
    datetime created_at
  }

  BILL {
    int bill_id PK
    int biller_id FK
    string customer_code
    string cycle
    decimal amount
    date due_date
    string status
  }

  PAYMENT {
    int payment_id PK
    int user_id FK
    int account_id FK
    int bill_id FK
    decimal amount
    decimal fee
    string method
    string status
    string provider_ref
    datetime created_at
    datetime updated_at
    string idempotency_key
  }

  TRANSACTION {
    int txn_id PK
    int account_id FK
    int payment_id FK
    string txn_type
    decimal amount
    decimal balance_after
    datetime posted_at
    string status
  }

  NOTIFICATION {
    int notif_id PK
    int user_id FK
    string channel
    string subject
    string content
    string ref_type
    int ref_id
    string status
    datetime sent_at
  }

  %% Relationships
  USER ||--o{ ACCOUNT : "1-N"
  USER ||--o{ USER_BILLER : "1-N"
  BILLER ||--o{ USER_BILLER : "1-N"
  BILLER ||--o{ BILL : "1-N"
  BILL ||--o| PAYMENT : "1-0..1"
  USER ||--o{ PAYMENT : "1-N"
  ACCOUNT ||--o{ TRANSACTION : "1-N"
  PAYMENT ||--o{ TRANSACTION : "1-N (debit/rollback)"
  USER ||--o{ NOTIFICATION : "1-N"
```

---

## 7) Ma trận Traceability (FR ↔ Use Case)

| FR  | Use Case                        |
|-----|---------------------------------|
| FR1 | UC-Login                        |
| FR2 | UC-ViewAccountSummary           |
| FR3 | UC-Transfer (User A)            |
| FR4 | UC-ManageBiller                 |
| FR5 | UC-QueryBills                   |
| FR6 | UC-ViewBillDetail               |
| FR7 | UC-PayBill                      |
| FR8 | UC-ViewPaymentHistory           |
| FR9 | UC-ConfigureAlerts              |
| FR10| UC-DownloadReceipt              |

---

## 8) Ghi chú triển khai & nộp

- Biểu đồ dùng **Mermaid** để GitHub render trực tiếp.  
- Use Case mô phỏng bằng `flowchart` cho tương thích GitHub.  
- Trường “idempotency_key” trong **PAYMENT** đảm bảo **an toàn khi retry** thanh toán.  
- **Error Rollback** minh họa bằng mô hình **saga/compensation** (refund/void + release hold).  
- Nộp bài: chỉ cần commit file **README.md** này vào repo GitHub.
