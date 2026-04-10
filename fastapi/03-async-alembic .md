# Runbook: FastAPI — Group 2: Database (Async SQLAlchemy + Alembic)

> **Series:** Backend Runbooks — FastAPI Edition  
> **Schema:** E-Commerce (same as Runbook 2)  
> **Covers:** Async engine, AsyncSession, rewriting sync queries, Alembic setup to rollback  

---

## Part A — Async SQLAlchemy

### Why bother going async?

| | Sync | Async |
|---|---|---|
| Route def | `def` or `async def` | must be `async def` |
| DB call blocks event loop? | ✅ Yes (bad under load) | ❌ No |
| Good for | Simple apps, scripts | High-concurrency APIs |
| SQLAlchemy session | `Session` | `AsyncSession` |
| DB driver (Postgres) | `psycopg2` | `asyncpg` |
| DB driver (MySQL) | `pymysql` | `aiomysql` |
| Query style | `db.query(Model)` | `await db.execute(select(Model))` |

> **Rule of thumb:** If you expect concurrent users or do heavy I/O, go async. For internal tools or low-traffic APIs, sync is fine and simpler.

---

### Step 1 — Install async drivers

```bash
pip install fastapi uvicorn sqlalchemy asyncpg
# MySQL:      pip install aiomysql
# SQL Server: no mature async driver yet — use sync for MSSQL
```

---

### Step 2 — Async `database.py`

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase

# Note the +asyncpg driver suffix
DATABASE_URL = "postgresql+asyncpg://user:password@localhost:5432/webstore"
# MySQL: "mysql+aiomysql://user:password@localhost:3306/webstore"

engine = create_async_engine(DATABASE_URL, echo=False)

# async_sessionmaker is the async equivalent of sessionmaker
AsyncSessionLocal = async_sessionmaker(
    bind=engine,
    expire_on_commit=False,   # important: prevents lazy-load errors after commit
)

class Base(DeclarativeBase):
    pass

# Async dependency — used in every route
async def get_db():
    async with AsyncSessionLocal() as session:
        yield session
```

> **`expire_on_commit=False`** — In sync SQLAlchemy, accessing an attribute after commit re-fetches it lazily. In async, that lazy fetch would block. Setting this to `False` keeps attribute values in memory after commit.

---

### Step 3 — Models stay the same

The ORM models (`models.py`) are **identical** to the sync version. SQLAlchemy models are not async or sync — only the engine and session are.

---

### Step 4 — Query syntax changes

This is the main shift. Every query now uses `select()` + `await db.execute()`.

#### Sync → Async translation table

| Sync | Async equivalent |
|---|---|
| `db.query(Model).all()` | `(await db.execute(select(Model))).scalars().all()` |
| `db.query(Model).first()` | `(await db.execute(select(Model))).scalars().first()` |
| `db.query(Model).filter(...)` | `select(Model).where(...)` |
| `db.query(Model).join(...)` | `select(Model).join(...)` |
| `db.add(obj)` | `db.add(obj)` ← same |
| `db.commit()` | `await db.commit()` |
| `db.refresh(obj)` | `await db.refresh(obj)` |
| `db.flush()` | `await db.flush()` |
| `db.delete(obj)` | `await db.delete(obj)` |

---

### Step 5 — Rewriting the product endpoints (async)

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from sqlalchemy.orm import joinedload, selectinload
from database import get_db
from models import Product, Category, Inventory
from schemas import ProductOut, ProductCreate

router = APIRouter()

# 🟢 List active products
@router.get("/", response_model=list[ProductOut])
async def list_products(
    skip: int = 0,
    limit: int = 20,
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(Product)
        .options(
            joinedload(Product.category),
            joinedload(Product.inventory),
        )
        .where(Product.is_active == True)
        .offset(skip)
        .limit(limit)
    )
    return result.scalars().all()


# 🟢 Get single product
@router.get("/{product_id}", response_model=ProductOut)
async def get_product(product_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(Product)
        .options(joinedload(Product.category), joinedload(Product.inventory))
        .where(Product.id == product_id)
    )
    product = result.scalars().first()
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    return product


# 🟡 Filter by category slug (JOIN)
@router.get("/by-category/{slug}", response_model=list[ProductOut])
async def products_by_category(slug: str, db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(Product)
        .join(Product.category)
        .options(joinedload(Product.category), joinedload(Product.inventory))
        .where(Category.slug == slug, Product.is_active == True)
    )
    return result.scalars().all()


# 🟡 Create product + seed inventory
@router.post("/", response_model=ProductOut, status_code=201)
async def create_product(payload: ProductCreate, db: AsyncSession = Depends(get_db)):
    product = Product(
        category_id=payload.category_id,
        name=payload.name,
        description=payload.description,
        price=payload.price,
    )
    db.add(product)
    await db.flush()   # get product.id without committing

    inventory = Inventory(product_id=product.id, quantity=payload.quantity)
    db.add(inventory)
    await db.commit()
    await db.refresh(product)

    result = await db.execute(
        select(Product)
        .options(joinedload(Product.category), joinedload(Product.inventory))
        .where(Product.id == product.id)
    )
    return result.scalars().first()
```

