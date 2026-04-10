# Runbook: FastAPI — Group 4: Auth (JWT + RBAC)

> **Series:** Backend Runbooks — FastAPI Edition  
> **Schema:** E-Commerce (same as Runbook 2)  
> **Covers:** Password hashing, JWT login, protected routes, refresh tokens, role-based access control  

---

## How Auth Works in This Runbook

```
POST /auth/register  →  hash password, store user
POST /auth/login     →  verify password, return access + refresh tokens
GET  /products/      →  verify JWT in Authorization header → allow
DELETE /products/1   →  verify JWT + check role == "admin" → allow or 403
POST /auth/refresh   →  exchange refresh token for new access token
```

Tokens are **JWTs** — signed strings the client stores and sends with every request. The server never stores them (stateless).

---

## Part A — Setup

### Step 1 — Install

```bash
pip install python-jose[cryptography] passlib[bcrypt]
```

- `python-jose` — creates and verifies JWTs
- `passlib[bcrypt]` — hashes and verifies passwords

---

### Step 2 — Config (`config.py`)

```python
# config.py
import os

SECRET_KEY        = os.getenv("SECRET_KEY", "change-this-in-production-use-openssl-rand")
ALGORITHM         = "HS256"
ACCESS_TOKEN_TTL  = 30          # minutes
REFRESH_TOKEN_TTL = 60 * 24 * 7 # minutes — 7 days
```

Generate a real secret key:
```bash
openssl rand -hex 32
```

---

### Step 3 — Add `User` and `Role` models (`models.py`)

```python
from sqlalchemy import Column, Integer, String, Boolean, ForeignKey, DateTime
from sqlalchemy.orm import relationship
from datetime import datetime
from database import Base

class Role(Base):
    __tablename__ = "roles"
    id   = Column(Integer, primary_key=True, index=True)
    name = Column(String(50), unique=True, nullable=False)  # "admin", "customer", "staff"

    users = relationship("User", back_populates="role")


class User(Base):
    __tablename__ = "users"
    id              = Column(Integer, primary_key=True, index=True)
    email           = Column(String(150), unique=True, nullable=False, index=True)
    hashed_password = Column(String(200), nullable=False)
    is_active       = Column(Boolean, default=True)
    role_id         = Column(Integer, ForeignKey("roles.id"), nullable=False)
    created_at      = Column(DateTime, default=datetime.utcnow)

    role = relationship("Role", back_populates="users")
```

> **Note:** `User` is separate from `Customer`. A `Customer` is a business entity (has orders, addresses). A `User` is an auth identity. They can be linked by a `user_id` FK on `Customer` when needed.

---

### Step 4 — Schemas (`schemas.py` additions)

```python
from pydantic import BaseModel, EmailStr

class UserRegister(BaseModel):
    email:    EmailStr
    password: str
    role:     str = "customer"   # default role on signup

class UserOut(BaseModel):
    id:        int
    email:     str
    is_active: bool
    role:      str               # role name, not id
    model_config = {"from_attributes": True}

class TokenOut(BaseModel):
    access_token:  str
    refresh_token: str
    token_type:    str = "bearer"

class TokenData(BaseModel):
    user_id: int | None = None
    role:    str | None = None

class RefreshRequest(BaseModel):
    refresh_token: str
```

---

## Part B — Core Auth Utilities

### Step 5 — Password hashing (`auth/password.py`)

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(plain: str) -> str:
    return pwd_context.hash(plain)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)
```

---

### Step 6 — JWT utilities (`auth/tokens.py`)

```python
from datetime import datetime, timedelta, timezone
from jose import jwt, JWTError
from config import SECRET_KEY, ALGORITHM, ACCESS_TOKEN_TTL, REFRESH_TOKEN_TTL
from schemas import TokenData

def _create_token(data: dict, expires_minutes: int) -> str:
    payload = data.copy()
    expire  = datetime.now(timezone.utc) + timedelta(minutes=expires_minutes)
    payload.update({"exp": expire})
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def create_access_token(user_id: int, role: str) -> str:
    return _create_token(
        {"sub": str(user_id), "role": role, "type": "access"},
        ACCESS_TOKEN_TTL,
    )

def create_refresh_token(user_id: int) -> str:
    return _create_token(
        {"sub": str(user_id), "type": "refresh"},
        REFRESH_TOKEN_TTL,
    )

def decode_token(token: str) -> TokenData:
    try:
        payload  = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id  = int(payload.get("sub"))
        role     = payload.get("role")
        return TokenData(user_id=user_id, role=role)
    except JWTError:
        return None
