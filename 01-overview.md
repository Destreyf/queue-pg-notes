# 1. **Overview**

You want to:
- Use a message queue to decouple data producers and consumers.
- Have lightweight Python producers and high-throughput consumers.
- Consumers should batch-insert into PostgreSQL (ideally using `COPY`).
- Use as few DB connections as possible.
- Handle message retries and dead-letter queues (DLQ).
- Deploy everything in Docker.
- Optionally, share Django models with the consumer.

---

# 2. **Choosing a Queue System**

**RabbitMQ** is a solid choice, but **[Redpanda](https://redpanda.com/)** (Kafka-compatible, easy Docker deployment) or **[NATS](https://nats.io/)** are also good. For this guide, we’ll use **RabbitMQ** for its simplicity and rich Python support.

---

# 3. **Architecture Diagram**

```
[Producer(s)] --> [RabbitMQ Queue] --> [Consumer(s)] --> [PostgreSQL]
                                         |
                                         v
                                 [Dead Letter Queue]
```

---

# 4. **Docker Compose Setup**

Here’s a minimal `docker-compose.yml` to run RabbitMQ and PostgreSQL:

```yaml
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"  # Management UI
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

volumes:
  pgdata:
```

---

# 5. **Python Producer Example**

Install dependencies:

```bash
pip install pika
```

**Producer code:**

```python
# producer.py
import pika
import json

RABBITMQ_URL = "amqp://user:pass@localhost:5672/"
QUEUE_NAME = "my_queue"

def send_message(data):
    connection = pika.BlockingConnection(pika.URLParameters(RABBITMQ_URL))
    channel = connection.channel()
    channel.queue_declare(queue=QUEUE_NAME, durable=True)
    channel.basic_publish(
        exchange='',
        routing_key=QUEUE_NAME,
        body=json.dumps(data),
        properties=pika.BasicProperties(delivery_mode=2)  # make message persistent
    )
    connection.close()

if __name__ == "__main__":
    for i in range(100):
        send_message({"id": i, "value": f"data_{i}"})
```

---

# 6. **Python Consumer with Batching and Dead Letter Queue**

Install dependencies:

```bash
pip install pika psycopg[binary]
```

**Consumer code:**

```python
# consumer.py
import pika
import json
import psycopg
import time

RABBITMQ_URL = "amqp://user:pass@rabbitmq:5672/"
QUEUE_NAME = "my_queue"
DLQ_NAME = "my_queue_dlq"
BATCH_SIZE = 100
RETRY_LIMIT = 5

POSTGRES_DSN = "postgresql://myuser:mypass@postgres:5432/mydb"

def setup_queues(channel):
    # Main queue with DLQ
    args = {
        'x-dead-letter-exchange': '',
        'x-dead-letter-routing-key': DLQ_NAME,
        'x-max-delivery-count': RETRY_LIMIT
    }
    channel.queue_declare(queue=DLQ_NAME, durable=True)
    channel.queue_declare(queue=QUEUE_NAME, durable=True, arguments=args)

def batch_insert(conn, rows):
    # Use COPY for high throughput
    with conn.cursor() as cur:
        with cur.copy("COPY my_table (id, value) FROM STDIN WITH (FORMAT csv)") as copy:
            for row in rows:
                copy.write_row([row['id'], row['value']])

def main():
    connection = pika.BlockingConnection(pika.URLParameters(RABBITMQ_URL))
    channel = connection.channel()
    setup_queues(channel)

    batch = []
    delivery_tags = []
    pg_conn = psycopg.connect(POSTGRES_DSN, autocommit=True)

    def callback(ch, method, properties, body):
        nonlocal batch, delivery_tags
        data = json.loads(body)
        batch.append(data)
        delivery_tags.append(method.delivery_tag)
        if len(batch) >= BATCH_SIZE:
            try:
                batch_insert(pg_conn, batch)
                for tag in delivery_tags:
                    ch.basic_ack(delivery_tag=tag)
            except Exception as e:
                print("Batch insert failed:", e)
                # Don't ack, messages will be retried
            finally:
                batch = []
                delivery_tags = []

    channel.basic_qos(prefetch_count=BATCH_SIZE)
    channel.basic_consume(queue=QUEUE_NAME, on_message_callback=callback, auto_ack=False)

    print("Consumer started. Waiting for messages...")
    try:
        channel.start_consuming()
    except KeyboardInterrupt:
        print("Stopping consumer...")
    finally:
        pg_conn.close()
        connection.close()

if __name__ == "__main__":
    main()
```

---

# 7. **Dead Letter Queue Consumer**

You can run a similar consumer for the DLQ, maybe just logging or storing failed messages for manual inspection.

```python
# dlq_consumer.py
import pika
import json

RABBITMQ_URL = "amqp://user:pass@rabbitmq:5672/"
DLQ_NAME = "my_queue_dlq"

def main():
    connection = pika.BlockingConnection(pika.URLParameters(RABBITMQ_URL))
    channel = connection.channel()
    channel.queue_declare(queue=DLQ_NAME, durable=True)

    def callback(ch, method, properties, body):
        data = json.loads(body)
        print("DLQ message:", data)
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(queue=DLQ_NAME, on_message_callback=callback, auto_ack=False)
    print("DLQ Consumer started. Waiting for messages...")
    channel.start_consuming()

if __name__ == "__main__":
    main()
```

---

# 8. **Sharing Django Models**

If you want to share Django models:
- Put your models in a separate Python package (e.g., `myapp/models.py`).
- Install your Django app as a dependency in the consumer.
- Use [Django’s standalone usage](https://docs.djangoproject.com/en/5.0/topics/settings/#calling-django-setup-is-required-for-standalone-django-usage) to load models, but **for high-throughput ingestion, use raw SQL or `psycopg` directly**.

---

# 9. **Deployment**

- Build Docker images for your producer and consumer.
- Use the same `docker-compose.yml` to orchestrate everything.
- Example Dockerfile for consumer:

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY consumer.py .
RUN pip install pika psycopg[binary]
CMD ["python", "consumer.py"]
```

---

# 10. **Summary of Key Points**

- **Producers**: Simple, stateless, send JSON to RabbitMQ.
- **Consumers**: Batch messages, use `COPY` for fast Postgres ingest, minimal DB connections.
- **Retry/Dead Letter**: Use RabbitMQ’s DLQ features, with a separate DLQ consumer.
- **Docker**: All components run in Docker for easy deployment.
- **Django Models**: Can be shared, but for ingestion, use direct DB access for speed.

---

# 11. **Further Reading**

- [RabbitMQ Dead Letter Queues](https://www.rabbitmq.com/dlx.html)
- [psycopg COPY documentation](https://www.psycopg.org/psycopg3/docs/api/copy.html)
- [Django standalone scripts](https://docs.djangoproject.com/en/5.2/topics/settings/#calling-django-setup-is-required-for-standalone-django-usage)

---