# 🧭 CATALOG CONTEXT — DOMAIN LAYER DESIGN DOCUMENTATION (MVP VERSION)

### Context Overview

The **Catalog Context** manages all product-related domain concerns within the e-commerce system.
Its primary responsibilities include:

* Managing product information, categories, and attributes.
* Maintaining availability.
* Handling product creation, updates, and removals.

The context is modeled as a **bounded context** under the **Catalog** subdomain.

---

## 1. Aggregate and Entity Model

### 1.1 Aggregate Root: `Product`

**Description:**
Represents a single sellable product in the e-commerce catalog.
It encapsulates all product-related attributes and ensures domain invariants such as valid pricing.

**Responsibilities:**

* Maintain descriptive, pricing, and dimensional data for the product.
* Ensure business rules regarding price, and product nature are enforced.

**Structure:**

```text
Product (Aggregate Root)
 ├── ProductId (Value Object)
 ├── Name (Value Object)
 ├── Description (Value Object)
 ├── Price (Value Object)
 ├── Weight (Value Object)
 ├── Dimensions (Value Object)
 ├── CategoryId (Value Object)
 ├── Nature (Enum)
 ├── CreatedAt (Value Object)
 ├── UpdatedAt (Value Object)
 ├── isActive (boolean)
```

---

## 2. Entities and Value Objects

### 2.1 Entity: `Product`

#### Fields:

| Field         | Type            | Description                                                 |
| ------------- | --------------- | ----------------------------------------------------------- |
| `product_id`          | `ProductId`     | Unique identity of the product.                             |
| `product_name`        | `Name`          | Product name, must be descriptive and unique per category.  |
| `product_description` | `Description`   | Explains the product details.                               |
| `product_price`       | `Price`         | Represents the product’s price in local currency.           |
| `product_weight`      | `Weight`        | Product’s physical weight (used for shipping calculations). |
| `product_dimensions`  | `Dimensions`    | Represents product physical size (length, width, height).   |
| `categoryId`  | `CategoryId`    | Reference to the associated category.                       |
| `product_nature`      | `ProductNature` | Defines product handling requirements.                      |
| `product_createdAt`   | `CreatedAt`     | Date/time of product creation.                              |
| `product_updatedAt`   | `UpdatedAt`     | Date/time of last modification.                             |
| `product_isActive`    | `boolean`       | Indicates if the product is available for sale.             |

#### Methods (Behavior):

| Method                                                                                      | Description                                                          |
| ------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `create(Name, Description, Price, Weight, Dimensions, CategoryId, ProductNature)`.          |
| `updateDetails(Name, Description, Price, Weight, Dimensions, CategoryId)`                   | Updates the product’s descriptive and pricing details.               |
| `activate()`                                                                                | Marks the product as available for sale.                             |
| `deactivate()`                                                                              | Marks the product as unavailable for sale.                           |
| `isAvailable()`                                                                             | Returns true if product is active.                 |
| `equals(Product)`                                                                           | Compares product identity based on `ProductId`.                      |

#### Domain Invariants:

* Product must always have a **name**, **price**, and **category**.
* `Price` must be positive.

---

### 2.2 Entity: `Category`

**Description:**
Represents a logical grouping for products.

| Field         | Type                  | Description                                       |
| ------------- | --------------------- | ------------------------------------------------- |
| `category_id`          | `CategoryId`          | Unique category identity.                         |
| `category_name`        | `CategoryName`        | Category name (e.g., “Electronics”, “Groceries”). |
| `category_description` | `CategoryDescription` | Optional descriptive text.                        |

**Methods:**

| Method                      | Description              |
| --------------------------- | ------------------------ |
| `create(name, description)` | Creates a new category.  |
| `rename(newName)`           | Updates category name.   |
| `equals(Category)`          | Identity equality check. |

**Domain Invariant:**

* Category name must be unique within the catalog.

---

## 3. Value Objects

### 3.1 `ProductId`

**Responsibilities:**

* Guarantees unique identity for a product.

```text
+ id: UUID
+ equals(ProductId): bool
+ toString(): string
```

---

### 3.2 `CategoryId`

**Responsibilities:**

* Represents a unique identifier for product categories.

```text
+ id: UUID
+ equals(CategoryId): bool
+ toString(): string
```

---

### 3.3 `Name`

**Responsibilities:**

* Represents a valid product name.

