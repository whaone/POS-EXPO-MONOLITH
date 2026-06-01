# Features & Requirements

Dokumentasi lengkap semua fitur dan requirement POS-EXPO-MONOLITH.

## 1. Core POS & Kasir (Front-End)

### 1.1 Multi-Session & Cash Control

**Requirement**: Kasir harus membuka sesi kas sebelum mulai melayani transaksi.

**Alur**:
1. Kasir login dengan credentials
2. Kasir input **saldo awal** kas (contoh: Rp 500.000 untuk kembalian)
3. Sistem mencatat waktu pembukaan & saldo awal
4. Kasir dapat melayani transaksi
5. Di akhir shift, kasir tutup kas
6. Sistem hitung:
   - **Expected Total**: Jumlah uang yang seharusnya ada berdasarkan transaksi
   - **Actual Total**: Uang tunai yang dihitung manual oleh kasir
   - **Variance**: Selisih (deteksi fraud/kecurangan atau kesalahan hitung)

**Entity**: `CashRegister`
```typescript
interface CashRegister {
  id: string;
  branchId: string;
  cashierId: string;
  openingBalance: number;      // Saldo awal (Rp 500.000)
  closingBalance: number;      // Saldo akhir (diinput manual)
  expectedTotal: number;       // Total expected dari transaksi
  actualTotal: number;         // Total actual (manual count)
  variance: number;            // Selisih (expected - actual)
  status: 'open' | 'closed';
  openedAt: Date;
  closedAt: Date;
  notes: string;               // Catatan jika ada variance
}
```

**API Endpoints**:
```
POST   /api/pos/cash-register/open
Body: {
  openingBalance: 500000
}
Response: { cashRegisterId, status: 'open', openedAt }

POST   /api/pos/cash-register/close
Body: {
  actualTotal: 1250000
}
Response: { 
  status: 'closed',
  expectedTotal: 1200000,
  actualTotal: 1250000,
  variance: 50000,  // Positive = surplus
  closedAt
}
```

---

### 1.2 Put On Hold / Parkir Tagihan

**Requirement**: Kasir dapat menunda/suspend satu order untuk melayani pelanggan lain.

**Use Case**: 
- Pelanggan A sudah bayar sebagian atau siap bayar
- Tiba-tiba pelanggan A ingin ambil barang lain yang tertinggal
- Kasir bisa "hold" pesanan A, lalu layani pelanggan B
- Setelah pelanggan B selesai, kasir "resume" pesanan A

**Entity**: `Transaction`
```typescript
interface Transaction {
  // ... existing fields
  status: 'completed' | 'pending' | 'cancelled' | 'on_hold';
  heldAt?: Date;
  heldByNotes?: string;  // Alasan hold
}
```

**API Endpoints**:
```
POST   /api/pos/transactions/:id/hold
Body: {
  notes: "Pelanggan ambil barang lagi"
}
Response: { status: 'on_hold', transactionId }

GET    /api/pos/hold-orders
Query: { branchId }
Response: [
  {
    transactionId,
    customerName,
    items: [...],
    totalAmount,
    heldAt,
    notes
  }
]

POST   /api/pos/transactions/:id/resume
Response: { status: 'pending', transactionId, items: [...] }
```

---

### 1.3 Flexible Payment Methods

**Requirement**: Mendukung berbagai metode pembayaran termasuk split payment.

**Metode Pembayaran**:
1. **Tunai (Cash)**: Pembayaran dengan uang fisik
2. **QRIS**: QR Code Indonesian Standard (e-wallet, bank transfer)
3. **Debit/Credit Card**: Via payment terminal/gateway
4. **Split Payment**: Kombinasi metode (contoh: 50% tunai + 50% QRIS)

**Entity**: `Payment`
```typescript
interface Payment {
  id: string;
  transactionId: string;
  method: 'cash' | 'qris' | 'debit' | 'credit' | 'transfer';
  amount: number;
  status: 'pending' | 'confirmed' | 'failed';
  referenceNumber?: string;  // Untuk non-cash
  timestamp: Date;
}

// Split payment example:
interface SplitPayment {
  transactionId: string;
  payments: Payment[];  // Multiple payments
  totalAmount: number;
  change?: number;  // Kembalian jika ada
}
```

