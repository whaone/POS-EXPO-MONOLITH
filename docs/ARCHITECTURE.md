# POS-EXPO-MONOLITH Architecture

Dokumentasi lengkap arsitektur Monolith untuk POS-EXPO-MONOLITH.

## 1. Pengantar Monolith Architecture

**Monolith** adalah arsitektur di mana seluruh aplikasi (frontend, backend, database) berjalan dalam satu proses/server tunggal.

**Keuntungan Monolith untuk tim/startup kecil**:
- ✅ Sederhana untuk development awal
- ✅ Deploy lebih mudah (satu Docker image)
- ✅ Performance overhead lebih rendah (no network latency between services)
- ✅ Debugging lebih mudah - single codebase
- ✅ Sharing code lebih mudah (shared types, utilities)
- ✅ Transaction ACID lebih reliable
- ✅ Cost-efficient untuk awal

**Tantangan Monolith**:
- ❌ Sulit untuk scaling horizontal
- ❌ Jika ada crash di satu fitur, bisa affect yang lain
- ❌ Deployment harus stop service seluruhnya
- ❌ Sulit scale resource per fitur

**Strategi: Layered Architecture dalam Monolith**
Untuk memudahkan migrasi ke microservices di masa depan, kami menggunakan **clear separation of concerns** dengan layer yang distinct:
- Presentation Layer (Frontend)
- API Layer (Routes & Controllers)
- Business Logic Layer (Services)
- Data Access Layer (ORM/Repository)
- Data Layer (Database & Cache)

---

## 2. Layered Architecture (Monolith)

```
┌───────────────────────────────────────────────────┐
│         PRESENTATION LAYER                         │
│  React SPA (Browser-based, Progressive Web App)   │
│  • Components, Pages, Hooks                        │
│  • Redux/Zustand State Management                  │
│  • IndexedDB (Offline Cache)                       │
└───────────────────────────────────────────────────┘
              ↓ (HTTP/REST + WebSocket)
┌───────────────────────────────────────────────────┐
│         MIDDLEWARE LAYER                           │
│  • Authentication (JWT verify)                     │
│  • Request Validation                              │
│  • Error Handling                                  │
│  • Logging & Rate Limiting                         │
└───────────────────────────────────────────────────┘
              ↓
┌───────────────────────────────────────────────────┐
│         API LAYER (Express Routes)                │
│  • POST /api/pos/transactions                      │
│  • GET /api/inventory/products                     │
│  • POST /api/customers                             │
│  etc...                                            │
└───────────────────────────────────────────────────┘
              ↓
┌───────────────────────────────────────────────────┐
│         BUSINESS LOGIC LAYER (Services)            │
│  • TransactionService                              │
│  • InventoryService                                │
│  • CustomerService                                 │
│  • PromotionService (Discount calculation)         │
│  • LoyaltyService                                  │
│  • BookingService                                  │
│  • No HTTP knowledge, pure business rules          │
│  • Easily testable                                 │
└───────────────────────────────────────────────────┘
              ↓
┌───────────────────────────────────────────────────┐
│         DATA ACCESS LAYER (ORM)                    │
│  • TypeORM/Prisma Repositories                     │
│  • Query building & optimization                   │
│  • Transaction management                          │
│  • Cache layer (Redis)                             │
└───────────────────────────────────────────────────┘
              ↓
┌───────────────────────────────────────────────────┐
│         DATA LAYER                                 │
│  • PostgreSQL (Primary DB)                         │
│  • Redis (Cache, Session Store)                    │
│  • File Storage (Receipts, Reports)                │
└───────────────────────────────────────────────────┘
```

---

## 3. Frontend Architecture

### 3.1 Technology Stack
- **Framework**: React 18 + TypeScript
- **State Management**: Redux Toolkit / Zustand
- **Offline Database**: IndexedDB + PouchDB
- **HTTP Client**: Axios + React Query
- **UI Framework**: Material-UI / Tailwind CSS
- **Build Tool**: Vite
- **Offline Support**: Service Worker + PWA

