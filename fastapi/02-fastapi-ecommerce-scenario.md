# Runbook: FastAPI — E-Commerce API (Practical Scenario)

> **Series:** Backend Runbooks — FastAPI Edition  
> **Scenario:** Webstore — products, customers, orders, inventory, reviews  
> **DB:** MySQL / PostgreSQL / SQL Server (SQLAlchemy works with all three)  
> **Level:** Beginner → Moderate  

---

## The Schema

8 tables covering the most common real-world relationships.

```
customers           ← standalone, referenced by orders and reviews
categories          ← standalone, referenced by products
products            ← belongs to category
inventory           ← one-to-one with product
orders              ← belongs to customer
order_items         ← join table between orders and products (many-to-many)
addresses           ← belongs to customer
reviews             ← belongs to customer + product
```

### Entity-Relationship Overview

```
customers ──< orders ──< order_items >── products ──< inventory
    │                                       │
    └──< addresses                          └──< categories
    └──< reviews >────────────────────── products
```

---

## Part 1 — Setup

### Step 1 — Install

```bash
pip install fastapi uvicorn sqlalchemy
pip install psycopg2-binary        # PostgreSQL
# pip install pymysql              # MySQL
# pip install pyodbc               # SQL Server
```

### Step 2 — `database.py`

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase

# Swap in your actual connection string
DATABASE_URL = "postgresql://user:password@localhost:5432/webstore"
# MySQL:      "mysql+pymysql://user:password@localhost:3306/webstore"
# SQL Server: "mssql+pyodbc://user:password@server/webstore?driver=ODBC+Driver+17+for+SQL+Server"

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

class Base(DeclarativeBase):
    pass

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

---

## Part 2 — Models

### Step 3 — `models.py`

```python
from datetime import datetime
from sqlalchemy import (
    Column, Integer, String, Text, Float,
    Boolean, DateTime, ForeignKey, Numeric
)
from sqlalchemy.orm import relationship
from database import Base


class Customer(Base):
    __tablename__ = "customers"
    id         = Column(Integer, primary_key=True, index=True)
    name       = Column(String(100), nullable=False)
    email      = Column(String(150), unique=True, nullable=False)
    phone      = Column(String(20))
    created_at = Column(DateTime, default=datetime.utcnow)

    orders    = relationship("Order",   back_populates="customer")
    addresses = relationship("Address", back_populates="customer")
    reviews   = relationship("Review",  back_populates="customer")


class Address(Base):
    __tablename__ = "addresses"
    id          = Column(Integer, primary_key=True, index=True)
    customer_id = Column(Integer, ForeignKey("customers.id"), nullable=False)
    line1       = Column(String(200), nullable=False)
    city        = Column(String(100), nullable=False)
    state       = Column(String(100))
    country     = Column(String(100), nullable=False)
    is_default  = Column(Boolean, default=False)

    customer = relationship("Customer", back_populates="addresses")


class Category(Base):
    __tablename__ = "categories"
    id   = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), unique=True, nullable=False)
    slug = Column(String(100), unique=True, nullable=False)

    products = relationship("Product", back_populates="category")


class Product(Base):
    __tablename__ = "products"
    id          = Column(Integer, primary_key=True, index=True)
    category_id = Column(Integer, ForeignKey("categories.id"), nullable=False)
    name        = Column(String(200), nullable=False)
    description = Column(Text)
    price       = Column(Numeric(10, 2), nullable=False)
    is_active   = Column(Boolean, default=True)

    category    = relationship("Category",   back_populates="products")
    inventory   = relationship("Inventory",  back_populates="product", uselist=False)
    order_items = relationship("OrderItem",  back_populates="product")
    reviews     = relationship("Review",     back_populates="product")


class Inventory(Base):
    __tablename__ = "inventory"
    id         = Column(Integer, primary_key=True, index=True)
    product_id = Column(Integer, ForeignKey("products.id"), unique=True, nullable=False)
    quantity   = Column(Integer, default=0, nullable=False)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    product = relationship("Product", back_populates="inventory")


class Order(Base):
    __tablename__ = "orders"
    id          = Column(Integer, primary_key=True, index=True)
    customer_id = Column(Integer, ForeignKey("customers.id"), nullable=False)
    status      = Column(String(50), default="pending")   # pending / paid / shipped / cancelled
    total       = Column(Numeric(10, 2), nullable=False)
    created_at  = Column(DateTime, default=datetime.utcnow)

    customer    = relationship("Customer",  back_populates="orders")
    items       = relationship("OrderItem", back_populates="order")


class OrderItem(Base):
    __tablename__ = "order_items"
    id         = Column(Integer, primary_key=True, index=True)
    order_id   = Column(Integer, ForeignKey("orders.id"),   nullable=False)
    product_id = Column(Integer, ForeignKey("products.id"), nullable=False)
    quantity   = Column(Integer, nullable=False)
    unit_price = Column(Numeric(10, 2), nullable=False)   # price at time of purchase

    order   = relationship("Order",   back_populates="items")
    product = relationship("Product", back_populates="order_items")


class Review(Base):
    __tablename__ = "reviews"
    id          = Column(Integer, primary_key=True, index=True)
    customer_id = Column(Integer, ForeignKey("customers.id"), nullable=False)
    product_id  = Column(Integer, ForeignKey("products.id"),  nullable=False)
    rating      = Column(Integer, nullable=False)     # 1–5
    comment     = Column(Text)
    created_at  = Column(DateTime, default=datetime.utcnow)

    customer = relationship("Customer", back_populates="reviews")
    product  = relationship("Product",  back_populates="reviews")
```

