# Hardware Integration Guide

Panduan lengkap integrasi hardware untuk POS-EXPO-MONOLITH termasuk printer thermal, barcode scanner, cash drawer, dan perangkat lainnya.

## 1. Thermal Printer Integration

### 1.1 Supported Printers

Printer thermal yang didukung:
- **Epson TM Series**: TM-T80, TM-T88, TM-T20, TM-U220
- **Star Micronics**: SM-S230i, SM-L90, SM-T300
- **Xprinter**: XP-58, XP-365B, XP-80C
- **Generic ESC/POS**: Printer dengan protokol ESC/POS standard

### 1.2 Connection Methods

#### USB Connection

```typescript
// src/backend/services/hardware/printer.service.ts
import { Injectable } from '@nestjs/common';
import usb from 'usb';

@Injectable()
export class PrinterService {
  private printer: any;
  private isConnected: boolean = false;

  // Epson TM-T88
  private readonly VENDOR_ID = 0x04b8;
  private readonly PRODUCT_ID = 0x0202;

  async connect(): Promise<boolean> {
    try {
      this.printer = usb.findByIds(this.VENDOR_ID, this.PRODUCT_ID);
      if (!this.printer) {
        throw new Error('Printer not found');
      }

      this.printer.open();
      this.isConnected = true;
      console.log('Thermal printer connected (USB)');
      return true;
    } catch (error) {
      console.error('Failed to connect printer:', error);
      this.isConnected = false;
      return false;
    }
  }

  async send(data: Buffer): Promise<void> {
    if (!this.isConnected) {
      throw new Error('Printer not connected');
    }

    return new Promise((resolve, reject) => {
      this.printer.getEndpoints()[0].transfer(data, (error) => {
        if (error) reject(error);
        else resolve();
      });
    });
  }

  isConnectedStatus(): boolean {
    return this.isConnected;
  }

  async disconnect(): Promise<void> {
    if (this.printer) {
      this.printer.close();
      this.isConnected = false;
    }
  }
}
```

#### Network (Ethernet/LAN)

```typescript
// src/backend/services/hardware/network-printer.service.ts
import * as net from 'net';

export class NetworkPrinterService {
  private socket: net.Socket | null = null;
  private host: string;
  private port: number = 9100;
  private isConnected: boolean = false;

  constructor(host: string, port?: number) {
    this.host = host;
    if (port) this.port = port;
  }

  async connect(): Promise<boolean> {
    return new Promise((resolve) => {
      this.socket = net.createConnection(this.port, this.host);

      this.socket.on('connect', () => {
        this.isConnected = true;
        console.log(`Connected to network printer at ${this.host}:${this.port}`);
        resolve(true);
      });

      this.socket.on('error', (error) => {
        console.error('Network printer error:', error);
        this.isConnected = false;
        resolve(false);
      });

      this.socket.on('close', () => {
        this.isConnected = false;
      });
    });
  }

  async send(data: Buffer): Promise<void> {
    return new Promise((resolve, reject) => {
      if (!this.socket || !this.isConnected) {
        reject(new Error('Printer not connected'));
        return;
      }

      this.socket!.write(data, (error) => {
        if (error) reject(error);
        else resolve();
      });
    });
  }

  isConnectedStatus(): boolean {
    return this.isConnected;
  }

  async disconnect(): Promise<void> {
    if (this.socket) {
      this.socket.destroy();
      this.isConnected = false;
    }
  }
}
```

#### Serial (COM Port)

```typescript
// src/backend/services/hardware/serial-printer.service.ts
import SerialPort from 'serialport';

export class SerialPrinterService {
  private port: SerialPort | null = null;
  private isConnected: boolean = false;

  async connect(portPath: string): Promise<boolean> {
    return new Promise((resolve) => {
      this.port = new SerialPort(portPath, {
        baudRate: 115200,
        dataBits: 8,
        stopBits: 1,
        parity: 'none'
      }, (error) => {
        if (error) {
          console.error('Serial port error:', error);
          resolve(false);
        } else {
          this.isConnected = true;
          console.log(`Connected to serial printer: ${portPath}`);
          resolve(true);
        }
      });
    });
  }

  async send(data: Buffer): Promise<void> {
    return new Promise((resolve, reject) => {
      if (!this.port || !this.isConnected) {
        reject(new Error('Serial printer not connected'));
        return;
      }

      this.port!.write(data, (error) => {
        if (error) reject(error);
        else resolve();
      });
    });
  }

  isConnectedStatus(): boolean {
    return this.isConnected;
  }
}
```