### 3.2 Directory Structure

```
src/frontend/
├── components/
│   ├── POS/                    # POS Kasir screens
│   │   ├── CashierScreen.tsx      # Main transaction screen
│   │   ├── ProductGrid.tsx        # Product list & search
│   │   ├── CartItems.tsx          # Cart display
│   │   ├── PaymentScreen.tsx      # Payment method selection
│   │   ├── BarcodeInput.tsx       # Barcode scanner input
│   │   └── HoldOrdersList.tsx     # Held/suspended orders
│   ├── Customers/              # Customer management
│   │   ├── CustomerSearch.tsx
│   │   ├── CustomerForm.tsx
│   │   └── LoyaltyDisplay.tsx
│   ├── Bookings/               # Booking system
│   │   ├── BookingCalendar.tsx
│   │   ├── BookingForm.tsx
│   │   └── BookingStatus.tsx
│   ├── Admin/                  # Admin dashboard
│   │   ├── Dashboard.tsx
│   │   ├── SalesChart.tsx
│   │   ├── ProductPerformance.tsx
│   │   └── StaffPerformance.tsx
│   └── Common/                 # Shared components
│       ├── Header.tsx
│       ├── Sidebar.tsx
│       ├── Modal.tsx
│       └── NotificationToast.tsx
├── pages/
│   ├── POS.tsx                 # POS page
│   ├── Inventory.tsx
│   ├── Dashboard.tsx
│   ├── Bookings.tsx
│   └── Settings.tsx
├── services/
│   ├── api.ts                  # Axios client
│   ├── offline-sync.ts         # IndexedDB sync logic
│   └── hardware.ts             # Hardware driver integration
├── store/                      # Redux state
│   ├── slices/
│   │   ├── auth.ts
│   │   ├── cart.ts
│   │   ├── customers.ts
│   │   ├── inventory.ts
│   │   ├── bookings.ts
│   │   ├── ui.ts
│   │   └── offline.ts
│   └── index.ts
├── hooks/
│   ├── useOfflineSync.ts
│   ├── useHardware.ts
│   └── useNavigation.ts
├── utils/
│   ├── formatters.ts           # Currency, date formatting
│   ├── validators.ts           # Input validation
│   ├── calculations.ts         # Discount, tax calculations
│   └── constants.ts            # App constants
├── db/
│   ├── schema.ts               # IndexedDB schema
│   ├── migrations.ts
│   └── seedData.ts
├── App.tsx
└── main.tsx
```

### 3.3 Offline-First Data Sync

**Key Concept**: Data is saved locally FIRST, then synced to backend

```typescript
// src/frontend/services/offline-sync.ts

export class OfflineSyncManager {
  private syncQueue: PendingAction[] = [];
  private isOnline: boolean = navigator.onLine;

  constructor(private db: IndexedDBService, private api: ApiClient) {
    window.addEventListener('online', () => this.handleOnline());
    window.addEventListener('offline', () => this.handleOffline());
  }

  // Main workflow: Local first
  async performAction(action: Action): Promise<void> {
    // 1. Save to local IndexedDB immediately
    await this.db.save(action);

    // 2. Update Redux state (UI responds instantly)
    store.dispatch(updateAction(action));

    // 3. Try to sync with backend
    if (this.isOnline) {
      try {
        await this.api.post('/sync', action);
        // Mark as synced
        await this.db.markSynced(action.id);
      } catch (error) {
        // If sync fails, add to queue for retry
        this.syncQueue.push(action);
      }
    } else {
      // If offline, add to queue
      this.syncQueue.push(action);
    }
  }

  // When online, flush queue
  private async handleOnline() {
    this.isOnline = true;
    console.log('Online - flushing sync queue');
    
    while (this.syncQueue.length > 0) {
      const action = this.syncQueue.shift()!;
      try {
        await this.api.post('/sync', action);
        await this.db.markSynced(action.id);
      } catch (error) {
        // Re-queue and stop trying
        this.syncQueue.unshift(action);
        console.error('Sync failed, will retry later', error);
        break;
      }
    }
  }

  private handleOffline() {
    this.isOnline = false;
    console.log('Offline mode - all changes saved locally');
  }
}
```