---

## Part 3 — Schemas

### Step 4 — `schemas.py`

```python
from datetime import datetime
from decimal import Decimal
from pydantic import BaseModel

cfg = {"from_attributes": True}   # Pydantic v2 shorthand used below


# ── Category ──────────────────────────────────────────────
class CategoryOut(BaseModel):
    id:   int
    name: str
    slug: str
    model_config = cfg


# ── Inventory ─────────────────────────────────────────────
class InventoryOut(BaseModel):
    quantity:   int
    updated_at: datetime
    model_config = cfg


# ── Product ───────────────────────────────────────────────
class ProductOut(BaseModel):
    id:          int
    name:        str
    description: str | None
    price:       Decimal
    is_active:   bool
    category:    CategoryOut
    inventory:   InventoryOut | None
    model_config = cfg

class ProductCreate(BaseModel):
    category_id: int
    name:        str
    description: str | None = None
    price:       Decimal
    quantity:    int = 0        # seeds inventory on create


# ── Customer ──────────────────────────────────────────────
class CustomerOut(BaseModel):
    id:    int
    name:  str
    email: str
    phone: str | None
    model_config = cfg

class CustomerCreate(BaseModel):
    name:  str
    email: str
    phone: str | None = None


# ── Order ─────────────────────────────────────────────────
class OrderItemOut(BaseModel):
    product_id: int
    quantity:   int
    unit_price: Decimal
    product:    ProductOut
    model_config = cfg

class OrderOut(BaseModel):
    id:         int
    status:     str
    total:      Decimal
    created_at: datetime
    customer:   CustomerOut
    items:      list[OrderItemOut]
    model_config = cfg

class OrderItemIn(BaseModel):
    product_id: int
    quantity:   int

class OrderCreate(BaseModel):
    customer_id: int
    items:       list[OrderItemIn]


# ── Review ────────────────────────────────────────────────
class ReviewOut(BaseModel):
    id:         int
    rating:     int
    comment:    str | None
    created_at: datetime
    customer:   CustomerOut
    model_config = cfg

class ReviewCreate(BaseModel):
    customer_id: int
    product_id:  int
    rating:      int
    comment:     str | None = None


# ── Aggregates (used by summary endpoints) ────────────────
class ProductSummary(BaseModel):
    product_id:   int
    product_name: str
    total_sold:   int
    revenue:      Decimal

class CustomerOrderSummary(BaseModel):
    customer_id:   int
    customer_name: str
    order_count:   int
    total_spent:   Decimal
```

---

## Part 4 — Endpoints

Each section shows the query complexity level.

---

### `routers/products.py`

#### 🟢 Simple — list all active products (single table)

