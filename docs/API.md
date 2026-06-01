# API Reference

Dokumentasi lengkap REST API endpoints POS-EXPO-MONOLITH.

## Base URL
```
Development: http://localhost:3000/api
Production:  https://api.pos-expo.com/api
```

## Authentication

### JWT Token
Setiap request (kecuali login) memerlukan header:
```
Authorization: Bearer <jwt_token>
```

### Login
```http
POST /auth/login
Content-Type: application/json

{
  "email": "cashier@example.com",
  "password": "password123"
}

Response 200:
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
  "expiresIn": 3600,
  "user": {
    "id": "user-123",
    "name": "Budi Santoso",
    "email": "cashier@example.com",
    "role": "cashier",
    "branchId": "branch-1"
  }
}
```

### Logout
```http
POST /auth/logout
Authorization: Bearer <token>

Response 200:
{
  "message": "Logout successful"
}
```

### Refresh Token
```http
POST /auth/refresh
Content-Type: application/json

{
  "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}

Response 200:
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "expiresIn": 3600
}
```

---

## POS Transactions

### Create Transaction
```http
POST /pos/transactions
Authorization: Bearer <token>
Content-Type: application/json

{
  "branchId": "branch-1",
  "customerId": "cust-123",
  "items": [
    {
      "productId": "prod-1",
      "quantity": 2,
      "unitPrice": 5000
    },
    {
      "productId": "prod-2",
      "quantity": 1,
      "unitPrice": 10000
    }
  ]
}

Response 201:
{
  "id": "txn-12345",
  "branchId": "branch-1",
  "transactionNumber": "20260601001234",
  "customerId": "cust-123",
  "items": [
    {
      "productId": "prod-1",
      "quantity": 2,
      "unitPrice": 5000,
      "subtotal": 10000
    }
  ],
  "subtotal": 20000,
  "discountAmount": 1000,
  "taxAmount": 1900,
  "totalAmount": 20900,
  "status": "pending",
  "createdAt": "2026-06-01T14:30:00Z"
}
```

### Get Transaction
```http
GET /pos/transactions/:transactionId
Authorization: Bearer <token>

Response 200:
{
  "id": "txn-12345",
  "transactionNumber": "20260601001234",
  "items": [...],
  "totalAmount": 20900,
  "status": "pending",
  "createdAt": "2026-06-01T14:30:00Z"
}
```

### Hold Order
```http
POST /pos/transactions/:transactionId/hold
Authorization: Bearer <token>
Content-Type: application/json

{
  "notes": "Pelanggan ambil barang tambahan"
}

Response 200:
{
  "id": "txn-12345",
  "status": "on_hold",
  "heldAt": "2026-06-01T14:35:00Z"
}
```

### Get Hold Orders
```http
GET /pos/hold-orders?branchId=branch-1
Authorization: Bearer <token>

Response 200:
[
  {
    "id": "txn-12345",
    "transactionNumber": "20260601001234",
    "customerName": "Budi",
    "items": [...],
    "totalAmount": 20900,
    "heldAt": "2026-06-01T14:35:00Z"
  }
]
```

### Resume Order
```http
POST /pos/transactions/:transactionId/resume
Authorization: Bearer <token>

Response 200:
{
  "id": "txn-12345",
  "status": "pending",
  "items": [...],
  "totalAmount": 20900
}
```

### Pay Transaction
```http
POST /pos/transactions/:transactionId/pay
Authorization: Bearer <token>
Content-Type: application/json

{
  "method": "cash",
  "amount": 25000,
  "change": 4100
}

Response 200:
{
  "id": "txn-12345",
  "status": "completed",
  "paymentMethod": "cash",
  "amount": 20900,
  "change": 4100,
  "completedAt": "2026-06-01T14:40:00Z"
}
```

### Split Payment
```http
POST /pos/transactions/:transactionId/pay/split
Authorization: Bearer <token>
Content-Type: application/json

{
  "payments": [
    {
      "method": "cash",
      "amount": 10000
    },
    {
      "method": "qris",
      "amount": 10900
    }
  ]
}

Response 200:
{
  "id": "txn-12345",
  "status": "completed",
  "payments": [
    { "method": "cash", "amount": 10000 },
    { "method": "qris", "amount": 10900, "referenceNumber": "..." }
  ]
}
```

---

## Cash Register

### Open Cash Register
```http
POST /pos/cash-register/open
Authorization: Bearer <token>
Content-Type: application/json

{
  "openingBalance": 500000
}

Response 201:
{
  "id": "cr-123",
  "branchId": "branch-1",
  "cashierId": "user-123",
  "openingBalance": 500000,
  "status": "open",
  "openedAt": "2026-06-01T08:00:00Z"
}
```

