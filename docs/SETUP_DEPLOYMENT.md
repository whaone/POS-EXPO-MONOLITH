# Setup & Deployment

Panduan lengkap setup development environment dan production deployment POS-EXPO-MONOLITH.

## 1. Development Setup

### 1.1 Prerequisites

- **Node.js**: v18.0.0 atau lebih baru
- **npm**: v9.0.0 atau lebih baru
- **Docker**: v20.0.0 atau lebih baru
- **Docker Compose**: v2.0.0 atau lebih baru
- **Git**: v2.30.0 atau lebih baru
- **PostgreSQL**: v14.0 (opsional, jika tidak pakai Docker)
- **Redis**: v7.0 (opsional, jika tidak pakai Docker)

### 1.2 Clone Repository

```bash
git clone https://github.com/whaone/POS-EXPO-MONOLITH.git
cd POS-EXPO-MONOLITH
```

### 1.3 Install Dependencies

```bash
# Install root dependencies
npm install

# Install backend dependencies
cd src/backend
npm install
cd ../..

# Install frontend dependencies
cd src/frontend
npm install
cd ../..
```

### 1.4 Environment Configuration

Buat file `.env` di root directory:

```bash
# Copy template
cp .env.example .env
```

Edit `.env`:

```env
# Application
NODE_ENV=development
PORT=3000
FRONTEND_URL=http://localhost:3001

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=pos_expo_dev
DB_USER=postgres
DB_PASSWORD=postgres
DB_LOGGING=true

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0

# JWT
JWT_SECRET=your-secret-key-change-this
JWT_EXPIRATION=3600
JWT_REFRESH_SECRET=your-refresh-secret
JWT_REFRESH_EXPIRATION=604800

# Authentication
BCRYPT_ROUNDS=10

# AWS S3 (untuk image upload)
AWS_REGION=ap-southeast-1
AWS_ACCESS_KEY_ID=your-key
AWS_SECRET_ACCESS_KEY=your-secret
AWS_S3_BUCKET=pos-expo-bucket

# Payment Gateway
QRIS_MERCHANT_ID=ID.MICROPAYMENT.YOUR_ID

# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password
SMTP_FROM=noreply@pos-expo.com

# Logging
LOG_LEVEL=debug
LOG_FORMAT=json
```

### 1.5 Database Setup

#### Option A: Using Docker (Recommended)

```bash
# Start PostgreSQL dan Redis dengan Docker Compose
docker-compose up -d

# Tunggu services ready (30 detik)
sleep 30

# Run database migrations
npm run db:migrate

# Seed database dengan data contoh
npm run db:seed
```

#### Option B: Manual Setup

```bash
# Create database
createdb pos_expo_dev

# Create user
createuser pos_user
psql -c "ALTER USER pos_user WITH PASSWORD 'password';"
psql -c "GRANT ALL PRIVILEGES ON DATABASE pos_expo_dev TO pos_user;"

# Run migrations
npm run db:migrate

# Seed database
npm run db:seed
```

### 1.6 Start Development Server

```bash
# Terminal 1: Backend (NestJS)
npm run start:dev:backend

# Terminal 2: Frontend (React/Expo)
npm run start:dev:frontend

# Or: Start both dengan concurrently
npm run dev
```

Akses aplikasi:
- **Frontend**: http://localhost:3001
- **Backend API**: http://localhost:3000/api
- **API Documentation**: http://localhost:3000/api/docs

---

## 2. Database Migrations

### 2.1 Create Migration

```bash
npm run db:migration:create -- --name=create_products_table
```

### 2.2 Run Migrations

```bash
# Run pending migrations
npm run db:migrate

# Run specific migration
npm run db:migrate -- --step=1

# Undo last migration
npm run db:migrate:undo

# Undo all migrations
npm run db:migrate:undo:all
```

### 2.3 Seed Database

```bash
# Run all seeders
npm run db:seed

# Run specific seeder
npm run db:seed -- --seed=20260601-products-seeder
```

---

## 3. Production Deployment

### 3.1 Build Docker Image

```bash
# Build image
docker build -t pos-expo-monolith:latest .

# Or build dengan specific tag
docker build -t pos-expo-monolith:v1.0.0 .

# List images
docker images
```

### 3.2 Push to Docker Registry