```python
@router.get("/", response_model=list[ProductOut])
def list_products(
    skip: int = 0,
    limit: int = 20,
    db: Session = Depends(get_db)
):
    return (
        db.query(Product)
        .options(joinedload(Product.category), joinedload(Product.inventory))
        .filter(Product.is_active == True)
        .offset(skip).limit(limit)
        .all()
    )
```

#### 🟢 Simple — get one product by id

```python
@router.get("/{product_id}", response_model=ProductOut)
def get_product(product_id: int, db: Session = Depends(get_db)):
    product = (
        db.query(Product)
        .options(joinedload(Product.category), joinedload(Product.inventory))
        .filter(Product.id == product_id)
        .first()
    )
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    return product
```

#### 🟡 Medium — filter products by category slug

```python
@router.get("/by-category/{slug}", response_model=list[ProductOut])
def products_by_category(slug: str, db: Session = Depends(get_db)):
    # JOIN products → categories, filter on categories.slug
    return (
        db.query(Product)
        .join(Product.category)
        .options(joinedload(Product.category), joinedload(Product.inventory))
        .filter(Category.slug == slug, Product.is_active == True)
        .all()
    )
```

#### 🟡 Medium — search products by name + optional price range

```python
@router.get("/search/", response_model=list[ProductOut])
def search_products(
    q:         str   = "",
    min_price: float = 0,
    max_price: float = 999999,
    db: Session = Depends(get_db)
):
    return (
        db.query(Product)
        .options(joinedload(Product.category), joinedload(Product.inventory))
        .filter(
            Product.is_active == True,
            Product.name.ilike(f"%{q}%"),
            Product.price >= min_price,
            Product.price <= max_price,
        )
        .all()
    )
```

#### 🟡 Medium — create product + seed inventory atomically

```python
@router.post("/", response_model=ProductOut, status_code=201)
def create_product(payload: ProductCreate, db: Session = Depends(get_db)):
    product = Product(
        category_id=payload.category_id,
        name=payload.name,
        description=payload.description,
        price=payload.price,
    )
    db.add(product)
    db.flush()   # get product.id without committing yet

    inventory = Inventory(product_id=product.id, quantity=payload.quantity)
    db.add(inventory)
    db.commit()
    db.refresh(product)
    return (
        db.query(Product)
        .options(joinedload(Product.category), joinedload(Product.inventory))
        .filter(Product.id == product.id)
        .first()
    )
```

---

### `routers/orders.py`

#### 🟢 Simple — get a single order with all its items

```python
@router.get("/{order_id}", response_model=OrderOut)
def get_order(order_id: int, db: Session = Depends(get_db)):
    order = (
        db.query(Order)
        .options(
            joinedload(Order.customer),
            joinedload(Order.items).joinedload(OrderItem.product)
                .joinedload(Product.category),
        )
        .filter(Order.id == order_id)
        .first()
    )
    if not order:
        raise HTTPException(status_code=404, detail="Order not found")
    return order
```

#### 🟡 Medium — list all orders for a customer

```python
@router.get("/by-customer/{customer_id}", response_model=list[OrderOut])
def orders_by_customer(customer_id: int, db: Session = Depends(get_db)):
    return (
        db.query(Order)
        .options(
            joinedload(Order.customer),
            joinedload(Order.items).joinedload(OrderItem.product),
        )
        .filter(Order.customer_id == customer_id)
        .order_by(Order.created_at.desc())
        .all()
    )
```

#### 🟡 Medium — place an order (transaction with inventory check)

```python
@router.post("/", response_model=OrderOut, status_code=201)
def place_order(payload: OrderCreate, db: Session = Depends(get_db)):
    total = Decimal("0.00")
    items = []

    for line in payload.items:
        product = db.query(Product).filter(Product.id == line.product_id).first()
        if not product:
            raise HTTPException(status_code=404, detail=f"Product {line.product_id} not found")

        inv = db.query(Inventory).filter(Inventory.product_id == line.product_id).first()
        if not inv or inv.quantity < line.quantity:
            raise HTTPException(status_code=400, detail=f"Insufficient stock for {product.name}")

        inv.quantity -= line.quantity          # deduct stock
        line_total = product.price * line.quantity
        total += line_total
        items.append(OrderItem(
            product_id=line.product_id,
            quantity=line.quantity,
            unit_price=product.price,
        ))

    order = Order(customer_id=payload.customer_id, total=total, status="pending")
    db.add(order)
    db.flush()   # get order.id

    for item in items:
        item.order_id = order.id
        db.add(item)

    db.commit()
    db.refresh(order)
    return (
        db.query(Order)
        .options(
            joinedload(Order.customer),
            joinedload(Order.items).joinedload(OrderItem.product)
                .joinedload(Product.category),
        )
        .filter(Order.id == order.id)
        .first()
    )
```