```text
+ value: string
+ validateNonEmpty()
+ equals(Name): bool
```

**Invariant:**

* Must not be empty or exceed reasonable character length (e.g., 100 chars).

---

### 3.4 `Description`

**Responsibilities:**

* Represents a human-readable product description.

```text
+ value: string
+ equals(Description): bool
```

---

### 3.5 `Price`

**Responsibilities:**

* Represents monetary value in Tanzanian Shillings (TZS).

```text
+ amount: decimal
+ currency: string = "TZS"
+ validatePositive()
+ add(Price)
+ subtract(Price)
```

**Invariant:**

* `amount` must be > 0.

---

### 3.6 `Weight`

**Responsibilities:**

* Represents product weight in kilograms or grams.

```text
+ value: float
+ unit: string = "kg"
+ validateNonNegative()
```

---

### 3.7 `Dimensions`

**Responsibilities:**

* Represents size in centimeters.

```text
+ length: float
+ width: float
+ height: float
+ volume(): float
```

**Invariant:**

* All dimensions must be > 0.

---

### 3.8 `CreatedAt` and `UpdatedAt`

**Responsibilities:**

* Represent timestamp metadata for lifecycle tracking.

```text
+ value: datetime
```

**Invariant:**

* Both are always required for persistence consistency.

---

## 4. Domain Service Interfaces

Domain services provide behavior that doesn’t naturally belong to a single entity.

### 4.1 `ICatalogDomainService`

**Responsibilities:**

* Validate product-category relationships.
* Apply catalog-wide policies such as uniqueness of product name within category.

```text
interface ICatalogDomainService {
    bool isProductNameUnique(Name name, CategoryId categoryId);
}
```

---

## 5. Repository Interfaces

Repositories abstract persistence while keeping the domain model pure.

### 5.1 `IProductRepository`

**Responsibilities:**

* Provide access to `Product` aggregates.
* Handle product persistence and retrieval.

```text
interface IProductRepository {
    ProductId nextIdentity();
    void save(Product product);
    Optional<Product> findById(ProductId id);
    List<Product> findByCategory(CategoryId categoryId);
    Optional<Product> findByName(Name name);
}
```

---

### 5.2 `ICategoryRepository`

**Responsibilities:**

* Manage storage and retrieval of product categories.

```text
interface ICategoryRepository {
    CategoryId nextIdentity();
    void save(Category category);
    Optional<Category> findById(CategoryId id);
    Optional<Category> findByName(CategoryName name);
    List<Category> findAll();
}
```

---

## 6. Enums

### 6.1 `ProductNature`

**Represents the nature or handling requirements of a product.**

```text
enum ProductNature {
    NORMAL,
    FRAGILE,
    PERISHABLE
}
```

---

## 7. Factories (Optional for MVP)

For future scaling, factories can encapsulate creation logic.

Example:

```text
ProductFactory.create(Name, Description, Price, Weight, Dimensions, CategoryId, ProductNature)
```

---

## 8. Summary of Domain Layer Components

| Type           | Name                                                                                                                    | Purpose                                    |
| -------------- | ----------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| Aggregate Root | `Product`                                                                                                               | Central aggregate controlling consistency. |
| Entity         | `Category`                                                                                                              | Represents product grouping.               |
| Value Objects  | `ProductId`, `CategoryId`, `Name`, `Description`, `Price`, `Weight`, `Dimensions`, `CreatedAt`, `UpdatedAt` | Ensure immutability and validation.        |
| Domain Service | `ICatalogDomainService`                                                                                                 | Handles cross-entity business logic.       |
| Repositories   | `IProductRepository`, `ICategoryRepository`                                                                             | Abstract persistence access.               |
| Enum           | `ProductNature`                                                                                                         | Defines handling requirements.             |

---

## 9. Design Principles Applied

| Principle                   | Description                                                                                                    |
| --------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **DDD Aggregation**         | Product acts as the aggregate root; all catalog state transitions are managed through it.                      |
| **Value Object Integrity**  | Each field (price, weight, name) encapsulates its own validation logic.                                        |
| **KISS**                    | Focuses on core attributes and operations only; no premature complexity (e.g., variants, images, SKUs).        |
| **DRY**                     | Common logic centralized in reusable value objects.                                                            |
| **Infrastructure Agnostic** | Domain logic is independent of persistence or frameworks.                                                      |