```

---

## Part C — Dependencies

### Step 7 — `get_current_user` dependency (`auth/dependencies.py`)

This is the reusable building block injected into any protected route.

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.orm import Session
from database import get_db
from models import User
from auth.tokens import decode_token

# Tells FastAPI where clients send the token (/auth/login is the token URL)
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

def get_current_user(
    token: str = Depends(oauth2_scheme),
    db:    Session = Depends(get_db),
) -> User:
    credentials_error = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Invalid or expired token",
        headers={"WWW-Authenticate": "Bearer"},
    )
    token_data = decode_token(token)
    if not token_data or not token_data.user_id:
        raise credentials_error

    user = db.query(User).filter(User.id == token_data.user_id).first()
    if not user or not user.is_active:
        raise credentials_error

    return user


def get_current_active_user(current_user: User = Depends(get_current_user)) -> User:
    if not current_user.is_active:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user
```

---

### Step 8 — Role-based dependency factory

```python
# auth/dependencies.py (continued)

def require_role(*roles: str):
    """
    Usage:
        @router.delete("/{id}", dependencies=[Depends(require_role("admin"))])
        @router.post("/",       dependencies=[Depends(require_role("admin", "staff"))])
    """
    def role_checker(current_user: User = Depends(get_current_user)):
        if current_user.role.name not in roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Requires role: {', '.join(roles)}",
            )
        return current_user
    return role_checker
```

---

## Part D — Auth Routes

### Step 9 — `routers/auth.py`

```python
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session
from database import get_db
from models import User, Role
from schemas import UserRegister, UserOut, TokenOut, RefreshRequest
from auth.password import hash_password, verify_password
from auth.tokens import create_access_token, create_refresh_token, decode_token

router = APIRouter(prefix="/auth", tags=["Auth"])


# ── Register ──────────────────────────────────────────────
@router.post("/register", response_model=UserOut, status_code=201)
def register(payload: UserRegister, db: Session = Depends(get_db)):
    existing = db.query(User).filter(User.email == payload.email).first()
    if existing:
        raise HTTPException(status_code=400, detail="Email already registered")

    role = db.query(Role).filter(Role.name == payload.role).first()
    if not role:
        raise HTTPException(status_code=400, detail=f"Role '{payload.role}' not found")

    user = User(
        email=payload.email,
        hashed_password=hash_password(payload.password),
        role_id=role.id,
    )
    db.add(user)
    db.commit()
    db.refresh(user)
    return {"id": user.id, "email": user.email, "is_active": user.is_active, "role": role.name}


# ── Login ─────────────────────────────────────────────────
# OAuth2PasswordRequestForm expects form data: username + password
# FastAPI/Swagger uses this automatically for the Authorize button
@router.post("/login", response_model=TokenOut)
def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db),
):
    user = db.query(User).filter(User.email == form_data.username).first()
    if not user or not verify_password(form_data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password",
        )
    if not user.is_active:
        raise HTTPException(status_code=400, detail="Account disabled")

    return TokenOut(
        access_token=create_access_token(user.id, user.role.name),
        refresh_token=create_refresh_token(user.id),
    )


# ── Refresh ───────────────────────────────────────────────
@router.post("/refresh", response_model=TokenOut)
def refresh(payload: RefreshRequest, db: Session = Depends(get_db)):
    token_data = decode_token(payload.refresh_token)
    if not token_data or not token_data.user_id:
        raise HTTPException(status_code=401, detail="Invalid refresh token")

    user = db.query(User).filter(User.id == token_data.user_id).first()
    if not user or not user.is_active:
        raise HTTPException(status_code=401, detail="User not found or inactive")

    return TokenOut(
        access_token=create_access_token(user.id, user.role.name),
        refresh_token=create_refresh_token(user.id),   # rotate refresh token
    )


# ── Me ────────────────────────────────────────────────────
@router.get("/me", response_model=UserOut)
def me(current_user: User = Depends(get_current_active_user)):
    return {
        "id":        current_user.id,
        "email":     current_user.email,
        "is_active": current_user.is_active,
        "role":      current_user.role.name,
    }
```

---

## Part E — Protecting Routes with RBAC

### Step 10 — Apply auth to existing routers

```python
# routers/products.py
from auth.dependencies import get_current_active_user, require_role

# 🟢 Public — anyone can browse products (no auth required)
@router.get("/", response_model=list[ProductOut])
def list_products(db: Session = Depends(get_db)):
    ...

# 🔒 Protected — must be logged in to see a single product
@router.get("/{product_id}", response_model=ProductOut)
def get_product(
    product_id: int,
    db: Session = Depends(get_db),
    current_user = Depends(get_current_active_user),   # ← inject here
):
    ...

# 🔒 Admin only — create product
@router.post(
    "/",
    response_model=ProductOut,
    status_code=201,
    dependencies=[Depends(require_role("admin"))],     # ← cleaner for role-only checks
)
def create_product(payload: ProductCreate, db: Session = Depends(get_db)):
    ...

# 🔒 Admin or staff — update inventory
@router.patch(
    "/{product_id}/inventory",
    dependencies=[Depends(require_role("admin", "staff"))],
)
def update_inventory(product_id: int, quantity: int, db: Session = Depends(get_db)):
    inv = db.query(Inventory).filter(Inventory.product_id == product_id).first()
    if not inv:
        raise HTTPException(status_code=404, detail="Inventory record not found")
    inv.quantity = quantity
    db.commit()
    return {"product_id": product_id, "quantity": quantity}


# 🔒 Admin only — delete product
@router.delete(
    "/{product_id}",
    status_code=204,
    dependencies=[Depends(require_role("admin"))],
)
def delete_product(product_id: int, db: Session = Depends(get_db)):
    product = db.query(Product).filter(Product.id == product_id).first()
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    db.delete(product)
    db.commit()
```

