## 1. **Assume a Table**

Suppose you have a table:

```sql
CREATE TABLE my_table (
    id INTEGER PRIMARY KEY,
    value TEXT
);
```

## 2. **Python Example: Streaming with `COPY`**

You want to stream rows from Python into Postgres as CSV lines, using `psycopg`'s `copy` context manager.

### **Install psycopg**

```bash
pip install psycopg[binary]
```

### **Code Example**

```python
import psycopg
import io

# Connection string (adjust as needed)
dsn = "postgresql://myuser:mypass@localhost:5432/mydb"

# Example data to insert
rows = [
    {"id": 1, "value": "foo"},
    {"id": 2, "value": "bar"},
    {"id": 3, "value": "baz"},
]

def row_to_csv(row):
    # Convert a dict to a CSV line (no header)
    return f'{row["id"]},{row["value"]}\n'

with psycopg.connect(dsn) as conn:
    with conn.cursor() as cur:
        # Use COPY FROM STDIN with CSV format
        with cur.copy("COPY my_table (id, value) FROM STDIN WITH (FORMAT csv)") as copy:
            for row in rows:
                # Write each row as a CSV line
                copy.write(row_to_csv(row))
```

### **Explanation**

- `cur.copy(...)` opens a context manager for streaming data into Postgres.
- `copy.write(...)` sends data to Postgres as if it were coming from a file.
- Each line must be a valid CSV row (no header).
- This is **much faster** than individual `INSERT` statements.

---

## 3. **Streaming Large Data**

If you have a generator or want to process data in batches, you can do:

```python
def generate_rows():
    for i in range(1000000):
        yield {"id": i, "value": f"value_{i}"}

with psycopg.connect(dsn) as conn:
    with conn.cursor() as cur:
        with cur.copy("COPY my_table (id, value) FROM STDIN WITH (FORMAT csv)") as copy:
            for row in generate_rows():
                copy.write(row_to_csv(row))
```

---

## 4. **Using a File-like Object (Advanced)**

If you have a file-like object (e.g., `io.StringIO`), you can use `copy.from_file()`:

```python
csv_buffer = io.StringIO()
for row in rows:
    csv_buffer.write(row_to_csv(row))
csv_buffer.seek(0)

with psycopg.connect(dsn) as conn:
    with conn.cursor() as cur:
        cur.copy("COPY my_table (id, value) FROM STDIN WITH (FORMAT csv)", source=csv_buffer)
```

---

## 5. **Best Practices**

- **Batching**: Accumulate a batch of rows, then stream them in one `COPY` operation.
- **Error Handling**: Wrap in try/except to handle DB errors.
- **Minimal Connections**: Reuse the same connection for multiple batches if possible.

---

## 6. **References**

- [psycopg3 COPY API](https://www.psycopg.org/psycopg3/docs/api/copy.html)
- [Postgres COPY docs](https://www.postgresql.org/docs/current/sql-copy.html)

---