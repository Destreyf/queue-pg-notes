# 🏗️ Multi-Stage Distroless Docker Build: Step-by-Step

## **1. Why Multi-Stage and Distroless?**

- **Multi-stage builds** let you use a full-featured image for building (compiling, installing dependencies), then copy only the results into a minimal runtime image.
- **Distroless images** contain only the language runtime and your app—no shell, no package manager, no OS utilities. This means:
  - **Tiny image size** (often < 100MB for Python apps)
  - **Minimal attack surface** (harder for attackers to exploit)
  - **Fewer vulnerabilities** (no unused packages)

---

## **2. Project Structure**

```
consumer/
├── consumer.py
├── requirements.txt
└── Dockerfile
```

---

## **3. requirements.txt**

```text
pika
psycopg[binary]
```
- `pika`: For RabbitMQ.
- `psycopg[binary]`: For fast PostgreSQL access.

---

## **4. Dockerfile (with Detailed Comments)**

```dockerfile
# ---- Stage 1: Build dependencies and app ----
FROM python:3.11-slim AS builder

WORKDIR /app

# Install build dependencies for psycopg (needed for binary wheels)
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# Install Python dependencies into a separate folder (/install)
COPY requirements.txt .
RUN pip install --upgrade pip && \
    pip install --prefix=/install --no-cache-dir -r requirements.txt

# Copy application code
COPY consumer.py .

# ---- Stage 2: Create minimal runtime image using distroless ----
FROM gcr.io/distroless/python3-debian11

# Set working directory in the new image
WORKDIR /app

# Copy installed Python packages from builder
COPY --from=builder /install /usr/local

# Copy application code from builder
COPY --from=builder /app/consumer.py .

# (Optional) Set a non-root user for extra security
USER nonroot

# Set the default command to run your consumer
CMD ["consumer.py"]
```

---

## **5. Step-by-Step Build and Run Instructions**

### **A. Build the Image**

```bash
docker build -t my-consumer:distroless .
```

### **B. Run the Container**

```bash
docker run --rm --network=host my-consumer:distroless
```
- Use `--network=host` for local development, or configure Docker networking as needed.
- If you need to pass environment variables (e.g., for DB or RabbitMQ connection strings), use `-e` flags:
  ```bash
  docker run -e RABBITMQ_URL=... -e POSTGRES_DSN=... my-consumer:distroless
  ```

---

## **6. Key Notes and Details**

### **Why Use `/install` and `--prefix`?**
- `pip install --prefix=/install ...` puts all dependencies in `/install/lib/python3.11/site-packages`.
- When you copy `/install` to `/usr/local` in the distroless image, Python will find them as usual.

### **Why No Shell?**
- Distroless images do **not** include `/bin/sh` or any shell.
- You **cannot** exec into the container and run shell commands.
- All debugging must be done in the build stage or by running the app in a non-distroless image.

### **Why Use `USER nonroot`?**
- Distroless images include a `nonroot` user for security.
- Running as non-root is a best practice for production.

### **How to Debug?**
- If you need to debug, temporarily use `python:3.11-slim` as your final image, or add a shell to the image for troubleshooting.

### **How Small Is It?**
- The final image will be **~80-100MB** (vs. 300-400MB for a full Python image).

### **How to Add More Files?**
- If your app has more files (modules, configs), copy them in the same way as `consumer.py`.

### **How to Use Environment Variables?**
- Distroless images support environment variables as usual via Docker’s `-e` flag or `ENV` in the Dockerfile.

### **How to Use with Docker Compose?**
- Just reference your built image in your `docker-compose.yml` as you would any other image.

---

## **7. Example: Docker Compose Service**

```yaml
services:
  consumer:
    image: my-consumer:distroless
    environment:
      - RABBITMQ_URL=amqp://user:pass@rabbitmq:5672/
      - POSTGRES_DSN=postgresql://myuser:mypass@postgres:5432/mydb
    depends_on:
      - rabbitmq
      - postgres
```

---

## **8. References**

- [Distroless Python images](https://github.com/GoogleContainerTools/distroless/blob/main/examples/python3.Dockerfile)
- [psycopg binary install docs](https://www.psycopg.org/psycopg3/docs/basic/install.html#binary-install)
- [Docker multi-stage builds](https://docs.docker.com/build/building/multi-stage/)

---

## **Summary Table**

| Stage      | Base Image                | Purpose                        |
|------------|--------------------------|--------------------------------|
| Builder    | python:3.11-slim         | Install/build dependencies     |
| Final      | gcr.io/distroless/python3-debian11 | Run app, minimal runtime      |

---