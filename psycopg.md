# PostgreSQL: Returning Structured Data to Python
Saved on: 2025-04-05
Topics covered:
- Creating PostgreSQL functions returning TABLE, composite, or JSON
- Using psycopg2 vs psycopg (v3)
- Fetching results as dicts, namedtuples, or JSON in Python

Great question! Returning a **structured result** (like a row, record, or custom type) from a PostgreSQL function and consuming it in **Python** is very common — and there are several clean ways to do it.

---

## ✅ Step-by-Step: Return Structure from PostgreSQL Function → Python

---

### 🎯 Goal:
Call a PostgreSQL function that returns a structured result (e.g., a composite type, table row, or `SETOF record`) and get it as a **dictionary, namedtuple, or dataclass** in Python.

---

## ✅ Option 1: Function Returns Table Row(s) → Use `psycopg2` or `psycopg`

This is the most common and recommended approach.

### 🔧 PostgreSQL Function Example:

```sql
-- Create a simple function that returns a table-like structure
CREATE OR REPLACE FUNCTION get_user_by_id(user_id INT)
RETURNS TABLE(id INT, name TEXT, email TEXT) AS $$
BEGIN
    RETURN QUERY
    SELECT u.id, u.name, u.email
    FROM users u
    WHERE u.id = user_id;
END;
$$ LANGUAGE plpgsql;
```

> 💡 You can also use `RETURNS SETOF your_table_name` if you’re returning rows from an existing table.

---

### 🐍 Python Code Using `psycopg2`:

```python
import psycopg2
from psycopg2.extras import RealDictCursor

conn = psycopg2.connect(
    dbname="your_db",
    user="your_user",
    password="your_pass",
    host="localhost"
)

cur = conn.cursor(cursor_factory=RealDictCursor)  # ← Returns dicts!

cur.callproc('get_user_by_id', [123])  # Call function with param

result = cur.fetchone()  # or fetchall() for multiple rows

if result:
    print(dict(result))  # {'id': 123, 'name': 'Alice', 'email': 'alice@example.com'}

cur.close()
conn.close()
```

> ✅ `RealDictCursor` gives you each row as a dictionary — super easy to work with!

---

## ✅ Option 2: Function Returns a Composite Type (Custom Struct)

### 🔧 PostgreSQL:

```sql
-- Define a custom type
CREATE TYPE user_summary AS (
    user_id INT,
    full_name TEXT,
    signup_date DATE
);

-- Function returning that type
CREATE OR REPLACE FUNCTION get_user_summary(uid INT)
RETURNS user_summary AS $$
DECLARE
    result user_summary;
BEGIN
    SELECT id, name, created_at::DATE
    INTO result
    FROM users
    WHERE id = uid;

    RETURN result;
END;
$$ LANGUAGE plpgsql;
```

---

### 🐍 Python:

```python
cur = conn.cursor()

cur.callproc('get_user_summary', [123])
row = cur.fetchone()

# Result is a tuple unless you map it
print(row)  # e.g., (123, 'Alice', datetime.date(2024, 1, 1))

# Optional: Convert to dict manually or use namedtuple
from collections import namedtuple

UserSummary = namedtuple('UserSummary', ['user_id', 'full_name', 'signup_date'])
user = UserSummary._make(row)

print(user.full_name)  # 'Alice'
```

> ⚠️ Without `RealDictCursor`, composite types come back as tuples. You’ll need to map field names yourself.

---

## ✅ Option 3: Return JSON (Flexible & Easy for APIs)

Sometimes it’s easier to return `JSON` or `JSONB` from Postgres and parse in Python.

### 🔧 PostgreSQL:

```sql
CREATE OR REPLACE FUNCTION get_user_json(user_id INT)
RETURNS JSON AS $$
SELECT json_build_object(
    'id', id,
    'name', name,
    'email', email
)
FROM users
WHERE id = user_id;
$$ LANGUAGE sql;
```

---

### 🐍 Python:

```python
cur = conn.cursor()
cur.callproc('get_user_json', [123])
result = cur.fetchone()[0]  # First column contains JSON string

import json
user_data = json.loads(result)
print(user_data['name'])  # 'Alice'
```

> ✅ Great for dynamic structures or when integrating with web APIs.

---

## ✅ Bonus: Using Modern `psycopg` (v3+) — Even Better!

If you’re using `psycopg` (the newer, async-friendly driver), you get automatic mapping to dicts/namedtuples out of the box.

```python
import psycopg

conn = await psycopg.AsyncConnection.connect("dbname=test user=postgres")

# Or sync:
# conn = psycopg.connect(...)

cur = conn.execute("SELECT * FROM get_user_by_id(%s)", (123,))

# Each row is a "Row" object — behaves like tuple AND dict!
row = cur.fetchone()
print(row['name'])        # ← dict-style access
print(row[1])             # ← tuple-style access

# Or convert all to dicts:
rows = cur.fetchall()
dict_rows = [dict(r) for r in rows]
```

> 💡 `psycopg3` is highly recommended for new projects — supports async, better typing, and automatic row mapping.

---

## ✅ Summary Table

| Approach                 | PostgreSQL Returns       | Python Output         | Best For                     |
|--------------------------|--------------------------|------------------------|------------------------------|
| `RETURNS TABLE(...)`     | Rows with named columns  | Dict (via RealDictCursor) | Most cases ✅                |
| Composite Type           | Tuple                    | NamedTuple / manual map | When reusing struct in SQL   |
| `RETURNS JSON`           | JSON string              | `json.loads()` result  | APIs, dynamic structures     |
| `psycopg3` Row objects   | Any                      | Dict/tuple hybrid      | Modern apps, async support   |

---

## 🛠 Pro Tips:

- Always use parameterized queries (`%s` or `$1`) — never concatenate strings!
- Use context managers (`with conn`, `with cur`) for auto-cleanup.
- Consider `asyncpg` if you need high-performance async I/O (but steeper learning curve).

---

## ❓ Need Help Adapting This?

Paste your PostgreSQL function definition + what structure you want in Python, and I’ll give you the exact code!

You got this — and I’ve got you 😊