---

### Step 11 — Customer can only access their own orders

```python
# routers/orders.py
@router.get("/my", response_model=list[OrderOut])
def my_orders(
    current_user: User = Depends(get_current_active_user),
    db: Session = Depends(get_db),
):
    # Customer sees only their own orders
    return (
        db.query(Order)
        .options(joinedload(Order.items).joinedload(OrderItem.product))
        .filter(Order.customer_id == current_user.id)
        .order_by(Order.created_at.desc())
        .all()
    )


@router.get("/{order_id}", response_model=OrderOut)
def get_order(
    order_id: int,
    current_user: User = Depends(get_current_active_user),
    db: Session = Depends(get_db),
):
    order = db.query(Order).filter(Order.id == order_id).first()
    if not order:
        raise HTTPException(status_code=404, detail="Order not found")

    # Admins and staff can see any order; customers only see their own
    if current_user.role.name == "customer" and order.customer_id != current_user.id:
        raise HTTPException(status_code=403, detail="Not your order")

    return order
```

---

## Part F — Project Structure

```
myapp/
├── main.py
├── config.py
├── database.py
├── models.py
├── schemas.py
├── auth/
│   ├── __init__.py
│   ├── password.py       ← hash / verify
│   ├── tokens.py         ← create / decode JWT
│   └── dependencies.py   ← get_current_user, require_role
└── routers/
    ├── auth.py
    ├── products.py
    ├── orders.py
    ├── reviews.py
    └── inventory.py
```

---

### Step 12 — Wire everything in `main.py`

```python
from fastapi import FastAPI
from database import Base, engine
from routers import auth, products, orders, reviews, inventory

Base.metadata.create_all(bind=engine)

app = FastAPI(title="Webstore API")

app.include_router(auth.router)
app.include_router(products.router, prefix="/products", tags=["Products"])
app.include_router(orders.router,   prefix="/orders",   tags=["Orders"])
app.include_router(reviews.router,  prefix="/reviews",  tags=["Reviews"])
app.include_router(inventory.router,prefix="/inventory",tags=["Inventory"])
```

---

## Auth Endpoint Map

| Method | Path | Auth required | Role |
|--------|------|---------------|------|
| POST | `/auth/register` | No | — |
| POST | `/auth/login` | No | — |
| POST | `/auth/refresh` | No (needs refresh token) | — |
| GET | `/auth/me` | ✅ Yes | Any |
| GET | `/products/` | No | — |
| GET | `/products/{id}` | ✅ Yes | Any |
| POST | `/products/` | ✅ Yes | admin |
| DELETE | `/products/{id}` | ✅ Yes | admin |
| PATCH | `/products/{id}/inventory` | ✅ Yes | admin, staff |
| GET | `/orders/my` | ✅ Yes | Any |
| GET | `/orders/{id}` | ✅ Yes | admin/staff = any; customer = own only |

---

## Gotchas

1. **Never store plain passwords** — always hash with bcrypt before inserting. Never log or return `hashed_password` in a response.
2. **Access token TTL should be short** — 15–30 minutes is standard. The refresh token carries the longer session (7–30 days).
3. **Rotate refresh tokens** — issue a new refresh token on every `/auth/refresh` call. This limits the damage if one is stolen.
4. **`OAuth2PasswordRequestForm` expects form data, not JSON** — the login endpoint takes `username` + `password` as form fields, not a JSON body. This matches the OAuth2 spec and works automatically with Swagger's Authorize button.
5. **`dependencies=[Depends(...)]` vs injecting in signature** — use `dependencies=[...]` when you only need the side effect (auth check). Inject into the function signature when you also need the `current_user` object inside the handler.
6. **Load the role relationship** — if you check `current_user.role.name` in a route, make sure the `role` relationship is loaded. Add `joinedload(User.role)` to your `get_current_user` query or it triggers a lazy load (which crashes in async).
7. **`EmailStr` requires `email-validator`** — install with `pip install email-validator` if you use `EmailStr` in Pydantic schemas.

---

## Alembic Migration for Auth Tables

```bash
alembic revision --autogenerate -m "add users and roles tables"
alembic upgrade head
```

Seed roles before any users:

```python
# seed.py — run once
from database import SessionLocal
from models import Role

db = SessionLocal()
for name in ["admin", "staff", "customer"]:
    if not db.query(Role).filter(Role.name == name).first():
        db.add(Role(name=name))
db.commit()
db.close()
```

```bash
python seed.py
```

---

*Next suggested → Group 3: Pagination, Filtering & Error Handling*