**API Endpoints**:
```
POST   /api/pos/transactions/:id/pay
Body: {
  method: 'cash',
  amount: 1250000,
  change?: 50000
}
Response: { transactionId, paid: true, status: 'completed' }

POST   /api/pos/transactions/:id/pay/split
Body: {
  payments: [
    { method: 'cash', amount: 500000 },
    { method: 'qris', amount: 750000 }
  ]
}
Response: { transactionId, totalPaid: 1250000, status: 'completed' }

POST   /api/pos/transactions/:id/qris
Response: {
  qrImage: "data:image/png;base64,...",
  amount: 1250000,
  timeout: 300000  // 5 minutes
}
```

---

### 1.4 Barcode Scanning & Product Search

**Requirement**: Kasir dapat scan barcode atau search produk secara manual.

**Features**:
- Scan barcode menggunakan barcode scanner
- Search produk by name/SKU dengan autocomplete
- Validasi barcode format (EAN-8, EAN-13, CODE128)
- Fuzzy search untuk error tolerance

**Entity**: `Product`
```typescript
interface Product {
  id: string;
  sku: string;
  barcode?: string;
  name: string;
  description?: string;
  category: string;
  price: number;
  cost: number;
  imageUrl?: string;
  isActive: boolean;
}
```

**API Endpoints**:
```
GET    /api/inventory/products/search
Query: { q: "Mie", branchId }
Response: [
  { id, sku, barcode, name, price, stock }
]

GET    /api/inventory/products/by-barcode/:barcode
Query: { branchId }
Response: { id, name, price, stock, category }

GET    /api/inventory/products/:id
Response: { id, sku, name, price, cost, category, images }
```

---

### 1.5 Cetak Struk via Printer Thermal

**Requirement**: Setelah transaksi selesai, sistem print struk ke printer thermal.

**Konten Struk**:
```
┌─────────────────────────────────┐
│          TOKO ABC               │
│   Jl. Merdeka No. 123           │
│      Jakarta 12345              │
├─────────────────────────────────┤
│ Tanggal: 01/06/2026 14:30:45   │
│ Kasir: Budi Santoso            │
│ No. Struk: 20260601001234      │
├─────────────────────────────────┤
│ DAFTAR PEMBELIAN                │
│ Item      Qty  Harga    Total   │
├─────────────────────────────────┤
│ Mie Instan 5x  @5.000 = 25.000 │
│ Teh Kotak  2x @10.000 = 20.000 │
│ Gula Pasir 1x @12.000 = 12.000 │
├─────────────────────────────────┤
│ Subtotal            : 57.000    │
│ Diskon (5%)         : 2.850     │
│ Pajak (10%)         : 5.415     │
├─────────────────────────────────┤
│ TOTAL               : 59.565    │
├─────────────────────────────────┤
│ Metode: QRIS                    │
│ Ref#: 20260601001234            │
├─────────────────────────────────┤
│    TERIMA KASIH                 │
│   Selamat berbelanja lagi!      │
└─────────────────────────────────┘
```

**API Endpoints**:
```
POST   /api/hardware/print-receipt/:transactionId
Response: { printed: true, timestamp }

POST   /api/hardware/reprint/:transactionId
Response: { printed: true, timestamp }
```

---

## 2. Customer & Loyalty Management (CRM)

### 2.1 Customer Profiling & Registration

**Requirement**: Kasir dapat registrasi pelanggan baru atau search pelanggan existing.

**Entity**: `Customer`
```typescript
interface Customer {
  id: string;
  email?: string;
  phone: string;
  name: string;
  loyaltyTier: 'STANDARD' | 'GOLD' | 'PLATINUM';
  loyaltyPoints: number;
  address?: string;
  city?: string;
  dateOfBirth?: Date;
  membershipDate: Date;
  isActive: boolean;
  lastPurchaseDate?: Date;
}
```

**API Endpoints**:
```
GET    /api/customers/search
Query: { q: "Budi", limit: 10 }
Response: [{ id, name, phone, loyaltyTier, loyaltyPoints }]

POST   /api/customers
Body: {
  name: "Budi Santoso",
  phone: "08123456789",
  email?: "budi@email.com",
  address?: "Jl. Merdeka"
}
Response: { customerId, createdAt }

GET    /api/customers/:id
Response: { id, name, phone, loyaltyPoints, membershipDate, ... }

PUT    /api/customers/:id
Body: { phone?, email?, address? }
Response: { updated: true }
```

