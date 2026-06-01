# POS-EXPO-MONOLITH

Sistem Point of Sale (POS) Terintegrasi untuk Retail dengan Fitur Enterprise-Grade (Monolith Architecture)

## 📋 Overview

POS-EXPO-MONOLITH adalah platform POS modern dengan **arsitektur Monolith** (single codebase, tapi dengan separasi layer yang jelas) yang menggabungkan:
- **Full-Stack**: Frontend, Backend, dan Database dalam satu project (monorepo)
- **Front-End**: Aplikasi kasir berbasis web/PWA dengan dukungan offline
- **Back-End**: API dan business logic dalam satu server
- **Hardware Integration**: Koneksi dengan printer thermal, barcode scanner
- **Fitur Enterprise**: Multi-branch, inventory real-time, CRM, loyalty program, booking system

## 🎯 Fitur Utama

### Core POS & Kasir (Front-End)
- ✅ Multi-Session & Cash Control (buka/tutup kas dengan saldo awal)
- ✅ Put On Hold / Parkir Tagihan (suspend order untuk melayani pelanggan lain)
- ✅ Flexible Payment Methods (tunai, QRIS, kartu kredit, split payment)
- ✅ Barcode Scanning & Product Search
- ✅ Offline-First dengan Sync Otomatis
- ✅ Cetak Struk via Printer Thermal

### Customer & Loyalty Management (CRM)
- ✅ Customer Profiling & Registration
- ✅ Loyalty Points & Rewards System
- ✅ Dynamic Pricelists (Per-Customer Discounts)
- ✅ Member Categories (Retail, Wholesale)

### Promosi & Diskon Pintar
- ✅ Conditional Discounts (BOGO, percentage, minimum purchase)
- ✅ Automated Price Markdown (Happy Hour, Expired Date)
- ✅ Promotion Rules Engine

### Multi-Shop & Multi-Warehouse
- ✅ Multi-Company / Multi-Branch Management
- ✅ Real-time Stock Checking across Stores & Warehouses
- ✅ Cross-Store Stock Transfer

### Booking System
- ✅ Product Reservation & Pre-order Management
- ✅ Booking Calendar & Schedule Management
- ✅ Booking Reminders & Notifications

### Analytics & Live Dashboard (Back-End)
- ✅ Product Performance Analysis (Fast/Slow Moving)
- ✅ Salesperson Analytics & Performance Tracking
- ✅ Sales & Revenue Reports
- ✅ Inventory Alerts & Stock Analysis

### Hardware Integration
- ✅ Printer Thermal Driver (Print Receipts)
- ✅ Barcode Scanner Integration
- ✅ Cash Drawer Control
- ✅ Customer Display Integration
- ✅ QRIS & Mobile Payment Support

## 🏗️ Arsitektur Sistem (Monolith)

```
┌─────────────────────────────────────────���──────────────────────┐
│                  POS-EXPO-MONOLITH (Single Server)              │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  FRONTEND LAYER (Served by Express/Static Server)        │  │
│  │  • React/Vue SPA (POS Kasir Screen)                       │  │
│  │  • Admin Dashboard                                        │  │
│  │  • Offline-First DB (IndexedDB)                           │  │
│  │  • Hardware Drivers                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          ↓                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  MIDDLEWARE LAYER                                         │  │
│  │  • Authentication (JWT)                                   │  │
│  │  • Request Validation                                     │  │
│  │  • Error Handling                                         │  │
│  │  • CORS & Security Headers                                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          ↓                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  API ROUTES & CONTROLLERS (Express Routes)               │  │
│  │  • /api/pos                                               │  │
│  │  • /api/inventory                                         │  │
│  │  • /api/customers                                         │  │
│  │  • /api/bookings                                          │  │
│  │  • /api/reports                                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          ↓                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  BUSINESS LOGIC LAYER (Services)                          │  │
│  │  • Transaction Processing                                 │  │
│  │  • Discount Calculation Engine                            │  │
│  │  • Inventory Management                                   │  │
│  │  • CRM & Loyalty Logic                                    │  │
│  │  • Booking Management                                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          ↓                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  DATA ACCESS LAYER (ORM/Query)                            │  │
│  │  • TypeORM / Prisma                                       │  │
│  │  • Database Queries                                       │  │
│  │  • Transaction Management                                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          ↓                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  DATA LAYER                                               │  │
│  │  • PostgreSQL Database                                    │  │
│  │  • Redis Cache                                            │  │
│  │  • File Storage                                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
                             ↓
         ┌────────────────────┼────────────────────┐
         ↓                    ↓                    ↓
    ┌─────────────┐    ┌──────────────┐    ┌──────────────┐
    │   Browser   │    │  Hardware    │    │  External    │
    │   (React)   │    │  (Printer,   │    │  Services    │
    │             │    │   Scanner)   │    │  (Payment)   │
    └─────────────┘    └──────────────┘    └──────────────┘
```