---

### Step 6 — Rewriting the order endpoints (async)

```python
from sqlalchemy import select, func
from sqlalchemy.orm import joinedload
from decimal import Decimal

# 🟢 Get single order (multi-level nested join)
@router.get("/{order_id}", response_model=OrderOut)
async def get_order(order_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(Order)
        .options(
            joinedload(Order.customer),
            joinedload(Order.items)
                .joinedload(OrderItem.product)
                .joinedload(Product.category),
        )
        .where(Order.id == order_id)
    )
    order = result.scalars().first()
    if not order:
        raise HTTPException(status_code=404, detail="Order not found")
    return order


# 🟠 Top-selling products (aggregate)
@router.get("/reports/top-products")
async def top_selling_products(limit: int = 10, db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(
            Product.id.label("product_id"),
            Product.name.label("product_name"),
            func.sum(OrderItem.quantity).label("total_sold"),
            func.sum(OrderItem.quantity * OrderItem.unit_price).label("revenue"),
        )
        .join(OrderItem, OrderItem.product_id == Product.id)
        .group_by(Product.id, Product.name)
        .order_by(func.sum(OrderItem.quantity).desc())
        .limit(limit)
    )
    return result.mappings().all()   # returns list of dict-like rows
```

---

### Step 7 — `joinedload` vs `selectinload` in async

```
joinedload  → adds a SQL JOIN to the main query (1 query total)
selectinload → runs a second SELECT ... WHERE id IN (...) (2 queries)
```

Use `selectinload` for **one-to-many** relations to avoid row duplication:

```python
# joinedload on a one-to-many can return duplicate parent rows
# selectinload is safer for collections
select(Order).options(
    selectinload(Order.items)    # ← prefer this for collections
        .joinedload(OrderItem.product)
)
```

---

### Step 8 — Creating tables async (for dev only)

```python
# main.py
from database import Base, engine

@app.on_event("startup")
async def startup():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
```

> Use Alembic (Part B below) for production. `create_all` is fine for local dev.

---

---

## Part B — Alembic Migrations

Alembic is to SQLAlchemy what EF Core migrations are to Entity Framework — it tracks schema changes as versioned scripts you can apply and roll back.

---

### Step 9 — Install and initialise

```bash
pip install alembic
alembic init alembic
```

This creates:
```
alembic/
├── env.py           ← you edit this
├── script.py.mako   ← migration template (leave alone)
└── versions/        ← migration files live here
alembic.ini          ← main config file
```

---

### Step 10 — Configure `alembic.ini`

```ini
# alembic.ini
sqlalchemy.url = postgresql://user:password@localhost:5432/webstore
# For async apps, this can stay sync — Alembic runs migrations synchronously
```

---

### Step 11 — Configure `alembic/env.py`

```python
# alembic/env.py  — the critical lines to change

from database import Base   # import your Base so Alembic sees your models
import models               # import models so they register against Base

# Replace the existing target_metadata line:
target_metadata = Base.metadata
```

Full relevant section:

