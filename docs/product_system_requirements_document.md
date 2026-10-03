# Product & System Requirements Document (PRD / SRD)

**Project Name:** Online Bookstore & Publishing Platform  
**Target Architecture:** Full-Stack Next.js (App Router), PostgreSQL (Prisma/Drizzle ORM), Flutterwave API  
**Deployment Target:** Vercel (or Node.js server) + Managed PostgreSQL (e.g., Supabase, Neon)

---

## 1. Executive Summary & Vision

The objective of this project is to build a scalable, mobile-responsive web application for online book sales and author/publisher self-management. The platform features an e-commerce storefront for customers, self-service portals for publishers, dynamic financial monitoring, and centralized administrative controls. 

### Key Technical Highlight: Dynamic Secret Key Injection
To meet the requirement of storing the Flutterwave Secret Key dynamically in the database via the Admin Dashboard (rather than standard `.env` variables), all payment request signatures and API calls are computed on the Next.js server runtime by querying encrypted settings records from PostgreSQL.

---

## 2. User Roles & Role-Based Access Control (RBAC)

The system strictly enforces RBAC at both the UI route level (middleware) and Server Action / API route level.

### User Roles Overview
*   **System Admin:** Full system management, secret key configuration, user provisioning, global platform settings, and book approval.
*   **Finance:** Monitoring transactions, payout reconciliations, revenue reporting, and approving author withdrawals.
*   **Publisher:** Book creation and management, inventory tracking, sales analytics, and payout requests.
*   **Customer / Guest:** Browsing, searching, adding to cart, completing checkout via Flutterwave, and viewing/downloading order receipts.

### RBAC Matrix

| Feature / Resource | Guest / Customer | Publisher | Finance | System Admin |
| :--- | :---: | :---: | :---: | :---: |
| Browse / Search Catalog | Read | Read | Read | Read |
| Add to Cart & Checkout (Flutterwave) | Write | Write | Write | Write |
| Submit Books for Listing | - | Write | - | Write/Approve |
| View Personal Book Sales | - | Read | - | Read |
| View All Platform Transactions | - | - | Read | Read |
| Approve / Process Payouts | - | - | Write | Write |
| Configure Flutterwave API Keys | - | - | - | **Full Access** |
| Manage System Users & Roles | - | - | - | **Full Access** |

---

## 3. Core Functional Requirements

### 3.1 Public Storefront & Customer Experience
*   **Hero Landing Page:**
    *   Dynamic banner featuring seasonal or promotional titles.
    *   **Featured Books Carousel:** Curated section controlled by Admin or driven by sales volume.
    *   **New Arrivals Grid:** Real-time query of recently published titles with cover images, prices, author names, and star ratings.
*   **Search & Filtering:** Multi-faceted filtering by genre, author, price range, format (e.g., Hardcover, Paperback, E-book), and publication date.
*   **Book Details Page:** Book summaries, metadata (ISBN, publisher, pages), customer reviews, stock availability, and immediate "Buy Now" / "Add to Cart" actions.
*   **Shopping Cart & Checkout:** Persistent local/session-based cart, address input, price breakdown (subtotal, tax, delivery fee), and seamless Flutterwave gateway modal or redirect.

### 3.2 Dynamic Payment Gateway Integration (Flutterwave)
*   **Admin API Key Storage:**
    *   Admin inputs `FLUTTERWAVE_PUBLIC_KEY`, `FLUTTERWAVE_SECRET_KEY`, and `FLUTTERWAVE_ENCRYPTION_KEY` into a secure admin form.
    *   Keys are encrypted using AES-256-GCM before being stored in the PostgreSQL `system_settings` table.
    *   Next.js API route fetches and decrypts keys at runtime on the server to instantiate payment payloads.
*   **Payment Flow:**
    1. Customer confirms checkout.
    2. Server creates a pending `Order` record and initializes a Flutterwave transaction using stored DB keys.
    3. Customer completes payment on Flutterwave.
    4. Flutterwave posts a payload to `/api/webhooks/flutterwave`.
    5. Server validates the signature header using the DB-stored secret key, updates `Order` status to `PAID`, adjusts book inventory, and releases downloads or triggers delivery workflow.

### 3.3 Dashboard Modules by Role

#### System Admin Dashboard
*   **API Key Management:** Encrypted forms for Flutterwave keys, webhook secret verification, and live/test mode toggles.
*   **User Management:** Provision, deactivate, or assign roles (`System Admin`, `Finance`, `Publisher`).
*   **Catalog Approval:** Review and approve/reject book listings submitted by Publishers.
*   **System Audit Logs:** Log critical administrative actions (e.g., changing payment keys, changing user roles).

#### Finance Dashboard
*   **Transaction Register:** Live feed of incoming purchases with search/filter by order ID, customer name, date, and status (`SUCCESS`, `PENDING`, `FAILED`).
*   **Payout Management:** Review sales balances accrued by Publishers, verify bank details, and mark payouts as completed or execute Flutterwave transfers.
*   **Revenue Analytics:** Charts showing Gross Merchandise Value (GMV), platform fee cuts, and net publisher payouts over customizable date ranges.

#### Publisher Dashboard
*   **Book Catalog Manager:** Interface to upload cover images, add metadata, upload digital assets (PDF/ePub for e-books), and manage physical inventory counts.
*   **Sales Performance Analytics:** Real-time graphs showing units sold, revenue generated, and royalty breakdown per book.
*   **Payout Requests:** Request withdrawal of earnings and register payout bank accounts.

