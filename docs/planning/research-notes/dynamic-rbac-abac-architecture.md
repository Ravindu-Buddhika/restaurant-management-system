# Dynamic Granular Role-Based Access Control (RBAC) / Attribute-Based Access Control (ABAC)

In modern software engineering, instead of hardcoding user roles (like `ADMIN`, `FINANCE_MANAGER`, or `SALES_REP`), production systems implement a flexible design called **Dynamic Granular Role-Based Access Control (RBAC)** or **Attribute-Based Access Control (ABAC)**.

This approach completely **decouples** the system's actions (**Permissions**) from user-defined labels (**Roles**).

---

## 1. Database Schema Design (Decoupling Roles & Permissions)

To allow client organizations to create custom roles with mixed permissions (e.g., combining Production and HR), the system stores atomic operations separately from user-created roles.

### Schema Blueprint

```
+------------------+         +-----------------------+         +------------------+
|   permissions    |         |   role_permissions    |         |      roles       |
+------------------+         +-----------------------+         +------------------+
| id (PK)          |<-------*| role_id (FK)          |*------->| id (PK)          |
| code (UNIQUE)    |         | permission_code (FK)  |         | name             |
| module           |         +-----------------------+         | tenant_id        |
| description      |                                           +------------------+
+------------------+                                                    ^
                                                                         |
                                                               +------------------+
                                                               |      users       |
                                                               +------------------+
                                                               | id (PK)          |
                                                               | role_id (FK)     |
                                                               | name             |
                                                               +------------------+
```

### Table Definitions & Data Examples

**`permissions`** (System-defined, immutable by end-users)
Stores every unique, atomic action available within the application.

| id | code                     | module     | description                |
|----|--------------------------|------------|-----------------------------|
| 1  | finance:reports:read     | Finance    | View financial statements   |
| 2  | finance:invoice:create   | Finance    | Create new invoices         |
| 3  | production:batch:create  | Production | Start a production batch    |
| 4  | hr:employee:view         | HR         | View employee directory     |

**`roles`** (User-created dynamic records)
Created dynamically by shop owners or system admins via the UI.

| id       | name                     | tenant_id |
|----------|--------------------------|-----------|
| role_101 | Floor Supervisor & HR Asst | shop_a  |
| role_102 | Store Manager            | shop_b    |

**`role_permissions`** (Junction Table)
Maps granular permissions to custom roles.

| role_id  | permission_code           |
|----------|----------------------------|
| role_101 | production:batch:create   |
| role_101 | hr:employee:view           |

---

## 2. Authentication Payload (JWT / Session)

When a user logs in, the backend fetches their assigned role, looks up the active permission codes from `role_permissions`, and embeds them into the JWT token payload.

```json
{
  "sub": "user_8923",
  "name": "Nimal Perera",
  "tenantId": "shop_a",
  "role": "Floor Supervisor & HR Asst",
  "permissions": [
    "production:batch:create",
    "hr:employee:view"
  ]
}
```

---

## 3. Backend Enforcement (Code Implementation)

**Never** write logic like `if (user.role == "PRODUCTION_MANAGER")`. Instead, evaluate whether the permission string exists in the user's execution context.

### Java / Spring Boot Example

```java
@RestController
@RequestMapping("/api/v1/production")
public class ProductionController {

    // Checked via authority string directly loaded from JWT permissions array
    @PreAuthorize("hasAuthority('production:batch:create')")
    @PostMapping("/batches")
    public ResponseEntity<BatchResponse> createBatch(@RequestBody CreateBatchRequest request) {
        return ResponseEntity.ok(productionService.createNewBatch(request));
    }
}
```

### Node.js / Express Middleware Example

```javascript
// Middleware to evaluate specific permission strings dynamically
const requirePermission = (permissionCode) => {
  return (req, res, next) => {
    const userPermissions = req.user?.permissions || [];

    if (!userPermissions.includes(permissionCode)) {
      return res.status(403).json({ 
        error: "AccessDenied", 
        message: `Missing required permission: ${permissionCode}` 
      });
    }
    
    next();
  };
};

// Route attachment
app.post(
  '/api/v1/hr/employees', 
  requirePermission('hr:employee:create'), 
  createEmployeeController
);
```

---

## 4. Frontend UI Guarding (React)

Dynamic rendering relies on helper components or custom hooks that query the stored permissions list.

```javascript
import React from 'react';
import { useAuth } from './AuthContext';

// Helper component for declarative permission checks
export const HasPermission = ({ code, children }) => {
  const { permissions } = useAuth();
  return permissions.includes(code) ? children : null;
};

// Dashboard component
export const Dashboard = () => {
  return (
    <div className="dashboard-grid">
      <HasPermission code="production:batch:create">
        <button className="btn-primary">Start New Batch</button>
      </HasPermission>

      <HasPermission code="finance:reports:read">
        <FinancialReportWidget />
      </HasPermission>
    </div>
  );
};
```

---

## 5. Advanced Fine-Grained Constraints (ABAC)

When a boolean permission check is insufficient (e.g., "User can create discount invoices, but only up to $500"), store **JSON Constraint Metadata** inside `role_permissions` or a user settings table.

### Extended Mapping Schema

```sql
CREATE TABLE role_permissions (
    role_id VARCHAR(50),
    permission_code VARCHAR(100),
    constraints JSONB DEFAULT NULL,
    PRIMARY KEY (role_id, permission_code)
);
```

### Sample Constraint JSON Data

```json
{
  "permission": "sales:discount:apply",
  "constraints": {
    "maxDiscountPercentage": 15,
    "maxDiscountAmountUSD": 500.00,
    "allowedLocations": ["STORE_COLOMBO_01", "STORE_COLOMBO_02"]
  }
}
```

### Evaluation Logic

```java
public boolean validateDiscount(UserPrincipal user, double requestedDiscountAmount) {
    PermissionConstraint constraint = user.getConstraintFor("sales:discount:apply");
    
    if (constraint == null) return false;
    
    double maxLimit = constraint.getNumericValue("maxDiscountAmountUSD");
    return requestedDiscountAmount <= maxLimit;
}
```

---

## Summary

This architecture allows end-users to create **any custom combination of business functions** without requiring a code redeployment or backend modification. By separating atomic **permissions** (system-defined, fixed) from **roles** (dynamic, tenant-specific groupings of permissions), organizations can:

- Let admins build custom roles on the fly through the UI
- Mix permissions across modules (e.g., Production + HR) into a single role
- Enforce access checks by permission string, not by hardcoded role name
- Add fine-grained constraints (like discount limits or allowed locations) on top of simple boolean permissions
- Scale to multi-tenant systems where each tenant defines its own roles independently