### 1.3 Receipt Generation

```typescript
// src/backend/services/hardware/receipt-builder.ts
export class ReceiptBuilder {
  private data: Buffer[] = [];
  private lineWidth: number = 32; // 80mm thermal paper

  // Initialize printer
  initialize(): this {
    // ESC @ - Reset printer
    this.data.push(Buffer.from([0x1B, 0x40]));
    return this;
  }

  // Set alignment (0=left, 1=center, 2=right)
  align(type: 'left' | 'center' | 'right'): this {
    const alignCode = {
      'left': 0x00,
      'center': 0x01,
      'right': 0x02
    }[type];

    this.data.push(Buffer.from([0x1B, 0x61, alignCode])); // ESC a
    return this;
  }

  // Print text
  text(content: string): this {
    this.data.push(Buffer.from(content, 'utf8'));
    return this;
  }

  // New line
  newLine(count: number = 1): this {
    this.data.push(Buffer.from('\n'.repeat(count)));
    return this;
  }

  // Set font size (1-4)
  setFontSize(width: number = 1, height: number = 1): this {
    const sizeCode = (width << 4) | height;
    this.data.push(Buffer.from([0x1D, 0x21, sizeCode])); // GS !
    return this;
  }

  // Print barcode
  barcode(code: string, type: 'CODE128' | 'EAN13' = 'CODE128'): this {
    const barcodeType = type === 'CODE128' ? 0x49 : 0x43;
    const codeLength = Buffer.from(code, 'utf8').length;

    this.data.push(Buffer.from([0x1D, 0x6B, barcodeType, codeLength])); // GS k
    this.data.push(Buffer.from(code, 'utf8'));
    return this;
  }

  // Cut paper (partial cut)
  cut(): this {
    this.data.push(Buffer.from([0x1D, 0x56, 0x41])); // GS V A
    return this;
  }

  // Cash drawer kick pulse
  kickDrawer(): this {
    this.data.push(Buffer.from([0x1B, 0x70, 0x00, 0x32, 0x00])); // ESC p
    return this;
  }

  // Horizontal line
  line(char: string = '─'): this {
    this.text(char.repeat(this.lineWidth) + '\n');
    return this;
  }

  // Get final buffer
  build(): Buffer {
    return Buffer.concat(this.data);
  }
}

// Usage in service
@Injectable()
export class ReceiptService {
  constructor(private printerService: PrinterService) {}

  async printTransactionReceipt(transaction: Transaction): Promise<void> {
    const receipt = new ReceiptBuilder()
      .initialize()
      .align('center')
      .setFontSize(2, 2)
      .text('TOKO ABC\n')
      .setFontSize(1, 1)
      .text(`No: ${transaction.transactionNumber}\n`)
      .text(`${this.formatDateTime(transaction.createdAt)}\n`)
      .newLine()
      .align('left')
      .line();

    // Print items
    for (const item of transaction.items) {
      const name = item.product.name.substring(0, 20);
      const qty = String(item.quantity).padStart(3, ' ');
      const price = String(item.unitPrice).padStart(8, ' ');
      const total = String(item.subtotal).padStart(8, ' ');

      receipt
        .text(`${name}\n`)
        .text(`${qty}x @ ${price} = ${total}\n`);
    }

    receipt
      .newLine()
      .line()
      .align('right')
      .text(`Subtotal: ${this.formatCurrency(transaction.subtotal)}\n`);

    if (transaction.discountAmount > 0) {
      receipt.text(`Diskon  : ${this.formatCurrency(transaction.discountAmount)}\n`);
    }

    receipt
      .text(`Pajak   : ${this.formatCurrency(transaction.taxAmount)}\n`)
      .setFontSize(2, 2)
      .text(`Total   : ${this.formatCurrency(transaction.totalAmount)}\n`)
      .setFontSize(1, 1)
      .newLine()
      .align('center')
      .text('Terima kasih!\n')
      .text('Thank you!\n')
      .newLine(3)
      .cut()
      .kickDrawer(); // Open drawer after printing

    const buffer = receipt.build();
    await this.printerService.send(buffer);
  }

  private formatCurrency(amount: number): string {
    return amount.toLocaleString('id-ID', {
      style: 'currency',
      currency: 'IDR',
      minimumFractionDigits: 0
    });
  }

  private formatDateTime(date: Date): string {
    return date.toLocaleString('id-ID', {
      year: 'numeric',
      month: '2-digit',
      day: '2-digit',
      hour: '2-digit',
      minute: '2-digit'
    });
  }
}
```

