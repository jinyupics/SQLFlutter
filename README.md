# SQLFlutter
UI = sql.f(database)
Database = sql.f (Database, UserAction)

State could be misleading, and it should not be too special to manage. We put them in the database and it is de-demonized thereafter.

## Routing is Data, Not State

Routing is super important **data** rather than state. When we want effective data, routing is always in the SQL.

Think of it like Google Sheets formulas:

```
| A (UserAction) | B (CurrentRoute)          |
|----------------|---------------------------|
| "click_login"  | =IF(A1="click_login", "/login", B0) |
| "submit_form"  | =IF(A2="submit_form", "/dashboard", B1) |
```

In SQL terms, we simply have a routing table:

```sql
CREATE TABLE routing (
  id INTEGER PRIMARY KEY,
  current_screen TEXT NOT NULL
);

-- That's it. One row, one truth.
SELECT current_screen FROM routing WHERE id = 1;
-- Result: '/dashboard'
```

When a user action happens, we update the table:

```sql
UPDATE routing SET current_screen = '/settings' WHERE id = 1;
```

The UI listens to this table. The screen you're on is just a `SELECT` away. No navigation stack, no router configuration, no state management - just data in a table.

Just like a spreadsheet cell updates automatically when its dependencies change, your route is a **computed value** from your data. No imperative `navigator.push()` - the route is simply what the SQL says it should be.

## Widget Content = SQL Query

Once we know where we are (the active path from the routing table), every widget's content is simply a SQL query result.

```sql
-- What should the header show?
SELECT title FROM screens WHERE path = '/dashboard';

-- What items are in the list?
SELECT * FROM tasks WHERE user_id = 1 AND completed = 0;

-- What's the badge count?
SELECT COUNT(*) FROM notifications WHERE read = 0;

-- What's the user's name in the avatar?
SELECT name, avatar_url FROM users WHERE id = 1;
```

Each widget binds to a query. When data changes, the widget updates. No `setState()`, no `notifyListeners()`, no rebuilding widget trees manually.

```
┌─────────────────────────────────────┐
│  Screen: /dashboard                 │  ← SELECT current_screen FROM routing
├─────────────────────────────────────┤
│  Header: "My Tasks"                 │  ← SELECT title FROM screens
├─────────────────────────────────────┤
│  ┌─────────────────────────────┐    │
│  │ □ Buy groceries             │    │  ← SELECT * FROM tasks
│  │ □ Call mom                  │    │
│  │ □ Finish report             │    │
│  └─────────────────────────────┘    │
├─────────────────────────────────────┤
│  [+ Add Task]  Notifications: 3     │  ← SELECT COUNT(*) FROM notifications
└─────────────────────────────────────┘
```

The entire UI is a reflection of the database. SQL is the single source of truth.

## User Actions = Upserts, Not New States

User actions should always be converted to **upserting** existing or new tables - never to creating new states.

```sql
-- User clicks "Add Task" button
INSERT INTO tasks (id, title, user_id, completed)
VALUES (uuid(), 'New task', 1, 0);

-- User toggles a task complete
UPDATE tasks SET completed = 1 WHERE id = 42;

-- User updates their profile (upsert pattern)
INSERT INTO users (id, name, email) VALUES (1, 'John', 'john@email.com')
ON CONFLICT(id) DO UPDATE SET name = excluded.name, email = excluded.email;

-- User navigates to settings
UPDATE routing SET current_screen = '/settings' WHERE id = 1;
```

Every user interaction is an `INSERT`, `UPDATE`, or `UPSERT`. That's it.

## Decoupled Architecture

**DB → UI** and **Action → DB** are completely decoupled:

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│   Action    │ ───►  │  Database   │ ───►  │     UI      │
│  (Upsert)   │       │  (Tables)   │       │  (Queries)  │
└─────────────┘       └─────────────┘       └─────────────┘
                            │
                            ▼
                      Single Source
                       of Truth
```

- **Action → DB**: User actions write to the database (INSERT/UPDATE)
- **DB → UI**: UI reads from the database (SELECT)

They don't know about each other. The database is the only connection between them.

This means:
- Actions don't care how the UI will render
- UI doesn't care what triggered the data change
- Testing is trivial: insert data, assert UI; trigger action, assert database
- Time travel / undo: just restore previous row states
