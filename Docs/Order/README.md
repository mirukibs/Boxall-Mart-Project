# 🚚 ORDER CONTEXT — DOMAIN LAYER DESIGN DOCUMENTATION (MVP VERSION)

---

## Context Overview

The **Order Context** governs the lifecycle of customer purchases, from cart creation to checkout, order creation, and delivery.
Its primary responsibilities include:

* Managing the customer's cart and items within it.
* Handling the creation and management of orders.
* Maintaining delivery and payment linkage.
* Ensuring domain invariants such as order totals, valid states, and transitions.

The context is modeled as a **bounded context** under the **Order** subdomain.

---

## 1. Aggregate and Entity Model

### 1.1 Aggregate Root: `Order`

**Description:**
Represents a customer's confirmed purchase.
Encapsulates all order-related state including items, customer reference, delivery information, and payment linkage.

**Responsibilities:**

* Maintain order integrity and invariants.
* Manage state transitions (`Created`, `Dispatched`, `InTransit`, `Delivered`).
* Calculate total costs (including delivery).
* Emit relevant domain events (e.g., `OrderCreated`, `OrderDispatched`).

**Structure:**

```text
Order (Aggregate Root)
 ├── OrderId (Value Object)
 ├── CustomerId (Value Object)
 ├── CartId (Value Object)
 ├── Items (List<OrderItem>)
 ├── TotalCost (Value Object)
 ├── DeliveryCost (Value Object)
 ├── DeliveryNotes (Value Object)
 ├── TransportMethod (Enum)
 ├── EstimatedDeliveryTime (Value Object)
 ├── PaymentId (Value Object)
 ├── Status (Enum)
 ├── CreatedAt (Value Object)
 ├── UpdatedAt (Value Object)
```

---

### 1.2 Entity: `Cart`

**Description:**
A temporary holding space for items before checkout.
Once checked out, it becomes the basis for an `Order`.

**Responsibilities:**

* Manage product additions and removals.
* Maintain accurate total cost and weight.
* Validate cart readiness for checkout.

**Structure:**

```text
Cart (Entity)
 ├── CartId (Value Object)
 ├── CustomerId (Value Object)
 ├── Items (List<CartItem>)
 ├── TotalCost (Value Object)
 ├── TotalWeight (Value Object)
 ├── CreatedAt (Value Object)
 ├── UpdatedAt (Value Object)
```

---

### 1.3 Entity: `OrderItem`

**Description:**
Represents a single product line within an order.

**Structure:**

```text
OrderItem (Entity)
 ├── ProductId (Value Object)
 ├── ProductName (Value Object)
 ├── Quantity (Value Object)
 ├── UnitPrice (Value Object)
 ├── Subtotal (Value Object)
 ├── Weight (Value Object)
```

---

### 1.4 Entity: `CartItem`

**Description:**
Represents a product temporarily stored in a customer's cart before checkout.

**Structure:**

```text
CartItem (Entity)
 ├── ProductId (Value Object)
 ├── ProductName (Value Object)
 ├── Quantity (Value Object)
 ├── UnitPrice (Value Object)
 ├── Weight (Value Object)
 ├── Subtotal (Value Object)
```

---

## 2. Value Objects

| Value Object              | Description                                     | Core Validation                          |
| ------------------------- | ----------------------------------------------- | ---------------------------------------- |
| **OrderId**               | Unique identity for an order.                   | Must be UUID.                            |
| **CartId**                | Unique identity for a cart.                     | Must be UUID.                            |
| **CustomerId**            | Reference to a registered customer.             | Must not be null.                        |
| **ProductId**             | Reference to product from Catalog Context.      | Must not be null.                        |
| **ProductName**           | Product name stored at order time.              | Non-empty.                               |
| **Quantity**              | Number of product units.                        | Must be ≥ 1.                             |
| **UnitPrice**             | Price per unit (TZS).                           | Must be > 0.                             |
| **Subtotal**              | Computed total per line = UnitPrice × Quantity. | Must equal computation.                  |
| **TotalCost**             | Total order cost.                               | Must equal sum of items + delivery cost. |
| **DeliveryCost**          | Cost of delivery.                               | ≥ 0.                                     |
| **TotalWeight**           | Total combined weight of products in cart.      | ≥ 0.                                     |
| **DeliveryNotes**         | Optional text instructions.                     | ≤ 500 chars.                             |
| **EstimatedDeliveryTime** | Predicted delivery timestamp.                   | Must be future date.                     |
| **PaymentId**             | Reference to payment transaction.               | Optional at creation.                    |
| **CreatedAt / UpdatedAt** | Audit timestamps.                               | Auto-generated.                          |

---

## 3. Enums

### 3.1 `OrderStatus`

Represents lifecycle stages of an order.

```text
enum OrderStatus {
    CREATED,
    DISPATCHED,
    IN_TRANSIT,
    DELIVERED,
    CANCELLED
}
```

### 3.2 `TransportMethod`

Represents the transport type based on item characteristics.

```text
enum TransportMethod {
    BIKE,
    CAR,
    TRUCK
}
```

---

## 4. Domain Invariants

* An `Order` must always have a **CustomerId**, **Items**, and **TotalCost**.
* `TotalCost` = sum of all `OrderItem.Subtotal` + `DeliveryCost`.
* Order transitions follow strict sequence:
  `CREATED → DISPATCHED → IN_TRANSIT → DELIVERED`
  (No skipping allowed.)