---

### 2.2 Loyalty Points & Rewards System

**Requirement**: Otomatisasi poin belanja, bisa ditukar jadi diskon.

**Perhitungan Poin**:
- Base: 1 poin per Rp 10.000 (configurable)
- Multiplier:
  - STANDARD: 1x
  - GOLD: 1.5x (15% lebih banyak poin)
  - PLATINUM: 2x (2x lipat poin)

**Contoh**:
- Belanja Rp 100.000 (tier STANDARD) → 10 poin
- Belanja Rp 100.000 (tier GOLD) → 15 poin
- Belanja Rp 100.000 (tier PLATINUM) → 20 poin

**Redemption**:
- 1000 poin = Rp 10.000 diskon
- Contoh: 50.000 poin = Rp 500.000 diskon (bisa dipakai untuk transaksi berikutnya)

**Entity**: `LoyaltyTransaction`
```typescript
interface LoyaltyTransaction {
  id: string;
  customerId: string;
  transactionId?: string;
  points: number;  // Positive=earn, Negative=redeem
  type: 'EARN' | 'REDEEM';
  amount?: number;  // Rupiah value
  description: string;
  createdAt: Date;
}
```

**API Endpoints**:
```
POST   /api/customers/:id/loyalty/earn
Body: {
  transactionId,
  amount: 100000
}
Response: { pointsEarned: 10, newBalance: 150 }

POST   /api/customers/:id/loyalty/redeem
Body: {
  points: 50000
}
Response: { 
  discountAmount: 500000,
  newBalance: 0,
  applied: true
}

GET    /api/customers/:id/loyalty/history
Query: { limit: 10, offset: 0 }
Response: [
  { type: 'EARN', points: 10, description: "Purchase #123", date },
  { type: 'REDEEM', points: -50000, description: "Redemption", date }
]
```

---

### 2.3 Dynamic Pricelists (Per-Customer Discounts)

**Requirement**: Harga berbeda untuk tipe pelanggan berbeda (retail vs wholesale).

**Entity**: `CustomerGroup`
```typescript
interface CustomerGroup {
  id: string;
  name: 'RETAIL' | 'WHOLESALE' | 'CORPORATE';
  discountPercentage: number;  // 0%, 5%, 10%, etc
}
```

**Alur**:
1. Kasir input/search customer
2. Sistem ambil customer group (RETAIL/WHOLESALE)
3. System apply discount otomatis pada setiap item

**Contoh**:
```
Product: Mie Instan
- Base Price: Rp 5.000
- Customer: Budi (RETAIL)
  → Harga: Rp 5.000 (0% discount)
- Customer: Toko Kelontong (WHOLESALE, 20% discount)
  → Harga: Rp 4.000
```

**API Endpoints**:
```
GET    /api/customers/groups
Response: [
  { id, name: 'RETAIL', discountPercentage: 0 },
  { id, name: 'WHOLESALE', discountPercentage: 20 },
  { id, name: 'CORPORATE', discountPercentage: 25 }
]

GET    /api/customers/:id/pricing
Query: { branchId }
Response: {
  customerId,
  group: 'WHOLESALE',
  discountPercentage: 20,
  applicableProducts: [...]
}
```

---

## 3. Promotions & Diskon Pintar

### 3.1 Conditional Discounts

**Jenis Diskon**:

1. **BOGO** (Buy One Get One)
   ```
   Beli 2 gratis 1
   Atau: Beli 3 ambil harga 2 termurah
   ```

2. **Percentage Discount**
   ```
   Diskon 10% untuk semua produk kategori "Minuman"
   Atau: Diskon 5% untuk total belanja > Rp 100.000
   ```

3. **Fixed Amount**
   ```
   Diskon Rp 50.000 untuk pembelian > Rp 500.000
   ```

4. **Minimum Purchase**
   ```
   Beli 5 item atau lebih → 15% off
   ```

**Entity**: `Promotion`
```typescript
interface Promotion {
  id: string;
  name: string;
  type: 'PERCENTAGE' | 'FIXED_AMOUNT' | 'BOGO' | 'MARKDOWN';
  value: number;
  minimumPurchase?: number;
  minimumQuantity?: number;
  startDate: Date;
  endDate: Date;
  applicableCategories?: string[];
  isActive: boolean;
}
```