---

## 2. Barcode Scanner Integration

### 2.1 Keyboard Wedge Scanner (Most Common)

```typescript
// src/frontend/services/hardware/barcode-scanner.ts
export class BarcodeScanner {
  private barcodeBuffer: string = '';
  private timeout: NodeJS.Timeout | null = null;
  private callback: ((barcode: string) => void) | null = null;
  private readonly TIMEOUT_MS = 2000; // 2 seconds

  initialize() {
    // Most USB barcode scanners emulate keyboard input
    document.addEventListener('keydown', (event) => {
      this.handleKeydown(event);
    });

    console.log('Barcode scanner initialized (keyboard mode)');
  }

  private handleKeydown(event: KeyboardEvent) {
    // Enter key ends input
    if (event.code === 'Enter') {
      if (this.barcodeBuffer.length > 0) {
        this.processScan(this.barcodeBuffer);
        this.barcodeBuffer = '';
        event.preventDefault();
      }
      return;
    }

    // Only capture printable characters
    if (event.key.length === 1 && !event.ctrlKey && !event.altKey && !event.metaKey) {
      this.barcodeBuffer += event.key;

      // Clear auto-timeout
      if (this.timeout) clearTimeout(this.timeout);

      // Re-set auto-timeout (for slow scanners)
      this.timeout = setTimeout(() => {
        if (this.barcodeBuffer.length > 0) {
          this.processScan(this.barcodeBuffer);
          this.barcodeBuffer = '';
        }
      }, this.TIMEOUT_MS);
    }
  }

  private processScan(barcode: string) {
    console.log('Barcode scanned:', barcode);

    // Validate format
    if (validateBarcode(barcode)) {
      if (this.callback) {
        this.callback(barcode);
      }
    } else {
      console.warn('Invalid barcode format:', barcode);
    }
  }

  onBarcode(callback: (barcode: string) => void) {
    this.callback = callback;
  }
}

// Barcode validation
function validateBarcode(barcode: string): boolean {
  // Check if matches common formats
  if (/^\d{8}$/.test(barcode)) return true;      // EAN-8
  if (/^\d{12,13}$/.test(barcode)) return true;  // UPC-A or EAN-13
  if (/^[A-Z0-9\-]+$/.test(barcode)) return true; // CODE128, CODE39
  return false;
}
```

### 2.2 Serial Port Scanner (Web Serial API)

```typescript
// src/frontend/services/hardware/serial-scanner.ts
export class SerialScanner {
  private port: SerialPort | null = null;
  private callback: ((barcode: string) => void) | null = null;
  private isConnected: boolean = false;

  async connect(): Promise<boolean> {
    try {
      // Request port from user (shows dialog)
      this.port = await navigator.serial.requestPort({
        filters: [
          { vendorId: 0x0463 }  // Zebex scanners
        ]
      });

      await this.port.open({ baudRate: 9600 });
      this.isConnected = true;
      this.startReading();
      return true;
    } catch (error) {
      console.error('Failed to connect serial scanner:', error);
      this.isConnected = false;
      return false;
    }
  }

  private async startReading() {
    if (!this.port?.readable) return;

    const reader = this.port.readable.getReader();
    let barcode = '';

    try {
      while (true) {
        const { value, done } = await reader.read();

        if (done) break;

        // Decode received bytes
        const text = new TextDecoder().decode(value);

        for (const char of text) {
          if (char === '\n' || char === '\r') {
            if (barcode.length > 0) {
              this.processScan(barcode);
              barcode = '';
            }
          } else if (char.charCodeAt(0) >= 32) {
            // Printable character
            barcode += char;
          }
        }
      }
    } catch (error) {
      console.error('Serial read error:', error);
      this.isConnected = false;
    }
  }

  private processScan(barcode: string) {
    if (this.callback) {
      this.callback(barcode.trim());
    }
  }

  onBarcode(callback: (barcode: string) => void) {
    this.callback = callback;
  }

  async disconnect() {
    if (this.port) {
      await this.port.close();
      this.isConnected = false;
    }
  }

  isConnectedStatus(): boolean {
    return this.isConnected;
  }
}
```

