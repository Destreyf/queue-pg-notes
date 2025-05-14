
# 🐳 **Full Example: PostgreSQL + PgBouncer with Custom Config and Healthchecks**

## **1. Directory Structure**

```
project/
├── docker-compose.yml
├── pgbouncer/
│   ├── pgbouncer.ini
│   └── userlist.txt
```

---

## **2. Custom PgBouncer Config (`pgbouncer/pgbouncer.ini`)**

```ini
[databases]
mydb = host=postgres port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = session
max_client_conn = 200
default_pool_size = 20
reserve_pool_size = 5
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
server_reset_query = DISCARD ALL
admin_users = myuser
```

---

## **3. PgBouncer Userlist (`pgbouncer/userlist.txt`)**

> **Note:** The password must be in MD5 format:  
> `md5` + md5(password + username)  
> You can generate it with `pg_md5` or online tools.

Example for user `myuser` with password `mypass`:

```text
"myuser" "md5e5e9fa1ba31ecd1ae84f75caaa474f3a663f05f4"
```

*(The above hash is just an example; generate your own for production!)*

---

## **4. Docker Compose File (`docker-compose.yml`)**

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypass
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser"]
      interval: 10s
      timeout: 5s
      retries: 5

  pgbouncer:
    image: edoburu/pgbouncer:1.20.1
    ports:
      - "6432:6432"
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ./pgbouncer/pgbouncer.ini:/etc/pgbouncer/pgbouncer.ini:ro
      - ./pgbouncer/userlist.txt:/etc/pgbouncer/userlist.txt:ro
    environment:
      - DB_USER=myuser
      - DB_PASSWORD=mypass
      - DB_HOST=postgres
      - DB_NAME=mydb
    healthcheck:
      test: ["CMD", "pg_isready", "-h", "localhost", "-p", "6432", "-U", "myuser"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

---

## **5. How to Generate MD5 Passwords for `userlist.txt`**

You can use the `pg_md5` tool (comes with Postgres) or Python:

```python
import hashlib
def pg_md5(password, username):
    s = password + username
    return 'md5' + hashlib.md5(s.encode('utf-8')).hexdigest()
print(pg_md5('mypass', 'myuser'))
```

---

## **6. How to Use**

1. **Build and start services:**
   ```bash
   docker-compose up -d
   ```

2. **Check health:**
   ```bash
   docker-compose ps
   # Both services should show "healthy" after a short time
   ```

3. **Connect your app to PgBouncer:**
   ```
   postgresql://myuser:mypass@localhost:6432/mydb
   ```

---

## **7. Healthcheck Details**

- **Postgres:** Uses `pg_isready` to check if the DB is accepting connections.
- **PgBouncer:** Uses `pg_isready` to check if PgBouncer is accepting connections on port 6432.

---

## **8. Tuning Tips**

- Adjust `default_pool_size` and `reserve_pool_size` in `pgbouncer.ini` to match your workload and Postgres `max_connections`.
- Use `pool_mode = session` for compatibility with `COPY FROM`.
- For most web apps, `pool_mode = transaction` is more efficient, but not compatible with `COPY FROM`.

---

## **9. Security Notes**

- Never commit real passwords or hashes to public repos.
- For production, restrict PgBouncer and Postgres ports to internal networks only.

---

## **10. References**

- [PgBouncer official docs](https://www.pgbouncer.org/config.html)
- [edoburu/pgbouncer Docker image](https://github.com/edoburu/docker-pgbouncer)
- [Postgres authentication](https://www.postgresql.org/docs/current/auth-password.html)

---