**API Endpoints**:
```
GET    /api/promotions/active
Query: { branchId, date }
Response: [
  {
    id,
    name: "Summer Sale",
    type: "PERCENTAGE",
    value: 20,
    minimumPurchase: 100000
  }
]

POST   /api/promotions/calculate
Body: {
  items: [...],
  customerId?,
  branchId
}
Response: {
  subtotal: 200000,
  discountAmount: 20000,  // Calculated by engine
  applicablePromotions: ["Promo A", "Promo B"]
}
```

---

### 3.2 Automated Price Markdown

**Requirement**: Harga otomatis turun untuk barang yang hampir expired atau jam-jam tertentu.

**Use Cases**:

1. **Happy Hour Markdown**
   ```
   Setiap hari 14:00-16:00 (2 jam):
   - Produk kategori "Makanan Ringan" diskon 30%
   ```

2. **Expired Date Markdown**
   ```
   - 7 hari sebelum expire: -10%
   - 3 hari sebelum expire: -30%
   - 1 hari sebelum expire: -50%
   ```

3. **Slow Moving Products**
   ```
   Produk yang tidak laku > 30 hari: -25%
   ```

**Entity**: `PriceMarkdown`
```typescript
interface PriceMarkdown {
  id: string;
  productId?: string;
  categoryId?: string;
  type: 'HAPPY_HOUR' | 'EXPIRY_BASED' | 'SLOW_MOVING';
  discountPercentage: number;
  startTime?: string;  // "14:00" untuk happy hour
  endTime?: string;    // "16:00"
  expiryDaysThreshold?: number;  // 7, 3, 1
  isActive: boolean;
}
```

**API Endpoints**:
```
POST   /api/promotions/markdowns
Body: {
  type: "HAPPY_HOUR",
  categoryId: "cat-123",
  discountPercentage: 30,
  startTime: "14:00",
  endTime: "16:00"
}
Response: { markdownId, createdAt }

GET    /api/promotions/markdowns/active
Query: { branchId }
Response: [{ id, type, discountPercentage, ... }]
```

---

## 4. Multi-Shop & Multi-Warehouse Management

### 4.1 Multi-Branch Support

**Requirement**: Satu sistem dapat mengelola banyak cabang toko.

**Entity**: `Branch`
```typescript
interface Branch {
  id: string;
  name: string;
  location: string;
  address: string;
  phone: string;
  openingTime: string;  // "08:00"
  closingTime: string;  // "21:00"
  isActive: boolean;
}
```

**API Endpoints**:
```
GET    /api/branches
Response: [
  { id, name: "Toko Pusat", location: "Jakarta", ... },
  { id, name: "Toko Cabang Bandung", location: "Bandung", ... }
]

GET    /api/branches/:id
Response: { id, name, location, ... }
```

---

### 4.2 Real-time Stock Checking

**Requirement**: Kasir di Toko A bisa cek stok di Toko B atau Gudang Pusat secara real-time.

**Use Case**:
1. Pelanggan ingin produk X
2. Toko A habis, tapi Toko B ada 5 unit
3. Kasir bisa transfer dari Toko B ke Toko A
4. Atau pelanggan disarankan ke Toko B

**Entity**: `Stock`
```typescript
interface Stock {
  id: string;
  productId: string;
  branchId: string;
  quantity: number;
  reservedQuantity: number;
  availableQuantity: number;  // quantity - reserved
  expiryDate?: Date;
}
```

**API Endpoints**:
```
GET    /api/inventory/stock/by-product/:productId
Response: [
  { branchId: "branch-1", name: "Toko Pusat", quantity: 10 },
  { branchId: "branch-2", name: "Bandung", quantity: 5 },
  { branchId: "gudang", name: "Gudang Pusat", quantity: 50 }
]

GET    /api/inventory/stock/available
Query: { productId, exclude_branchId }
Response: [
  { branchId, branchName, availableQuantity, transferCost? }
]
```

---

### 4.3 Cross-Store Stock Transfer

**Requirement**: Bisa transfer stok dari satu cabang ke cabang lain.

