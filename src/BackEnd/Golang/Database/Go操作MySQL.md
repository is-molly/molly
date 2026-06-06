---
order: 1
date: 2026-06-03
category:
  - 后端
  - Golang
---

# Go操作MySQL

<!-- more -->

## 一、MySQL 基础回顾

### 1.1 数据库基本概念

- **DB（数据库）**：存储数据的集合，以数据表等形式组成
- **DBMS（数据库管理系统）**：操作和管理数据库的系统，如 MySQL、Oracle 等
- **DBS（数据库系统）**：由数据库和数据库管理软件组成的整体

### 1.2 关系型数据库核心概念

| 概念 | 说明 |
|------|------|
| 表（Table） | 数据库中存储数据的基本单位，以二维表形式组织 |
| 列（Column） | 表中的一个字段，有对应的数据类型 |
| 行（Row） | 表中的一条记录 |
| 主键（Primary Key） | 唯一标识表中每一行的列，不允许为 NULL |

### 1.3 常用数据类型

| 类型 | 说明 |
|------|------|
| `INT` | 整数类型 |
| `DECIMAL(M,D)` | 精确浮点数，适合金额存储 |
| `VARCHAR(N)` | 可变长度字符串，最大 65535 |
| `TEXT` | 长文本类型 |
| `DATETIME` | 日期时间类型 `YYYY-MM-DD HH:MM:SS` |
| `DATE` | 日期类型 `YYYY-MM-DD` |

### 1.4 SQL 语言分类

- **DDL（数据定义语言）**：`CREATE`、`ALTER`、`DROP`、`TRUNCATE`
- **DML（数据操作语言）**：`INSERT`、`UPDATE`、`DELETE`
- **DQL（数据查询语言）**：`SELECT`
- **DCL（数据控制语言）**：`GRANT`、`REVOKE`
- **TCL（事务控制语言）**：`COMMIT`、`ROLLBACK`

### 1.5 基本 SQL 语句

```sql
-- 创建数据库
CREATE DATABASE mydb CHARACTER SET utf8mb4;

-- 创建表
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL,
    age INT DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 插入数据
INSERT INTO users (username, age) VALUES ('张三', 25);

-- 查询数据
SELECT id, username, age FROM users WHERE age > 18 ORDER BY id DESC LIMIT 10;

-- 更新数据
UPDATE users SET age = 26 WHERE id = 1;

-- 删除数据
DELETE FROM users WHERE id = 1;
```

### 1.6 约束

- **NOT NULL**：字段值不能为 NULL
- **UNIQUE**：字段值必须唯一，允许 NULL
- **PRIMARY KEY**：NOT NULL + UNIQUE，唯一标识每行
- **FOREIGN KEY**：外键约束，保证引用完整性
- **DEFAULT**：设置字段默认值

## 二、database/sql 包概述

Go 语言通过 `database/sql` 标准库提供了统一的数据库操作接口。该包定义了一套通用的 API，具体的数据库连接由第三方驱动实现。

```go
import (
    "database/sql"
    _ "github.com/go-sql-driver/mysql" // 导入 MySQL 驱动，init() 自动注册
)
```

> **注意**：驱动包使用匿名导入 `_`，因为代码中不直接调用驱动包，而是通过 `database/sql` 间接使用。

### 2.1 核心接口

`database/sql` 包定义了两个核心接口，由驱动实现：

- **`sql.DB`**：数据库连接句柄，代表一个连接池，并发安全
- **`sql.Tx`**：事务句柄，代表一个数据库事务
- **`sql.Rows`**：查询结果集迭代器
- **`sql.Row`**：单行查询结果
- **`sql.Stmt`**：预处理语句

### 2.2 驱动注册机制

```go
// go-sql-driver/mysql 驱动的 init() 函数内部实现：
func init() {
    sql.Register("mysql", &MySQLDriver{})
}
```

## 三、连接 MySQL 数据库

### 3.1 DSN（数据源名称）格式

```
[username[:password]@][protocol[(address)]]/dbname[?param1=value1&...]
```

