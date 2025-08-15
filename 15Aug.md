
### **System Architecture Report: RBAC and User Lifecycle Management**

#### **Executive Summary**

The system implements a robust, event-driven architecture for managing user identity and access control. It is built upon a granular Role-Based Access Control (RBAC) model, where permissions are the fundamental unit of authority. These permissions are aggregated into roles, which are then assigned to users. The user creation lifecycle is handled through a secure, asynchronous invitation flow, ensuring that the core `auth-service` is decoupled from external-facing concerns like email notifications. This design is secure, scalable, and adheres to modern microservice best practices.

---

### **Part 1: The Core Data Model**

The entire RBAC system is built upon a clear and robust data model within the `auth-service`'s PostgreSQL database. The relationships between `User`, `Role`, and `Permission` are the foundation of all security logic.

#### **Entity Relationship Diagram (ERD)**

This diagram illustrates the database schema. The key is the `RolePermission` join table, which creates a many-to-many relationship between Roles and Permissions, allowing for maximum flexibility.

```mermaid
erDiagram
    User {
        string id PK
        string name
        string email
        string password
        boolean isActive
        string invitationToken
        datetime invitationExpires
        string roleId FK
    }
    Role {
        string id PK
        string name
        string description
    }
    Permission {
        string id PK
        string name
        string description
    }
    RolePermission {
        string roleId FK
        string permissionId FK
    }

    User }|--|| Role : "is assigned"
    Role ||--|{ RolePermission : "has"
    Permission ||--|{ RolePermission : "belongs to"
```

**Model Breakdown:**

1.  **Permission:** The most granular unit of authority (e.g., `create:product`, `read:user`). These are static and defined in the seed script. They represent a single action a user can perform.
2.  **Role:** A named collection of permissions (e.g., "Supervisor", "Auditor"). A role defines a job function within the system.
3.  **User:** An individual entity that is assigned exactly one Role. The user inherits all permissions granted by that role.
4.  **RolePermission:** The join table that links Roles and Permissions. This is what allows a single Role to have many Permissions, and a single Permission to be part of many Roles.

---

### **Part 2: The User Invitation & Creation Flow**

This is the most complex process, involving multiple services communicating asynchronously. It is designed to be resilient and decoupled. A user is created in a "pending" state and must activate their own account.

#### **Sequence Diagram: Full Invitation Lifecycle**

This diagram shows the end-to-end process from an administrator's initial click to the new user being fully active in the system.

```mermaid
sequenceDiagram
    actor Admin
    participant API Gateway
    participant Auth Service
    participant Kafka Broker
    participant Notification Service
    actor New User

    Admin->>API Gateway: POST /users/invite (name, email, roleId)
    API Gateway->>Auth Service: FORWARD request

    activate Auth Service
    Note over Auth Service: 1. Verify admin permissions (write:user).
    Auth Service->>Auth DB: INSERT INTO User (isActive: false, password: null)
    Note over Auth Service: 2. Generate and store unique invitationToken.
    Auth Service->>Kafka Broker: PRODUCE event to 'auth_events' topic (type: USER_INVITED)
    Auth Service-->>API Gateway: 201 Created
    API Gateway-->>Admin: 201 Created
    deactivate Auth Service

    Kafka Broker-->>Notification Service: DELIVER USER_INVITED event

    activate Notification Service
    Note over Notification Service: 3. Parse event payload (email, token).
    Notification Service->>SMTP Server (Gmail): SEND invitation email
    deactivate Notification Service

    New User->>Email Client: Opens email, clicks invitation link
    New User->>API Gateway: POST /auth/complete-invitation (token, password)
    API Gateway->>Auth Service: FORWARD request

    activate Auth Service
    Note over Auth Service: 4. Find user by unique invitationToken.
    Auth Service->>Auth DB: SELECT * FROM User WHERE invitationToken = ?
    Note over Auth Service: 5. Hash new password.
    Auth Service->>Auth DB: UPDATE User SET password=?, isActive=true, invitationToken=null
    Note over Auth Service: 6. Generate new JWT for the user.
    Auth Service-->>API Gateway: 200 OK (with JWT)
    API Gateway-->>New User: 200 OK (with JWT)
    deactivate Auth Service
```

