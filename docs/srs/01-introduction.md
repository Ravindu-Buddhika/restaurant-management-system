# Software Requirements Specification (SRS)
## Restaurant Management System

---

## 1. Introduction
### 1.1 Purpose
This document details the functional and non-functional requirements for the Restaurant Management System, covering Phase 1 (POS & Core Management), Phase 2 (Self-Checkout Kiosk), and Phase 3 (Online Platform).

---

## 2. System Features & Functional Requirements

### 2.1 Core System & POS (Version 1.0.0)
* **FR-01: User Authentication & Role Management**
    * System must support Roles: Admin, Manager, Cashier, Kitchen Staff.
* **FR-02: Menu & Category Management**
    * Admins can create, update, and disable menu items with custom prices and variations.
* **FR-03: POS Billing & Receipt Generation**
    * Cashiers can add items to cart, apply discounts, select payment types, and print receipts.

### 2.2 Self-Checkout Kiosk (Version 2.0.0)
* **FR-04: Customer Kiosk Ordering**
    * Customers can browse visually, customize meals, and process card payments directly.

---

## 3. Non-Functional Requirements

* **NFR-01: Performance:** POS transaction response time must be under 1 second.
* **NFR-02: Security:** All passwords must be hashed using BCrypt, and APIs secured via JWT.
* **NFR-03: Availability:** The offline/local POS module should continue functioning if internet connectivity drops.