常用参数：

| 参数 | 说明 | 示例 |
|------|------|------|
| `charset` | 字符集 | `charset=utf8mb4` |
| `parseTime` | 将 `DATETIME` 解析为 `time.Time` | `parseTime=true` |
| `loc` | 时区设置 | `loc=Local` |
| `timeout` | 连接超时 | `timeout=10s` |
| `readTimeout` | 读超时 | `readTimeout=30s` |
| `writeTimeout` | 写超时 | `writeTimeout=30s` |

### 3.2 建立连接

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    _ "github.com/go-sql-driver/mysql"
)

// 数据库配置结构体
type DBConfig struct {
    Host     string // 数据库主机
    Port     int    // 数据库端口
    User     string // 用户名
    Password string // 密码
    DBName   string // 数据库名
}

func main() {
    // DSN 格式：user:password@tcp(host:port)/dbname?charset=utf8mb4&parseTime=True&loc=Local
    dsn := "root:123456@tcp(127.0.0.1:3306)/mydb?charset=utf8mb4&parseTime=True&loc=Local"

    // 打开数据库连接（并不会立即建立连接，只是验证 DSN 格式）
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        log.Fatal("打开数据库失败:", err)
    }
    // 确保程序退出时关闭数据库连接
    defer db.Close()

    // Ping 验证数据库连接是否可用
    if err = db.Ping(); err != nil {
        log.Fatal("连接数据库失败:", err)
    }

    fmt.Println("数据库连接成功")
}
```

### 3.3 连接池配置

```go
// 设置连接池参数
db.SetMaxOpenConns(25)                 // 最大打开连接数，0 表示不限制
db.SetMaxIdleConns(10)                 // 最大空闲连接数，默认 2
db.SetConnMaxLifetime(5 * time.Minute)  // 连接最大存活时间，0 表示永不过期
db.SetConnMaxIdleTime(2 * time.Minute)  // 连接最大空闲时间（Go 1.15+）
```

> **最佳实践**：`sql.DB` 是并发安全的，应该作为全局变量使用，不要频繁 `Open` 和 `Close`。

## 四、CRUD 操作

### 4.1 查询单行数据 - QueryRow

```go
// User 用户结构体
type User struct {
    ID        int       // 用户ID
    Username  string    // 用户名
    Age       int       // 年龄
    CreatedAt time.Time // 创建时间
}

