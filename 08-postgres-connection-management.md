## **General Guidance**

- **PostgreSQL is not designed for thousands of concurrent connections.**
- Each connection is a process and consumes memory and CPU.
- **Too many connections** can overwhelm the server, causing context switching and memory exhaustion.

### **Rule of Thumb**

- **2–4 active connections per CPU core** is a common starting point for OLTP (transactional) workloads.
- For **batch or analytical workloads**, you may want even fewer.

#### **Example:**
- If your server has **8 CPU cores**, start with **16–32 max connections**.

### **Why so few?**
- Most connections are **idle** most of the time in web apps.
- If you need to support more clients, use a **connection pooler** (like [PgBouncer](https://www.pgbouncer.org/)).

---

## **PostgreSQL Default**

- The default `max_connections` is **100**.
- This is often **too high** for small servers, and **too low** for large ones.

---

## **How to Tune**

1. **Estimate memory usage per connection:**
   - Each connection can use **5–10 MB** of RAM (sometimes more).
   - Multiply by your desired max connections and ensure it fits in RAM.

2. **Set `max_connections` in `postgresql.conf`:**
   ```conf
   max_connections = 32
   ```

3. **Use a connection pooler** if you need to support many clients (hundreds or thousands).

---

## **Summary Table**

| Cores | Recommended max_connections |
|-------|----------------------------|
| 2     | 4–8                        |
| 4     | 8–16                       |
| 8     | 16–32                      |
| 16    | 32–64                      |

---

## **Best Practice**

- **Start low** (2–4 per core), monitor, and increase only if needed.
- **Use PgBouncer** for high connection counts.

---

### **References**

- [PostgreSQL Wiki: Tuning Your PostgreSQL Server](https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server)
- [PostgreSQL Documentation: max_connections](https://www.postgresql.org/docs/current/runtime-config-connection.html)
- [PgBouncer](https://www.pgbouncer.org/)

---

**In summary:**  
**2–4 connections per core** is a safe starting point. For high concurrency, use a connection pooler.