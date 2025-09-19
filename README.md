# GWDG-projects

Here’s a detailed but tidy recap of what you nailed in this thread:

# Big picture: SQLAlchemy vs. SQLModel

* **SQLAlchemy** is the core DB toolkit + ORM (engines, sessions, transactions, queries, relationships).
* **SQLModel** is a thin, friendlier layer on top of SQLAlchemy + Pydantic: typed models that can double as DB tables *and* API schemas (great with FastAPI).
* For async behavior, docs + best practices come from **SQLAlchemy**; SQLModel reuses that machinery.

# Sessions, sessionmaker, and async best practices

* A **Session** is your “unit of work” (tracks objects, groups operations into a transaction).
* Use **one `AsyncSession` per request/task**; never share across concurrent coroutines.
* Create **one global session factory** (prefer `async_sessionmaker`) next to the engine:

  * Configure once (`expire_on_commit=False`, `autoflush`, etc.).
  * Get **new, consistent sessions** on demand.
  * Avoid **ad-hoc** `AsyncSession(a_engine)` scattered across files (prevents config drift).
* Provide a single helper:

  * `get_a_session()` as an `@asynccontextmanager` that yields a fresh session, and guarantees **rollback on error** and **close**.
  * Choose **one transaction policy** repo-wide:

    * **Explicit commits** in handlers, *or*
    * **Unit-of-work** via `async with session.begin():` (auto-commit/rollback; don’t call `commit()` inside).

# `expire_on_commit=False` (why we set it at the factory)

* Default SQLAlchemy behavior **expires** objects after `commit()`; reading attributes can trigger an **implicit DB refresh**.
* In **async**, implicit I/O can happen where you can’t `await`, causing errors/surprises.
* Setting **`expire_on_commit=False`** avoids surprise I/O; when you truly need fresh DB values, call **`await session.refresh(obj)`** explicitly.

# Relationship loading in async (no hidden I/O)

* Avoid plain lazy access (`obj.rel`) that triggers implicit I/O.
* Two safe patterns:

  * **Eager load** when querying (e.g., `options(selectinload(Model.rel))`)—great for collections; avoids N+1 queries.
  * **On demand** with **`await obj.awaitable_attrs.rel`** if you didn’t eager-load.

# Many-to-many with a link model

* `link_model=UserAccessGroupLink` tells SQLModel which **bridge table** connects `User` ↔ `AccessGroup`.
* Appending/removing from `user.access_groups` inserts/deletes rows in the link table; with `lazy="selectin"` groups load efficiently.

# Engines and `init_db()`

* You kept **two engines**:

  * **Async engine (`a_engine`)** for app runtime (awaitable queries).
  * **Sync engine (`engine`)** for bootstrap tasks like `create_all()` (or migrations); you *can* also do this with async engine via `run_sync`.
* `init_db()` calls `SQLModel.metadata.create_all(engine, checkfirst=True)`:

  * Creates **missing** tables that your models define in the DB pointed to by `engine`.
  * It does **not** alter existing tables; for schema evolution use **Alembic** migrations.

# `create_all()` behavior

* It runs `CREATE TABLE` **only** for tables defined in your models that **don’t exist yet**.
* It does **not** drop or modify columns/indexes.
* Useful for dev/test/bootstrap; not a replacement for migrations.

# Code organization you settled on

* **`db_models.py`**: **only models** (tables, relationships, events).
* **`db_methods.py`** (or `db.py`): engines, global `AsyncSessionFactory`, `get_a_session()`, `init_db()`, and any DB setup (e.g., SQLite PRAGMAs).
* Everywhere else: `async with get_a_session() as session:` for DB work.

# Sequential vs concurrent use of a session

* Multiple `await session.exec(...)` calls **in the same `async with` block** are **sequential**—totally fine.
* **Don’t** reuse the same session across multiple coroutines concurrently (e.g., `asyncio.gather`); if you need parallelism, give each coroutine its **own session** from the factory.

# Why not ad-hoc `AsyncSession(a_engine)`?

* **Config drift** (inconsistent options), **async pitfalls** (implicit refresh), **no centralized cleanup**, **harder testing**, **no single source of truth**, **weaker concurrency hygiene**, and **more boilerplate**.
* The factory enforces consistency and gives one place to change policy and add logging/metrics.

# Practical examples you walked through

* **Inserting a new user** (with/without groups), using `get_a_session()`, `commit()`, and `refresh()`.
* **Rewriting ad-hoc session code** to use the factory.
* **Combining queries** (optional join) to reduce round trips.

# Issue/PR communication and commits

* Wrote a clear **issue comment** proposing: global `AsyncSessionFactory`, `get_a_session()`, one transaction policy, eager/awaitable relationship rules, and moving engines out of `db_models.py`.
* Crafted **commit messages** (Conventional Commits) and a command including **“Refs #34”**.

---

## TL;DR “team rules” you can pin

* One **global** `AsyncSessionFactory` next to the engine; **never** ad-hoc sessions.
* **One `AsyncSession` per request/task**; guaranteed rollback/close via `get_a_session()`.
* Pick **explicit commits** *or* **unit-of-work**—don’t mix.
* Set **`expire_on_commit=False`**; **`await session.refresh(obj)`** when needed.
* For relationships in async: **`selectinload`** or **`awaitable_attrs`**.
* Models in **`db_models.py`**; engines + factory + helper in **`db_methods.py`**; use Alembic for schema changes.

   