// QueryUserByID 根据ID查询单个用户
func QueryUserByID(db *sql.DB, id int) (*User, error) {
    var user User
    sqlStr := "SELECT id, username, age, created_at FROM users WHERE id = ?"
    // QueryRow 最多返回一行，使用 Scan 将列值映射到变量
    err := db.QueryRow(sqlStr, id).Scan(&user.ID, &user.Username, &user.Age, &user.CreatedAt)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, nil // 没有找到记录，返回 nil
        }
        return nil, fmt.Errorf("查询用户失败: %w", err)
    }
    return &user, nil
}
```

### 4.2 查询多行数据 - Query

```go
// QueryUsersByAge 根据年龄查询用户列表
func QueryUsersByAge(db *sql.DB, minAge int) ([]User, error) {
    sqlStr := "SELECT id, username, age, created_at FROM users WHERE age >= ?"
    rows, err := db.Query(sqlStr, minAge)
    if err != nil {
        return nil, fmt.Errorf("查询用户列表失败: %w", err)
    }
    // 必须关闭 Rows，释放连接
    defer rows.Close()

    var users []User
    // Next 迭代结果集
    for rows.Next() {
        var user User
        err := rows.Scan(&user.ID, &user.Username, &user.Age, &user.CreatedAt)
        if err != nil {
            return nil, fmt.Errorf("扫描行数据失败: %w", err)
        }
        users = append(users, user)
    }
    // 检查迭代过程中是否有错误
    if err = rows.Err(); err != nil {
        return nil, fmt.Errorf("迭代结果集出错: %w", err)
    }
    return users, nil
}
```

### 4.3 插入数据 - Exec

```go
// InsertUser 插入新用户，返回自增ID
func InsertUser(db *sql.DB, username string, age int) (int64, error) {
    sqlStr := "INSERT INTO users (username, age) VALUES (?, ?)"
    // Exec 用于执行不返回行的 SQL（INSERT、UPDATE、DELETE）
    result, err := db.Exec(sqlStr, username, age)
    if err != nil {
        return 0, fmt.Errorf("插入用户失败: %w", err)
    }
    // LastInsertId 获取自增主键ID
    id, err := result.LastInsertId()
    if err != nil {
        return 0, fmt.Errorf("获取自增ID失败: %w", err)
    }
    return id, nil
}
```

### 4.4 更新数据 - Exec

```go
// UpdateUserAge 更新用户年龄
func UpdateUserAge(db *sql.DB, id int, newAge int) (int64, error) {
    sqlStr := "UPDATE users SET age = ? WHERE id = ?"
    result, err := db.Exec(sqlStr, newAge, id)
    if err != nil {
        return 0, fmt.Errorf("更新用户失败: %w", err)
    }
    // RowsAffected 获取影响的行数
    affected, err := result.RowsAffected()
    if err != nil {
        return 0, fmt.Errorf("获取影响行数失败: %w", err)
    }
    return affected, nil
}
```

### 4.5 删除数据 - Exec

```go
// DeleteUserByID 根据ID删除用户
func DeleteUserByID(db *sql.DB, id int) (int64, error) {
    sqlStr := "DELETE FROM users WHERE id = ?"
    result, err := db.Exec(sqlStr, id)
    if err != nil {
        return 0, fmt.Errorf("删除用户失败: %w", err)
    }
    return result.RowsAffected()
}
```

## 五、预处理语句

预处理语句（Prepared Statement）将 SQL 模板与参数值分离，有两大优势：

1. **防止 SQL 注入**：参数值不会被当作 SQL 代码执行
2. **提升性能**：SQL 只需编译一次，可多次执行

```go
// BatchInsertUsers 批量插入用户（使用预处理语句）
func BatchInsertUsers(db *sql.DB, users []User) error {
    // Prepare 预编译 SQL 语句
    stmt, err := db.Prepare("INSERT INTO users (username, age) VALUES (?, ?)")
    if err != nil {
        return fmt.Errorf("预编译SQL失败: %w", err)
    }
    defer stmt.Close()

    for _, user := range users {
        _, err := stmt.Exec(user.Username, user.Age)
        if err != nil {
            return fmt.Errorf("批量插入失败: %w", err)
        }
    }
    return nil
}
```

### 5.1 占位符说明

MySQL 使用 `?` 作为占位符。不同数据库的占位符不同：

| 数据库 | 占位符 |
|--------|--------|
| MySQL | `?` |
| PostgreSQL | `$1, $2, ...` |
| SQLite | `?` |
| Oracle | `:name` |

```go
// 安全写法：使用占位符
db.Query("SELECT * FROM users WHERE username = ?", userInput)