* A `Cart` cannot checkout if empty.
* Product quantity in cart cannot exceed available stock (to be validated by domain service).

---

## 5. Behaviors / Methods

### 5.1 `Order` Aggregate Root

| Method                                                                            | Description                                                      |
| --------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| `create(CartId, CustomerId, Items, DeliveryCost, TransportMethod, DeliveryNotes)` | Initializes a new order from a cart. Emits `OrderCreated` event. |
| `markAsDispatched()`                                                              | Changes status to `DISPATCHED`.                                  |
| `markAsInTransit()`                                                               | Changes status to `IN_TRANSIT`.                                  |
| `markAsDelivered()`                                                               | Changes status to `DELIVERED`.                                   |
| `cancel()`                                                                        | Cancels the order if not yet delivered.                          |
| `calculateTotalCost()`                                                            | Computes total from items + delivery.                            |
| `equals(Order)`                                                                   | Identity comparison via `OrderId`.                               |

---

### 5.2 `Cart` Entity

| Method                                                         | Description                               |
| -------------------------------------------------------------- | ----------------------------------------- |
| `addItem(ProductId, ProductName, Quantity, UnitPrice, Weight)` | Adds or updates a cart item.              |
| `removeItem(ProductId)`                                        | Removes product from cart.                |
| `updateItemQuantity(ProductId, Quantity)`                      | Updates quantity for an item.             |
| `calculateTotals()`                                            | Updates total cost and weight.            |
| `clear()`                                                      | Empties the cart.                         |
| `isEmpty()`                                                    | Returns true if no items exist.           |
| `checkout()`                                                   | Converts cart to order (through service). |

---

## 6. Domain Events

| Event             | Description                         | Trigger                    |
| ----------------- | ----------------------------------- | -------------------------- |
| `OrderCreated`    | Raised when a new order is created. | After `Order.create()`     |
| `OrderDispatched` | Raised when an order is dispatched. | After `markAsDispatched()` |
| `OrderDelivered`  | Raised when an order is delivered.  | After `markAsDelivered()`  |
| `OrderCancelled`  | Raised when an order is cancelled.  | After `cancel()`           |

**Example:**

```text
class OrderCreated {
    + orderId: OrderId
    + customerId: CustomerId
    + totalCost: TotalCost
    + occurredAt: datetime
}
```

---

## 7. Domain Service Interfaces

Domain services encapsulate logic that doesn’t fit neatly within a single entity.

### 7.1 `IOrderDomainService`

**Responsibilities:**

* Validate transport method choice based on weight and nature.
* Coordinate cost calculations and delivery time estimation.

```text
interface IOrderDomainService {
    TransportMethod determineTransportMethod(TotalWeight weight);
    EstimatedDeliveryTime estimateDeliveryTime(TransportMethod method);
    TotalCost calculateTotal(List<OrderItem> items, DeliveryCost delivery);
}
```

---

### 7.2 `ICartDomainService`

**Responsibilities:**

* Validate cart consistency and readiness for checkout.

```text
interface ICartDomainService {
    bool canCheckout(Cart cart);
}
```

---

## 8. Repository Interfaces

Repositories abstract persistence and provide aggregate-level access.

### 8.1 `IOrderRepository`

```text
interface IOrderRepository {
    OrderId nextIdentity();
    void save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomer(CustomerId customerId);
}
```

### 8.2 `ICartRepository`

```text
interface ICartRepository {
    CartId nextIdentity();
    void save(Cart cart);
    Optional<Cart> findById(CartId id);
    Optional<Cart> findByCustomer(CustomerId customerId);
    void remove(CartId id);
}
```

---

## 9. Design Principles Applied

| Principle                   | Description                                                                                        |
| --------------------------- | -------------------------------------------------------------------------------------------------- |
| **DDD Aggregation**         | `Order` is the aggregate root; `OrderItem` and related VOs form its internal consistency boundary. |
| **Value Object Integrity**  | All fields (price, quantity, notes, etc.) encapsulate validation internally.                       |
| **KISS**                    | Keeps order lifecycle simple and explicit for MVP.                                                 |
| **DRY**                     | Shared logic (e.g., total calculation) centralized in services and value objects.                  |
| **Infrastructure Agnostic** | Domain is fully decoupled from persistence or frameworks.                                          |

---

## 10. Summary of Domain Layer Components

| Type                | Name                                                                                       | Purpose                                           |
| ------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------- |
| **Aggregate Root**  | `Order`                                                                                    | Central order lifecycle management.               |
| **Entities**        | `Cart`, `OrderItem`, `CartItem`                                                            | Support aggregate operations and cart management. |
| **Value Objects**   | `OrderId`, `CartId`, `CustomerId`, `ProductId`, `Quantity`, `UnitPrice`, `TotalCost`, etc. | Encapsulate invariants and business rules.        |
| **Domain Services** | `IOrderDomainService`, `ICartDomainService`                                                | Handle cross-entity logic.                        |
| **Repositories**    | `IOrderRepository`, `ICartRepository`                                                      | Abstract data access.                             |
| **Enums**           | `OrderStatus`, `TransportMethod`                                                           | Define state and delivery modes.                  |
| **Events**          | `OrderCreated`, `OrderDispatched`, `OrderDelivered`, `OrderCancelled`                      | Signal domain state changes.                      |

---

## ✅ MVP Domain Boundary Summary

The **Order Context** connects directly with:

* **Customer Context** — via `CustomerId`.
* **Catalog Context** — via `ProductId`.