### Close Cash Register
```http
POST /pos/cash-register/close
Authorization: Bearer <token>
Content-Type: application/json

{
  "actualTotal": 1250000
}

Response 200:
{
  "id": "cr-123",
  "status": "closed",
  "openingBalance": 500000,
  "expectedTotal": 1200000,
  "actualTotal": 1250000,
  "variance": 50000,
  "variancePercentage": 4.17,
  "closedAt": "2026-06-01T21:00:00Z"
}
```

---

## Inventory

### Get Products
```http
GET /inventory/products?branchId=branch-1&category=minuman&limit=20&offset=0
Authorization: Bearer <token>

Response 200:
{
  "data": [
    {
      "id": "prod-1",
      "sku": "SKU001",
      "barcode": "8991001234567",
      "name": "Mie Instan",
      "category": "makanan",
      "price": 5000,
      "stock": 150
    }
  ],
  "total": 250,
  "limit": 20,
  "offset": 0
}
```

### Search Products
```http
GET /inventory/products/search?q=mie&branchId=branch-1
Authorization: Bearer <token>

Response 200:
[
  {
    "id": "prod-1",
    "name": "Mie Instan Goreng",
    "sku": "SKU001",
    "price": 5000,
    "stock": 150
  },
  {
    "id": "prod-2",
    "name": "Mie Kuning",
    "sku": "SKU002",
    "price": 8000,
    "stock": 75
  }
]
```

### Get Product by Barcode
```http
GET /inventory/products/by-barcode/8991001234567?branchId=branch-1
Authorization: Bearer <token>

Response 200:
{
  "id": "prod-1",
  "name": "Mie Instan",
  "sku": "SKU001",
  "price": 5000,
  "stock": 150,
  "category": "makanan"
}
```

### Get Product Detail
```http
GET /inventory/products/prod-1
Authorization: Bearer <token>

Response 200:
{
  "id": "prod-1",
  "sku": "SKU001",
  "barcode": "8991001234567",
  "name": "Mie Instan Goreng",
  "description": "Mie instan dengan rasa goreng yang nikmat",
  "category": "makanan",
  "price": 5000,
  "cost": 2500,
  "images": ["url1", "url2"],
  "isActive": true
}
```

### Get Stock Levels
```http
GET /inventory/stock?branchId=branch-1&limit=50
Authorization: Bearer <token>

Response 200:
[
  {
    "id": "stock-1",
    "productId": "prod-1",
    "productName": "Mie Instan",
    "quantity": 150,
    "reservedQuantity": 10,
    "availableQuantity": 140,
    "expiryDate": "2026-12-31"
  }
]
```

### Check Available Stock
```http
GET /inventory/stock/available?productId=prod-1
Authorization: Bearer <token>

Response 200:
[
  {
    "branchId": "branch-1",
    "branchName": "Toko Pusat",
    "quantity": 150,
    "availableQuantity": 140
  },
  {
    "branchId": "branch-2",
    "branchName": "Toko Bandung",
    "quantity": 75,
    "availableQuantity": 75
  },
  {
    "branchId": "gudang",
    "branchName": "Gudang Pusat",
    "quantity": 500,
    "availableQuantity": 500
  }
]
```

### Create Stock Transfer
```http
POST /inventory/transfers
Authorization: Bearer <token>
Content-Type: application/json

{
  "fromBranchId": "branch-1",
  "toBranchId": "branch-2",
  "productId": "prod-1",
  "quantity": 20
}

Response 201:
{
  "id": "transfer-123",
  "fromBranch": { "id": "branch-1", "name": "Toko Pusat" },
  "toBranch": { "id": "branch-2", "name": "Toko Bandung" },
  "product": { "id": "prod-1", "name": "Mie Instan" },
  "quantity": 20,
  "status": "pending",
  "createdAt": "2026-06-01T10:00:00Z"
}
```

---

## Customers

### Get Customers
```http
GET /customers?limit=20&offset=0
Authorization: Bearer <token>

Response 200:
{
  "data": [
    {
      "id": "cust-1",
      "name": "Budi Santoso",
      "phone": "08123456789",
      "email": "budi@email.com",
      "loyaltyTier": "GOLD",
      "loyaltyPoints": 250
    }
  ],
  "total": 150
}
```

### Search Customers
```http
GET /customers/search?q=budi
Authorization: Bearer <token>

Response 200:
[
  {
    "id": "cust-1",
    "name": "Budi Santoso",
    "phone": "08123456789",
    "loyaltyTier": "GOLD",
    "loyaltyPoints": 250
  }
]
```

