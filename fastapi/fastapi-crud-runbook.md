# Runbook: FastAPI — Create a CRUD API Endpoint

> **Series:** Backend Runbooks — FastAPI Edition  
> **Level:** Beginner-friendly  
> **Goal:** Spin up a working GET endpoint from a single table, then extend it to a joined multi-table query.

---

## Part 1 — Single Table Endpoint

### Step 1 — Install dependencies

```bash
pip install fastapi uvicorn sqlalchemy psycopg2-binary
# swap psycopg2-binary for aiosqlite if using SQLite
```

---

### Step 2 — Project layout

```
myapp/
├── main.py
├── database.py
├── models.py
├── schemas.py
└── routers/
    └── users.py
```

---

### Step 3 — Wire up the database connection (`database.py`)

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase

DATABASE_URL = "postgresql://user:password@localhost:5432/mydb"
# SQLite alternative: "sqlite:///./mydb.db"

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

class Base(DeclarativeBase):
    pass

# Dependency — call this in every route
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

---

### Step 4 — Define the ORM model (`models.py`)

```python
from sqlalchemy import Column, Integer, String
from database import Base

class User(Base):
    __tablename__ = "users"

    id      = Column(Integer, primary_key=True, index=True)
    name    = Column(String, nullable=False)
    email   = Column(String, unique=True, nullable=False)
```

---

### Step 5 — Define Pydantic schemas (`schemas.py`)

> Schemas validate request/response data. Keep them separate from ORM models.

```python
from pydantic import BaseModel

# What you return from the API
class UserOut(BaseModel):
    id:    int
    name:  str
    email: str

    model_config = {"from_attributes": True}  # Pydantic v2

# What you accept in a POST body
class UserCreate(BaseModel):
    name:  str
    email: str
```

---

### Step 6 — Write the router (`routers/users.py`)

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from database import get_db
from models import User
from schemas import UserOut, UserCreate

router = APIRouter(prefix="/users", tags=["users"])

# GET all users
@router.get("/", response_model=list[UserOut])
def list_users(db: Session = Depends(get_db)):
    return db.query(User).all()

# GET single user by id
@router.get("/{user_id}", response_model=UserOut)
def get_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

# POST — create a user
@router.post("/", response_model=UserOut, status_code=201)
def create_user(payload: UserCreate, db: Session = Depends(get_db)):
    user = User(**payload.model_dump())
    db.add(user)
    db.commit()
    db.refresh(user)
    return user

# DELETE a user
@router.delete("/{user_id}", status_code=204)
def delete_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    db.delete(user)
    db.commit()
```

---

### Step 7 — Mount the router in `main.py`

```python
from fastapi import FastAPI
from database import Base, engine
from routers import users

Base.metadata.create_all(bind=engine)  # creates tables if they don't exist

app = FastAPI(title="My API")
app.include_router(users.router)
```

---

### Step 8 — Run the server

```bash
uvicorn main:app --reload
```

Open `http://localhost:8000/docs` — you get a free interactive Swagger UI.

---

---

## Part 2 — Multi-Table Joined Endpoint

**Scenario:** Each `Post` belongs to a `User`. Return posts with their author's name.

---

### Step 9 — Add the second model (`models.py`)

```python
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship
from database import Base

class User(Base):
    __tablename__ = "users"
    id    = Column(Integer, primary_key=True, index=True)
    name  = Column(String, nullable=False)
    email = Column(String, unique=True, nullable=False)

    posts = relationship("Post", back_populates="author")

class Post(Base):
    __tablename__ = "posts"
    id        = Column(Integer, primary_key=True, index=True)
    title     = Column(String, nullable=False)
    body      = Column(String)
    user_id   = Column(Integer, ForeignKey("users.id"), nullable=False)

    author    = relationship("User", back_populates="posts")
```

---

### Step 10 — Add a joined schema (`schemas.py`)