#### 🟠 Moderate — top-selling products (aggregate across 3 tables)

```python
from sqlalchemy import func

@router.get("/reports/top-products", response_model=list[ProductSummary])
def top_selling_products(limit: int = 10, db: Session = Depends(get_db)):
    # JOIN order_items → products, GROUP BY product, SUM qty and revenue
    rows = (
        db.query(
            Product.id.label("product_id"),
            Product.name.label("product_name"),
            func.sum(OrderItem.quantity).label("total_sold"),
            func.sum(OrderItem.quantity * OrderItem.unit_price).label("revenue"),
        )
        .join(OrderItem, OrderItem.product_id == Product.id)
        .group_by(Product.id, Product.name)
        .order_by(func.sum(OrderItem.quantity).desc())
        .limit(limit)
        .all()
    )
    return [
        ProductSummary(
            product_id=r.product_id,
            product_name=r.product_name,
            total_sold=r.total_sold,
            revenue=r.revenue,
        )
        for r in rows
    ]
```

#### 🟠 Moderate — customer spend summary (aggregate across 3 tables)

```python
@router.get("/reports/customer-spend", response_model=list[CustomerOrderSummary])
def customer_spend(limit: int = 10, db: Session = Depends(get_db)):
    rows = (
        db.query(
            Customer.id.label("customer_id"),
            Customer.name.label("customer_name"),
            func.count(Order.id).label("order_count"),
            func.sum(Order.total).label("total_spent"),
        )
        .join(Order, Order.customer_id == Customer.id)
        .filter(Order.status != "cancelled")
        .group_by(Customer.id, Customer.name)
        .order_by(func.sum(Order.total).desc())
        .limit(limit)
        .all()
    )
    return [
        CustomerOrderSummary(
            customer_id=r.customer_id,
            customer_name=r.customer_name,
            order_count=r.order_count,
            total_spent=r.total_spent,
        )
        for r in rows
    ]
```

---

### `routers/reviews.py`

#### 🟢 Simple — get reviews for a product

```python
@router.get("/product/{product_id}", response_model=list[ReviewOut])
def product_reviews(product_id: int, db: Session = Depends(get_db)):
    return (
        db.query(Review)
        .options(joinedload(Review.customer))
        .filter(Review.product_id == product_id)
        .order_by(Review.created_at.desc())
        .all()
    )
```

#### 🟡 Medium — average rating per product (aggregate)

```python
from sqlalchemy import func

@router.get("/ratings/", response_model=list[dict])
def average_ratings(db: Session = Depends(get_db)):
    rows = (
        db.query(
            Product.id,
            Product.name,
            func.round(func.avg(Review.rating), 2).label("avg_rating"),
            func.count(Review.id).label("review_count"),
        )
        .join(Review, Review.product_id == Product.id)
        .group_by(Product.id, Product.name)
        .order_by(func.avg(Review.rating).desc())
        .all()
    )
    return [
        {"product_id": r[0], "name": r[1], "avg_rating": r[2], "review_count": r[3]}
        for r in rows
    ]
```

---

### `routers/inventory.py`

#### 🟡 Medium — products low on stock (JOIN + filter)

```python
@router.get("/low-stock/", response_model=list[ProductOut])
def low_stock(threshold: int = 5, db: Session = Depends(get_db)):
    return (
        db.query(Product)
        .join(Product.inventory)
        .options(joinedload(Product.category), joinedload(Product.inventory))
        .filter(Inventory.quantity <= threshold, Product.is_active == True)
        .order_by(Inventory.quantity.asc())
        .all()
    )
```