### 3.4 Hardware Integration (Frontend)

```typescript
// src/frontend/services/hardware.ts

export class HardwareManager {
  private scanner: BarcodeScanner;
  private printer: PrinterService;  // Communicates with backend
  private drawer: DrawerService;    // Communicates with backend

  async initialize() {
    // Scanner runs locally (keyboard emulation)
    this.scanner.initialize();
    this.scanner.onBarcode((barcode) => this.handleBarcodeScanned(barcode));

    // Printer & Drawer commands sent to backend via API
    // Backend handles actual hardware communication
  }

  private async handleBarcodeScanned(barcode: string) {
    // Look up product in local cache first
    const product = this.getProductFromCache(barcode);
    
    if (product) {
      // Add to cart (Redux state update)
      store.dispatch(addToCart(product));
      showNotification(`Added: ${product.name}`, 'success');
    } else {
      showNotification('Product not found', 'error');
    }
  }

  async printReceipt(transactionId: string) {
    // Send print command to backend
    await fetch(`/api/hardware/print-receipt/${transactionId}`, {
      method: 'POST'
    });
  }

  async openCashDrawer() {
    // Send drawer kick command to backend
    await fetch('/api/hardware/drawer/kick', {
      method: 'POST'
    });
  }
}
```

---

## 4. Backend Architecture

### 4.1 Request-Response Flow

```
HTTP Request (from browser)
    ↓
Express Router → match route
    ↓
Middleware Pipeline
  ├─ CORS & Security Headers
  ├─ Body Parser (JSON)
  ├─ Request Logging
  ├─ Authentication (JWT verify)
  ├─ Input Validation
  └─ Error Boundary
    ↓
Controller (extract & validate params)
    ↓
Service (business logic)
  ├─ Call other services
  ├─ Call Repository/ORM
  ├─ Perform calculations
  └─ Handle errors
    ↓
Repository/ORM (TypeORM/Prisma)
  ├─ Build query
  ├─ Execute query
  ├─ Manage transactions
  └─ Update cache
    ↓
Database Query
    ↓
Response → JSON
  ├─ 200 OK + Data
  ├─ 400 Bad Request
  ├─ 401 Unauthorized
  ├─ 404 Not Found
  └─ 500 Server Error
```

### 4.2 Service Layer (Core Business Logic)