```python
# Nested schema — embed the author inside the post response
class AuthorOut(BaseModel):
    id:   int
    name: str
    model_config = {"from_attributes": True}

class PostOut(BaseModel):
    id:     int
    title:  str
    body:   str | None
    author: AuthorOut          # <-- nested join result
    model_config = {"from_attributes": True}

class PostCreate(BaseModel):
    title:   str
    body:    str | None = None
    user_id: int
```

---

### Step 11 — Write the joined router (`routers/posts.py`)

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session, joinedload
from database import get_db
from models import Post, User
from schemas import PostOut, PostCreate

router = APIRouter(prefix="/posts", tags=["posts"])

# GET all posts — SQLAlchemy eager-loads the related User in one query
@router.get("/", response_model=list[PostOut])
def list_posts(db: Session = Depends(get_db)):
    return (
        db.query(Post)
        .options(joinedload(Post.author))   # JOIN users ON posts.user_id = users.id
        .all()
    )

# GET posts by a specific user — explicit filter + join
@router.get("/by-user/{user_id}", response_model=list[PostOut])
def posts_by_user(user_id: int, db: Session = Depends(get_db)):
    return (
        db.query(Post)
        .options(joinedload(Post.author))
        .filter(Post.user_id == user_id)
        .all()
    )

# POST — create a post
@router.post("/", response_model=PostOut, status_code=201)
def create_post(payload: PostCreate, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == payload.user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    post = Post(**payload.model_dump())
    db.add(post)
    db.commit()
    db.refresh(post)
    # re-query to get the eager-loaded author in the response
    return db.query(Post).options(joinedload(Post.author)).filter(Post.id == post.id).first()
```

---

### Step 12 — Register the new router in `main.py`

```python
from routers import users, posts

app.include_router(users.router)
app.include_router(posts.router)
```

---

### Step 13 — Raw SQL join (alternative, when ORM is overkill)

```python
from sqlalchemy import text

@router.get("/raw", response_model=list[PostOut])
def list_posts_raw(db: Session = Depends(get_db)):
    sql = text("""
        SELECT p.id, p.title, p.body,
               u.id AS user_id, u.name AS user_name
        FROM   posts p
        JOIN   users u ON u.id = p.user_id
    """)
    rows = db.execute(sql).mappings().all()
    # manually shape into dicts matching your schema
    return [
        {
            "id":     r["id"],
            "title":  r["title"],
            "body":   r["body"],
            "author": {"id": r["user_id"], "name": r["user_name"]},
        }
        for r in rows
    ]
```

---

---

## Quick Reference Cheatsheet

| Concept | FastAPI pattern |
|---|---|
| Route decorator | `@router.get("/path")` |
| Path parameter | `def fn(id: int)` in signature |
| Query parameter | `def fn(skip: int = 0, limit: int = 10)` |
| Request body | `payload: MySchema` (Pydantic model) |
| DB session injection | `db: Session = Depends(get_db)` |
| 404 response | `raise HTTPException(status_code=404, detail="...")` |
| Eager join | `.options(joinedload(Model.relation))` |
| Nested response | Pydantic model with a nested model as a field |
| Auto docs | `http://localhost:8000/docs` (Swagger) |

---

## Common Gotchas

1. **`from_attributes = True` is required** in Pydantic v2 when serialising SQLAlchemy ORM objects. Without it you get a validation error.
2. **`db.refresh(obj)` after commit** repopulates the object with DB-generated values (e.g. auto-increment id).
3. **`joinedload` vs `lazy loading`** — by default SQLAlchemy lazy-loads relations, which causes N+1 queries. Always use `joinedload` (or `selectinload`) when you know you need the relation.
4. **`Base.metadata.create_all`** is fine for development. Use **Alembic** for production migrations.
5. **`--reload` flag** in uvicorn is for development only. Remove it in production.

---

## Next Steps

- Add **Alembic** for database migrations
- Add **authentication** with `python-jose` + `passlib` (JWT)
- Switch to **async** routes with `asyncpg` + `SQLAlchemy async session` for higher throughput
- Add **pagination** with `skip` and `limit` query params

---

*Next runbook → Django REST Framework (DRF) edition*