---

### **Part 3: Authentication & Authorization Flow**

Once a user is active, their identity and permissions are managed via JSON Web Tokens (JWTs).

#### **Flow 3.1: User Login and JWT Creation**

When a user logs in, the `auth-service` queries the database for all associated permissions and embeds them directly into the JWT payload. This makes the JWT a self-contained "passport" of the user's identity and capabilities.

```mermaid
sequenceDiagram
    actor User
    participant API Gateway
    participant Auth Service
    participant Auth DB

    User->>API Gateway: POST /login (email, password)
    API Gateway->>Auth Service: FORWARD request

    activate Auth Service
    Auth Service->>Auth DB: SELECT * FROM User WHERE email = ?
    Note over Auth Service: 1. Verify password hash with bcrypt.
    Auth Service->>Auth DB: SELECT p.name FROM Permission p JOIN RolePermission rp ON p.id = rp.permissionId WHERE rp.roleId = ?
    Note over Auth Service: 2. Collect all permission strings for the user's role.
    Note over Auth Service: 3. Create JWT payload: { id, name, email, role, permissions: ['read:product', ...] }
    Auth Service-->>API Gateway: 200 OK (with JWT)
    API Gateway-->>User: 200 OK (with JWT)
    deactivate Auth Service
```

#### **Flow 3.2: Authenticated & Authorized API Request**

When the frontend makes a request to a protected page, it includes the JWT. The frontend logic (`WithPermission` HOC and conditional rendering) uses the permission list within the JWT to grant or deny access.

```mermaid
sequenceDiagram
    actor User
    participant Frontend App
    participant API Gateway
    participant Protected Service

    User->>Frontend App: Clicks on "Products" link
    
    activate Frontend App
    Note over Frontend App: 1. Sidebar checks: hasPermission('read:product')? -> YES
    Frontend App->>API Gateway: GET /products (Authorization: Bearer <JWT>)
    deactivate Frontend App

    API Gateway->>Protected Service: FORWARD request

    activate Protected Service
    Note over Protected Service: 2. Middleware verifies JWT signature (optional but good practice).
    Note over Protected Service: 3. Although frontend already checked, the route can re-verify if needed.
    Protected Service->>Protected Service: Perform business logic (e.g., fetch products)
    Protected Service-->>API Gateway: 200 OK (with product data)
    API Gateway-->>Frontend App: 200 OK (with product data)
```

---

### **Part 4: Frontend Implementation**

The frontend implements a two-layer security model to provide a seamless user experience.

1.  **Route-Level Security:** The `WithPermission` HOC acts as a hard gatekeeper for entire pages. If a user tries to access a URL directly, this HOC checks their permissions before rendering the page component.
2.  **Component-Level Security:** Within a page, UI elements (buttons, form fields, links) are conditionally rendered using the `hasPermission` helper from the `authStore`. This ensures a user never even sees an action they are not authorized to perform.

#### **Frontend Logic Flowchart**

This diagram shows the decision process on the frontend for rendering any protected content.

```mermaid
graph TD
    A[User attempts to access a page or view a component] --> B{Is Authenticated?<br/>Token exists and is valid};
    B -- No --> C[Redirect to Login Page];
    B -- Yes --> D{Permission Required?};
    D -- No --> F[Render Content];
    D -- Yes --> E{User has required permission?};
    E -- No --> G[Hide Component OR<br/>Redirect to Dashboard/Error];
    E -- Yes --> F;
```

This comprehensive system ensures that user access is controlled securely, efficiently, and consistently from the database all the way to the final pixel rendered in the user's browser.