**Entity**: `StockTransfer`
```typescript
interface StockTransfer {
  id: string;
  fromBranchId: string;
  toBranchId: string;
  productId: string;
  quantity: number;
  status: 'PENDING' | 'IN_TRANSIT' | 'COMPLETED';
  createdAt: Date;
  completedAt?: Date;
  notes?: string;
}
```

**API Endpoints**:
```
POST   /api/inventory/transfers
Body: {
  fromBranchId: "branch-1",
  toBranchId: "branch-2",
  productId: "prod-123",
  quantity: 5
}
Response: { transferId, status: 'PENDING' }

GET    /api/inventory/transfers/:id
Response: { id, fromBranch, toBranch, product, quantity, status }

PUT    /api/inventory/transfers/:id
Body: { status: 'COMPLETED' }
Response: { status: 'COMPLETED', completedAt }
```

---

## 5. Booking System

### 5.1 Product Reservation & Pre-order

**Requirement**: Pelanggan dapat pre-order/reserve produk untuk pickup nanti.

**Use Cases**:
- Pelanggan mau pesan barang tapi mau diambil 3 hari lagi
- Barang ready sesuai jadwal yang dijanjikan
- Sistem send reminder 1 hari sebelum pickup

**Entity**: `Booking`
```typescript
interface Booking {
  id: string;
  customerId: string;
  productId: string;
  branchId: string;
  quantity: number;
  bookingDate: Date;  // Tanggal mau ambil
  bookingTime: string;  // "14:00"
  status: 'PENDING' | 'CONFIRMED' | 'COMPLETED' | 'CANCELLED';
  notes?: string;
  createdAt: Date;
  completedAt?: Date;
}
```

**API Endpoints**:
```
POST   /api/bookings
Body: {
  customerId: "cust-123",
  productId: "prod-456",
  quantity: 2,
  bookingDate: "2026-06-10",
  bookingTime: "14:00",
  notes: "Booking untuk resepsi"
}
Response: { bookingId, status: 'PENDING', confirmationCode: "BK-001" }

GET    /api/bookings/:id
Response: { id, customer, product, quantity, bookingDate, status }

PUT    /api/bookings/:id
Body: { status: 'CONFIRMED' | 'COMPLETED' | 'CANCELLED' }
Response: { status: 'CONFIRMED' }

DELETE /api/bookings/:id
Response: { cancelled: true }
```

---

### 5.2 Booking Calendar & Schedule

**Requirement**: Sistem tunjukkan slot available untuk booking.

**API Endpoints**:
```
GET    /api/bookings/available-slots
Query: { productId, date, branchId }
Response: [
  { time: "08:00", available: true },
  { time: "09:00", available: true },
  { time: "10:00", available: false },  // Sudah full
  { time: "11:00", available: true }
]

GET    /api/bookings/calendar
Query: { productId, year, month }
Response: {
  2026: {
    6: {
      1: { bookings: 0, available: 5 },
      2: { bookings: 2, available: 3 },
      ...
    }
  }
}
```

---

### 5.3 Booking Reminders & Notifications

**Requirement**: Sistem kirim reminder ke pelanggan.

**Timing**:
- 1 hari sebelum booking date: SMS/notifikasi "Booking Anda besok jam 14:00"
- Hari H pukul 08:00: Reminder "Jangan lupa booking Anda jam 14:00"

**API Endpoints**:
```
GET    /api/bookings/reminders
Query: { status: 'PENDING' }
Response: [
  {
    bookingId,
    customerPhone,
    customerEmail,
    message: "Booking Anda besok...",
    scheduleTime: "2026-06-09T14:00:00Z"
  }
]

POST   /api/bookings/send-reminder/:id
Response: { sent: true, method: 'sms', timestamp }
```

---

## 6. Analytics & Reporting

### 6.1 Product Performance Analysis

**Metrics**:
- **Fast Moving**: Produk yang terjual cepat (top 20)
- **Slow Moving**: Produk yang jarang terjual (bottom 20)
- **Revenue Contribution**: Kontribusi revenue per produk

**API Endpoints**:
```
GET    /api/reports/products/performance
Query: { branchId, startDate, endDate }
Response: [
  {
    productId,
    name,
    category,
    unitsSold: 150,
    revenue: 750000,
    averagePrice: 5000,
    rank: 'FAST_MOVING'
  }
]

GET    /api/reports/products/slow-moving
Query: { branchId, days: 30 }
Response: [
  {
    productId,
    name,
    unitsSold: 2,
    lastSaleDate: "2026-05-20",
    suggestedAction: "MARKDOWN" | "DISCONTINUE"
  }
]
```