## 📂 Struktur Project (Monolith)

```
POS-EXPO-MONOLITH/
├── src/
│   ├── frontend/                      # Frontend code (served by Express)
│   │   ├── components/
│   │   │   ├── POS/
│   │   │   │   ├── CashierScreen.tsx
│   │   │   │   ├── PaymentScreen.tsx
│   │   │   │   ├── CartItems.tsx
│   │   │   │   └── ProductGrid.tsx
│   │   │   ├── Customers/
│   │   │   ├── Bookings/
│   │   │   ├── Admin/
│   │   │   └── Common/
│   │   ├── pages/
│   │   │   ├── POS.tsx
│   │   │   ├── Inventory.tsx
│   │   │   ├── Dashboard.tsx
│   │   │   └── Settings.tsx
│   │   ├── services/
│   │   │   ├── api.ts                 # Axios client
│   │   │   ├── offline-sync.ts        # IndexedDB sync
│   │   │   └── hardware.ts            # Hardware drivers
│   │   ├── store/
│   │   │   ├── cart.ts
│   │   │   ├── auth.ts
│   │   │   └── index.ts
│   │   ├── hooks/
│   │   ├── utils/
│   │   ├── db/
│   │   │   └── schema.ts
│   │   ├── App.tsx
│   │   └── main.tsx
│   │
│   ├── backend/                       # Backend code (Express Server)
│   │   ├── routes/
│   │   │   ├── auth.routes.ts
│   │   │   ├── pos.routes.ts
│   │   │   ├── inventory.routes.ts
│   │   │   ├── customers.routes.ts
│   │   │   ├── bookings.routes.ts
│   │   │   ├── promotions.routes.ts
│   │   │   └── reports.routes.ts
│   │   ├── controllers/
│   │   │   ├── auth.controller.ts
│   │   │   ├── pos.controller.ts
│   │   │   ├── inventory.controller.ts
│   │   │   ├── customers.controller.ts
│   │   │   ├── bookings.controller.ts
│   │   │   ├── promotions.controller.ts
│   │   │   └── reports.controller.ts
│   │   ├── services/
│   │   │   ├── auth.service.ts
│   │   │   ├── pos.service.ts
│   │   │   ├── inventory.service.ts
│   │   │   ├── customers.service.ts
│   │   │   ├── loyalty.service.ts
│   │   │   ├── bookings.service.ts
│   │   │   ├── promotions.service.ts  # Discount engine
│   │   │   ├── reports.service.ts
│   │   │   └── notification.service.ts
│   │   ├── models/
│   │   │   ├── entities/              # TypeORM/Prisma entities
│   │   │   │   ├── User.entity.ts
│   │   │   │   ├── Transaction.entity.ts
│   │   │   │   ├── Product.entity.ts
│   │   │   │   ├── Customer.entity.ts
│   │   │   │   ├── Booking.entity.ts
│   │   │   │   ├── Promotion.entity.ts
│   │   │   │   ├── Stock.entity.ts
│   │   │   │   └── ...
│   │   │   └── dto/
│   │   │       ├── transaction.dto.ts
│   │   │       ├── customer.dto.ts
│   │   │       └── ...
│   │   ├── middleware/
│   │   │   ├── auth.middleware.ts
│   │   │   ├── error-handler.ts
│   │   │   ├── validator.ts
│   │   │   ├── rate-limiter.ts
│   │   │   └── cors.ts
│   │   ├── utils/
│   │   │   ├── logger.ts
│   │   │   ├── cache.ts
│   │   │   ├── calculations.ts
│   │   │   ├── formatters.ts
│   │   │   └── validators.ts
│   │   ├── config/
│   │   │   ├── database.ts
│   │   │   ├── cache.ts
│   │   │   ├── jwt.ts
│   │   │   └── env.ts
│   │   ├── websocket/
│   │   │   ├── handlers/
│   │   │   ├── setup.ts
│   │   │   └── namespaces.ts
│   │   ├── jobs/
│   │   │   ├── sync-transactions.ts
│   │   │   ├── generate-reports.ts
│   │   │   └── booking-reminders.ts
│   │   ├── app.ts                     # Express app setup
│   │   ├── server.ts                  # Server bootstrap
│   │   └── index.ts
│   │
│   └── shared/
│       ├── types/
│       │   ├── transaction.types.ts
│       │   ├── customer.types.ts
│       │   ├── booking.types.ts
│       │   └── ...
│       ├── constants/
│       │   ├── payment-methods.ts
│       │   ├── user-roles.ts
│       │   └── ...
│       └── utils/
│           ├── formatters.ts
│           ├── validators.ts
│           └── ...
│
├── public/
│   ├── index.html
│   ├── manifest.json                 # PWA manifest
│   └── service-worker.js             # Service worker
│
├── database/
│   ├── migrations/                   # Database migrations
│   │   ├── 001_init.sql
│   │   ├── 002_add_bookings.sql
│   │   └── ...
│   └── seeds/                        # Seed data
│       ├── users.seed.ts
│       ├── products.seed.ts
│       └── ...
│
├── docker-compose.yml
├── Dockerfile                        # Single Docker image for monolith
├── .env.example
├── package.json                      # Single package.json for all dependencies
├── tsconfig.json
├── webpack.config.js or vite.config.ts
└── README.md
```

