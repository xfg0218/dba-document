---
title: "编程语言连接数据库"
weight: 3
bookToc: true
bookCollapseSection: false
---

## 编程语言连接数据库

目前数据库一般支持HA的连接，即一个`Coordinator`内的一个节点异常后会链接到另外的一个节点，不会影响业务的正常运行。在`JDBC`配置时需要采用 [高可用链接字符串(Connection URL/DSN)](https://www.postgresql.org/docs/16/libpq-connect.html#LIBPQ-CONNSTRING) 的方式连接。适用于不同的编程语言中使用，`URL`例如:

```SQL
postgres://username1:password1@CoordinatorIP:CoordinatorPort,CoordinatorIP:CoordinatorPort/database?sslmode=disable
```

`username1`: Coordinator1数据库用户名

`password1`: Coordinator1数据库密码 

`CoordinatorIP`: Coordinator1数据库IP

`CoordinatorPort`: Coordinator1数据库端口

`database`: 数据库名

`sslmode`: 表示是否启用 SSL 连接。

## URL 高级连接参数

### 插入数据时的参数

如果在批量加载数据时需要添加以下的参数，为了在加载数据时按照`batchSize`进行批量写入，可以添加以下参数：

```sql
preferQueryMode=simple&reWriteBatchedInserts=true

```

**`preferQueryMode`**: 指定使用哪种模式来执行对数据库的查询：simple 表示（`Q`执行，不解析，不绑定，仅文本模式），`extended` 表示始终使用绑定 / 执行消息，`extendedForPrepared` 表示仅对预编译语句使用 `extended` 模式，`endedCacheEverything` 表示使用扩展协议并尝试在查询缓存中缓存每一条语句（包括 `Statement.execute (String sql)）`。取值为 `extended | extendedForPrepared | extendedCacheEverything | simple` 。

**`reWriteBatchedInserts`**: 这会将批量插入语句从 `insert into foo (col1, col2, col3) values (1, 2, 3)` 改为 `insert into foo (col1, col2, col3) values (1, 2, 3), (4, 5, 6)`，这样能提升 2 - 3 倍的性能。


### 在高并发读取时的参数
```sql
preparedStatementCacheQueries=512&defaultRowFetchSize=500&prepareThreshold=10
```

**`preparedStatementCacheQueries`**：一个会话中最大能缓存的PreparedStatement数量，默认值256，在高并发读取时可以适当调整为512.

**`defaultRowFetchSize`**：每次以游标的形式从数据库中获取的行数，默认值0不限制，可以适当调整为500，每次获取500行。

**`prepareThreshold`**：确定在切换到使用服务器端预编译语句之前，PreparedStatement所需的执行次数。默认值为 5，这意味着在同一个PreparedStatement对象执行第 5 次时开始使用服务器端预编译语句。

> [PostgreSQL JDBC 官方参数详解](https://jdbc.postgresql.org/documentation/head/connect.html)

## **`JAVA 链接案例`**

### 驱动下载

建议选择使用与 PostgreSQL 与 Greenplum 兼容的驱动。

- [PostgreSQL 官方驱动](https://jdbc.postgresql.org/)

- [Greenplum 官方驱动](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/datadirect-datadirect_jdbc.html#configuring-prepared-statement-execution)

- [PostgreSQL® Extensions to the JDBC API](https://jdbc.postgresql.org/documentation/server-prepare/)

### 连接案例

- [Java JDBC 操作代码示例](https://gitee.com/gaoky2020/ymatrix-labs/tree/master/04_example_javacase/LoadDataTOYMatrix)

## **`Python 链接案例`**

### 驱动下载
- [Python 官方驱动](https://pypi.org/project/psycopg2/)

```python
# 轻量版驱动
pip install psycopg2-binary

# 本地编译
pip install psycopg2
```

### 链接案例
```python

import psycopg2

#  1. 连接数据库（替换为你的实际参数）
conn = psycopg2.connect(
    host="localhost",
    database="testdb",
    user="postgres",
    password="yourpassword"
)

# 2. 创建游标
cur = conn.cursor()

# 3. 创建表（如果不存在）
cur.execute("""
    CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100),
        email VARCHAR(100) UNIQUE
    )
""")

# 4. 插入一条数据
cur.execute(
    "INSERT INTO users (name, email) VALUES (%s, %s)",
    ("Alice", "alice@example.com")
)

# 5. 查询数据
cur.execute("SELECT * FROM users")
rows = cur.fetchall()
for row in rows:
    print(f"ID: {row[0]}, Name: {row[1]}, Email: {row[2]}")

# 6. 关闭连接
conn.commit()  # 提交事务
cur.close()
conn.close()
```

## **Go 链接案例**

### 驱动下载

[pgx](https://github.com/jackc/pgx) 是一个用于 PostgreSQL 的纯 Go 语言驱动程序和工具包。

### 链接案例
```go
package main

import (
	"context"
	"fmt"
	"os"

	"github.com/jackc/pgx/v5"
)

func main() {
	// urlExample := "postgres://username:password@localhost:5432/database_name"
	conn, err := pgx.Connect(context.Background(), os.Getenv("DATABASE_URL"))
	if err != nil {
		fmt.Fprintf(os.Stderr, "Unable to connect to database: %v\n", err)
		os.Exit(1)
	}
	defer conn.Close(context.Background())

	var name string
	var weight int64
	err = conn.QueryRow(context.Background(), "select name, weight from widgets where id=$1", 42).Scan(&name, &weight)
	if err != nil {
		fmt.Fprintf(os.Stderr, "QueryRow failed: %v\n", err)
		os.Exit(1)
	}

	fmt.Println(name, weight)
}
```

> 如果对与兼容性要求高，性能不是很高的场景可以使用[pg](https://pkg.go.dev/github.com/lib/pq) 的连接驱动。

## **C 链接案例**

建议使用 [psqlODBC](https://odbc.postgresql.org/?spm=a2c4g.14484438.10004.13) 的驱动连接 PostgreSQL 。下载并配置驱动的方式如下：

### 驱动下载
```shell
sudo yum install -y unixODBC unixODBC-devel postgresql-odbc
```

### 配置驱动

在`etc/`下创建件`odbcinst.ini`和`odbc.ini`配置文件内容如下:   

```shell
# cat /etc/odbcinst.ini
[PostgreSQL]
Description = ODBC for PostgreSQL
Driver      = /usr/lib/psqlodbcw.so
Setup       = /usr/lib/libodbcpsqlS.so
Driver64    = /usr/lib64/psqlodbcw.so
Setup64     = /usr/lib64/libodbcpsqlS.so
FileUsage = 1

# cat /etc/odbc.ini
[pg]
Description = Test to pg
Driver = PostgreSQL
Database = test
# Servername 集群的 Master,Standby的IP地址
Servername = <Master,Standby>
UserName = username
Password = password
# Master,Standby 的集群端口
Port = <Master Port,Standby Port>
ReadOnly = 0
```

### C 链接代码

编写`odbc_test.c`代码，案例如下：

```C 
#include <stdio.h>
#include <sql.h>
#include <sqlext.h>

void check_error(SQLRETURN ret, SQLHANDLE handle, SQLSMALLINT type) {
    if (!SQL_SUCCEEDED(ret)) {
        SQLCHAR sqlstate[6], message[256];
        SQLSMALLINT len;
        SQLError(handle, type, sqlstate, NULL, message, sizeof(message), &len);
        fprintf(stderr, "ERROR: %s (SQLSTATE %s)\n", message, sqlstate);
        exit(1);
    }
}

int main() {
    SQLHENV env;
    SQLHDBC dbc;
    SQLHSTMT stmt;
    SQLRETURN ret;
    SQLCHAR connstr[] = "DSN=testdb";

    // 1. 初始化环境
    SQLAllocHandle(SQL_HANDLE_ENV, SQL_NULL_HANDLE, &env);
    SQLSetEnvAttr(env, SQL_ATTR_ODBC_VERSION, (void*)SQL_OV_ODBC3, 0);

    // 2. 建立连接
    SQLAllocHandle(SQL_HANDLE_DBC, env, &dbc);
    ret = SQLDriverConnect(dbc, NULL, connstr, SQL_NTS, NULL, 0, NULL, SQL_DRIVER_COMPLETE);
    check_error(ret, dbc, SQL_HANDLE_DBC);

    // 3. 创建表
    SQLAllocHandle(SQL_HANDLE_STMT, dbc, &stmt);
    SQLExecDirect(stmt, "CREATE TABLE IF NOT EXISTS users(id SERIAL PRIMARY KEY, name VARCHAR(50))", SQL_NTS);
    printf("Table created\n");

    // 4. 插入数据
    SQLExecDirect(stmt, "INSERT INTO users(name) VALUES ('CentOS User')", SQL_NTS);
    printf("Data inserted\n");

    // 5. 查询数据
    SQLExecDirect(stmt, "SELECT * FROM users", SQL_NTS);
    SQLCHAR name[50];
    while (SQLFetch(stmt) == SQL_SUCCESS) {
        SQLGetData(stmt, 2, SQL_C_CHAR, name, sizeof(name), NULL);
        printf("User: %s\n", name);
    }

    // 6. 清理资源
    SQLFreeHandle(SQL_HANDLE_STMT, stmt);
    SQLDisconnect(dbc);
    SQLFreeHandle(SQL_HANDLE_DBC, dbc);
    SQLFreeHandle(SQL_HANDLE_ENV, env);
    return 0;
}
```

## **`C# 连接案例`**

### 驱动下载


```c#
sudo rpm -Uvh https://packages.microsoft.com/config/centos/7/packages-microsoft-prod.rpm
sudo yum install -y dotnet-sdk-6.0

```

### 链接案例

创建项目并编写代码

```C#
dotnet new console -n PgOdbcDemo
cd PgOdbcDemo
```

编辑 Program.cs 文件
```C# 
using System;
using System.Data.Odbc;

class Program
{
    static void Main()
    {
        // 使用DSN或直接连接字符串
        string connStr = "Dsn=testdb;";
        
        using (var conn = new OdbcConnection(connStr))
        {
            conn.Open();
            Console.WriteLine("Connected to PostgreSQL via ODBC");

            // 查询版本
            var cmd = new OdbcCommand("SELECT version(), current_user", conn);
            using (var reader = cmd.ExecuteReader())
            {
                while (reader.Read())
                {
                    Console.WriteLine($"Version: {reader[0]}");
                    Console.WriteLine($"Current User: {reader[1]}");
                }
            }
        }
    }
}

```

### 运行项目
```C#
dotnet run
```

## **`Ruby 连接案例`**

### 驱动下载

推荐使用[ruby-pg](https://github.com/ged/ruby-pg)作为`Ruby`的`PostgreSQL`驱动。

> 要求Ruby 2.7 及以上版本。
> PostgreSQL 10 及以上版本。

```Ruby
yum install -y gem

gem install pg -- --with-pg-config=<path to pg_config>

bundle config build.pg --with-pg-config=<path to pg_config>
```

### 链接案例

```Ruby
  #!/usr/bin/env ruby

  require 'pg'

  # Output a table of current connections to the DB
  conn = PG.connect( dbname: 'sales' )
  conn.exec( "SELECT * FROM pg_stat_activity" ) do |result|
    puts "     PID | User             | Query"
    result.each do |row|
      puts " %7d | %-16s | %s " %
        row.values_at('pid', 'usename', 'query')
    end
  end
```

## **`Rust 连接案例`**

### 驱动下载

推荐使用[rust-postgres](https://github.com/sfackler/rust-postgres) 作为数据库的连接驱动。


### 链接案例

```rust
use tokio_postgres::{NoTls, Error};
 
async fn main() -> Result<(), Error> {
 
    let conn_str = "host=localhost user=your_username password=your_password dbname=your_dbname";
    let (client, connection) = tokio_postgres::connect(conn_str, NoTls).await?;
 
 tokio::spawn(async move {
        if let Err(e) = connection.await {
            eprintln!("连接池错误: {}", e);
        }
    });
 
 println!("连接成功！");
 
    let stmt = client.prepare("SELECT version();").await?;
    let rows = client.query(&stmt, &[]).await?;
 
    for row in rows {
        let version: String = row.get(0);
        println!("PostgreSQL版本: {}", version);
    }
 
    Ok(())
}
```

## **`NodeJS 连接案例`**

### 驱动下载

推荐使用[postgres](https://github.com/porsager/postgres)作为链接驱动。

```NodeJS
$ npm install postgres
```



### 链接案例

推荐使用 Async/Await 的方法。

``` NodeJS
const { Client } = require('pg');

async function main() {
  const client = new Client({
    host: 'localhost',
    user: 'postgres',
    password: 'yourpassword',
    database: 'testdb',
  });

  try {
    await client.connect();
    
    await client.query('BEGIN');
    const insertRes = await client.query(
      'INSERT INTO users (name) VALUES ($1) RETURNING id',
      ['Bob']
    );
    await client.query('COMMIT');
    
    console.log('Inserted ID:', insertRes.rows[0].id);
  } catch (err) {
    await client.query('ROLLBACK');
    console.error('Error:', err);
  } finally {
    await client.end();
  }
}

main();
```

## **`PHP 连接案例`**

### 驱动下载

驱动建议安装7.4 及以上的版本。

```php
# 安装驱动
sudo yum install php-pgsql

# 验证安装的驱动
php -m | grep pgsql

```


### 链接案例

```php
<?php
$host = 'localhost';
$dbname = 'testdb';
$user = 'postgres';
$password = 'yourpassword';

try {
    $pdo = new PDO("pgsql:host=$host;dbname=$dbname", $user, $password, [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_PERSISTENT => true,  
        PDO::ATTR_TIMEOUT => 5         
    ]);

    $pdo->exec("
        CREATE TABLE IF NOT EXISTS users (
            id SERIAL PRIMARY KEY,
            name VARCHAR(100),
            email VARCHAR(100) UNIQUE
        )
    ");

    $stmt = $pdo->prepare("INSERT INTO users (name, email) VALUES (?, ?)");
    $stmt->execute(['Alice', 'alice@example.com']);
    echo "Inserted ID: " . $pdo->lastInsertId() . "\n";

    $stmt = $pdo->query("SELECT * FROM users");
    while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
        echo "ID: {$row['id']}, Name: {$row['name']}\n";
    }

} catch (PDOException $e) {
    die("Error: " . $e->getMessage());
}
?>
```


## **`Perl 连接案例`**

### 驱动下载

建议使用Perl5 的5.40.0 及更高版本

```perf
# Debian/Ubuntu 安装方式
sudo apt-get install libdbd-pg-perl

# CentOS/RHEL 安装方式
sudo yum install perl-DBD-Pg
```

### 链接案例

```perl
#!/usr/bin/perl
use strict;
use warnings;
use DBI;

my $dsn = "DBI:Pg:dbname=testdb;host=localhost;port=5432";
my $user = "postgres";
my $password = "yourpassword";

my $dbh = DBI->connect($dsn, $user, $password, {
    RaiseError => 1,       # 自动报错
    AutoCommit => 0,       # 关闭自动提交（启用事务）
    pg_server_prepare => 1 # 启用服务器端预处理
}) or die $DBI::errstr;

$dbh->do(<<'SQL');
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100) UNIQUE
)
SQL

my $insert = $dbh->prepare(
    "INSERT INTO users (name, email) VALUES (?, ?) RETURNING id"
);
$insert->execute('Alice', 'alice@example.com');
my $new_id = $insert->fetchrow_arrayref->[0];
print "Inserted ID: $new_id\n";

my $sth = $dbh->prepare("SELECT * FROM users");
$sth->execute();
while (my $row = $sth->fetchrow_hashref) {
    print "ID: $row->{id}, Name: $row->{name}\n";
}

$dbh->commit;

$sth->finish;
$dbh->disconnect;
```

## Excel 链接数据库

### 驱动下载

建议从[psqlODBC 驱动安装包](https://www.postgresql.org/ftp/odbc/versions.old/msi/)下载ODBC 驱动。

### 链接案例

链接案例请参考[使用 Excel 访问 PostgreSQL](https://www.rockdata.net/zh-cn/tutorial/excel-odbc-connect/)

