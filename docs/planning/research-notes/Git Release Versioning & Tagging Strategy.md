# Git Release Versioning & Tagging Strategy

## 1. Overview
This document outlines the standard Release Versioning and Branching Strategy for the Restaurant Management System. Managing multiple phases—starting with the Core Management & POS system, expanding to Self-Checkout, and eventually launching an Online Customer Platform—requires structured version control to maintain stability and clean project history.

---

## 2. Why Use Version Control & Tagging?

1. **Clear Milestones & Project History:**
   Allows tracking the exact code state corresponding to major project milestones (e.g., Version 1.0 vs. Version 2.0).
2. **Stable Rollbacks:**
   If a newly deployed feature in a later version breaks the system, you can quickly checkout and redeploy the previous stable version tag.
3. **Traceability:**
   Helps team members, code reviewers, and recruiters understand the progressive expansion of the project over time.
4. **Automated CI/CD Integration:**
   Modern deployment tools (e.g., GitHub Actions, Docker) can automatically trigger production builds whenever a new version tag (e.g., `v1.0.0`) is pushed.

---

## 3. Industry Standards: Semantic Versioning (SemVer)

We follow **Semantic Versioning (SemVer 2.0.0)**, which uses a `MAJOR.MINOR.PATCH` format (e.g., `v1.0.0`).

$$\text{Format: } \mathbf{vX.Y.Z}$$

* **MAJOR (`X`):** Incremented for major architecture changes or breaking changes (e.g., introducing a new module like Self-Checkout or Online Ordering).
* **MINOR (`Y`):** Incremented for new functionality added in a backward-compatible manner (e.g., adding a new report export feature to the POS).
* **PATCH (`Z`):** Incremented for backward-compatible bug fixes or minor patches.

### Project Roadmap Versioning Plan

| Version Tag | Target Scope / Phase | Description |
| :--- | :--- | :--- |
| **`v1.0.0`** | **Phase 1: Core System & POS** | Admin Panel, Inventory, Menu Management, Staff Access, & POS Billing. |
| **`v1.1.0`** | **Phase 1 Enhancements** | Adding extra analytics or bug fixes to the core POS system. |
| **`v2.0.0`** | **Phase 2: Self-Checkout Kiosk** | Self-Checkout frontend integration & kiosk payment processing. |
| **`v3.0.0`** | **Phase 3: Online Customer Platform** | Web platform for online food ordering, customer authentication, & order tracking. |

---

## 4. Branching Strategy

To keep the repository organized, we use a simplified **Git Flow** branching model:

* **`main` / `master`:** Represents production-ready, stable code. Only merged when a version release is ready.
* **`develop`:** Active development branch where feature branches are merged.
* **`feature/<feature-name>`:** Individual feature branches created from `develop` (e.g., `feature/pos-billing`, `feature/jwt-auth`).

---

## 5. Practical Guide: Step-by-Step Commands

### Step 1: Feature Development
Create a new branch from `develop` for a specific task:
```bash
git checkout develop
git pull origin develop
git checkout -b feature/pos-billing

Work on your feature, commit changes, and push:

```bash
git add .
git commit -m "feat(pos): add cart calculations and bill printing logic"
git push origin feature/pos-billing

### Step 2: Merge into develop
Once the feature is finished and tested:
```bash
git checkout develop
git merge feature/pos-billing
git push origin develop

### Step 3: Preparing a Version Release
When Phase 1 (Core POS) is complete and thoroughly tested:

## 1 Merge develop into main:
```bash
git checkout main
git pull origin main
git merge develop

## 2 Create a Lightweight or Annotated Git Tag:
Annotated tags store the creator name, email, date, and a message.
```bash
git tag -a v1.0.0 -m "Release v1.0.0: Core Restaurant Management & POS System"

## 3 Push the Tag to GitHub:
```bash
git push origin v1.0.0
# Or push all local tags:
git push origin --tags

## 6. Managing & Navigating Releases