```typescript
// src/backend/services/pos.service.ts

@Injectable()
export class PosService {
  constructor(
    private transactionRepo: Repository<Transaction>,
    private productRepo: Repository<Product>,
    private customerRepo: Repository<Customer>,
    private stockRepo: Repository<Stock>,
    private promotionService: PromotionService,
    private loyaltyService: LoyaltyService,
    private cacheService: CacheService,
    private logger: LoggerService
  ) {}

  async createTransaction(dto: CreateTransactionDto): Promise<Transaction> {
    // Validate items & check stock
    const items = await this.validateAndPrepareItems(dto.items);

    // Calculate discounts
    const discount = await this.promotionService.calculateDiscount(
      items,
      dto.customerId
    );

    // Create transaction
    const transaction = new Transaction();
    transaction.items = items;
    transaction.subtotal = items.reduce((sum, item) => 
      sum + item.subtotal, 0
    );
    transaction.discount = discount;
    transaction.tax = (transaction.subtotal - discount) * 0.1;
    transaction.total = transaction.subtotal - discount + transaction.tax;
    transaction.cashierId = dto.cashierId;
    transaction.branchId = dto.branchId;

    // Save to database
    await this.transactionRepo.save(transaction);

    // Award loyalty points
    if (dto.customerId) {
      const points = await this.loyaltyService.calculateAndAwardPoints(
        dto.customerId,
        transaction.total - discount
      );
      transaction.loyaltyPoints = points;
    }

    // Update stock
    for (const item of items) {
      await this.stockRepo.update(
        { productId: item.productId, branchId: dto.branchId },
        { quantity: () => 'quantity - ' + item.quantity }
      );
    }

    // Invalidate cache
    await this.cacheService.invalidate('transactions:' + dto.branchId);

    // Broadcast via WebSocket
    this.eventEmitter.emit('transaction:created', transaction);

    return transaction;
  }

  async holdOrder(transactionId: string): Promise<void> {
    const transaction = await this.transactionRepo.findOneOrFail({
      where: { id: transactionId }
    });

    transaction.status = 'on_hold';
    transaction.heldAt = new Date();

    await this.transactionRepo.save(transaction);
    this.eventEmitter.emit('order:held', transaction);
  }

  async resumeOrder(transactionId: string): Promise<Transaction> {
    const transaction = await this.transactionRepo.findOneOrFail({
      where: { id: transactionId }
    });

    if (transaction.status !== 'on_hold') {
      throw new BadRequestException('Order is not on hold');
    }

    transaction.status = 'pending';
    transaction.heldAt = null;

    return await this.transactionRepo.save(transaction);
  }

  async closeCashRegister(branchId: string, cashierId: string): Promise<CashRegister> {
    // Get all transactions for this cashier
    const transactions = await this.transactionRepo.find({
      where: {
        branchId,
        cashierId,
        createdAt: MoreThan(new Date(Date.now() - 24 * 60 * 60 * 1000))
      }
    });

    // Calculate expected total
    const expectedTotal = transactions.reduce((sum, t) => 
      sum + t.totalAmount, 0
    );

    // Create cash register closing record
    const cashRegister = new CashRegister();
    cashRegister.branchId = branchId;
    cashRegister.cashierId = cashierId;
    cashRegister.expectedTotal = expectedTotal;
    cashRegister.status = 'closed';
    cashRegister.closedAt = new Date();

    return await this.cashRegisterRepo.save(cashRegister);
  }
}
```

### 4.3 Promotion Engine (Discount Calculation)

```typescript
// src/backend/services/promotions.service.ts

@Injectable()
export class PromotionService {
  constructor(
    private promotionRepo: Repository<Promotion>,
    private cacheService: CacheService
  ) {}

  async calculateDiscount(
    items: TransactionItem[],
    customerId?: string,
    branchId?: string
  ): Promise<number> {
    // Get active promotions for branch
    const cacheKey = `promotions:${branchId}`;
    let promotions = await this.cacheService.get<Promotion[]>(cacheKey);
    
    if (!promotions) {
      promotions = await this.promotionRepo.find({
        where: {
          isActive: true,
          startDate: LessThanOrEqual(new Date()),
          endDate: MoreThanOrEqual(new Date())
        }
      });
      await this.cacheService.set(cacheKey, promotions, 3600); // 1 hour TTL
    }

    let totalDiscount = 0;

    for (const promo of promotions) {
      switch (promo.type) {
        case 'PERCENTAGE':
          // e.g., 10% off entire purchase
          totalDiscount += this.calculatePercentageDiscount(items, promo);
          break;

        case 'FIXED_AMOUNT':
          // e.g., Rp50,000 off if purchase > Rp500,000
          if (this.meetsMinimumRequirement(items, promo)) {
            totalDiscount += promo.value;
          }
          break;

        case 'BOGO':
          // Buy 2 get 1 free
          totalDiscount += this.calculateBogoDiscount(items, promo);
          break;

        case 'MARKDOWN':
          // Discount on specific items (e.g., expired soon)
          totalDiscount += this.calculateMarkdownDiscount(items, promo);
          break;
      }
    }

    // Apply customer group discount (if applicable)
    if (customerId) {
      const customerDiscount = await this.getCustomerGroupDiscount(customerId);
      totalDiscount += customerDiscount;
    }

    return Math.min(totalDiscount, this.calculateSubtotal(items));
  }

  private calculateBogoDiscount(items: TransactionItem[], promo: Promotion): number {
    // Find eligible items (quantity >= 2)
    const eligibleItems = items.filter(item => item.quantity >= 2);
    
    let discount = 0;
    for (const item of eligibleItems) {
      // For every 2 bought, get 1 free
      const freeCount = Math.floor(item.quantity / 3);
      discount += item.unitPrice * freeCount;
    }

    return discount;
  }

  private meetsMinimumRequirement(items: TransactionItem[], promo: Promotion): boolean {
    const subtotal = items.reduce((sum, item) => sum + item.subtotal, 0);
    return subtotal >= (promo.minimumPurchase || 0);
  }
}
```