### Create Customer
```http
POST /customers
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Budi Santoso",
  "phone": "08123456789",
  "email": "budi@email.com",
  "address": "Jl. Merdeka No. 123"
}

Response 201:
{
  "id": "cust-1",
  "name": "Budi Santoso",
  "phone": "08123456789",
  "email": "budi@email.com",
  "loyaltyTier": "STANDARD",
  "loyaltyPoints": 0,
  "createdAt": "2026-06-01T10:00:00Z"
}
```

### Get Customer Detail
```http
GET /customers/cust-1
Authorization: Bearer <token>

Response 200:
{
  "id": "cust-1",
  "name": "Budi Santoso",
  "phone": "08123456789",
  "email": "budi@email.com",
  "address": "Jl. Merdeka No. 123",
  "loyaltyTier": "GOLD",
  "loyaltyPoints": 250,
  "membershipDate": "2025-01-15",
  "totalSpent": 2500000,
  "transactionCount": 125
}
```

### Get Loyalty Info
```http
GET /customers/cust-1/loyalty
Authorization: Bearer <token>

Response 200:
{
  "customerId": "cust-1",
  "currentPoints": 250,
  "tier": "GOLD",
  "tierBenefit": "1.5x points multiplier",
  "pointsToNextTier": 750,
  "pointsExpiry": null,
  "redeemableAmount": 2500  // Rp 25,000
}
```

### Redeem Loyalty Points
```http
POST /customers/cust-1/loyalty/redeem
Authorization: Bearer <token>
Content-Type: application/json

{
  "points": 100
}

Response 200:
{
  "customerId": "cust-1",
  "pointsRedeemed": 100,
  "discountAmount": 1000,
  "remainingPoints": 150,
  "appliedAt": "2026-06-01T14:00:00Z"
}
```

---

## Bookings

### Get Bookings
```http
GET /bookings?status=pending&limit=20
Authorization: Bearer <token>

Response 200:
{
  "data": [
    {
      "id": "book-1",
      "customer": { "id": "cust-1", "name": "Budi", "phone": "081234" },
      "product": { "id": "prod-1", "name": "Mie Instan" },
      "quantity": 5,
      "bookingDate": "2026-06-05",
      "bookingTime": "14:00",
      "status": "pending",
      "createdAt": "2026-06-01T10:00:00Z"
    }
  ],
  "total": 15
}
```

### Create Booking
```http
POST /bookings
Authorization: Bearer <token>
Content-Type: application/json

{
  "customerId": "cust-1",
  "productId": "prod-1",
  "branchId": "branch-1",
  "quantity": 5,
  "bookingDate": "2026-06-05",
  "bookingTime": "14:00",
  "notes": "Untuk acara pertemuan"
}

Response 201:
{
  "id": "book-1",
  "bookingNumber": "BK-2026060101",
  "status": "pending",
  "confirmationCode": "BK-ABC123",
  "createdAt": "2026-06-01T10:00:00Z"
}
```

### Get Available Slots
```http
GET /bookings/available-slots?productId=prod-1&date=2026-06-05
Authorization: Bearer <token>

Response 200:
{
  "date": "2026-06-05",
  "slots": [
    { "time": "08:00", "available": true },
    { "time": "09:00", "available": true },
    { "time": "10:00", "available": false },
    { "time": "11:00", "available": true },
    { "time": "12:00", "available": false },
    { "time": "13:00", "available": true },
    { "time": "14:00", "available": true },
    { "time": "15:00", "available": false },
    { "time": "16:00", "available": true }
  ]
}
```

### Complete Booking
```http
POST /bookings/book-1/complete
Authorization: Bearer <token>

Response 200:
{
  "id": "book-1",
  "status": "completed",
  "completedAt": "2026-06-05T14:00:00Z"
}
```

### Cancel Booking
```http
DELETE /bookings/book-1
Authorization: Bearer <token>

Response 200:
{
  "id": "book-1",
  "status": "cancelled",
  "cancelledAt": "2026-06-01T15:00:00Z"
}
```

---

## Promotions

### Get Active Promotions
```http
GET /promotions/active?branchId=branch-1
Authorization: Bearer <token>

Response 200:
[
  {
    "id": "promo-1",
    "name": "Summer Sale",
    "type": "PERCENTAGE",
    "value": 20,
    "minimumPurchase": 100000,
    "startDate": "2026-06-01",
    "endDate": "2026-06-30"
  },
  {
    "id": "promo-2",
    "name": "BOGO Minuman",
    "type": "BOGO",
    "value": 50,
    "startDate": "2026-06-01",
    "endDate": "2026-06-30"
  }
]
```

