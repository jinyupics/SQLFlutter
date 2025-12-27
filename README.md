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

In SQL terms:

```sql
-- The current route is derived from user actions and conditions
SELECT
  CASE
    WHEN last_action = 'click_login' THEN '/login'
    WHEN last_action = 'submit_form' AND is_authenticated = 1 THEN '/dashboard'
    WHEN last_action = 'logout' THEN '/'
    ELSE current_route
  END AS route
FROM app_state
```

Just like a spreadsheet cell updates automatically when its dependencies change, your route is a **computed value** from your data. No imperative `navigator.push()` - the route is simply what the SQL says it should be.
