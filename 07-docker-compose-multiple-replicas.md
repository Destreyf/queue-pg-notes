Absolutely! Here’s a **detailed example** of a `docker-compose.yml` that:

- Runs **multiple instances** of your consumer (scalable, e.g., 3–5 replicas)
- Uses **automatic restarts** (`restart: always` or `restart: unless-stopped`)
- Connects to RabbitMQ and Postgres services
- Is easy to scale up or down

---

## **1. Example `docker-compose.yml`**

```yaml
version: '3.8'

services:
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: user
      RABBITMQ_DEFAULT_PASS: pass

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

  consumer:
    image: my-consumer:distroless  # <-- Build this image as shown previously
    restart: always # you might want to use "unless-stopped" (without quotes) instead
    environment:
      RABBITMQ_URL: amqp://user:pass@rabbitmq:5672/
      POSTGRES_DSN: postgresql://myuser:mypass@postgres:5432/mydb
    depends_on:
      - rabbitmq
      - postgres
    deploy:
      replicas: 3  # <--- Set to 3-5 as needed
    # For Compose v3, 'deploy' is only supported in Swarm mode.
    # For local/dev, use 'docker-compose up --scale consumer=3'
    # Remove 'deploy' if not using Swarm.

volumes:
  pgdata:
```

---

## **2. How to Scale Consumers**

### **A. For Local Development (non-Swarm):**

- **Remove the `deploy:` block** (it’s ignored by default Compose).
- Use the `--scale` flag:

```bash
docker-compose up --build --scale consumer=3
```
- This will run 3 instances. Change the number as needed.

### **B. For Docker Swarm (Production):**

- **Keep the `deploy:` block**.
- Deploy the stack:

```bash
docker stack deploy -c docker-compose.yml mystack
```
- Swarm will run the specified number of replicas and handle restarts.

---

## **3. Automatic Restarts**

- `restart: always` ensures that if a consumer crashes or the host reboots, Docker will restart it automatically.

---

## **4. Notes**

- All consumer instances will compete for messages from the same RabbitMQ queue (this is how you scale out consumers).
- Each consumer should be stateless and able to process any message.
- You can increase or decrease the number of consumers at any time.

---

## **5. Example: Scaling Up and Down**

- To scale to 5 consumers:

```bash
docker-compose up --scale consumer=5
```

- To scale down to 2 consumers:

```bash
docker-compose up --scale consumer=2
```

Or, if running in Swarm, update the `replicas` value and redeploy.

---

## **6. Full Example Directory Structure**

```
project/
├── consumer/
│   ├── consumer.py
│   ├── requirements.txt
│   └── Dockerfile
└── docker-compose.yml
```

---

## **7. Summary Table**

| Service   | Replicas | Restart Policy | Notes                        |
|-----------|----------|---------------|------------------------------|
| rabbitmq  | 1        | default       | Management UI on 15672       |
| postgres  | 1        | default       | Data persisted in volume     |
| consumer  | 3–5      | always        | Scalable, stateless workers  |

---