// 危险写法：拼接字符串（有 SQL 注入风险）
db.Query("SELECT * FROM users WHERE username = '" + userInput + "'")
```

## 六、事务操作

事务是一组不可分割的操作，要么全部成功，要么全部失败回滚。

```go
// TransferMoney 转账示例：演示事务操作
func TransferMoney(db *sql.DB, fromID, toID int, amount float64) error {
    // Begin 开启事务
    tx, err := db.Begin()
    if err != nil {
        return fmt.Errorf("开启事务失败: %w", err)
    }
    // 使用 defer 确保异常时回滚
    defer func() {
        if err != nil {
            tx.Rollback() // 回滚事务
        }
    }()

    // 扣款操作
    sql1 := "UPDATE accounts SET balance = balance - ? WHERE id = ? AND balance >= ?"
    result, err := tx.Exec(sql1, amount, fromID, amount)
    if err != nil {
        return fmt.Errorf("扣款失败: %w", err)
    }
    affected, _ := result.RowsAffected()
    if affected == 0 {
        return fmt.Errorf("余额不足")
    }

    // 到账操作
    sql2 := "UPDATE accounts SET balance = balance + ? WHERE id = ?"
    _, err = tx.Exec(sql2, amount, toID)
    if err != nil {
        return fmt.Errorf("到账失败: %w", err)
    }

    // Commit 提交事务
    err = tx.Commit()
    if err != nil {
        return fmt.Errorf("提交事务失败: %w", err)
    }
    return nil
}
```

> **注意**：`sql.Tx` 上的所有操作必须使用 `tx.Exec`、`tx.Query` 等方法，不能使用 `db.Exec`，否则不在事务范围内。

## 七、NULL 值处理

数据库中字段可能为 `NULL`，Go 中基本类型无法表示 `NULL`，需要使用 `sql` 包的特殊类型：

| Go 类型 | 说明 |
|---------|------|
| `sql.NullString` | 可空的字符串 |
| `sql.NullInt64` | 可空的 int64 |
| `sql.NullFloat64` | 可空的 float64 |
| `sql.NullBool` | 可空的 bool |
| `sql.NullTime` | 可空的 time.Time |

```go
// UserWithNull 包含可空字段的用户结构体
type UserWithNull struct {
    ID       int
    Username string
    Email    sql.NullString // email 字段可能为 NULL
    Age      sql.NullInt64  // age 字段可能为 NULL
}

// QueryUserWithNull 查询包含NULL字段的用户
func QueryUserWithNull(db *sql.DB, id int) (*UserWithNull, error) {
    var user UserWithNull
    sqlStr := "SELECT id, username, email, age FROM users WHERE id = ?"
    err := db.QueryRow(sqlStr, id).Scan(&user.ID, &user.Username, &user.Email, &user.Age)
    if err != nil {
        return nil, err
    }
    // 判断 NULL 值
    if user.Email.Valid {
        fmt.Println("Email:", user.Email.String)
    } else {
        fmt.Println("Email 为 NULL")
    }
    return &user, nil
}
```

还可以使用指针类型处理 NULL：

```go
type UserPointer struct {
    ID    int
    Email *string // *string 可直接接收 NULL，为 nil 表示 NULL
}
```

## 八、完整示例：用户管理模块

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "time"

    _ "github.com/go-sql-driver/mysql"
)

// User 用户模型
type User struct {
    ID        int       `json:"id"`         // 用户ID
    Username  string    `json:"username"`   // 用户名
    Age       int       `json:"age"`        // 年龄
    CreatedAt time.Time `json:"created_at"` // 创建时间
}

// UserDAO 用户数据访问对象
type UserDAO struct {
    db *sql.DB // 数据库连接
}

// NewUserDAO 创建UserDAO实例
func NewUserDAO(dsn string) (*UserDAO, error) {
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        return nil, err
    }
    // 配置连接池
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(10)
    db.SetConnMaxLifetime(5 * time.Minute)

    if err = db.Ping(); err != nil {
        return nil, err
    }
    return &UserDAO{db: db}, nil
}

// Close 关闭数据库连接
func (dao *UserDAO) Close() error {
    return dao.db.Close()
}

// Create 创建用户
func (dao *UserDAO) Create(username string, age int) (int64, error) {
    result, err := dao.db.Exec("INSERT INTO users (username, age) VALUES (?, ?)", username, age)
    if err != nil {
        return 0, err
    }
    return result.LastInsertId()
}

// GetByID 根据ID查询用户
func (dao *UserDAO) GetByID(id int) (*User, error) {
    var user User
    err := dao.db.QueryRow("SELECT id, username, age, created_at FROM users WHERE id = ?", id).
        Scan(&user.ID, &user.Username, &user.Age, &user.CreatedAt)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, nil
        }
        return nil, err
    }
    return &user, nil
}

// List 查询用户列表
func (dao *UserDAO) List(page, pageSize int) ([]User, error) {
    offset := (page - 1) * pageSize
    rows, err := dao.db.Query(
        "SELECT id, username, age, created_at FROM users ORDER BY id DESC LIMIT ? OFFSET ?",
        pageSize, offset,
    )
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var users []User
    for rows.Next() {
        var user User
        if err := rows.Scan(&user.ID, &user.Username, &user.Age, &user.CreatedAt); err != nil {
            return nil, err
        }
        users = append(users, user)
    }
    return users, rows.Err()
}

// Update 更新用户
func (dao *UserDAO) Update(user *User) error {
    _, err := dao.db.Exec("UPDATE users SET username = ?, age = ? WHERE id = ?",
        user.Username, user.Age, user.ID)
    return err
}

// Delete 删除用户
func (dao *UserDAO) Delete(id int) error {
    _, err := dao.db.Exec("DELETE FROM users WHERE id = ?", id)
    return err
}

func main() {
    dsn := "root:123456@tcp(127.0.0.1:3306)/mydb?charset=utf8mb4&parseTime=True&loc=Local"
    dao, err := NewUserDAO(dsn)
    if err != nil {
        log.Fatal("初始化数据库失败:", err)
    }
    defer dao.Close()

    // 创建用户
    id, err := dao.Create("张三", 25)
    if err != nil {
        log.Fatal("创建用户失败:", err)
    }
    fmt.Printf("创建用户成功，ID: %d\n", id)

    // 查询用户
    user, err := dao.GetByID(int(id))
    if err != nil {
        log.Fatal("查询用户失败:", err)
    }
    fmt.Printf("用户信息: %+v\n", user)
}
```

