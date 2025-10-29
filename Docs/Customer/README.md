# 🧭 CUSTOMER CONTEXT — DOMAIN LAYER DESIGN DOCUMENTATION (MVP VERSION)

### Context Overview

The **Customer Context** manages all customer-related domain concerns within the e-commerce system.
Its primary responsibilities include:

* Managing customer registration and identity.
* Managing contact information and delivery addresses.
* Managing profile updates (email, phone number, address).
* Emitting domain events related to customer lifecycle changes.

The context is modeled as a **bounded context** under the **Customer** subdomain.

---

## 1. Aggregate and Entity Model

### 1.1 Aggregate Root: `Customer`

**Description:**
Represents an individual customer in the system.
It is the main entry point for all customer-related business logic and state transitions.

**Responsibilities:**

* Maintain customer identity and contact details.
* Manage the lifecycle of a customer profile.
* Trigger domain events upon registration or profile update.
* Ensure domain invariants through value objects and internal validation.

**Structure:**

```text
Customer (Aggregate Root)
 ├── CustomerId (Value Object)
 ├── Name (Value Object)
 │     ├── FirstName
 │     └── LastName
 ├── Email (Value Object)
 ├── PhoneNumber (Value Object)
 ├── Address (Value Object)
 │     ├── Street
 │     ├── City
 │     ├── Region
 │     ├── LandMark
 ├── RegistrationDate (Value Object)
 ├── isVerified (boolean)
```

---

## 2. Entities and Value Objects

### 2.1 Entity: `Customer`

#### Fields:

| Field              | Type               | Description                                 |
| ------------------ | ------------------ | ------------------------------------------- |
| `id`               | `CustomerId`       | Unique identity of the customer.            |
| `name`             | `Name`             | Customer’s full name.                       |
| `email`            | `Email`            | Email for communication and authentication. |
| `phoneNumber`      | `PhoneNumber`      | Validated contact number.                   |
| `address`          | `Address`          | Primary delivery address.                   |
| `registrationDate` | `RegistrationDate` | Date the customer registered.               |
| `isVerified`       | `bool`             | Indicates if the account is verified.       |

#### Methods (Behavior):

| Method                                        | Description                                                       |
| --------------------------------------------- | ----------------------------------------------------------------- |
| `register(Name, Email, PhoneNumber, Address)` | Registers a new customer and raises a `CustomerRegistered` event. |
| `updateContactInfo(Email, PhoneNumber)`       | Updates email and phone number after validation.                  |
| `updateAddress(Address)`                      | Updates address while maintaining domain integrity.               |
| `markVerified()`                              | Marks a customer as verified.                                     |
| `getFullName()`                               | Returns concatenated full name.                                   |
| `equals(Customer)`                            | Identity equality check based on `CustomerId`.                    |

#### Domain Invariants:

* A customer **must have** a valid name, email, and phone number.
* The address must always be **non-null** and valid.
* The customer’s identity is immutable after creation.

---

## 3. Value Objects

Each value object enforces its own validation logic, immutability, and equality rules.

### 3.1 `CustomerId`

**Responsibilities:**

* Guarantees uniqueness and identity comparison.

```text
+ id: UUID
+ equals(CustomerId): bool
+ toString(): string
```

### 3.2 `Name`

**Responsibilities:**

* Represents a valid human-readable name.
* Enforces non-empty strings and alphabetic validation.

```text
+ firstName: string
+ lastName: string
+ getFullName(): string
```

**Invariants:**

* `firstName` and `lastName` cannot be null or empty.
* Must contain only alphabetic characters (with optional accents).

---

### 3.3 `Email`

**Responsibilities:**

* Represents a properly formatted email address.
* Enforces email regex validation.

```text
+ value: string
+ validateFormat()
+ equals(Email): bool
```

**Invariant:**

* Must match standard email format (e.g., `user@example.com`).

---

### 3.4 `PhoneNumber`

**Responsibilities:**

* Represents a valid phone number in international format.
* Enforces country prefix (e.g., `+255...` for Tanzania).

```text
+ value: string
+ validateFormat()
+ equals(PhoneNumber): bool
```

**Invariant:**

* Must follow E.164 standard (`+255XXXXXXXXX`).

---

### 3.5 `Address`