### 4.4 Loyalty Program Service

```typescript
// src/backend/services/loyalty.service.ts

@Injectable()
export class LoyaltyService {
  constructor(
    private customerRepo: Repository<Customer>,
    private loyaltyTransactionRepo: Repository<LoyaltyTransaction>
  ) {}

  async calculateAndAwardPoints(customerId: string, amount: number): Promise<number> {
    const customer = await this.customerRepo.findOne({
      where: { id: customerId }
    });

    if (!customer) throw new NotFoundException('Customer not found');

    // Base points: 1 point per 10,000 rupiah
    let points = Math.floor(amount / 10000);

    // Multiplier based on customer tier
    if (customer.loyaltyTier === 'GOLD') points *= 1.5;
    if (customer.loyaltyTier === 'PLATINUM') points *= 2;

    // Award points
    customer.loyaltyPoints += points;
    await this.customerRepo.save(customer);

    // Record transaction
    await this.loyaltyTransactionRepo.save({
      customer,
      points,
      type: 'EARN',
      description: 'Purchase'
    });

    return points;
  }

  async redeemPoints(customerId: string, points: number): Promise<number> {
    const customer = await this.customerRepo.findOne({
      where: { id: customerId }
    });

    if (!customer) throw new NotFoundException('Customer not found');
    if (customer.loyaltyPoints < points) {
      throw new BadRequestException('Insufficient loyalty points');
    }

    // 1000 points = Rp 10,000 discount
    const discount = (points / 1000) * 10000;

    customer.loyaltyPoints -= points;
    await this.customerRepo.save(customer);

    // Record transaction
    await this.loyaltyTransactionRepo.save({
      customer,
      points: -points,
      type: 'REDEEM',
      description: 'Redemption for discount'
    });

    return discount;
  }
}
```

### 4.5 Booking Service