## 🚀 Quick Start

### Prerequisites
- Node.js 18+ & npm/yarn
- Docker & Docker Compose (optional)
- PostgreSQL 14+ (or use Docker)
- Redis 7+ (or use Docker)

### Development Setup

```bash
# Clone repository
git clone https://github.com/whaone/POS-EXPO-MONOLITH.git
cd POS-EXPO-MONOLITH

# Install dependencies
npm install

# Setup environment variables
cp .env.example .env.local

# Start development services (PostgreSQL, Redis)
docker-compose up -d

# Run migrations
npm run db:migrate

# Seed sample data
npm run db:seed

# Start development server (both frontend & backend)
npm run dev
```

### Access Points
- **POS Kasir (Frontend)**: http://localhost:3000
- **Admin Dashboard**: http://localhost:3000/admin
- **Backend API**: http://localhost:3000/api (internal)
- **API Documentation (Swagger)**: http://localhost:3000/api/docs
- **WebSocket Connection**: ws://localhost:3000

## 🔧 Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | React 18, TypeScript, Redux/Zustand, Tailwind CSS |
| **Backend** | Node.js, Express.js, TypeScript |
| **Database** | PostgreSQL, Redis |
| **Offline DB** | IndexedDB, PouchDB |
| **ORM** | TypeORM atau Prisma |
| **Real-time** | Socket.io (WebSocket) |
| **Hardware** | Node-USB, Thermal Printer API, Web Serial API |
| **Build** | Vite (Frontend), TypeScript compiler (Backend) |
| **Containerization** | Docker, Docker Compose |
| **Testing** | Jest, React Testing Library |
| **API Docs** | Swagger/OpenAPI |

## 📖 Documentation

- [Architecture Detail](docs/ARCHITECTURE.md) - Deep dive into monolith design
- [Features & Requirements](docs/FEATURES.md) - Detailed feature specifications
- [Hardware Integration](docs/HARDWARE.md) - Printer, scanner, drawer setup
- [API Reference](docs/API.md) - Complete API endpoints
- [Setup & Deployment](docs/SETUP.md) - Production deployment guide
- [Database Schema](docs/DATABASE.md) - Complete schema documentation

## 🤝 Contributing

Kontribusi sangat diterima! Silakan baca [CONTRIBUTING.md](CONTRIBUTING.md) untuk guidelines.

## 📄 License

MIT License - See [LICENSE](LICENSE) file

## 👥 Team

- **Author**: [@whaone](https://github.com/whaone)

## 📞 Support

Untuk pertanyaan atau issues, buka [GitHub Issues](https://github.com/whaone/POS-EXPO-MONOLITH/issues)

---

**Last Updated**: June 1, 2026
