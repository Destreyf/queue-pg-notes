# 🐳 **PgBouncer + PostgreSQL with Docker: Quick Tutorial**

## **1. What is PgBouncer?**

- **PgBouncer** is a lightweight connection pooler for PostgreSQL.
- It allows many clients to connect to PgBouncer, which then maintains a smaller pool of actual connections to Postgres.
- This reduces resource usage and improves scalability.

---

## **2. Example `docker-compose.yml`**

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

  pgbouncer:
    image: edoburu/pgbouncer:1.20.1
    ports:
      - "6432:6432"
    environment:
      - DB_USER=myuser
      - DB_PASSWORD=mypass
      - DB_HOST=postgres
      - DB_NAME=mydb
      - POOL_MODE=session
      - MAX_CLIENT_CONN=100
      - DEFAULT_POOL_SIZE=20
    depends_on:
      - postgres

volumes:
  pgdata:
```

**Notes:**
- PgBouncer listens on port **6432** (default).
- The `edoburu/pgbouncer` image is a popular, maintained Docker image.
- Environment variables configure PgBouncer for a simple setup.

---

## **3. Connecting Your App**

- **To use PgBouncer**, point your app to `pgbouncer:6432` instead of `postgres:5432`.
- Example DSN:
  ```
  postgresql://myuser:mypass@pgbouncer:6432/mydb
  ```

---

## **4. How Does `COPY FROM` Work with PgBouncer?**

- **`COPY FROM` and `COPY TO` require a "session" or "transaction" pool mode.**
- **`POOL_MODE=session`** (the default in the example above) is safest for `COPY`.
- In **`POOL_MODE=transaction`**, `COPY` may fail because PgBouncer can’t guarantee the connection stays assigned for the whole operation.
- **Never use `POOL_MODE=statement` with `COPY`!**

**Summary:**  
- Use `POOL_MODE=session` for `COPY FROM` to work reliably.

---

## **5. Basic PgBouncer Tuning**

- **`max_client_conn`**: Max client connections PgBouncer will accept (e.g., 100–1000).
- **`default_pool_size`**: Max server connections per database/user pair (e.g., 10–50).
- **`reserve_pool_size`**: Extra connections for spikes (optional).
- **`pool_mode`**: Use `session` for `COPY`, `transaction` for most web apps.

**Example tuning for a small server:**
```yaml
    environment:
      - POOL_MODE=session
      - MAX_CLIENT_CONN=200
      - DEFAULT_POOL_SIZE=20
      - RESERVE_POOL_SIZE=5
```

- **Rule of thumb:**  
  `default_pool_size * number_of_databases * number_of_users` should not exceed your Postgres `max_connections`.

---

## **6. Customizing PgBouncer Further**

For more advanced setups, you can mount your own `pgbouncer.ini` and `userlist.txt` files as volumes.  
See [edoburu/pgbouncer docs](https://github.com/edoburu/docker-pgbouncer) for details.

---

## **7. Quick Start Commands**

```bash
docker-compose up -d
# Wait for containers to start
docker-compose logs pgbouncer
```

---

## **8. Summary Table**

| Setting            | Description                                 | Example Value |
|--------------------|---------------------------------------------|---------------|
| pool_mode          | Pooling mode (`session` for COPY)           | session       |
| max_client_conn    | Max client connections to PgBouncer         | 100–1000      |
| default_pool_size  | Max Postgres connections per db/user        | 10–50         |
| reserve_pool_size  | Extra connections for spikes                | 5             |

---

## **9. References**

- [PgBouncer Docs](https://www.pgbouncer.org/config.html)
- [edoburu/pgbouncer Docker image](https://github.com/edoburu/docker-pgbouncer)
- [Postgres COPY command](https://www.postgresql.org/docs/current/sql-copy.html)

---