---

## 4. Mobile Responsiveness & Design System Requirements

All views must adhere to a **Mobile-First Responsive Design Strategy** using Fluid CSS / Tailwind CSS breakpoints (`sm: 640px`, `md: 768px`, `lg: 1024px`, `xl: 1280px`).

*   **Navigation Architecture:**
    *   **Mobile (`< 768px`):** Slide-out drawer menu (hamburger), sticky bottom bar for Quick Cart / Search, and collapsible dashboard sidebar headers.
    *   **Desktop (`>= 768px`):** Top bar with category dropdowns and full left-rail sidebar for dashboards.
*   **Data Tables (Finance & Admin):** Dynamic switching from full horizontal data grids on desktop to card-based stacks on mobile screens to eliminate horizontal scrolling issues.
*   **Touch Targets:** Minimum interactive touch size of $44 \times 44\text{ px}$ across mobile viewports.

---

## 5. Database Schema (PostgreSQL)

```sql
-- Role Enum
CREATE TYPE user_role AS ENUM ('SYSTEM_ADMIN', 'FINANCE', 'PUBLISHER', 'CUSTOMER');
CREATE TYPE order_status AS ENUM ('PENDING', 'PAID', 'CANCELLED', 'REFUNDED');
CREATE TYPE book_status AS ENUM ('DRAFT', 'PENDING_APPROVAL', 'PUBLISHED', 'ARCHIVED');

-- Users Table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    role user_role NOT NULL DEFAULT 'CUSTOMER',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- System Settings (Dynamic Keys)
CREATE TABLE system_settings (
    key VARCHAR(100) PRIMARY KEY,
    value_encrypted TEXT NOT NULL,
    iv VARCHAR(255) NOT NULL,
    updated_by UUID REFERENCES users(id),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Books Table
CREATE TABLE books (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    publisher_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    isbn VARCHAR(20) UNIQUE,
    author VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    cover_image_url TEXT,
    file_url TEXT, -- For e-books
    is_featured BOOLEAN DEFAULT false,
    status book_status DEFAULT 'DRAFT',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Orders Table
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID REFERENCES users(id),
    total_amount DECIMAL(10, 2) NOT NULL,
    status order_status DEFAULT 'PENDING',
    tx_ref VARCHAR(100) UNIQUE NOT NULL, -- Flutterwave Tx Ref
    flw_transaction_id VARCHAR(100),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Order Items Table
CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID REFERENCES orders(id) ON DELETE CASCADE,
    book_id UUID REFERENCES books(id),
    quantity INT NOT NULL,
    price_at_purchase DECIMAL(10, 2) NOT NULL
);
```

---

## 6. Architecture & Tech Stack

*   **Frontend & Framework:** Next.js 14+ (App Router, Server Actions, React Server Components).
*   **Styling:** Tailwind CSS + Shadcn/UI for accessible, mobile-first design.
*   **Database & ORM:** PostgreSQL with Prisma ORM or Drizzle ORM.
*   **Authentication:** NextAuth.js (Auth.js) or Lucia Auth with JWT strategy storing user `role` in session token.
*   **Payment Gateway:** Flutterwave v3 API via custom server HTTP client.
*   **Security & Encryption:** Node.js `crypto` module (`aes-256-gcm`) for encrypting/decrypting system configuration settings at rest.

---

## 7. Dynamic API Key Management Implementation Design

To bypass environment variables and maintain payment configuration within the PostgreSQL database via Admin settings, implement a dedicated encryption service wrapper:

### Key Retrieval Flow
1. Admin enters Flutterwave secret key in Dashboard.
2. Next.js Server Action encrypts key using an internal master key (`MASTER_ENCRYPTION_KEY` — the sole static secret in `.env` or derived via KMS) and saves the encrypted string + IV to `system_settings`.
3. When initialized, the `PaymentService` queries the database for the active key, decrypts it, and invokes Flutterwave endpoints.

```typescript
// lib/payments/flutterwave.ts
import { db } from "@/lib/db";
import { decryptSecret } from "@/lib/crypto";

export async function getFlutterwaveClient() {
  const settingsRecord = await db.systemSettings.findUnique({
    where: { key: "FLUTTERWAVE_SECRET_KEY" }
  });

  if (!settingsRecord) {
    throw new Error("Flutterwave Secret Key is not configured in Admin Dashboard.");
  }

  const secretKey = decryptSecret(settingsRecord.value_encrypted, settingsRecord.iv);
  
  return {
    initializePayment: async (payload: PaymentPayload) => {
      const response = await fetch("https://api.flutterwave.com/v3/payments", {
        method: "POST",
        headers: {
          Authorization: `Bearer ${secretKey}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify(payload),
      });
      return response.json();
    }
  };
}
```

---

## 8. Non-Functional Requirements (NFRs)

*   **Performance:**
    *   First Contentful Paint (FCP) $< 1.2\text{s}$ on 4G networks.
    *   Lighthouse Mobile Score $\ge 90$ across Performance, Accessibility, and Best Practices.
*   **Security:**
    *   Strict HTTPS enforcement.
    *   Role verification middleware on all `/dashboard/*` routes.
    *   Webhook signature matching to prevent spoofed purchase notifications.
    *   Input sanitization to prevent XSS and SQL injection.
*   **Scalability:**
    *   Stateless application layer hosted on edge environments.
    *   Indexed PostgreSQL fields on `isbn`, `publisher_id`, `status`, `tx_ref`, and `created_at`.