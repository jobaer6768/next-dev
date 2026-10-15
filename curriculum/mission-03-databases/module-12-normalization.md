# Module 12 — Database Normalization, Postgres Installation & Application Insights

> Turn a spreadsheet-shaped mess into a set of well-formed tables that never lie.

---

## Why this module matters

The brief from Module 11 gave you entities and relationships. The next question is: *how do I split them into tables so I never have to update the same fact in two places?*

Normalization is the answer. It's a set of rules — First, Second, Third Normal Form — that, when applied, removes redundant data and protects the integrity of the database.

This module is dense. Take it slowly. By the end you should be able to take a "messy spreadsheet" and produce a normalized schema with a clean ER diagram.

---

## Learning objectives

- Explain anomalies (update, insertion, deletion) and how normalization prevents them.
- Define 1NF, 2NF, 3NF.
- Identify and resolve functional dependencies.
- Resolve M:N relationships with a junction table.
- Install and use Postgres + pgAdmin.
- Trace data flow through an application.

---

## 1. The three anomalies

Consider a single table that stores `orders` and their `customer` info inline:

```
orders:
  id, customer_name, customer_email, customer_city, product_name, quantity, total
  1,  Rumi,    rumi@x.com,    Dhaka,    Mouse,         2,         200
  2,  Rumi,    rumi@x.com,    Dhaka,    Keyboard,      1,         500
  3,  Nadia,   nadia@x.com,   Chittagong, Mouse,      1,         100
```

**Update anomaly** — Rumi moves to Sylhet. You have to update two rows. If you miss one, the data is inconsistent.

**Insertion anomaly** — You want to add a new customer who hasn't ordered yet. You can't — there's no order to put them in.

**Deletion anomaly** — You delete Nadia's only order. Now you have no record that Nadia was ever a customer.

Normalization fixes all three by separating the entities into their own tables.

---

## 2. Functional dependency

A **functional dependency** `X → Y` means: given X, Y is determined.

In the `orders` table above:

- `id → customer_name, customer_email, product_name, quantity, total`
- `customer_email → customer_name, customer_city`
- `(id, product_name) → quantity, unit_price`

Normalization replaces tables with smaller ones whose dependencies are *about* one entity.

---

## 3. First Normal Form (1NF)

A table is in 1NF if:

1. Every cell holds an **atomic** (single) value.
2. There are no **repeating groups**.
3. Every row is unique (has a primary key).

**Violation:**

```
order_id | items
1        | "Mouse, Keyboard, Cable"
```

**Fix:** create an `order_items` table.

```sql
CREATE TABLE order_items (
  order_id   UUID REFERENCES orders(id),
  product_id UUID REFERENCES products(id),
  quantity   INT NOT NULL,
  PRIMARY KEY (order_id, product_id)
);
```

**1NF = no lists, no JSON-as-row, no comma-separated values.**

---

## 4. Second Normal Form (2NF)

A table is in 2NF if it is in 1NF **and** every non-key column depends on the *whole* primary key, not just part of it.

**Violation:**

```
order_items(order_id, product_id, quantity, product_name, product_price)
```

`product_name` depends only on `product_id`, not on `(order_id, product_id)`.

**Fix:** move product info to a `products` table. `order_items` keeps only `quantity`.

---

## 5. Third Normal Form (3NF)

A table is in 3NF if it is in 2NF **and** no non-key column depends on *another non-key column* (no transitive dependencies).

**Violation:**

```
orders(id, customer_id, customer_name, customer_email, customer_city, total)
```

`customer_name` depends on `customer_id` (not on `id`). Transitive.

**Fix:** move customer info to a `customers` table. `orders` keeps only `customer_id`.

---

## 6. The normalized SaaS schema (the one we'll use in Mission 4)

```
users(id, email, name, password_hash, created_at, updated_at)

workspaces(id, name, owner_id → users.id, created_at, updated_at)

memberships(id, user_id → users.id, workspace_id → workspaces.id,
            role, status, invited_at, joined_at, created_at, updated_at)
  -- UNIQUE(user_id, workspace_id)

projects(id, workspace_id → workspaces.id, name, description, created_by → users.id,
         created_at, updated_at)

tasks(id, project_id → projects.id, title, description,
      assignee_id → users.id NULL, status, priority,
      due_at NULL, created_by → users.id, created_at, updated_at)

comments(id, task_id → tasks.id, author_id → users.id, body, created_at, updated_at)

attachments(id, task_id → tasks.id, uploader_id → users.id, url, size, mime, created_at)

plans(id, code, name, price_cents, features JSONB, created_at)

subscriptions(id, workspace_id → workspaces.id UNIQUE, plan_id → plans.id,
              status, current_period_end, created_at, updated_at)
```

