# 💊 Dawaa Pharmacy (دواء) — AI Automation & Inventory System

![n8n](https://img.shields.io/badge/n8n-Workflow_Automation-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)
![Google Gemini](https://img.shields.io/badge/Gemini_2.5_Flash-AI_Extraction-4285F4?style=for-the-badge&logo=googlegemini&logoColor=white)
![WhatsApp Cloud API](https://img.shields.io/badge/WhatsApp-Cloud_API-25D366?style=for-the-badge&logo=whatsapp&logoColor=white)
![Telegram Bot API](https://img.shields.io/badge/Telegram-Pharmacist_Bot-26A5E4?style=for-the-badge&logo=telegram&logoColor=white)
![Google Sheets](https://img.shields.io/badge/Google_Sheets-Live_Database-34A853?style=for-the-badge&logo=googlesheets&logoColor=white)

An end-to-end **AI-powered pharmacy order, prescription parsing, inventory, and operations automation system** built with **n8n**. Customers send medicine names or prescription photos via WhatsApp (in English or Arabic), Gemini extracts and matches items against a live catalog, controlled medications route to a Telegram pharmacist approval desk, and background schedules automatically expire unpaid holds, restock inventory, monitor low stock, and log errors.

---

## ✨ Key Features

- **Bilingual WhatsApp Intake (Text & Prescription Photos):** Accepts natural text requests or uploaded prescription images via Meta WhatsApp Cloud API.
- **AI Prescription Extraction:** Uses **Gemini 2.5 Flash** to parse medicine names and quantities, with automatic fallback escalation to a pharmacist if AI parsing fails.
- **Smart Inventory & Controlled-Drug Routing:**
  - **OTC Medicines:** Auto-reserved with instant stock deduction and WhatsApp confirmation.
  - **Controlled Medicines (e.g., Xanax):** Placed in `pending_pharmacist` status and sent to the Telegram pharmacists group with inline **✅ Approve** / **❌ Reject** buttons.
- **Interactive Pharmacist Callback Loop:** One-tap Telegram approval updates reservation rows, deducts catalog stock, edits the original Telegram message to prevent duplicate actions, and notifies the customer on WhatsApp.
- **Automated 15-Minute Expiration & Restock:** Scans overdue `reserved` rows, marks them `expired`, notifies customers via WhatsApp, and returns stock quantities to the catalog.
- **Daily 22:00 Low-Stock Reorder Reports:** Compares `stock <= reorder_level` every night and sends consolidated reorder alerts to Telegram and Gmail.
- **Centralized Error Handling:** Captures failures from any workflow, classifies severity and suggested fixes, logs deduplicated fingerprints to Google Sheets, alerts Telegram, and emails admins on `critical` errors.

---

## 📂 Repository Structure

```text
Dawaa-pharmacy/
├── Dawaa 1 - Requests & Reservations.json   # Main WhatsApp intake, AI parsing & Telegram approval flows
├── Error Handling(Dawaa) .json              # Error monitor, 15-min expiry/restock & 22:00 low-stock jobs
├── index.html                               # Interactive web simulator & workflow explorer (GitHub Pages)
└── README.md
```

---

## 🏗️ Workflow Architecture

### 1. `Dawaa 1 - Requests & Reservations.json`
Contains two core pipelines:
1. **Customer Request & Reservation Pipeline:**
   `WhatsApp Trigger` → `Normalize Message` → `Text or Image` → *(Photo download or Text prompt)* → `AI Read Request (Gemini 2.5 Flash)` → `Read Catalog` → `Match Catalog` → `Route Request`:
   - **Nothing Found:** Replies on WhatsApp asking for a clearer medicine name or photo.
   - **Pharmacist Review / Controlled Item:** Saves `pending_pharmacist` rows → Alerts Telegram group with inline Approve/Reject buttons → Informs customer that a pharmacist is reviewing.
   - **Auto-Reserved (OTC):** Saves `reserved` rows → Deducts catalog stock → Sends WhatsApp confirmation.
2. **Pharmacist Decision Callback Pipeline:**
   `Telegram Trigger (callback_query)` → `Parse Decision` → `Find Pending Lines` → `Read Catalog1` → `Apply Decision` → `Answer Query a callback` & `Mark Message Done` → `Line Updates` → `Update Lines` → `If (Approved?)`:
   - **Approved (`true`):** `Stock Rows` → `Take Stock (Approved)` → `Tell Customer: Approved` (WhatsApp).
   - **Rejected (`false`):** `(Rejected)Send message` (WhatsApp).

### 2. `Error Handling(Dawaa) .json`
Contains three automated operations workflows:
1. **Global Error Monitor:**
   `Error Trigger` → `Classify Error` → `Append row in sheet` → `Send a text message` (Telegram) → `If (severity === critical)` → `Send a message` (Gmail).
2. **Reservation Expiration & Restock (Every 15 Minutes):**
   `Schedule Trigger` → `Read Reservations` → `Read Catalog`:
   - **Top Branch:** `Expired lines` → `Update row in sheet` (`status = expired`) → `Send message` (WhatsApp).
   - **Bottom Branch:** `Restock Totals` → `Update row in sheet1` (returns expired quantities to `Catalog`).
3. **Daily Low-Stock Reorder Alert (`0 22 * * *`):**
   `Schedule Trigger1` → `Read Stock` → `Find Low Stock` → `Send a text message1` (Telegram) & `Send a message1` (Gmail).

---

## 📊 Google Sheets Database Schema (`Dawaa Pharmacy Data`)

Create a Google Spreadsheet with three tabs using the exact headers below:

### Tab 1: `Catalog`
| Column | Description | Example |
| :--- | :--- | :--- |
| `sku` | Unique product identifier | `P001` |
| `name` | Medicine brand name | `Panadol` |
| `active_ingredient` | Generic active compound | `paracetamol` |
| `strength` | Dosage strength | `500 mg` |
| `form` | Packaging / form factor | `tablets x24` |
| `price_egp` | Unit price in EGP | `45` |
| `stock` | Current available units | `29` |
| `reorder_level` | Low-stock alert threshold | `10` |
| `controlled` | Requires pharmacist approval (`yes`/`no`) | `no` |

### Tab 2: `Reservations`
| Column | Description | Example |
| :--- | :--- | :--- |
| `res_id` | Unique reservation line ID | `R-261117-H6GY-1` |
| `created_at` | Creation timestamp | `2026-11-17 8:37` |
| `customer_id` | Customer WhatsApp number | `201060034670` |
| `wa_phone_id` | Business WhatsApp Phone ID | `1384774488060812` |
| `customer_name` | Customer profile name | `Mostafa` |
| `sku` | Matched catalog SKU | `P001` |
| `name` | Medicine name | `Panadol` |
| `qty` | Requested quantity | `1` |
| `total_egp` | Line total in EGP | `45` |
| `status` | `reserved`, `pending_pharmacist`, `expired`, `rejected` | `reserved` |
| `expires_at` | Hold expiration timestamp | `2026-11-18 8:34` |
| `pharmacist` | Approving pharmacist username | `sasa50pwg` |
| `note` | Special flags (e.g., `controlled`) | `controlled` |

### Tab 3: `Error logs`
`Time` | `Workflow` | `Node` | `Type` | `Severity` | `Message` | `Suggested Fix` | `Execution ID` | `Execution Link` | `Fingerprint` | `Epoch Minute` | `Alerted` | `Status`

---

## 🚀 Setup & Installation

1. **Prerequisites:**
   - Self-hosted or cloud **n8n** instance (with HTTPS webhook URL, e.g., ngrok or reverse proxy).
   - **Meta WhatsApp Business Cloud API** app & test/live phone number.
   - **Telegram Bot** created via `@BotFather` and added to your pharmacists group.
   - **Google Cloud OAuth2** credentials enabled for Google Sheets & Gmail APIs.
   - **Google Gemini API Key** (`gemini-2.5-flash`).
2. **Import Workflows:**
   - In n8n, go to **Workflows → Import from File** and import both `.json` files.
3. **Configure Credentials & Parameters:**
   - Re-link your Google Sheets, Gmail, WhatsApp Cloud API, Telegram, and Gemini credentials.
   - Update your Telegram **Chat ID**, Gmail recipient address, and Google Sheet document IDs.
   - Set **Workflow Settings → Timezone** to `Africa/Cairo` (or your local timezone) so the `0 22 * * *` cron runs at 22:00 local time.
4. **Link Error Workflow:**
   - Open `Dawaa 1 - Requests & Reservations` → **Settings → Error Workflow** → select `Error Handling(Dawaa)`.
5. **Activate & Test:**
   - Publish both workflows and send a WhatsApp message (e.g., `Panadol 500` or `1x Xanax`) to test both auto-reservation and pharmacist approval flows.

> **Security Note:** Before committing exported n8n `.json` files to a public repository, verify that no hardcoded API keys, bot tokens, webhook secrets, or personal phone numbers remain inside node parameters.

---

## 👨‍💻 Author

**Mostafa Mahmoud (Mostafa Abdelmobdy)** — AI & Workflow Automation Engineer
- GitHub: [@MostafaAbdelmobdy](https://github.com/MostafaAbdelmobdy)
- LinkedIn: [Mostafa Abdelmobdy](https://www.linkedin.com/in/mostafa-abdelmobdy)
- Portfolio: [mostafaabdelmobdy.github.io](https://mostafaabdelmobdy.github.io)
