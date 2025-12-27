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

## No Service Layer. No Model Layer.

Traditional architecture:
```
Action → Controller → Service → Repository → Model → Database
                                                         ↓
UI ← ViewModel ← UseCase ← Service ← Repository ← Model ←
```

SQLFlutter:
```
Action → Database → UI
```

That's it. No services. No models. No repositories. No DTOs. No mappers.

- **Tables are your models** - the schema defines the shape of your data
- **SQL is your service layer** - queries contain the business logic
- **The database is your repository** - it already knows how to persist and retrieve

Why add layers that just pass data through? The database already does the job.

---

## 愿景：像写Excel一样写Flutter

### 当前的问题

传统Drift写法需要太多步骤：

| 步骤 | 传统 Drift |
|------|-----------|
| 1 | 定义Table类 |
| 2 | 运行build_runner生成代码 |
| 3 | 写DAO方法 |
| 4 | 用StreamBuilder包装 |
| 5 | 处理async/await |

**5步才能显示一个数据。太重了。**

### 我们想要的

```dart
Text("共 ${sql.watch('SELECT COUNT(*) FROM tasks')} 个任务")
```

**1步。完事。**

就像Excel公式一样 —— 写什么得什么，数据变了自动刷新。

### 理想的API

| 方法 | 用途 | 响应式 |
|------|------|--------|
| `sql.watch()` | 查询数据 | ✓ 自动刷新 |
| `sql.run()` | 增删改 | ✗ 执行一次 |

### 完整示例

```dart
class TaskListPage extends StatelessWidget {
  final int userId;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // 标题
        Text("${sql.watch('SELECT name FROM users WHERE id = $userId')} 的任务"),
        Text("共 ${sql.watch('SELECT COUNT(*) FROM tasks WHERE user_id = $userId')} 项"),

        // 列表
        Expanded(
          child: ListView(
            children: [
              for (var task in sql.watch('SELECT * FROM tasks WHERE user_id = $userId ORDER BY due_date'))
                ListTile(
                  leading: Checkbox(
                    value: task.done == 1,
                    onChanged: (_) => sql.run('UPDATE tasks SET done = 1 - done WHERE id = ${task.id}'),
                  ),
                  title: Text(task.title),
                  subtitle: Text("${task.dueDate}"),
                  onTap: () => sql.run('UPDATE routing SET current_screen = "/task/${task.id}"'),
                ),
            ],
          ),
        ),

        // 添加按钮
        ElevatedButton(
          onPressed: () => sql.run('INSERT INTO tasks (user_id, title) VALUES ($userId, "新任务")'),
          child: Text("添加"),
        ),
      ],
    );
  }
}
```

### 效果

```
┌─────────────────────────────────────────────┐
│  张三 的任务                                 │
│  共 5 项                                     │
├─────────────────────────────────────────────┤
│                                             │
│  ☐ 买牛奶                                   │
│    2024-01-15                               │
│                                             │
│  ☑ 给妈妈打电话                              │
│    2024-01-14                               │
│                                             │
│  ☐ 完成报告                                  │
│    2024-01-16                               │
│                                             │
│  ☐ 预约牙医                                  │
│    2024-01-20                               │
│                                             │
│  ☑ 交水电费                                  │
│    2024-01-13                               │
│                                             │
├─────────────────────────────────────────────┤
│              [ ＋ 添加 ]                     │
└─────────────────────────────────────────────┘
```

**交互：**
- 点击 ☐ → `sql.run(UPDATE...)` → 自动变成 ☑
- 点击 [添加] → `sql.run(INSERT...)` → 列表自动多一行，"共 6 项"
- 点击某行 → 路由表更新 → 自动跳转详情页

**全部自动刷新，零 setState，零 StreamBuilder。**

### 核心理念

> SQL查询像读变量一样简单。
>
> 写应用像写Excel一样自然。

这就是SQLFlutter想要实现的未来。