## 九、最佳实践

### 9.1 全局 DB 实例

```go
var db *sql.DB

func init() {
    var err error
    db, err = sql.Open("mysql", "root:123456@tcp(127.0.0.1:3306)/mydb?charset=utf8mb4&parseTime=True")
    if err != nil {
        log.Fatal(err)
    }
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(10)
}
```

### 9.2 参数始终使用占位符

```go
// ✅ 正确：使用占位符
db.Query("SELECT * FROM users WHERE id = ?", id)

// ❌ 错误：直接拼接字符串
db.Query(fmt.Sprintf("SELECT * FROM users WHERE id = %d", id))
```

### 9.3 及时关闭 Rows

```go
rows, err := db.Query("SELECT ...")
if err != nil {
    return err
}
defer rows.Close() // 确保关闭，归还连接
```

### 9.4 检查 Rows.Err

```go
for rows.Next() {
    // 扫描数据...
}
// 迭代结束后检查错误
if err := rows.Err(); err != nil {
    return err
}
```

### 9.5 使用 sqlmock 进行单元测试

```go
import "github.com/DATA-DOG/go-sqlmock"

func TestGetUserByID(t *testing.T) {
    db, mock, _ := sqlmock.New()
    defer db.Close()

    rows := sqlmock.NewRows([]string{"id", "username", "age"}).
        AddRow(1, "张三", 25)

    mock.ExpectQuery("SELECT (.+) FROM users WHERE id = ?").
        WithArgs(1).
        WillReturnRows(rows)

    // 使用 mock db 进行测试...
}
```

### 9.6 常用第三方库

| 库 | 说明 |
|----|------|
| [go-sql-driver/mysql](https://github.com/go-sql-driver/mysql) | MySQL 官方驱动 |
| [jmoiron/sqlx](https://github.com/jmoiron/sqlx) | 对 database/sql 的扩展，支持结构体映射 |
| [gorm.io/gorm](https://gorm.io) | 全功能 ORM 框架 |
| [DATA-DOG/go-sqlmock](https://github.com/DATA-DOG/go-sqlmock) | SQL Mock 测试库 |

## 十、参考链接

- [Go database/sql 官方文档](https://pkg.go.dev/database/sql)
- [go-sql-driver/mysql GitHub](https://github.com/go-sql-driver/mysql)
- [Go database/sql tutorial](http://go-database-sql.org/)