#### 🟠 Moderate — products with NO inventory record (LEFT OUTER JOIN)

```python
from sqlalchemy.orm import outerjoin

@router.get("/missing/", response_model=list[ProductOut])
def products_without_inventory(db: Session = Depends(get_db)):
    # LEFT JOIN to find products that have no inventory row at all
    return (
        db.query(Product)
        .outerjoin(Product.inventory)
        .options(joinedload(Product.category))
        .filter(Inventory.id == None)
        .all()
    )
```

---

## Part 5 — Register All Routers

### `main.py`

```python
from fastapi import FastAPI
from database import Base, engine
from routers import products, orders, reviews, inventory, customers

Base.metadata.create_all(bind=engine)

app = FastAPI(title="Webstore API")

app.include_router(products.router,  prefix="/products",  tags=["Products"])
app.include_router(orders.router,    prefix="/orders",    tags=["Orders"])
app.include_router(reviews.router,   prefix="/reviews",   tags=["Reviews"])
app.include_router(inventory.router, prefix="/inventory", tags=["Inventory"])
app.include_router(customers.router, prefix="/customers", tags=["Customers"])
```

---

## Endpoint Map

| Method | Path | Complexity | What it does |
|--------|------|------------|--------------|
| GET | `/products/` | 🟢 Simple | List active products (paginated) |
| GET | `/products/{id}` | 🟢 Simple | Single product with category + stock |
| POST | `/products/` | 🟡 Medium | Create product + seed inventory |
| GET | `/products/by-category/{slug}` | 🟡 Medium | JOIN products → categories |
| GET | `/products/search/` | 🟡 Medium | Filter by name + price range |
| GET | `/orders/{id}` | 🟢 Simple | Order with customer + line items |
| GET | `/orders/by-customer/{id}` | 🟡 Medium | All orders for a customer |
| POST | `/orders/` | 🟡 Medium | Place order + deduct inventory |
| GET | `/orders/reports/top-products` | 🟠 Moderate | SUM + GROUP BY across 3 tables |
| GET | `/orders/reports/customer-spend` | 🟠 Moderate | SUM + COUNT across 3 tables |
| GET | `/reviews/product/{id}` | 🟢 Simple | Reviews for a product |
| GET | `/reviews/ratings/` | 🟡 Medium | AVG rating GROUP BY product |
| GET | `/inventory/low-stock/` | 🟡 Medium | Products below stock threshold |
| GET | `/inventory/missing/` | 🟠 Moderate | LEFT JOIN — products with no stock record |

---

## Key Patterns Illustrated

| Pattern | Where used |
|---|---|
| Single table query | `list_products`, `get_product` |
| INNER JOIN | `products_by_category`, `low_stock` |
| LEFT OUTER JOIN | `products_without_inventory` |
| Multi-level nested join | `get_order` (order → items → product → category) |
| Aggregate: SUM + GROUP BY | `top_selling_products`, `customer_spend` |
| Aggregate: AVG + COUNT | `average_ratings` |
| Filter on joined table column | `products_by_category` (filter on `Category.slug`) |
| Transaction with flush | `place_order` (inventory check + deduct in one commit) |
| `db.flush()` vs `db.commit()` | `create_product`, `place_order` |
| Pagination with offset/limit | `list_products` |

---

## Gotchas Specific to This Schema

1. **`unit_price` on `OrderItem`** — always snapshot the price at the time of purchase. Never recalculate from the current `product.price` — prices change.
2. **`db.flush()` before relating** — when you need the auto-generated `id` (e.g. `order.id`) before the transaction commits, call `flush()`. It sends the INSERT to the DB but holds the transaction open.
3. **`uselist=False`** on `Product.inventory` — since inventory is one-to-one, this makes SQLAlchemy return a single object instead of a list.
4. **`outerjoin` for missing rows** — the standard `.join()` is an INNER JOIN and silently drops rows with no match. Use `.outerjoin()` + `filter(RelatedModel.id == None)` to find orphaned records.
5. **`func.sum` returns `None` if no rows** — wrap in `coalesce` if needed: `func.coalesce(func.sum(...), 0)`.

---

*Next runbook → Django REST Framework (DRF) — same schema, different framework*
