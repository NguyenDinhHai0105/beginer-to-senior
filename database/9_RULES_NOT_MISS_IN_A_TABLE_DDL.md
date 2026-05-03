# Mandatory rules when writing DDL for a table

This file summarizes rules not to be missed when designing database tables. Each item includes a short explanation and SQL examples.

---

## 1. Standard audit columns
Every table should have at least 5–7 audit columns to trace history and support optimistic locking:

- `created_at` (TIMESTAMP WITH TIME ZONE): record creation time; typically not modified after creation.
- `updated_at` (TIMESTAMP WITH TIME ZONE): last update time; updated automatically on changes.
- `created_by` (BIGINT): id of the creator (if applicable).
- `updated_by` (BIGINT): id of the last updater (if applicable).
- `version` (INTEGER/BIGINT): version for optimistic locking.
- `is_deleted` (BOOLEAN) and `deleted_at` (TIMESTAMP) — if using soft deletes.

Example (Postgres):
```sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  usr_username VARCHAR(50) NOT NULL,
  usr_email VARCHAR(255) NOT NULL,
  usr_full_name VARCHAR(150),

  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_by BIGINT NULL,
  updated_by BIGINT NULL,
  version INTEGER NOT NULL DEFAULT 1,

  is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
  deleted_at TIMESTAMPTZ NULL
);
```
Note: `updated_at` is usually maintained by a trigger or ORM.

---

## 2. Column comments
Add comments for important columns to help readers understand meaning, e.g.:
```sql
COMMENT ON COLUMN users.usr_username IS 'Login name (unique identifier for the user)';
```
Comments help onboard new team members and avoid misunderstandings.

---

## 3. Soft delete
Avoid hard-deleting records unless necessary. Add `is_deleted` and `deleted_at` to mark deletions:

```sql
ALTER TABLE users
  ADD COLUMN is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
  ADD COLUMN deleted_at TIMESTAMPTZ NULL;
```
Default queries should filter out `is_deleted = true` (in the application or via views). Create `active_users` view if needed.

---

## 4. Column name prefixes
Use a short table prefix for column names (e.g., `usr_` for `users`) to avoid name collisions when joining tables:

- users: `usr_id`, `usr_email`
- feed: `fd_id`, `fd_title`

Example:
```sql
SELECT u.usr_email, p.prd_name
FROM users u
JOIN purchases p ON u.id = p.user_id;
```

---

## 5. Vertical partitioning when there are many columns
If a table has many columns, separate rarely-used or large columns into a child table to reduce I/O and improve cache efficiency. Example splitting `feed` and `feed_content`:

```sql
CREATE TABLE feed (
  fd_id BIGSERIAL PRIMARY KEY,
  fd_title VARCHAR(200) NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE feed_content (
  fc_id BIGSERIAL PRIMARY KEY,
  fc_feed_id BIGINT NOT NULL REFERENCES feed(fd_id),
  fc_body TEXT NOT NULL,
  fc_media JSONB NULL
);
```
Listing feeds without bodies will only read the `feed` table — less I/O.

---

## 6. Choose appropriate data types and lengths
Pick the smallest type that satisfies requirements:
- Use INTEGER or BIGINT appropriately (use 32-bit if the ID space is small). Use 64-bit only when needed.
- Use VARCHAR(n) with limits when possible (e.g., username 50). Use TEXT for unbounded content.
- Use JSONB (Postgres) for structured JSON data for efficient querying.

Example: email `VARCHAR(255)`, username `VARCHAR(50)`.

---

## 7. Add NOT NULL where possible
Set `NOT NULL` for required columns to simplify application checks and help the optimizer.

```sql
usr_email VARCHAR(255) NOT NULL
```
Combine `NOT NULL` with sensible defaults where appropriate.

---

## 8. Index sensibly
Create indexes for columns used in WHERE / JOIN / ORDER BY. Avoid redundant indexes that slow writes.

Example:
```sql
CREATE UNIQUE INDEX ux_users_email ON users (usr_email);
CREATE INDEX idx_users_created_at ON users (created_at DESC);
```
Use composite or partial indexes when suitable (e.g., index only non-deleted records):
```sql
CREATE INDEX idx_users_email_active ON users (usr_email) WHERE is_deleted = false;
```

---

## 9. Apply normalization (3NF)
Design according to 3rd Normal Form to avoid redundancy and maintain consistency. If many columns repeat grouped information (e.g., address), split into separate tables.

Example: move addresses to `user_address`:
```sql
CREATE TABLE user_address (
  ua_id BIGSERIAL PRIMARY KEY,
  ua_user_id BIGINT NOT NULL REFERENCES users(id),
  ua_address_line VARCHAR(255),
  ua_city VARCHAR(100),
  ua_country VARCHAR(100)
);
```

---

## Combined example — a `users` table following the rules

```sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,

  usr_username VARCHAR(50) NOT NULL,
  usr_email VARCHAR(255) NOT NULL,
  usr_full_name VARCHAR(150),

  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_by BIGINT NULL,
  updated_by BIGINT NULL,
  version INTEGER NOT NULL DEFAULT 1,

  is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
  deleted_at TIMESTAMPTZ NULL
);

COMMENT ON COLUMN users.usr_username IS 'Login name (unique)';
COMMENT ON COLUMN users.usr_email IS 'User email';

CREATE UNIQUE INDEX ux_users_email ON users (usr_email) WHERE is_deleted = false;
CREATE INDEX idx_users_updated_at ON users (updated_at DESC);
```

Final notes:
- Syntax varies across databases (MySQL, Postgres, MSSQL): `TIMESTAMPTZ` is Postgres; MySQL uses `DATETIME(6)`.
- Automating `updated_at` updates may require triggers or ORM support.

If desired, a sample trigger to auto-update `updated_at` and an example of optimistic locking (using `version`) can be added.