```typescript
// src/backend/services/bookings.service.ts

@Injectable()
export class BookingsService {
  constructor(
    private bookingRepo: Repository<Booking>,
    private productRepo: Repository<Product>,
    private stockRepo: Repository<Stock>,
    private customerRepo: Repository<Customer>,
    private notificationService: NotificationService
  ) {}

  async createBooking(dto: CreateBookingDto): Promise<Booking> {
    // Validate product exists
    const product = await this.productRepo.findOne({
      where: { id: dto.productId }
    });
    if (!product) throw new NotFoundException('Product not found');

    // Check available stock
    const stock = await this.stockRepo.findOne({
      where: {
        productId: dto.productId,
        branchId: dto.branchId
      }
    });

    if (!stock || stock.quantity - stock.reservedQuantity < dto.quantity) {
      throw new BadRequestException('Insufficient stock available');
    }

    // Create booking
    const booking = new Booking();
    booking.customer = await this.customerRepo.findOne({
      where: { id: dto.customerId }
    });
    booking.product = product;
    booking.quantity = dto.quantity;
    booking.bookingDate = dto.bookingDate;
    booking.bookingTime = dto.bookingTime;
    booking.status = 'pending';

    await this.bookingRepo.save(booking);

    // Reserve stock
    stock.reservedQuantity += dto.quantity;
    await this.stockRepo.save(stock);

    // Send notification
    await this.notificationService.sendBookingConfirmation(booking);

    return booking;
  }

  async completeBooking(bookingId: string): Promise<void> {
    const booking = await this.bookingRepo.findOne({
      where: { id: bookingId },
      relations: ['product', 'customer']
    });

    if (!booking) throw new NotFoundException('Booking not found');

    // Update status
    booking.status = 'completed';
    await this.bookingRepo.save(booking);

    // Release reserved stock
    const stock = await this.stockRepo.findOne({
      where: {
        productId: booking.product.id,
        branchId: booking.branchId
      }
    });

    if (stock) {
      stock.reservedQuantity = Math.max(0, stock.reservedQuantity - booking.quantity);
      stock.quantity -= booking.quantity;
      await this.stockRepo.save(stock);
    }

    // Send notification
    await this.notificationService.sendBookingCompleted(booking);
  }

  async getAvailableSlots(productId: string, date: Date): Promise<string[]> {
    // Return available time slots for a date
    const bookings = await this.bookingRepo.find({
      where: {
        productId,
        bookingDate: date,
        status: Not('cancelled')
      }
    });

    const allSlots = this.generateTimeSlots('08:00', '17:00', 30); // 30-min slots
    const bookedSlots = bookings.map(b => b.bookingTime);

    return allSlots.filter(slot => !bookedSlots.includes(slot));
  }

  private generateTimeSlots(start: string, end: string, interval: number): string[] {
    // Generate array of time slots
    const slots: string[] = [];
    // ... implementation
    return slots;
  }
}
```

### 4.6 Controller Layer

```typescript
// src/backend/controllers/pos.controller.ts

@Controller('api/pos')
export class PosController {
  constructor(
    private posService: PosService,
    private hardwareService: HardwareService
  ) {}

  @Post('transactions')
  @UseGuards(AuthGuard)
  async createTransaction(
    @Body() dto: CreateTransactionDto,
    @Req() req: Request
  ) {
    const transaction = await this.posService.createTransaction({
      ...dto,
      cashierId: req.user.id,
      branchId: req.user.branchId
    });

    return {
      statusCode: 201,
      message: 'Transaction created successfully',
      data: transaction
    };
  }

  @Post('transactions/:id/hold')
  @UseGuards(AuthGuard)
  async holdOrder(@Param('id') id: string) {
    await this.posService.holdOrder(id);

    return {
      statusCode: 200,
      message: 'Order placed on hold'
    };
  }

  @Get('hold-orders')
  @UseGuards(AuthGuard)
  async getHoldOrders(@Req() req: Request) {
    const orders = await this.posService.getHoldOrders(req.user.branchId);

    return {
      statusCode: 200,
      data: orders
    };
  }

  @Post('cash-register/close')
  @UseGuards(AuthGuard)
  async closeCashRegister(@Req() req: Request) {
    const result = await this.posService.closeCashRegister(
      req.user.branchId,
      req.user.id
    );

    return {
      statusCode: 200,
      message: 'Cash register closed',
      data: result
    };
  }

  @Post('hardware/print-receipt/:transactionId')
  @UseGuards(AuthGuard)
  async printReceipt(@Param('transactionId') transactionId: string) {
    await this.hardwareService.printReceipt(transactionId);

    return {
      statusCode: 200,
      message: 'Receipt printed'
    };
  }

  @Post('hardware/drawer/kick')
  @UseGuards(AuthGuard)
  async kickCashDrawer() {
    await this.hardwareService.kickDrawer();

    return {
      statusCode: 200,
      message: 'Cash drawer opened'
    };
  }
}
```

---

## 5. Database Design

### 5.1 Core Tables