```python
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context

# --- ADD THESE ---
from database import Base
import models
# -----------------

config = context.config
fileConfig(config.config_file_name)

target_metadata = Base.metadata   # ← was None, now points to your models
```

---

### Step 12 — Create your first migration

**Option A — autogenerate** (Alembic diffs your models vs the DB)

```bash
alembic revision --autogenerate -m "create initial tables"
```

This produces a file in `alembic/versions/` like:

```python
# alembic/versions/abc123_create_initial_tables.py

def upgrade() -> None:
    op.create_table(
        "customers",
        sa.Column("id",    sa.Integer(), primary_key=True),
        sa.Column("name",  sa.String(100), nullable=False),
        sa.Column("email", sa.String(150), nullable=False, unique=True),
        ...
    )
    # ... all your tables

def downgrade() -> None:
    op.drop_table("customers")
    # ... in reverse order
```

**Option B — blank migration** (write it yourself)

```bash
alembic revision -m "add phone column to customers"
```

Then fill in manually:

```python
def upgrade() -> None:
    op.add_column("customers", sa.Column("phone", sa.String(20), nullable=True))

def downgrade() -> None:
    op.drop_column("customers", "phone")
```

---

### Step 13 — Apply migrations

```bash
# Apply all pending migrations
alembic upgrade head

# Apply exactly one step forward
alembic upgrade +1

# Check current version
alembic current

# Show full history
alembic history --verbose
```

---

### Step 14 — Roll back migrations

```bash
# Roll back one step
alembic downgrade -1

# Roll back to a specific revision id
alembic downgrade abc123

# Roll back everything (back to empty DB)
alembic downgrade base
```

---

### Step 15 — Common migration operations

```python
# Add a column
op.add_column("products", sa.Column("sku", sa.String(50), nullable=True))

# Drop a column
op.drop_column("products", "sku")

# Rename a column
op.alter_column("products", "name", new_column_name="title")

# Add an index
op.create_index("ix_products_sku", "products", ["sku"], unique=True)

# Drop an index
op.drop_index("ix_products_sku", table_name="products")

# Add a foreign key
op.create_foreign_key("fk_orders_customer", "orders", "customers", ["customer_id"], ["id"])

# Run raw SQL in a migration
op.execute("UPDATE products SET is_active = true WHERE is_active IS NULL")
```

---

### Step 16 — Alembic with async engine

If your app uses an async engine, Alembic still runs sync by default. To support async in `env.py`:

```python
# alembic/env.py — async variant
import asyncio
from sqlalchemy.ext.asyncio import create_async_engine

DATABASE_URL = "postgresql+asyncpg://user:password@localhost:5432/webstore"

def run_migrations_online():
    connectable = create_async_engine(DATABASE_URL)

    async def run_async_migrations():
        async with connectable.connect() as connection:
            await connection.run_sync(do_run_migrations)
        await connectable.dispose()

    asyncio.run(run_async_migrations())

def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()
```

---

### Alembic Workflow Cheatsheet

```
Make model change
      ↓
alembic revision --autogenerate -m "description"
      ↓
Review the generated file in alembic/versions/
      ↓
alembic upgrade head
      ↓
Something wrong? → alembic downgrade -1
```

---

## Gotchas

1. **Always review autogenerated migrations** — Alembic misses things like check constraints, partial indexes, and some column type changes. Never blindly apply without reading the file.
2. **Import all models in `env.py`** — if a model isn't imported, Alembic doesn't know it exists and won't generate its table.
3. **`expire_on_commit=False` is essential in async** — without it, accessing any attribute after `await db.commit()` triggers a lazy load that crashes in async context.
4. **`scalars()` vs `mappings()`** — use `.scalars().all()` when returning ORM model instances. Use `.mappings().all()` for raw column results (aggregates, custom selects).
5. **`selectinload` for collections** — `joinedload` on a one-to-many relation multiplies rows in the result set. Use `selectinload` instead.
6. **Never use `alembic downgrade base` in production** — it drops everything. Downgrade to a specific known-good revision instead.

---

*Next in series → Group 4: Auth (JWT + RBAC)*