### Calculate Discount
```http
POST /promotions/calculate
Authorization: Bearer <token>
Content-Type: application/json

{
  "items": [
    {
      "productId": "prod-1",
      "quantity": 2,
      "unitPrice": 5000,
      "categoryId": "cat-1"
    },
    {
      "productId": "prod-2",
      "quantity": 1,
      "unitPrice": 10000,
      "categoryId": "cat-2"
    }
  ],
  "customerId": "cust-1",
  "branchId": "branch-1"
}

Response 200:
{
  "subtotal": 20000,
  "discountAmount": 2000,
  "discounts": [
    {
      "promotionId": "promo-1",
      "promotionName": "Summer Sale",
      "amount": 2000
    }
  ],
  "finalAmount": 18000
}
```

---

## Reports

### Sales Report - Daily
```http
GET /reports/sales/daily?branchId=branch-1&date=2026-06-01
Authorization: Bearer <token>

Response 200:
{
  "date": "2026-06-01",
  "branchId": "branch-1",
  "totalTransactions": 120,
  "totalRevenue": 2400000,
  "discountAmount": 120000,
  "taxAmount": 240000,
  "paymentMethods": {
    "cash": 1200000,
    "qris": 1000000,
    "card": 200000
  },
  "topProducts": [
    { "productId": "prod-1", "name": "Mie Instan", "sold": 250 }
  ]
}
```

### Product Performance
```http
GET /reports/products/performance?branchId=branch-1&startDate=2026-05-01&endDate=2026-06-01
Authorization: Bearer <token>

Response 200:
[
  {
    "productId": "prod-1",
    "name": "Mie Instan",
    "category": "makanan",
    "unitsSold": 500,
    "revenue": 2500000,
    "averagePrice": 5000,
    "rank": "FAST_MOVING"
  }
]
```

### Staff Performance
```http
GET /reports/staff/performance?branchId=branch-1&startDate=2026-05-01&endDate=2026-06-01
Authorization: Bearer <token>

Response 200:
[
  {
    "staffId": "user-1",
    "staffName": "Budi Santoso",
    "totalTransactions": 500,
    "totalRevenue": 10000000,
    "averageTransactionValue": 20000,
    "rank": 1,
    "commission": 500000
  }
]
```

---

## Hardware

### Print Receipt
```http
POST /hardware/print-receipt/:transactionId
Authorization: Bearer <token>

Response 200:
{
  "printed": true,
  "timestamp": "2026-06-01T14:40:00Z"
}
```

### Open Cash Drawer
```http
POST /hardware/drawer/kick
Authorization: Bearer <token>

Response 200:
{
  "opened": true,
  "timestamp": "2026-06-01T14:40:00Z"
}
```

### Get Hardware Status
```http
GET /hardware/status
Authorization: Bearer <token>

Response 200:
{
  "printer": {
    "connected": true,
    "type": "usb",
    "model": "Epson TM-T88"
  },
  "scanner": {
    "connected": true,
    "type": "keyboard-wedge"
  },
  "drawer": {
    "connected": true,
    "status": "closed"
  },
  "payment": {
    "qris": true,
    "merchantId": "ID.MICROPAYMENT.xxx"
  }
}
```

### Generate QRIS Code
```http
POST /pos/qris
Authorization: Bearer <token>
Content-Type: application/json

{
  "transactionId": "txn-12345",
  "amount": 20900
}

Response 200:
{
  "qrImage": "data:image/png;base64,...",
  "amount": 20900,
  "expiresIn": 300000,
  "timeout": 300000
}
```

---

## Error Responses

### 400 Bad Request
```json
{
  "statusCode": 400,
  "message": "Invalid request",
  "errors": [
    {
      "field": "email",
      "message": "Email is required"
    }
  ]
}
```

### 401 Unauthorized
```json
{
  "statusCode": 401,
  "message": "Unauthorized",
  "error": "Invalid or expired token"
}
```

### 403 Forbidden
```json
{
  "statusCode": 403,
  "message": "Forbidden",
  "error": "You don't have permission to access this resource"
}
```

### 404 Not Found
```json
{
  "statusCode": 404,
  "message": "Not Found",
  "error": "Transaction not found"
}
```

### 500 Internal Server Error
```json
{
  "statusCode": 500,
  "message": "Internal Server Error",
  "error": "Something went wrong"
}
```

---

## Rate Limiting

- **Limit**: 1000 requests per hour per user
- **Headers**: 
  ```
  X-RateLimit-Limit: 1000
  X-RateLimit-Remaining: 999
  X-RateLimit-Reset: 1234567890
  ```

---

## Pagination

Semua list endpoint support pagination:
- `limit`: Jumlah item per halaman (default: 20, max: 100)
- `offset`: Offset dari awal (default: 0)
- `page`: Alternative ke offset (page 1 = offset 0)

Response:
```json
{
  "data": [...],
  "total": 150,
  "limit": 20,
  "offset": 0,
  "page": 1,
  "pages": 8
}
```

---

**Document Version**: 1.0  
**Last Updated**: June 1, 2026