```sql
-- Users & Branches
CREATE TABLE branches (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name VARCHAR(255) NOT NULL,
  location VARCHAR(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  name VARCHAR(255) NOT NULL,
  role ENUM('admin', 'manager', 'cashier') NOT NULL,
  branch_id UUID NOT NULL REFERENCES branches(id),
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Products & Inventory
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  sku VARCHAR(100) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  category VARCHAR(100),
  price DECIMAL(12, 2) NOT NULL,
  cost DECIMAL(12, 2),
  barcode VARCHAR(50),
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE stock (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  product_id UUID NOT NULL REFERENCES products(id),
  branch_id UUID NOT NULL REFERENCES branches(id),
  quantity INT NOT NULL DEFAULT 0,
  reserved_quantity INT DEFAULT 0,
  expiry_date DATE,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(product_id, branch_id)
);

-- Transactions
CREATE TABLE transactions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  branch_id UUID NOT NULL REFERENCES branches(id),
  cashier_id UUID NOT NULL REFERENCES users(id),
  customer_id UUID REFERENCES customers(id),
  transaction_number VARCHAR(50) UNIQUE NOT NULL,
  subtotal DECIMAL(12, 2) NOT NULL,
  discount_amount DECIMAL(12, 2) DEFAULT 0,
  tax_amount DECIMAL(12, 2) DEFAULT 0,
  total_amount DECIMAL(12, 2) NOT NULL,
  payment_method VARCHAR(50),
  status ENUM('completed', 'pending', 'cancelled', 'on_hold') DEFAULT 'completed',
  held_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_branch_id (branch_id),
  INDEX idx_created_at (created_at)
);

CREATE TABLE transaction_items (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  transaction_id UUID NOT NULL REFERENCES transactions(id) ON DELETE CASCADE,
  product_id UUID NOT NULL REFERENCES products(id),
  quantity INT NOT NULL,
  unit_price DECIMAL(12, 2) NOT NULL,
  discount_amount DECIMAL(12, 2) DEFAULT 0,
  subtotal DECIMAL(12, 2) NOT NULL
);

-- Customers & Loyalty
CREATE TABLE customers (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email VARCHAR(255),
  phone VARCHAR(20),
  name VARCHAR(255) NOT NULL,
  loyalty_tier VARCHAR(50) DEFAULT 'STANDARD',
  loyalty_points INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE loyalty_transactions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  customer_id UUID NOT NULL REFERENCES customers(id),
  transaction_id UUID REFERENCES transactions(id),
  points INT NOT NULL,
  type ENUM('earn', 'redeem') NOT NULL,
  description VARCHAR(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bookings
CREATE TABLE bookings (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  customer_id UUID NOT NULL REFERENCES customers(id),
  product_id UUID NOT NULL REFERENCES products(id),
  branch_id UUID NOT NULL REFERENCES branches(id),
  quantity INT NOT NULL,
  booking_date DATE NOT NULL,
  booking_time TIME NOT NULL,
  status ENUM('pending', 'confirmed', 'completed', 'cancelled') DEFAULT 'pending',
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_booking_date (booking_date),
  INDEX idx_customer_id (customer_id)
);

-- Promotions
CREATE TABLE promotions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name VARCHAR(255) NOT NULL,
  type ENUM('percentage', 'fixed_amount', 'bogo', 'markdown') NOT NULL,
  value DECIMAL(12, 2) NOT NULL,
  minimum_purchase DECIMAL(12, 2),
  start_date TIMESTAMP,
  end_date TIMESTAMP,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Cash Register
CREATE TABLE cash_registers (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  branch_id UUID NOT NULL REFERENCES branches(id),
  cashier_id UUID NOT NULL REFERENCES users(id),
  opening_balance DECIMAL(12, 2) NOT NULL,
  closing_balance DECIMAL(12, 2),
  expected_total DECIMAL(12, 2),
  actual_total DECIMAL(12, 2),
  variance DECIMAL(12, 2),
  status ENUM('open', 'closed') DEFAULT 'open',
  opened_at TIMESTAMP NOT NULL,
  closed_at TIMESTAMP
);
```

---

## 6. API Endpoints

### 6.1 Authentication
```
POST   /api/auth/login              User login
POST   /api/auth/logout             User logout
POST   /api/auth/refresh            Refresh JWT token
```

