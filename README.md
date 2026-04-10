# Backend Runbooks

> Personal reference library for building API endpoints across frameworks and languages.  
> Each runbook is a sequential, copy-paste-friendly guide — from setup to moderately complex queries.  
> All runbooks share a common [E-Commerce scenario](#the-scenario) so concepts are directly comparable across frameworks.

---

## Quick Jump

| Framework | Language | Status |
|---|---|---|
| [FastAPI](#fastapi) | Python | 🟢 In progress |
| [Django REST Framework](#django-rest-framework-drf) | Python | 🔲 Planned |
| [Flask](#flask) | Python | 🔲 Planned |
| [Express](#express) | Node.js | 🔲 Planned |
| [NestJS](#nestjs) | Node.js / TypeScript | 🔲 Planned |
| [Fastify](#fastify) | Node.js / TypeScript | 🔲 Planned |
| [Spring Boot](#spring-boot) | Java | 🔲 Planned |
| [ASP.NET Core Web API](#aspnet-core-web-api) | C# | 🔲 Planned |

---

## The Scenario

All runbooks use the same **e-commerce schema** so you can compare patterns across frameworks without re-learning the domain.

📄 [`_shared/ecommerce-schema.md`](./_shared/ecommerce-schema.md)

**8 tables:**

```
customers ──< orders ──< order_items >── products ──< inventory
    │                                        │
    └──< addresses                           └──< categories
    └──< reviews >─────────────────────── products
```

**Endpoint complexity levels used throughout:**
- 🟢 Simple — single table, basic CRUD
- 🟡 Medium — joins, filters, transactions
- 🟠 Moderate — aggregates, GROUP BY, LEFT JOIN, ownership rules

---

## FastAPI

> Python · SQLAlchemy · Pydantic · Uvicorn

| # | Runbook | Topics |
|---|---|---|
| 01 | [Core Setup & First Endpoint](./fastapi/01-core-setup.md) | Project layout, DB connection, ORM model, Pydantic schema, router, single table CRUD, joined query |
| 02 | [E-Commerce Scenario](./fastapi/02-ecommerce-scenario.md) | Full 8-table schema, 14 endpoints from simple to moderate, aggregates, transactions, LEFT JOIN |
| 03 | [Async SQLAlchemy + Alembic](./fastapi/03-async-alembic.md) | AsyncSession, asyncpg, sync→async query translation, Alembic init, autogenerate, upgrade, rollback |
| 04 | [Auth — JWT + RBAC](./fastapi/04-auth-jwt-rbac.md) | Password hashing, access + refresh tokens, protected routes, role-based access control, ownership checks |

---

## Django REST Framework (DRF)

> Python · Django ORM · DRF Serializers · ViewSets

| # | Runbook | Topics |
|---|---|---|
| 01 | Core Setup & First Endpoint | _Planned_ |
| 02 | E-Commerce Scenario | _Planned_ |
| 03 | Auth — Token + JWT | _Planned_ |

---

## Flask

> Python · SQLAlchemy · Flask-RESTful / Flask-Smorest

| # | Runbook | Topics |
|---|---|---|
| 01 | Core Setup & First Endpoint | _Planned_ |
| 02 | E-Commerce Scenario | _Planned_ |

---

## Express

> Node.js · Sequelize / Prisma · JavaScript

| # | Runbook | Topics |
|---|---|---|
| 01 | Core Setup & First Endpoint | _Planned_ |
| 02 | E-Commerce Scenario | _Planned_ |
| 03 | Auth — JWT + Middleware | _Planned_ |

---

## NestJS

> Node.js · TypeScript · TypeORM / Prisma · Guards

| # | Runbook | Topics |
|---|---|---|
| 01 | Core Setup & First Endpoint | _Planned_ |
| 02 | E-Commerce Scenario | _Planned_ |
| 03 | Auth — JWT + Guards + RBAC | _Planned_ |

---

## Fastify

> Node.js · TypeScript · Prisma

| # | Runbook | Topics |
|---|---|---|
| 01 | Core Setup & First Endpoint | _Planned_ |
| 02 | E-Commerce Scenario | _Planned_ |

---

## Spring Boot

> Java · Spring Data JPA · Hibernate · Maven

| # | Runbook | Topics |
|---|---|---|
| 01 | Core Setup & First Endpoint | _Planned_ |
| 02 | E-Commerce Scenario | _Planned_ |
| 03 | Auth — Spring Security + JWT | _Planned_ |

---

## ASP.NET Core Web API

> C# · Entity Framework Core · .NET 8+

| # | Runbook | Topics |
|---|---|---|
| 01 | Core Setup & First Endpoint | _Planned_ |
| 02 | E-Commerce Scenario | _Planned_ |
| 03 | Auth — JWT + Identity + RBAC | _Planned_ |

---

## Cross-Framework Cheatsheet

> How the same concept maps across frameworks — updated as runbooks are completed.

| Concept | FastAPI | DRF | Express | NestJS | Spring Boot | ASP.NET |
|---|---|---|---|---|---|---|
| Route definition | `@router.get("/")` | `@action` / ViewSet | `router.get("/")` | `@Get("/")` | `@GetMapping("/")` | `[HttpGet]` |
| ORM / query builder | SQLAlchemy | Django ORM | Sequelize / Prisma | TypeORM / Prisma | Spring Data JPA | EF Core |
| Request body | Pydantic model | Serializer | `req.body` | DTO class | `@RequestBody` | `[FromBody]` |
| Dependency injection | `Depends()` | — | middleware | `@Injectable()` | `@Autowired` | Constructor DI |
| Auth middleware | `Depends(get_current_user)` | `IsAuthenticated` | middleware | `AuthGuard` | `@PreAuthorize` | `[Authorize]` |
| Migrations | Alembic | `manage.py migrate` | Sequelize CLI / Prisma | TypeORM CLI / Prisma | Flyway / Liquibase | `dotnet ef migrations` |
| Validation | Pydantic | Serializer `.is_valid()` | `express-validator` / Zod | class-validator | Bean Validation | Data Annotations |
| Auto API docs | `/docs` (Swagger) | `/api/` (Browsable) | Manual / Swagger | `/api` (Swagger) | Springdoc | Swashbuckle |

---

## Repo Structure

```
backend-runbooks/
├── README.md
├── _shared/
│   └── ecommerce-schema.md
├── fastapi/
│   ├── 01-core-setup.md
│   ├── 02-ecommerce-scenario.md
│   ├── 03-async-alembic.md
│   └── 04-auth-jwt-rbac.md
├── drf/
├── flask/
├── express/
├── nestjs/
├── fastify/
├── spring-boot/
└── dotnet-webapi/
```

---

## Purpose

Built as a personal reference for:
- Quickly recalling how to wire up an endpoint in a framework not used recently
- Interview preparation across multiple backend stacks
- Comparing how the same pattern (auth, pagination, joins) is handled differently per framework

---

*Runbooks are living documents — updated as new patterns are encountered.*