### 2.3 Usage in POS Component

```typescript
// src/frontend/components/POS/CashierScreen.tsx
import React, { useEffect, useState } from 'react';
import { useDispatch } from 'react-redux';
import { BarcodeScanner } from '@/services/hardware/barcode-scanner';

export function CashierScreen() {
  const dispatch = useDispatch();
  const [scanner, setScanner] = useState<BarcodeScanner | null>(null);

  useEffect(() => {
    const barcodeScanner = new BarcodeScanner();
    barcodeScanner.initialize();

    barcodeScanner.onBarcode((barcode) => {
      // Look up product by barcode
      const product = lookupProductByBarcode(barcode);

      if (product) {
        // Add to cart
        dispatch(addToCart({
          productId: product.id,
          quantity: 1,
          price: product.price
        }));

        showNotification(`Added: ${product.name}`, 'success');
      } else {
        showNotification('Product not found', 'error');
      }
    });

    setScanner(barcodeScanner);
  }, [dispatch]);

  return (
    <div className="cashier-screen">
      {/* Main POS interface */}
    </div>
  );
}
```

---

## 3. Cash Drawer Integration

### 3.1 ESC/POS Drawer Kick (via Printer)

```typescript
// src/backend/services/hardware/cash-drawer.service.ts
@Injectable()
export class CashDrawerService {
  constructor(private printerService: PrinterService) {}

  async openDrawer(): Promise<boolean> {
    try {
      // ESC p command (Epson standard)
      const command = Buffer.from([0x1B, 0x70, 0x00, 0x32, 0x00]);
      await this.printerService.send(command);

      console.log('Cash drawer opened');
      return true;
    } catch (error) {
      console.error('Failed to open cash drawer:', error);
      return false;
    }
  }

  // Alternative for Star printers
  async openDrawerStar(): Promise<boolean> {
    try {
      // Star Micronics drawer command
      const command = Buffer.from([0x07]);
      await this.printerService.send(command);

      return true;
    } catch (error) {
      console.error('Failed to open Star drawer:', error);
      return false;
    }
  }
}
```

### 3.2 Safety Manager

```typescript
// src/backend/services/pos/cash-register.service.ts
@Injectable()
export class CashRegisterService {
  private drawerStates: Map<string, DrawerState> = new Map();

  constructor(private drawerService: CashDrawerService) {}

  async openCashRegister(branchId: string, cashierId: string): Promise<void> {
    const key = `${branchId}-${cashierId}`;

    // Check if already open
    if (this.drawerStates.get(key)?.isOpen) {
      throw new BadRequestException('Cash drawer already open');
    }

    // Try to open
    const success = await this.drawerService.openDrawer();
    if (!success) {
      throw new InternalServerErrorException('Failed to open cash drawer');
    }

    // Record state
    this.drawerStates.set(key, {
      isOpen: true,
      openedAt: new Date(),
      openedByCashierId: cashierId
    });

    // Auto-timeout after 10 minutes
    setTimeout(() => {
      this.drawerStates.delete(key);
    }, 10 * 60 * 1000);
  }

  getDrawerState(branchId: string, cashierId: string): DrawerState | undefined {
    return this.drawerStates.get(`${branchId}-${cashierId}`);
  }
}

interface DrawerState {
  isOpen: boolean;
  openedAt: Date;
  openedByCashierId: string;
}
```

---

## 4. QRIS Payment (Indonesian Standard)

### 4.1 Generate QRIS Code