```bash
# Login to Docker Hub
docker login

# Tag image
docker tag pos-expo-monolith:latest username/pos-expo-monolith:latest
docker tag pos-expo-monolith:latest username/pos-expo-monolith:v1.0.0

# Push image
docker push username/pos-expo-monolith:latest
docker push username/pos-expo-monolith:v1.0.0
```

### 3.3 Production Environment

Buat `.env.production`:

```env
# Application
NODE_ENV=production
PORT=3000
FRONTEND_URL=https://pos.example.com

# Database
DB_HOST=db.example.com
DB_PORT=5432
DB_NAME=pos_expo_prod
DB_USER=prod_user
DB_PASSWORD=change-this-strong-password
DB_LOGGING=false
DB_SSL=true

# Redis
REDIS_HOST=redis.example.com
REDIS_PORT=6379
REDIS_PASSWORD=change-this-strong-password
REDIS_DB=0

# JWT
JWT_SECRET=change-this-very-long-random-secret
JWT_EXPIRATION=3600
JWT_REFRESH_SECRET=change-this-very-long-random-secret
JWT_REFRESH_EXPIRATION=604800

# AWS S3
AWS_REGION=ap-southeast-1
AWS_ACCESS_KEY_ID=xxxxx
AWS_SECRET_ACCESS_KEY=xxxxx
AWS_S3_BUCKET=pos-expo-prod

# QRIS
QRIS_MERCHANT_ID=ID.MICROPAYMENT.PROD_ID

# Email
SMTP_HOST=smtp.sendgrid.net
SMTP_PORT=587
SMTP_USER=apikey
SMTP_PASSWORD=your-sendgrid-key
SMTP_FROM=noreply@pos.example.com

# Logging
LOG_LEVEL=info
LOG_FORMAT=json

# CDN
CDN_URL=https://cdn.example.com
```

### 3.4 Kubernetes Deployment

#### Create deployment.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: pos-expo

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: pos-expo-config
  namespace: pos-expo
data:
  NODE_ENV: production
  LOG_LEVEL: info

---
apiVersion: v1
kind: Secret
metadata:
  name: pos-expo-secrets
  namespace: pos-expo
type: Opaque
stringData:
  DB_HOST: postgres.default.svc.cluster.local
  DB_NAME: pos_expo_prod
  DB_USER: prod_user
  DB_PASSWORD: change-this
  REDIS_HOST: redis.default.svc.cluster.local
  REDIS_PASSWORD: change-this
  JWT_SECRET: change-this-very-long-random-secret

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pos-expo-app
  namespace: pos-expo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pos-expo-app
  template:
    metadata:
      labels:
        app: pos-expo-app
    spec:
      containers:
      - name: pos-expo
        image: username/pos-expo-monolith:v1.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: pos-expo-config
              key: NODE_ENV
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: pos-expo-secrets
              key: DB_HOST
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: pos-expo-secrets
              key: DB_NAME
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: pos-expo-secrets
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: pos-expo-secrets
              key: DB_PASSWORD
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: pos-expo-service
  namespace: pos-expo
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
  selector:
    app: pos-expo-app

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: pos-expo-hpa
  namespace: pos-expo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: pos-expo-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

Deploy ke Kubernetes:

```bash
# Apply configuration
kubectl apply -f deployment.yaml

# Check deployment status
kubectl get deployment -n pos-expo
kubectl get pods -n pos-expo

# View logs
kubectl logs -n pos-expo deployment/pos-expo-app

# Port forward untuk testing
kubectl port-forward -n pos-expo svc/pos-expo-service 3000:80
```

### 3.5 Docker Compose for Production

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: pos_expo_prod
      POSTGRES_USER: prod_user
      POSTGRES_PASSWORD: change-this
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U prod_user"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass change-this
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    image: username/pos-expo-monolith:v1.0.0
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      DB_HOST: postgres
      DB_NAME: pos_expo_prod
      DB_USER: prod_user
      DB_PASSWORD: change-this
      REDIS_HOST: redis
      REDIS_PASSWORD: change-this
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - app
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

### 3.6 Nginx Configuration

