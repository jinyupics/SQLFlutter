# SQLFlutter
UI = sql.f(database)
Database = sql.f (Database, UserAction)

状态管理容易让人困惑，它不应该被特殊对待。我们把状态放进数据库，从此不再神秘。

## 路由是数据，不是状态

路由是非常重要的**数据**，而不是状态。当我们需要有效的数据时，路由永远在SQL里。

就像Google表格的公式：

```
| A (用户操作)    | B (当前路由)                      |
|----------------|-----------------------------------|
| "click_login"  | =IF(A1="click_login", "/login", B0) |
| "submit_form"  | =IF(A2="submit_form", "/dashboard", B1) |
```

用SQL来表达，我们只需要一个路由表：

```sql
CREATE TABLE routing (
  id INTEGER PRIMARY KEY,
  current_screen TEXT NOT NULL
);

-- 就这样。一行数据，一个真相。
SELECT current_screen FROM routing WHERE id = 1;
-- 结果: '/dashboard'
```

当用户操作发生时，我们更新这个表：

```sql
UPDATE routing SET current_screen = '/settings' WHERE id = 1;
```

UI监听这个表。你在哪个页面，只需要一个 `SELECT`。没有导航栈，没有路由配置，没有状态管理 - 只是表里的数据。

就像电子表格单元格在依赖项变化时自动更新一样，你的路由是从数据**计算**出来的值。不需要命令式的 `navigator.push()` - 路由就是SQL查询的结果。

## Widget内容 = SQL查询

一旦我们知道在哪里（从路由表获取当前路径），每个widget的内容就是一个SQL查询结果。

```sql
-- 标题显示什么？
SELECT title FROM screens WHERE path = '/dashboard';

-- 列表里有什么？
SELECT * FROM tasks WHERE user_id = 1 AND completed = 0;

-- 角标数字是多少？
SELECT COUNT(*) FROM notifications WHERE read = 0;

-- 头像里的用户名是什么？
SELECT name, avatar_url FROM users WHERE id = 1;
```

每个widget绑定一个查询。数据变了，widget就更新。不需要 `setState()`，不需要 `notifyListeners()`，不需要手动重建widget树。

```
┌─────────────────────────────────────┐
│  页面: /dashboard                   │  ← SELECT current_screen FROM routing
├─────────────────────────────────────┤
│  标题: "我的任务"                    │  ← SELECT title FROM screens
├─────────────────────────────────────┤
│  ┌─────────────────────────────┐    │
│  │ □ 买菜                       │    │  ← SELECT * FROM tasks
│  │ □ 给妈妈打电话                │    │
│  │ □ 完成报告                   │    │
│  └─────────────────────────────┘    │
├─────────────────────────────────────┤
│  [+ 添加任务]  通知: 3              │  ← SELECT COUNT(*) FROM notifications
└─────────────────────────────────────┘
```

整个UI是数据库的映射。SQL是唯一的真相来源。

## 用户操作 = Upsert，不是新状态

用户操作应该始终转换为**upsert**（插入或更新）现有或新的表 - 永远不要创建新的状态。

```sql
-- 用户点击"添加任务"按钮
INSERT INTO tasks (id, title, user_id, completed)
VALUES (uuid(), '新任务', 1, 0);

-- 用户切换任务完成状态
UPDATE tasks SET completed = 1 WHERE id = 42;

-- 用户更新个人资料（upsert模式）
INSERT INTO users (id, name, email) VALUES (1, 'John', 'john@email.com')
ON CONFLICT(id) DO UPDATE SET name = excluded.name, email = excluded.email;

-- 用户导航到设置页
UPDATE routing SET current_screen = '/settings' WHERE id = 1;
```

每个用户交互就是一个 `INSERT`、`UPDATE` 或 `UPSERT`。就这么简单。

## 解耦架构

**DB → UI** 和 **Action → DB** 完全解耦：

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│   操作      │ ───►  │   数据库    │ ───►  │     UI      │
│  (Upsert)   │       │   (表)      │       │   (查询)    │
└─────────────┘       └─────────────┘       └─────────────┘
                            │
                            ▼
                       唯一真相来源
```

- **操作 → DB**：用户操作写入数据库（INSERT/UPDATE）
- **DB → UI**：UI从数据库读取（SELECT）

它们互不知道对方的存在。数据库是它们之间唯一的连接。

这意味着：
- 操作不关心UI如何渲染
- UI不关心是什么触发了数据变化
- 测试变得简单：插入数据，断言UI；触发操作，断言数据库
- 时间旅行/撤销：只需恢复之前的行状态

## 没有Service层。没有Model层。

传统架构：
```
操作 → Controller → Service → Repository → Model → 数据库
                                                       ↓
UI ← ViewModel ← UseCase ← Service ← Repository ← Model ←
```

SQLFlutter：
```
操作 → 数据库 → UI
```

就这样。没有service。没有model。没有repository。没有DTO。没有mapper。

- **表就是你的model** - schema定义了数据的形状
- **SQL就是你的service层** - 查询包含了业务逻辑
- **数据库就是你的repository** - 它本来就知道如何持久化和检索

为什么要添加那些只是传递数据的层？数据库已经把活干完了。

---

## 愿景：像写Excel一样写Flutter

### 当前的问题

传统Drift需要太多步骤：

| 步骤 | 传统 Drift |
|------|------------|
| 1 | 定义Table类 |
| 2 | 运行build_runner生成代码 |
| 3 | 写DAO方法 |
| 4 | 用StreamBuilder包装 |
| 5 | 处理async/await |

**显示一个数据需要5步。太重了。**

### 我们想要的

```dart
Text("共 ${sql.watch('SELECT COUNT(*) FROM tasks')} 个任务")
```

**1步。搞定。**

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

**全部自动刷新。零 setState。零 StreamBuilder。**

### 核心理念

> SQL查询像读变量一样简单。
>
> 写应用像写Excel一样自然。

这就是SQLFlutter想要实现的未来。