---

### 6.2 Salesperson Performance Analytics

**Metrics**:
- Total transactions
- Total revenue
- Average transaction value
- Customer satisfaction
- Peak hours
- Commission calculation

**API Endpoints**:
```
GET    /api/reports/staff/performance
Query: { branchId, startDate, endDate }
Response: [
  {
    staffId,
    staffName,
    totalTransactions: 250,
    totalRevenue: 5000000,
    averageTransactionValue: 20000,
    rank: 1,
    commission: 500000  // 10% of revenue
  }
]

GET    /api/reports/staff/:staffId/detail
Query: { startDate, endDate }
Response: {
  staffId,
  name,
  transactions: [...],
  topProducts: [...],
  peakHours: [9, 13, 17],
  customerRepeat: 25  // 25% repeat customers
}
```

---

### 6.3 Sales & Revenue Reports

**API Endpoints**:
```
GET    /api/reports/sales/daily
Query: { branchId, date }
Response: {
  date,
  totalTransactions: 120,
  totalRevenue: 2400000,
  discountAmount: 120000,
  taxAmount: 240000,
  paymentMethods: {
    cash: 1200000,
    qris: 1000000,
    card: 200000
  },
  topProducts: [...]
}

GET    /api/reports/sales/monthly
Query: { branchId, year, month }
Response: {
  period: "June 2026",
  totalRevenue: 72000000,
  totalTransactions: 3600,
  averageTransactionValue: 20000,
  dailyBreakdown: [...]
}

GET    /api/reports/sales/export
Query: { format: 'pdf' | 'excel', startDate, endDate, branchId }
Response: Binary file (PDF/Excel)
```

---

### 6.4 Inventory & Stock Analysis

**API Endpoints**:
```
GET    /api/reports/inventory/stock-level
Query: { branchId }
Response: [
  {
    productId,
    name,
    currentStock: 50,
    minimumStock: 20,
    status: 'HEALTHY' | 'LOW' | 'CRITICAL',
    reorderQuantity: 100
  }
]

GET    /api/reports/inventory/aging
Query: { branchId, days: 60 }
Response: [
  {
    productId,
    name,
    stock: 5,
    daysSinceLastSale: 45,
    action: 'MARKDOWN' | 'MONITOR' | 'NONE'
  }
]
```

---

## 7. Hardware Features

### 7.1 Printer Thermal

- Print struk otomatis setelah transaksi
- Reprint struk dari history
- Print barcode untuk inventory
- Print laporan per shift

### 7.2 Barcode Scanner

- Scan barcode untuk cepat input produk
- Validasi format barcode
- Support multiple format (EAN, CODE128, etc)

### 7.3 Cash Drawer

- Buka drawer setelah pembayaran cash
- Monitor status drawer (open/close)
- Safety control untuk deteksi fraud

### 7.4 QRIS Payment

- Generate QR code untuk QRIS payment
- Monitor payment status
- Support e-wallet (OVO, GoPay, DANA, LinkAja)

---

## Summary Feature Matrix

| Feature | Frontend | Backend | Hardware | Priority |
|---------|----------|---------|----------|----------|
| Cash Control | ✅ | ✅ | ✅ Drawer | HIGH |
| Hold Orders | ✅ | ✅ | - | HIGH |
| Flexible Payment | ✅ | ✅ | ✅ Printer | HIGH |
| Barcode Scan | ✅ | ✅ | ✅ Scanner | HIGH |
| Customer CRM | ✅ | ✅ | - | MEDIUM |
| Loyalty Points | ✅ | ✅ | - | MEDIUM |
| Dynamic Pricing | ✅ | ✅ | - | MEDIUM |
| Promotions Engine | ✅ | ✅ | - | MEDIUM |
| Multi-Branch | ✅ | ✅ | - | MEDIUM |
| Booking System | ✅ | ✅ | - | LOW |
| Reports & Analytics | ✅ | ✅ | - | MEDIUM |

---

**Document Version**: 1.0  
**Last Updated**: June 1, 2026
