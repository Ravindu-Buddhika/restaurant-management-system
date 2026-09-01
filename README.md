# Multi-Channel Restaurant Management System

A robust, full-stack enterprise solution designed to manage complex restaurant operations across multiple ordering channels including Cashier/POS checkout, Online customer ordering, and Self-service table ordering.

---

## 🏗️ Architecture & Tech Stack

- **Frontend:** React.js, Tailwind CSS, Axios, React Router
- **Backend:** Java, Spring Boot (Spring Security, Spring Data JPA, REST APIs)
- **Database & Services:** Supabase (PostgreSQL), Supabase Realtime Engine
- **Authentication & Security:** JWT / OAuth2 via Spring Security

---

## 🔥 Key Features

- **Multi-Channel Order Processing:**
    - **POS / Cashier Checkout:** Fast in-house billing, bill splitting, and table settlement.
    - **Online Customer Ordering:** Direct menu browsing, cart management, and order placement.
    - **Self-Ordering System:** Table-based QR/Web self-ordering interface for dine-in guests.
- **Realtime Kitchen Display System (KDS):** Direct order routing to the kitchen using realtime triggers.
- **Inventory & Stock Tracking:** Automatic stock updates upon order confirmation.
- **Role-Based Access Control (RBAC):** Distinct permissions for Admin, Cashier, Kitchen Staff, and Customers.

---

## 🛠️ Project Structure

```text
├── restaurant-management-backend/   # Spring Boot Application
├── restaurant-management-frontend/  # React Web Application
└── README.md