```nginx
# nginx.conf
upstream app {
  server app:3000;
}

server {
  listen 80;
  server_name _;

  # Redirect HTTP to HTTPS
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl http2;
  server_name pos.example.com;

  ssl_certificate /etc/nginx/ssl/cert.pem;
  ssl_certificate_key /etc/nginx/ssl/key.pem;

  # Security headers
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
  add_header X-Frame-Options "SAMEORIGIN" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header X-XSS-Protection "1; mode=block" always;

  # Gzip compression
  gzip on;
  gzip_types text/plain text/css text/javascript application/json application/javascript;
  gzip_min_length 1024;

  location / {
    proxy_pass http://app;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_cache_bypass $http_upgrade;

    # Timeouts
    proxy_connect_timeout 60s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
  }

  # Static files caching
  location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
  }
}
```

### 3.7 Database Backup & Restore

```bash
# Backup database
pg_dump -h db.example.com -U prod_user pos_expo_prod > backup.sql

# Restore database
psql -h db.example.com -U prod_user pos_expo_prod < backup.sql

# Automated backup (cron job)
# 0 2 * * * pg_dump -h db.example.com -U prod_user pos_expo_prod | gzip > /backups/pos-expo-$(date +\%Y\%m\%d).sql.gz
```

---

## 4. CI/CD Pipeline

### 4.1 GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main, production]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_DB: pos_expo_test
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
    - uses: actions/checkout@v3

    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Run linter
      run: npm run lint

    - name: Run tests
      run: npm run test:cov
      env:
        DB_HOST: localhost
        DB_NAME: pos_expo_test

    - name: Upload coverage
      uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
    - uses: actions/checkout@v3

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2

    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: |
          ${{ secrets.DOCKER_USERNAME }}/pos-expo-monolith:latest
          ${{ secrets.DOCKER_USERNAME }}/pos-expo-monolith:${{ github.sha }}
        cache-from: type=registry,ref=${{ secrets.DOCKER_USERNAME }}/pos-expo-monolith:buildcache
        cache-to: type=registry,ref=${{ secrets.DOCKER_USERNAME }}/pos-expo-monolith:buildcache,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/production'

    steps:
    - uses: actions/checkout@v3

    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/pos-expo-app \
          pos-expo=${{ secrets.DOCKER_USERNAME }}/pos-expo-monolith:${{ github.sha }} \
          -n pos-expo
      env:
        KUBECONFIG: ${{ secrets.KUBECONFIG }}

    - name: Verify deployment
      run: |
        kubectl rollout status deployment/pos-expo-app -n pos-expo
      env:
        KUBECONFIG: ${{ secrets.KUBECONFIG }}
```

---

## 5. Monitoring & Logging

### 5.1 Winston Logging

```typescript
// src/backend/config/logger.ts
import * as winston from 'winston';

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
  ],
});

if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.simple(),
  }));
}
```

### 5.2 Health Check Endpoint

```typescript
// src/backend/health.controller.ts
@Controller('health')
export class HealthController {
  constructor(
    private db: Database,
    private redis: Redis,
  ) {}

  @Get()
  async check() {
    const checks = {
      status: 'ok',
      timestamp: new Date(),
      database: await this.checkDatabase(),
      redis: await this.checkRedis(),
    };

    return checks;
  }

  private async checkDatabase() {
    try {
      await this.db.query('SELECT 1');
      return { status: 'ok' };
    } catch (error) {
      return { status: 'error', message: error.message };
    }
  }

  private async checkRedis() {
    try {
      await this.redis.ping();
      return { status: 'ok' };
    } catch (error) {
      return { status: 'error', message: error.message };
    }
  }
}
```

---

## 6. Useful Commands

```bash
# Development
npm run dev                 # Start both frontend & backend
npm run start:dev:backend   # Start backend only
npm run start:dev:frontend  # Start frontend only

# Testing
npm test                    # Run all tests
npm run test:watch         # Run tests in watch mode
npm run test:cov           # Run tests with coverage

# Database
npm run db:migrate         # Run migrations
npm run db:seed            # Seed database
npm run db:reset           # Reset database (dev only)

# Linting
npm run lint               # Run ESLint
npm run lint:fix           # Fix linting issues
npm run format             # Format code with Prettier

# Building
npm run build              # Build for production
npm run build:docker       # Build Docker image

# Deployment
npm run deploy:dev         # Deploy to dev
npm run deploy:staging     # Deploy to staging
npm run deploy:prod        # Deploy to production

# Docker
docker-compose up          # Start services
docker-compose down        # Stop services
docker-compose logs -f     # View logs
docker-compose ps          # List services
```

---

**Document Version**: 1.0  
**Last Updated**: June 1, 2026