```typescript
// src/backend/services/payment/qris.service.ts
import QRCode from 'qrcode';

@Injectable()
export class QrisService {
  async generateQRIS(transaction: Transaction): Promise<string> {
    const qrisString = this.buildQRISString({
      merchantId: 'ID.MICROPAYMENT.YOUR_MERCHANT_ID',
      merchantName: 'TOKO ABC',
      merchantCity: 'JAKARTA',
      amount: transaction.totalAmount,
      transactionId: transaction.id
    });

    // Generate QR code image (base64)
    const qrImage = await QRCode.toDataURL(qrisString, {
      width: 300,
      margin: 2,
      errorCorrectionLevel: 'H'
    });

    return qrImage;
  }

  private buildQRISString(data: any): string {
    // Simplified QRIS format (EMVCo standard)
    // Full spec: https://www.bi.go.id/id/sistem-pembayaran/bi-fast/qris/

    const qris = {
      '00': '01',                                      // Payload format indicator
      '01': '12',                                      // Point of initiation (dynamic)
      '29': this.encodeTag29(data.merchantId),       // Merchant account info
      '04': String(data.amount).padStart(12, '0'),  // Transaction amount
      '53': '360',                                    // Currency (IDR)
      '58': 'ID',                                     // Country
      '59': data.merchantName.substring(0, 25),     // Merchant name
      '60': data.merchantCity.substring(0, 15)      // Merchant city
    };

    let encoded = '';
    for (const [tag, value] of Object.entries(qris)) {
      if (value) {
        const strValue = String(value);
        encoded += `${tag}${String(strValue.length).padStart(2, '0')}${strValue}`;
      }
    }

    // Add CRC-16
    const crc = this.calculateCRC16(encoded + '6304');
    encoded += `6304${crc}`;

    return encoded;
  }

  private encodeTag29(merchantId: string): string {
    const gai = '00000007223002'; // Global Unique Identifier
    const rai = merchantId;

    const value = gai + rai;
    return value;
  }

  private calculateCRC16(data: string): string {
    let crc = 0xFFFF;

    for (let i = 0; i < data.length; i += 2) {
      const byte = parseInt(data.substr(i, 2), 16);
      crc ^= byte << 8;

      for (let j = 0; j < 8; j++) {
        crc = crc << 1;
        if (crc & 0x10000) {
          crc = (crc ^ 0x1021) & 0xFFFF;
        }
      }
    }

    return crc.toString(16).toUpperCase().padStart(4, '0');
  }
}
```

### 4.2 Display QRIS in Frontend

```typescript
// src/frontend/components/POS/QRISPayment.tsx
import React, { useEffect, useState } from 'react';

export function QRISPayment({ transaction }: { transaction: Transaction }) {
  const [qrImage, setQrImage] = useState<string>('');
  const [paymentStatus, setPaymentStatus] = useState<'pending' | 'confirmed' | 'expired'>('pending');

  useEffect(() => {
    // Get QRIS from backend
    async function loadQRIS() {
      const response = await fetch(`/api/pos/qris/${transaction.id}`);
      const data = await response.json();
      setQrImage(data.qrImage);

      // Poll for payment confirmation
      checkPaymentStatus();
    }

    loadQRIS();
  }, [transaction.id]);

  async function checkPaymentStatus() {
    const interval = setInterval(async () => {
      const response = await fetch(`/api/pos/transactions/${transaction.id}`);
      const data = await response.json();

      if (data.paymentStatus === 'confirmed') {
        setPaymentStatus('confirmed');
        clearInterval(interval);
      }
    }, 2000); // Check every 2 seconds

    // Stop after 5 minutes
    setTimeout(() => {
      clearInterval(interval);
      setPaymentStatus('expired');
    }, 5 * 60 * 1000);
  }

  return (
    <div className="qris-payment">
      <h3>Scan QRIS Code</h3>

      {qrImage && (
        <img src={qrImage} alt="QRIS Code" className="qr-code" />
      )}

      <p className="amount">
        Jumlah: Rp {transaction.totalAmount.toLocaleString('id-ID')}
      </p>

      <div className="status">
        {paymentStatus === 'pending' && (
          <span className="pending">⏳ Menunggu pembayaran...</span>
        )}
        {paymentStatus === 'confirmed' && (
          <span className="confirmed">✓ Pembayaran diterima</span>
        )}
        {paymentStatus === 'expired' && (
          <span className="expired">× Kode QR kadaluarsa</span>
        )}
      </div>
    </div>
  );
}
```