This is a **3NF** schema:

- Every non-key column depends on the primary key, the whole key, and nothing but the key.
- M:N (User ↔ Workspace) is resolved by `memberships` with role and status.
- Each entity is a table; relationships are foreign keys.

---

## 7. Junction tables — the M:N pattern

```sql
CREATE TABLE memberships (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  role        TEXT NOT NULL CHECK (role IN ('owner', 'admin', 'member')),
  status      TEXT NOT NULL CHECK (status IN ('invited', 'active', 'suspended')),
  invited_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  joined_at   TIMESTAMPTZ,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (user_id, workspace_id)
);
```

The `UNIQUE (user_id, workspace_id)` prevents a user from being added to the same workspace twice. The `role` and `status` columns are *attributes* of the relationship itself — a common reason junction tables exist.

---

## 8. Installing Postgres (recap with focus on app integration)

### macOS

```bash
brew install postgresql@16
brew services start postgresql@16
psql postgres
```

### Ubuntu / WSL

```bash
sudo apt install -y postgresql-16
sudo service postgresql start
sudo -u postgres psql
```

### Docker (any OS)

```bash
docker run -d --name pg \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_DB=next_level \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

docker exec -it pg psql -U postgres -d next_level
```

### pgAdmin

A web-based GUI at `http://localhost:5050` if you run `pgadmin4` in Docker. It is great for:

- Browsing tables and indexes.
- Writing and running ad-hoc SQL.
- Visualizing ER diagrams.
- Examining query plans.

For real work we use `psql` (or any SQL client like **TablePlus**, **DBeaver**, **DataGrip**).

---

## 9. Tracing data flow through an application

```
   ┌──────────┐
   │  Client  │  HTTP POST /api/v1/tasks
   └────┬─────┘
        │  JSON body: { title, projectId, assigneeId }
        ▼
   ┌──────────┐
   │  Express │  Zod validates body
   │ handler  │  → usersService.create(input)
   └────┬─────┘
        │  validated input
        ▼
   ┌──────────┐
   │ Service  │  Business rules (e.g. user is member of project)
   └────┬─────┘
        │  clean input
        ▼
   ┌──────────┐
   │   Repo   │  INSERT INTO tasks (...) RETURNING *
   └────┬─────┘
        │  row
        ▼
   ┌──────────┐
   │ Postgres │  writes to disk, updates indexes, logs WAL
   └──────────┘
```

**Every boundary validates.** HTTP body is not the same shape as the service input is not the same shape as the SQL parameter.

---

## 10. Common normalization mistakes

1. **Over-normalizing.** Some denormalization is fine for performance (a `comment_count` on `tasks` is OK if you measure the read cost).
2. **JSON everywhere.** JSONB has its place (settings, variable-shape metadata). For core entities, use columns.
3. **Storing derived data** (e.g. `total = quantity * unit_price` on every row). Compute on read.
4. **Forgetting `created_at`/`updated_at`.** Add them to every table.

---

## Exercise

1. Take a 10-row spreadsheet of fake "orders" with customer info, products, and quantities inline. Identify all 1NF, 2NF, and 3NF violations.
2. Normalize it into 4 tables: `customers`, `products`, `orders`, `order_items`.
3. Write the `CREATE TABLE` SQL with proper primary keys, foreign keys, and `ON DELETE` behavior.
4. Insert the 10 rows of seed data spread across the new tables.
5. Write a query that reproduces the original spreadsheet (using `JOIN`).
6. Install Postgres (or use Docker). Run your schema. Run your query.

When done, you have the foundation of the Mission 3 capstone: a normalized 3NF schema for a brief of your choice.

---

## AI Industry Note (2026)

- **"Just use MongoDB / use JSONB / use a vector store" is sometimes right, but for your *core* domain, a normalized relational schema is still the right default in 2026.** A senior engineer can defend that choice in 30 seconds: ACID, joins, mature tooling, a 50-year track record. JSONB and vector stores are *augmentations*, not replacements.
- **Schema review is a senior interview question in 2026.** "Take this brief, design the schema, justify each table, identify the anomalies you've prevented." If you can answer that in 15 minutes on a whiteboard, you are hired. This module is your whiteboard rehearsal.
- **What to read next:** Chris Date's *"Database Design and Relational Theory"* (the modern Codd). The normalization chapters alone are worth the price of the book.
