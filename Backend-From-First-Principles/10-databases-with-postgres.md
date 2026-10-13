# 10 — Mastering Databases with Postgres

**Video:** [#12 Mastering Databases with Postgres](https://www.youtube.com/watch?v=F7Vwp2Xo5Do) (2h 45m)

Why databases exist, relational vs non-relational, why this series picks Postgres, the data types that matter, migrations, and then a full schema (with enums, constraints, one-to-one / one-to-many / many-to-many, referential integrity), seeding, the queries behind each API, parameterized queries, indexes, and triggers.

The video **skips SQL and Postgres basics** on purpose (`SELECT`, `GROUP BY`, etc.). Learn those separately. Tool used for browsing data: **TablePlus**. Migration tool: **dbmate**.

---

## Contents

1. [What is a database, and why disk-based](#what-is-a-database-and-why-disk-based)
2. [DBMS and why not text files](#dbms-and-why-not-text-files)
3. [Relational vs non-relational](#relational-vs-non-relational)
4. [Why Postgres](#why-postgres)
5. [Postgres data types worth knowing](#postgres-data-types-worth-knowing)
6. [Migrations](#migrations)
7. [Schema design: the project management platform](#schema-design-the-project-management-platform)
8. [Seeding](#seeding)
9. [Queries behind the APIs](#queries-behind-the-apis)
10. [Parameterized queries](#parameterized-queries)
11. [Dynamic filter, sort, pagination](#dynamic-filter-sort-pagination)
12. [Insert and update](#insert-and-update)
13. [Indexes](#indexes)
14. [Triggers](#triggers)
15. [Summary](#summary)
16. [Interview questions](#interview-questions)

---

## What is a database, and why disk-based

[0:00](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=0s)

**Persistence** = data survives after the program that created it stops. A to-do app that forgets everything on close is useless.

In the broadest sense, a "database" is any structured storage: a phone's contact list, the browser's `localStorage`/cookies, even a text file. The common thread: a persistent system offering **create, read, update, delete (CRUD)**.

In a backend context, "database" means a **disk-based** database (HDD or SSD).

| | RAM (primary memory) | Disk (secondary storage) |
|---|---|---|
| Speed | Very fast | Slower |
| Cost / capacity | Expensive, small (8–128 GB typical) | Cheap, large (512 GB–2 TB+) |
| Used for | **Caches** (Redis, in-memory) | **Databases** (Postgres, MongoDB, ...) |

Databases need *space* more than raw speed, so they trade some speed for capacity. How they lay data out on disk is a deep topic; for an app engineer, it's enough to know the distinction.

---

## DBMS and why not text files

[8:46](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=526s)

A **DBMS** (Database Management System) is the software that stores data *and* gives efficient CRUD over it. Responsibilities:

| Responsibility | Meaning |
|---|---|
| **Data organisation** | Store data so reads/writes/updates are efficient |
| **Access** | Provide CRUD operations |
| **Integrity** | Data is accurate and valid. E.g. a `payment` column only ever holds numbers |
| **Security** | Users and roles, protecting from unauthorised access |

(Plus scaling, load balancing, etc.)

### Why not just use text files?

[12:21](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=741s)

1. **Parsing** — to find a customer you have to read the file, split lines and compare fields in application code. Slow and error-prone, and slower still in interpreted languages. A bug can corrupt data.
2. **No structure** — nothing enforces "this field is a number". Files take anything.
3. **Concurrency** — two users read `amount = 40` at the same time. One adds 20 (→ 60), the other subtracts 20 (→ 20). Whoever writes last wins, and you can't predict which. A text file has no mechanism to coordinate this.

A DBMS exists to solve exactly these problems.

---

## Relational vs non-relational

[18:36](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=1116s)

| | Relational | Non-relational (NoSQL) |
|---|---|---|
| Structure | **Tables, rows, columns**. Relationships via **foreign keys** | e.g. MongoDB: **collections** of **documents** |
| Schema | **Predefined and strict.** Columns and types fixed up front | Flexible. Each document can differ |
| Query language | **SQL** | Varies by product |
| Examples | PostgreSQL, MySQL, SQLite, SQL Server | MongoDB, DynamoDB |
| Strength | **Data integrity**: you can rely on the shape and consistency of data | Speed of prototyping, schema freedom |
| Weakness | Rigid, needs migrations | Integrity is pushed into **application code**, which is easier to get wrong |

Terminology mapping: table ≈ collection, row ≈ document.

**Use cases from the video:**

- **CRM** (contacts, sales opportunities): needs accurate, consistent, related data → **relational (Postgres)**.
- **CMS** (articles with images, code blocks, embeds, unpredictable shapes) → **non-relational** fits naturally. (But see the JSONB point below.)

---

## Why Postgres

[26:18](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=1578s)

1. **Open source and free**; self-hostable.
2. **Sticks to the SQL standard** → easier to migrate to MySQL etc. later.
3. **Extensible**: ~1,400-page docs, huge feature set, extension system.
4. **Reliable and scalable.**
5. **Excellent JSON support** (`jsonb` with indexing and querying). So you rarely need a separate NoSQL DB for dynamic data.

> MySQL may beat it in some benchmarks, but until you serve millions of users and hit a very specific bottleneck, **Postgres is the default first choice**.

SQL = the language. PostgreSQL = the database system that runs it.

---

## Postgres data types worth knowing

[33:26](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=2006s)

### Numbers

| Type | Notes |
|---|---|
| `serial` / `bigserial` | Auto-incrementing integer. Use `bigserial` in production when you want an integer primary key. |
| `smallint` < `integer` < `bigint` | Increasing capacity |
| `decimal(p, s)` / `numeric(p, s)` | Exact. `numeric(10,2)` = up to 10 digits total, 2 after the decimal point (e.g. `12345678.90`) |
| `real`, `double precision` (floats) | Approximate but fast |

**Rule:** money, prices, anything where accuracy matters → `decimal`. Measurements where tiny error doesn't matter (area, scientific values) → float (faster).

### Strings

[40:42](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=2442s)

| Type | Behaviour |
|---|---|
| `char(n)` | **Pads** with spaces to length `n`. Only use if length is always fixed (e.g. 2-letter day codes). Otherwise **never**. |
| `varchar(n)` | Variable length up to `n`. Error if exceeded. |
| `text` | Any length (up to ~1 GB). |

**Recommendation: use `text`.** Postgres's own docs recommend it; there's no performance difference vs `varchar`. `varchar(255)` is a MySQL convention and means nothing in Postgres, but people copy it. Changing the limit later needs a risky migration, and a future reader will wonder why `255`. Enforce length in application code instead.

### Others

| Type | Notes |
|---|---|
| `boolean` | true / false |
| `date`, `time`, `timestamp`, `timestamptz` | `timestamptz` (timestamp **with time zone**) stores the time zone info; the usual choice |
| `interval` | e.g. "10 days" |
| `uuid` | Native type. Popular for primary keys |
| `json` vs `jsonb` | `json` is stored as plain text. **`jsonb`** is parsed into a binary format for faster queries and indexing. **Prefer `jsonb`.** |
| arrays | e.g. `integer[]`, `text[]` |
| Misc | network addresses, MACs, geometric points, XML |

---

## Migrations

[54:02](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=3242s)

In production you **never** just open a GUI and type SQL against the database. There'd be no record of what changed, when or who did it, and no way to version or roll back.

**Migrations** = ordered SQL files, committed to Git alongside code.

```
db/
  migrations/
    20260101_create_users_table.sql
    20260102_seed_data.sql
    20260103_add_indexes_and_triggers.sql
```

A CLI tool (dbmate, golang-migrate, ...) applies them **in sequence**. File names are sequential numbers or **timestamps** so ordering is unambiguous.

Each file has two sections:

```sql
-- migrate:up
CREATE TABLE ...;

-- migrate:down
DROP TABLE ...;
```

| Section | Purpose |
|---|---|
| **up** | Apply the change |
| **down** | **Revert** exactly that change, for rollback |

(Some modern teams skip down-migrations, but it's the standard approach.)

The tool maintains a bookkeeping table (`schema_migrations`) with the **current version**, so it knows which files are already applied and never re-runs them. Re-running `CREATE TABLE` would just fail with "already exists".

**Why migrations:**

1. **Track changes over time** in version control.
2. **Roll back** when something breaks.
3. Everyone (dev, CI, prod) goes through the same steps to reach the same schema.

```bash
dbmate new create_users_table   # creates db/migrations/<timestamp>_create_users_table.sql
dbmate up                       # apply pending migrations
dbmate down                     # roll back the last one
# needs DATABASE_URL in .env
```

---

## Schema design: the project management platform

[1:01:46](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=3706s)

Same domain as the REST API video: **users, user_profiles, projects, tasks, project_members**.

### Enums

[1:03:48](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=3828s)

```sql
CREATE TYPE project_status AS ENUM ('active', 'completed', 'archived');
CREATE TYPE task_status    AS ENUM ('pending', 'in_progress', 'completed', 'cancelled');
CREATE TYPE member_role    AS ENUM ('owner', 'admin', 'member');
```

Why enums instead of plain `text`:

1. **Data integrity at the database level.** An invalid value is rejected by the DB, not only by app code.
2. **Documentation.** Someone reading the migrations sees the *allowed values* at a glance, without digging through application code. The video calls this the more important reason.

### Table and column conventions

- Table names: **plural, lowercase** (`users`, `projects`). Plural is the industry norm, though teams may choose singular. Be consistent.
- Column names: **`snake_case`**. Postgres is case-insensitive for unquoted names, so `fullName` becomes `fullname`. Using camelCase would force you to double-quote it everywhere.
- Each table typically has `id`, `created_at`, `updated_at`.
- Most columns should be `NOT NULL`. The video suggests **70%+ of columns** should carry that constraint, because scripts and bugs otherwise leave nulls that make data hard to trust.

### users

```sql
CREATE TABLE users (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email         text NOT NULL UNIQUE,
  full_name     text NOT NULL,
  password_hash text NOT NULL,       -- never store the plaintext password
  created_at    timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now()
);
```

- **`PRIMARY KEY`** = uniquely identifies a row. It **implies `NOT NULL` + `UNIQUE`**, and is indexed automatically.
- `DEFAULT gen_random_uuid()` lets Postgres generate the ID, so the app doesn't pass one.
- **`UNIQUE`** on email: a second row with the same email fails at the DB level.
- `created_at` defaults to the current time.

### One-to-one: user_profiles

[1:17:45](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=4665s)

```sql
CREATE TABLE user_profiles (
  user_id     uuid PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  avatar_url  text,
  bio         text,
  phone       text,
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now()
);
```

**Why a separate table?** Profile data grows (social links, websites, ...). Splitting it out avoids repeatedly migrating the core `users` table. Common for profile, preferences and metadata.

**How:** the child's primary key **is** the parent's key (`user_id` is both PK and FK). No separate `id`. Optional fields have no `NOT NULL`.

### One-to-many: projects → tasks

[1:20:21](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=4821s)

```sql
CREATE TABLE projects (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name        text NOT NULL,
  description text,
  status      project_status NOT NULL DEFAULT 'active',
  owner_id    uuid NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE tasks (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id  uuid NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  title       text NOT NULL,
  description text,
  priority    integer NOT NULL DEFAULT 1 CHECK (priority BETWEEN 1 AND 5),
  status      task_status NOT NULL DEFAULT 'pending',
  due_date    date,
  assigned_to uuid REFERENCES users(id) ON DELETE SET NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now()
);
```

**One-to-many:** the "many" side (`tasks`) holds a **foreign key** to the "one" side (`projects`). It is *not* a primary key there. One project, many tasks.

### Referential integrity (`ON DELETE ...`)

[1:21:53](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=4913s)

Protects data across tables when a referenced row is deleted:

| Option | When the parent row is deleted... |
|---|---|
| **`RESTRICT`** | ...the delete **fails** if children exist. (`projects.owner_id`: can't delete a user who owns projects.) |
| **`CASCADE`** | ...the children are **deleted too**. (`tasks.project_id`: delete a project → delete its tasks.) |
| **`SET NULL`** | ...the FK is set to `NULL`. (`tasks.assigned_to`: deleted user → task becomes unassigned. Column must allow null.) |
| **`SET DEFAULT`** | ...the FK is set to the column default. |

A **foreign key constraint** also guarantees you can't insert a value that doesn't exist in the referenced table.

### Constraints summary

| Constraint | Effect |
|---|---|
| `NOT NULL` | value required |
| `UNIQUE` | no duplicates in the column |
| `PRIMARY KEY` | `NOT NULL` + `UNIQUE` + identity |
| `FOREIGN KEY ... REFERENCES` | value must exist in the other table |
| `CHECK (cond)` | custom rule, e.g. `priority BETWEEN 1 AND 5`. Insert/update fails otherwise |
| `DEFAULT` | value used when not supplied |

### Many-to-many: project_members (linking table)

[1:31:42](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=5502s)

A user can be in many projects, and a project can have many users (look at it both ways and both sides are "many").

```sql
CREATE TABLE project_members (
  project_id uuid NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  user_id    uuid NOT NULL REFERENCES users(id)    ON DELETE CASCADE,
  role       member_role NOT NULL DEFAULT 'member',
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (project_id, user_id)       -- composite primary key
);
```

- Use a **linking table** with two foreign keys.
- The **composite primary key** `(project_id, user_id)` means a given user can be in a given project only once.
- Data about the *relationship* (like `role`) lives here, because a role only makes sense for "this user in this project".

### Relationship recipe

| Relationship | Implementation |
|---|---|
| **1 : 1** | Child's primary key = parent's key (also FK) |
| **1 : N** | Child has a FK column to the parent |
| **N : M** | Linking table, with a composite primary key of two FKs |

### Down migration

[1:37:51](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=5871s)

Reverse the up, in **reverse order** (tables first, then types):

```sql
-- migrate:down
DROP TABLE project_members;
DROP TABLE tasks;
DROP TABLE projects;
DROP TABLE user_profiles;
DROP TABLE users;
DROP TYPE member_role;
DROP TYPE task_status;
DROP TYPE project_status;
```

---

## Seeding

[1:40:26](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=6026s)

In production, data arrives through real user flows. In **development** you need sample data to test against. **Seeding** = inserting test data. Best practice: put it in a **separate migration** file (`dbmate new seed_data`).

The video uses **CTEs** (`WITH ... AS`) for readability: insert users, `RETURNING id, email`, then insert profiles from that result, then projects, and so on.

---

## Queries behind the APIs

[1:44:31](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=6271s)

### GET /users — list with embedded profile

Tip: **write the `FROM` first** so you know where the data comes from. Use short **aliases**.

```sql
SELECT u.*, to_jsonb(up.*) AS profile
FROM users u
LEFT JOIN user_profiles up ON u.id = up.user_id
ORDER BY u.created_at DESC;
```

- **`LEFT JOIN`**, not `INNER JOIN`: a user who never created a profile has no row in `user_profiles`. An inner join would drop them. A left join keeps the user.
- `to_jsonb(up.*)` turns the profile row into a JSON object nested as `profile`, so the API returns user + profile in one call.
- **Always `ORDER BY`.** SQL gives no guaranteed row order otherwise. Default to `created_at DESC`.

(Inner and left joins are the ones used most.)

### GET /users/:userId

Same query with `WHERE u.id = $1` and no list ordering.

---

## Parameterized queries

[1:53:14](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=6794s)

```sql
SELECT u.*, to_jsonb(up.*) AS profile
FROM users u
LEFT JOIN user_profiles up ON u.id = up.user_id
WHERE u.id = $1;          -- $1 is a placeholder; value supplied separately
```

The query has an **empty slot**; the value is provided *separately* at execution. The database treats the value strictly as **data** (a string), never as SQL.

> If someone passes `'; DELETE FROM users; --` it's just a weird string, not a command. This prevents **SQL injection**. If you instead build the SQL by **concatenating strings**, you're vulnerable.

In real code, the driver or ORM does this for you (`$1, $2` in Postgres drivers). Always use it for any user-supplied value. The flow: route → handler → service → repository, which passes the ID into the parameter slot.

---

## Dynamic filter, sort, pagination

[1:59:58](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=7198s)

A list API supports `page`, `limit`, `letter` (filter), `sortBy`, `sortOrder`. Real code **builds the query dynamically** from whichever params were supplied. The SQL that results looks like:

```sql
SELECT u.*, to_jsonb(up.*) AS profile
FROM users u
LEFT JOIN user_profiles up ON u.id = up.user_id
WHERE u.full_name ILIKE $1 || '%'        -- $1 = 'j'  → names starting with J
ORDER BY u.email ASC                     -- sort column & direction from params
LIMIT $2 OFFSET $3;
```

| Piece | Notes |
|---|---|
| `ILIKE` | case-insensitive `LIKE`; `'j%'` = starts with j |
| Filter | only add the `WHERE` clause **if the client sent the param** |
| Defaults | `page = 1`, `limit = 10/20`, `sortBy = created_at`, `sortOrder = desc` |
| Sort column | **Allow-list** the columns a client may sort by (e.g. `full_name`, `email`, `created_at`). Never pass arbitrary user text into `ORDER BY` |
| Pagination | `LIMIT n OFFSET (page - 1) * n`. API pages start at 1, **SQL offset starts at 0** |

Most list endpoints should be paginated unless there's a reason not to.

---

## Insert and update

[2:11:15](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=7875s)

### POST /users

```sql
INSERT INTO users (email, full_name, password_hash)
VALUES ($1, $2, $3)
RETURNING *;
```

`RETURNING *` gives you the created row (with its generated `id` and timestamps) to send back with `201`.

### PATCH /users/:id/profile — partial update

The client may send any subset of `bio`, `phone`, `avatar_url`. Your code builds the `SET` list only from the fields it received:

```sql
UPDATE user_profiles
SET bio = $1, phone = $2           -- avatar_url wasn't sent, so it's untouched
WHERE user_id = $3
RETURNING *;
```

### The `updated_at` problem

After the update, `updated_at` still shows the old time. Two fixes:

1. Set it manually in every update query (`updated_at = now()`).
2. Use a **database trigger** (next sections). Preferred.

---

## Indexes

[2:18:57](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=8337s)

**Analogy:** the index of a book lists chapter → page, so you don't flip through 50 pages to find chapter 4.

Without an index, the database does a **sequential scan**: it reads every row, in different places on disk, checking the condition. Fine for 6 rows, painful for millions or billions.

An **index** is (conceptually) a **lookup table** for a column: value → location of the row on disk. It's stored compactly in one place, so lookups are fast. (Under the hood this is typically a B-tree. The video doesn't go into it.) Indexes can be **ascending or descending**, so `ORDER BY created_at DESC` can be answered straight from a descending index with no re-sorting.

### When to create an index

A column is a candidate if it appears in:

1. a **`WHERE`** clause,
2. a **`JOIN`** condition, or
3. an **`ORDER BY`**.

...**and** that query runs frequently.

- **Primary keys are indexed automatically.**
- **Foreign keys are not.** Index them yourself if you join/filter on them.

### The cost

Every `INSERT`/`UPDATE` must also update the index, so there's write **overhead**. Don't index everything. Start with the three rules, monitor, and **drop indexes** that turn out unused.

### The video's indexes

```sql
-- migrate:up
CREATE INDEX idx_users_email                 ON users (email);               -- look up user by email
CREATE INDEX idx_users_created_at_desc       ON users (created_at DESC);     -- default list order
CREATE INDEX idx_tasks_project_id            ON tasks (project_id);          -- tasks of a project (FK in join)
CREATE INDEX idx_tasks_assigned_to           ON tasks (assigned_to);         -- tasks of a user (FK in join)
CREATE INDEX idx_tasks_created_at_desc       ON tasks (created_at DESC);     -- default list order
CREATE INDEX idx_tasks_status                ON tasks (status);              -- filter by status
CREATE INDEX idx_project_members_project_id  ON project_members (project_id);
CREATE INDEX idx_project_members_user_id     ON project_members (user_id);
```

(`users.email` already has an index because of `UNIQUE`; the video creates one for the explanation. The principle is what matters: FK columns and filter/sort columns.)

---

## Triggers

[2:41:15](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&t=9675s)

A **trigger** runs a function automatically when something happens (e.g. a row is updated). Use one to maintain `updated_at` so application code never has to.

```sql
-- migrate:up
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS trigger AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
-- ...repeat for user_profiles, projects, tasks, project_members

-- migrate:down
DROP TRIGGER trg_users_updated_at ON users;
-- ...
DROP FUNCTION set_updated_at();
```

Name triggers clearly so they're easy to drop in the down migration. After applying (`dbmate up`), re-running the profile update shows `updated_at` correctly set to the current time.

---

## Summary

1. A **database** gives persistence plus efficient CRUD. Backend databases are **disk-based**; caches live in RAM.
2. A **DBMS** adds organisation, access, integrity and security, and solves text-file problems (parsing, structure, concurrency).
3. **Relational** = strict schema, strong integrity, SQL. **Non-relational** = flexible schema, integrity left to the app. Postgres + `jsonb` covers most "dynamic data" needs.
4. Use **Postgres**: open source, standard SQL, extensible, reliable, great JSON.
5. Types: `decimal` for money, `text` over `varchar(n)`, `timestamptz`, `uuid`, `jsonb`.
6. **Migrations**: ordered up/down SQL files in Git, applied by a tool that tracks the version.
7. Design: **enums** for fixed sets, **constraints** (`NOT NULL`, `UNIQUE`, `CHECK`, `PK`, `FK`), **`ON DELETE`** rules, 1:1 / 1:N / N:M patterns, snake_case plural tables.
8. **Always `ORDER BY`**, paginate lists, `LEFT JOIN` when the related row may be missing.
9. **Parameterize every query.** Never concatenate user input into SQL.
10. **Indexes** speed `WHERE`/`JOIN`/`ORDER BY` columns at the cost of write overhead. Index foreign keys yourself.
11. **Triggers** keep `updated_at` correct automatically.

## Interview questions

**Q: Why are databases disk-based and caches RAM-based?**
Disk is cheap and large but slower. RAM is fast but expensive and small. Databases need capacity and durability, caches need speed.

**Q: Why not store data in text files?**
Parsing is slow and error-prone, there's no enforced structure, and concurrent writes cause lost updates. A DBMS adds indexing, types, constraints, and concurrency control.

**Q: Relational vs non-relational, and when to pick each?**
Relational: fixed schema, relationships, strong integrity, SQL (CRM, finance). Non-relational: flexible documents (CMS, rapid prototypes). With Postgres `jsonb` you can often keep one database.

**Q: `decimal` vs `float`?**
`decimal`/`numeric` is exact (use for prices). Floats are approximate but faster (measurements).

**Q: `varchar(255)` vs `text` in Postgres?**
No performance difference. `text` is recommended. `255` is a MySQL habit with no meaning in Postgres, and changing a length later requires a migration.

**Q: What is a migration and why use them?**
A versioned SQL file applied by a tool. It gives an auditable history in Git, repeatable setup across environments, and rollback through down-migrations.

**Q: What does a primary key imply?**
`NOT NULL` + `UNIQUE`, an automatic index, and it serves as the row's identifier.

**Q: Explain `ON DELETE RESTRICT` vs `CASCADE` vs `SET NULL`.**
RESTRICT blocks deleting a parent that has children. CASCADE deletes the children too. SET NULL nulls the FK in the children (column must be nullable).

**Q: How do you model many-to-many?**
A linking table with two foreign keys and a composite primary key. Relationship-specific data (role) goes in that table.

**Q: Why `LEFT JOIN` for user + profile?**
A user may not have a profile row. `INNER JOIN` would drop those users, `LEFT JOIN` keeps them with `NULL` profile data.

**Q: What is SQL injection and how do parameterized queries stop it?**
Attackers inject SQL via user input that is concatenated into queries. Parameterized queries send SQL and values separately, so values are always treated as data.

**Q: What is an index, and what's the trade-off?**
A lookup structure that avoids scanning the whole table for `WHERE`/`JOIN`/`ORDER BY`. It costs extra storage and slows writes, so index the columns used frequently.

**Q: Are foreign keys indexed automatically in Postgres?**
No. Primary keys (and unique constraints) are. Index foreign keys you filter or join on.

**Q: Why use a trigger for `updated_at`?**
It guarantees the column is maintained on every update regardless of which code path or script made the change.

**Q: Why should list queries always have `ORDER BY`?**
Without it, row order isn't guaranteed, so results (and pagination) can change between calls.