### 6.2 POS Transactions
```
POST   /api/pos/transactions        Create transaction
GET    /api/pos/transactions/:id    Get transaction
POST   /api/pos/transactions/:id/hold    Hold order
POST   /api/pos/transactions/:id/resume  Resume order
GET    /api/pos/hold-orders         Get all held orders
POST   /api/pos/cash-register/close Close cash register
```

### 6.3 Inventory
```
GET    /api/inventory/products      Get products
GET    /api/inventory/stock         Get stock levels
POST   /api/inventory/transfers     Create stock transfer
```

### 6.4 Customers
```
GET    /api/customers               List customers
POST   /api/customers               Create customer
GET    /api/customers/:id/loyalty   Get loyalty info
POST   /api/customers/:id/loyalty/redeem  Redeem points
```

### 6.5 Bookings
```
GET    /api/bookings                List bookings
POST   /api/bookings                Create booking
GET    /api/bookings/available-slots Get available slots
POST   /api/bookings/:id/complete   Complete booking
DELETE /api/bookings/:id            Cancel booking
```

### 6.6 Reports
```
GET    /api/reports/sales           Sales report
GET    /api/reports/products        Product performance
GET    /api/reports/staff           Staff performance
GET    /api/reports/export          Export report
```

### 6.7 Hardware
```
POST   /api/hardware/print-receipt/:id  Print receipt
POST   /api/hardware/drawer/kick        Open cash drawer
GET    /api/hardware/status             Get hardware status
```

---

## 7. Real-time Updates (WebSocket)

```typescript
// src/backend/websocket/setup.ts
import { Server as SocketIOServer } from 'socket.io';

export function setupWebSocket(httpServer: any) {
  const io = new SocketIOServer(httpServer, {
    cors: { origin: '*' }
  });

  io.on('connection', (socket) => {
    // Join branch namespace
    socket.on('join-branch', (branchId: string) => {
      socket.join(`branch-${branchId}`);
    });
  });

  return io;
}

// Broadcast from services
export class NotificationService {
  constructor(private io: SocketIOServer) {}

  broadcastStockUpdate(branchId: string, stockData: any) {
    this.io.to(`branch-${branchId}`).emit('inventory:stock-updated', stockData);
  }

  broadcastNewTransaction(branchId: string, transaction: any) {
    this.io.to(`branch-${branchId}`).emit('pos:transaction-created', transaction);
  }

  broadcastBookingUpdate(branchId: string, booking: any) {
    this.io.to(`branch-${branchId}`).emit('booking:status-changed', booking);
  }
}
```

---

## 8. Deployment

### 8.1 Single Dockerfile (Monolith)

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy all code
COPY . .

# Install dependencies
RUN npm ci

# Build frontend
RUN npm run build:frontend

# Build backend (compile TypeScript)
RUN npm run build:backend

# Expose port
EXPOSE 3000

# Start server (serves both frontend + backend)
CMD ["npm", "start"]
```

### 8.2 Docker Compose (Development)

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://user:pass@postgres:5432/pos_db
      REDIS_URL: redis://redis:6379
      NODE_ENV: development
      JWT_SECRET: dev-secret-key
    depends_on:
      - postgres
      - redis
    volumes:
      - .:/app
      - /app/node_modules

  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: pos_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

---

## 9. Performance Optimization

### Frontend
- Code splitting & lazy loading
- Service Worker for asset caching
- IndexedDB for offline data
- Compression (gzip/brotli)
- Image optimization

### Backend
- Database indexes on frequently queried columns
- Redis caching layer
- Query optimization (select specific columns)
- Connection pooling
- Request rate limiting

### Monitoring
```
- API response times (p50, p95, p99)
- Database query latency
- Cache hit ratio
- Offline sync queue depth
- Transaction throughput
```

---

**Document Version**: 1.0  
**Last Updated**: June 1, 2026  
**Architecture Pattern**: Monolith with Layered Architecture