**Responsibilities:**

* Represents the delivery or primary contact address.

```text
+ street: string
+ city: string
+ region: string
+ landmark: string
+ fullAddress(): string
```

**Invariants:**

* `city`, `region`, `street`, and `landmark` cannot be empty.

---

### 3.6 `RegistrationDate`

**Responsibilities:**

* Represents the date/time of registration.

```text
+ value: datetime
```

**Invariant:**

* Must always be set during registration, never null.

---

## 4. Domain Events

### 4.1 `CustomerRegistered`

**Description:**
Emitted when a new customer successfully registers.

| Property     | Type         | Description                   |
| ------------ | ------------ | ----------------------------- |
| `customerId` | `CustomerId` | ID of the registered customer |
| `name`       | `Name`       | Full name                     |
| `email`      | `Email`      | Registered email              |
| `timestamp`  | `datetime`   | Event emission time           |

**Event Trigger:**
Emitted within the `Customer.register()` method after successful creation.

**Purpose:**
To notify other bounded contexts (e.g., Notification, Marketing, or Order) that a new customer has joined the system.

---

## 5. Domain Service Interfaces

Domain services are stateless and encapsulate logic that doesn’t belong to a specific entity or value object.

### 5.1 `ICustomerDomainService`

Handles cross-entity business logic or policies.

**Example Responsibilities:**

* Verify if a given email or phone number already exists (delegated to repository).
* Apply complex validation rules for registration beyond a single entity.

```text
interface ICustomerDomainService {
    bool emailExists(Email email);
    bool phoneExists(PhoneNumber phone);
}
```

---

## 6. Repository Interfaces

Repositories abstract persistence and data access logic while remaining domain-layer agnostic.

### 6.1 `ICustomerRepository`

**Responsibilities:**

* Provide access to `Customer` aggregates.
* Handle persistence while isolating infrastructure details.

**Interface Definition:**

```text
interface ICustomerRepository {
    CustomerId nextIdentity();
    void save(Customer customer);
    Optional<Customer> findById(CustomerId id);
    Optional<Customer> findByEmail(Email email);
    Optional<Customer> findByPhone(PhoneNumber phone);
}
```

---

## 7. Enums

### 7.1 `VerificationStatus`

**Represents the verification state of the customer.**

```text
enum VerificationStatus {
    PENDING,
    VERIFIED
}
```

*(Note: This may be embedded into `Customer` as `isVerified` boolean for simplicity in MVP0.)*

---

## 8. Factories (Optional for MVP)

In future iterations, factories can encapsulate customer creation logic if it grows complex.

Example:

```text
CustomerFactory.create(Name, Email, PhoneNumber, Address)
```

---

## 9. Summary of the Domain Layer Components

| Type           | Name                                                                        | Purpose                                   |
| -------------- | --------------------------------------------------------------------------- | ----------------------------------------- |
| Aggregate Root | `Customer`                                                                  | Core entry point for customer management. |
| Entity         | `Customer`                                                                  | Represents individual customer record.    |
| Value Objects  | `CustomerId`, `Name`, `Email`, `PhoneNumber`, `Address`, `RegistrationDate` | Enforce immutability and validation.      |
| Domain Event   | `CustomerRegistered`                                                        | Signals customer registration.            |
| Domain Service | `ICustomerDomainService`                                                    | Handles cross-entity validation.          |
| Repository     | `ICustomerRepository`                                                       | Provides persistence abstraction.         |
| Enum           | `VerificationStatus`                                                        | Tracks verification state.                |

---

## 10. Design Principles Applied

| Principle                   | Description                                                                                      |
| --------------------------- | ------------------------------------------------------------------------------------------------ |
| **DDD Aggregation**         | Customer acts as the aggregate root, ensuring encapsulated consistency.                          |
| **Value Object Integrity**  | Each field encapsulates its own validation logic.                                                |
| **KISS**                    | Only essential domain logic is represented; no premature complexity.                             |
| **DRY**                     | Shared logic (validation, identity, equality) is centralized in value objects.                   |
| **Infrastructure Agnostic** | Repositories and services are interfaces; no dependency on frameworks or persistence mechanisms. |
| **Event-Driven Readiness**  | Domain events prepare the model for future integrations (notifications, analytics, etc.).        |