---

## 5. Hardware Configuration

### 5.1 Configuration File

```json
// hardware-config.json
{
  "printer": {
    "enabled": true,
    "type": "usb",
    "vendor": "0x04b8",
    "product": "0x0202",
    "paperWidth": 80,
    "autocut": true,
    "paperCutType": "partial",
    "characterEncoding": "UTF-8"
  },
  "scanner": {
    "enabled": true,
    "type": "keyboard-wedge",
    "timeout": 2000,
    "validBarcodeFormats": ["EAN8", "EAN13", "CODE128", "CODE39"],
    "enableValidation": true
  },
  "drawer": {
    "enabled": true,
    "type": "escp",
    "autoCloseTime": 10000
  },
  "payment": {
    "qris": {
      "enabled": true,
      "merchantId": "ID.MICROPAYMENT.YOUR_ID",
      "timeout": 300000
    }
  }
}
```

### 5.2 Hardware Manager Service

```typescript
// src/backend/services/hardware/hardware-manager.service.ts
@Injectable()
export class HardwareManagerService {
  private config: any;
  private devices: Map<string, HardwareDevice> = new Map();

  async initialize(configPath: string) {
    // Load config
    this.config = JSON.parse(fs.readFileSync(configPath, 'utf-8'));

    // Initialize printer
    if (this.config.printer.enabled) {
      const printer = new PrinterService();
      await printer.connect();
      this.devices.set('printer', printer);
    }

    // Initialize drawer
    if (this.config.drawer.enabled) {
      const drawer = new CashDrawerService(this.devices.get('printer') as PrinterService);
      this.devices.set('drawer', drawer);
    }

    console.log('Hardware initialized');
  }

  getDevice(name: string): any {
    return this.devices.get(name);
  }

  async healthCheck(): Promise<Map<string, boolean>> {
    const status = new Map<string, boolean>();

    for (const [name, device] of this.devices) {
      try {
        status.set(name, device.isConnectedStatus?.() ?? false);
      } catch (error) {
        status.set(name, false);
      }
    }

    return status;
  }
}
```

---

## 6. Error Handling

```typescript
// src/backend/services/hardware/hardware-errors.ts
export class HardwareNotConnectedError extends Error {
  constructor(deviceName: string) {
    super(`Hardware device ${deviceName} is not connected`);
    this.name = 'HardwareNotConnectedError';
  }
}

export class HardwareTimeoutError extends Error {
  constructor(deviceName: string, timeout: number) {
    super(`Hardware operation on ${deviceName} timed out after ${timeout}ms`);
    this.name = 'HardwareTimeoutError';
  }
}

// Global error handler middleware
export function handleHardwareError(error: any, deviceName: string) {
  if (error instanceof HardwareNotConnectedError) {
    console.warn(`Continuing without ${deviceName}`);
    return;
  }

  if (error instanceof HardwareTimeoutError) {
    console.error(`${deviceName} timeout, attempting reconnection...`);
  }

  console.error(`Hardware error [${deviceName}]:`, error);
}
```

---

## 7. Testing

```typescript
// tests/hardware/printer.test.ts
describe('PrinterService', () => {
  let printer: PrinterService;

  beforeEach(() => {
    printer = new PrinterService();
  });

  it('should connect to printer', async () => {
    const connected = await printer.connect();
    expect(connected).toBe(true);
    expect(printer.isConnectedStatus()).toBe(true);
  });

  it('should generate receipt buffer', () => {
    const receipt = new ReceiptBuilder()
      .initialize()
      .text('Test Receipt\n')
      .newLine()
      .cut();

    const buffer = receipt.build();
    expect(buffer).toBeInstanceOf(Buffer);
    expect(buffer.length).toBeGreaterThan(0);
  });
});
```

---

**Document Version**: 1.0  
**Last Updated**: June 1, 2026
