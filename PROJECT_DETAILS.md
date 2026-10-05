# Arupro – Offline-First Accounting ERP (Project Details)

## 1. Goal
One accounting/ERP system that runs on **PC/laptop and mobile**, shows screens **by user role**, stores data in **local PC, VPS and Firebase**, and **keeps working without internet**, syncing automatically when the connection returns.

## 2. Platforms
| Platform | How |
|---|---|
| PC / Laptop (Windows, Linux, Mac) | Installable web app (PWA) in Chrome/Edge; optional Electron wrapper later |
| Mobile (Android / iOS) | Same PWA ("Add to Home Screen"); optional Capacitor wrapper to produce an APK |
| One codebase | Single front-end for all devices, responsive layout (sidebar on desktop, bottom menu on mobile) |

## 3. Architecture
```
 Phone / PC browser app (PWA)
   └─ IndexedDB (local copy of all data + outbox of unsynced changes)
        │  auto-sync when any server is reachable
        ▼
 Local PC server (Node + SQLite)  <── replicates both ways ──>  VPS server (Node + SQLite)
        │                                                          │
        └──────────────── one-way mirror ──────────────────────────┘
                                   ▼
                        Firebase Firestore (cloud backup / viewing)
```
- **Front-end:** HTML/JS PWA with service worker (app loads offline), IndexedDB for data.
- **Server:** Node.js + Express + SQLite, same code runs on the local PC and on the VPS.
- **Server-to-server replication:** local server and VPS push/pull changes to each other, so either one can be down.
- **Firebase:** every change is mirrored to Firestore by a background worker with retry; restore script can rebuild a server from Firebase.
- **Auth:** JWT tokens, bcrypt passwords, shared secret across servers.

## 4. Offline and Auto-Sync (core requirement)
1. Every save goes first to **IndexedDB on the device** and into an **outbox queue**. The app never waits for internet.
2. A sync loop runs every ~15 s, on "back online" event, and when the app is opened. It tries servers in order: Local PC → VPS.
3. **Push** the outbox, then **pull** everything newer than the last sequence number.
4. **Conflicts:** last-write-wins per record by timestamp. Stock and ledgers are computed from append-only documents (invoices, vouchers, stock adjustments), so two offline users never overwrite each other's totals.
5. **Document numbers** include a device code (e.g. `S-A7K-0001`) so offline devices never create duplicate invoice numbers.
6. Status badge always visible: Online / Offline / Syncing / "N changes pending".
7. **Offline login:** after the first online login, a salted hash is cached on the device so users can log in with no internet.
8. Local backup: one-click JSON export from Settings.

## 5. Roles and Permissions (enforced on the server, hidden in the UI)
| Role | Sees / can do |
|---|---|
| **Admin** | Everything: all modules, reports, users, settings, sync config |
| **Accountant** | Sales, purchases, vouchers, parties, chart of accounts, all reports; items/stock read-only |
| **Store Keeper** | Items, stock, stock adjustments, goods-received (purchases); no profit/ledger/price reports |
| **Sales** | Create sales invoices, customers, view items; sees only sales invoices |
| **Viewer** | Read-only access to reports and lists |
Admin can add roles/users, deactivate users, and reset passwords.

## 6. Modules
1. **Dashboard** – today's sales, receivables, payables, cash/bank balance, low-stock alerts, pending sync (role-based cards).
2. **Sales** – invoices with item lines, tax %, discount, paid amount (cash/bank/credit), print/PDF.
3. **Purchases** – supplier bills / goods received, stock increases automatically.
4. **Vouchers** – receipt, payment, expense, journal.
5. **Parties** – customers and suppliers with opening balance and statement.
6. **Items & Stock** – item master, units, cost/sale price, opening qty, stock adjustments, reorder level, valuation.
7. **Chart of Accounts** – asset, liability, equity, income, purchase, expense; default chart pre-loaded.
8. **Reports** – Trial Balance, Profit & Loss (with opening/closing stock), Party Ledger/Statement, Receivables/Payables ageing, Stock report, Day Book, Cash/Bank Book.
9. **Users** – admin-only user and role management.
10. **Settings** – server URLs (local + VPS), company info, device name, sync now, backup/restore, logout.

## 7. Accounting Rules
- Full **double-entry**: every invoice/voucher posts balanced debit/credit lines.
- Sale: Dr Customer (or Cash) / Cr Sales + Output Tax. Purchase: Dr Purchases + Input Tax / Cr Supplier. Payments settle the party.
- COGS = Opening stock + Purchases − Closing stock.
- Trial balance must always balance; opening balances offset to a Capital account.

## 8. Data Model (synced collections)
`accounts`, `parties`, `items`, `invoices` (sale/purchase + lines), `vouchers`, `stockadj`, plus server-only `users`.
Each record: `id`, `updatedAt`, `deleted` flag (soft delete), device id.

## 9. Tech Stack
Node 20+, Express, SQLite (better-sqlite3), JWT, bcryptjs, firebase-admin (Firestore), vanilla JS PWA (or React/Vite), IndexedDB, Service Worker, Docker for VPS, Caddy/Nginx for HTTPS.

## 10. Deployment
- **Local PC:** `npm install && npm start`; app opens at `http://localhost:3000`; other devices on the LAN use the PC's IP.
- **VPS:** Docker container behind HTTPS (Let's Encrypt). HTTPS is required for installable mobile app and offline caching.
- **Firebase:** create a project, enable Firestore, put service-account key path in the server's environment.
- **Config (env):** `JWT_SECRET` (same on all servers), `DB_PATH`, `UPSTREAM_URL` (VPS address for the local server), `FIREBASE_SERVICE_ACCOUNT`, `ADMIN_PASSWORD`.

## 11. Security
bcrypt hashes, JWT expiry, per-role checks on every sync request, HTTPS on VPS, Firebase mirror excludes password hashes, regular automatic DB backups.

## 12. Build Phases
| Phase | Scope |
|---|---|
| 1 | Server (auth, roles, sync API), PWA shell, offline storage, login |
| 2 | Parties, items, sales, purchases, vouchers, double-entry engine |
| 3 | Reports, stock, dashboard, printing |
| 4 | Local↔VPS replication, Firebase mirror and restore |
| 5 | Testing (offline/online scenarios, multi-device conflicts), packaging (Docker, APK/Electron), user manual |

## 13. Open Questions
1. Country/currency and tax type (GST, VAT, none)?
2. Number of users and devices?
3. Need barcode scanning, multi-branch/warehouse, or payroll?
4. Is an Android APK required, or is the installable web app enough?
5. Invoice print format (A4, thermal 80 mm)?
