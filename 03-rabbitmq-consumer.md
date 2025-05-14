Absolutely! Here’s a **complete, minimal, and customizable example** of a **RabbitMQ consumer** in Python that:

- Consumes messages from a RabbitMQ queue,
- Batches them up to a configurable size,
- Uses `psycopg`’s `COPY FROM STDIN` for high-speed bulk insert into PostgreSQL.

This is designed for **maximum throughput** and **easy batch size tuning**.

---

## 1. **Assumptions**

- **RabbitMQ** is running and accessible.
- **PostgreSQL** is running and accessible.
- The table exists:

    ```sql
    CREATE TABLE my_table (
        id INTEGER PRIMARY KEY,
        value TEXT
    );
    ```

- Messages are JSON objects like: `{"id": 123, "value": "foo"}`

---

## 2. **Install Dependencies**

```bash
pip install pika psycopg[binary]
```

---

## 3. **Consumer Code**

```python
import pika
import psycopg
import json
import time

# --- CONFIGURABLE PARAMETERS ---
RABBITMQ_URL = "amqp://user:pass@localhost:5672/"
QUEUE_NAME = "my_queue"
POSTGRES_DSN = "postgresql://myuser:mypass@localhost:5432/mydb"
BATCH_SIZE = 100  # <--- Change this to tune batch size
BATCH_TIMEOUT = 2.0  # seconds: flush batch if this much time passes

# --- CSV conversion helper ---
def row_to_csv(row):
    # Escape commas and newlines if needed for real data!
    return f'{row["id"]},{row["value"]}\n'

def main():
    # Set up RabbitMQ connection
    connection = pika.BlockingConnection(pika.URLParameters(RABBITMQ_URL))
    channel = connection.channel()
    channel.queue_declare(queue=QUEUE_NAME, durable=True)

    # Set up Postgres connection
    pg_conn = psycopg.connect(POSTGRES_DSN, autocommit=True)
    cur = pg_conn.cursor()

    # Batch state
    batch = []
    delivery_tags = []
    last_flush = time.time()

    def flush_batch():
        nonlocal batch, delivery_tags, last_flush
        if not batch:
            return
        try:
            with cur.copy(
                "COPY my_table (id, value) FROM STDIN WITH (FORMAT csv)"
            ) as copy:
                for row in batch:
                    copy.write(row_to_csv(row))
            # Acknowledge all messages in the batch
            for tag in delivery_tags:
                channel.basic_ack(delivery_tag=tag)
            print(f"Inserted batch of {len(batch)} rows")
        except Exception as e:
            print("Batch insert failed:", e)
            # Do not ack, messages will be retried
        finally:
            batch.clear()
            delivery_tags.clear()
            last_flush = time.time()

    def on_message(ch, method, properties, body):
        nonlocal batch, delivery_tags, last_flush
        data = json.loads(body)
        batch.append(data)
        delivery_tags.append(method.delivery_tag)
        # Flush if batch size reached
        if len(batch) >= BATCH_SIZE:
            flush_batch()

    # Set prefetch to batch size for efficiency
    channel.basic_qos(prefetch_count=BATCH_SIZE)
    channel.basic_consume(
        queue=QUEUE_NAME, on_message_callback=on_message, auto_ack=False
    )

    print("Consumer started. Waiting for messages...")
    try:
        while True:
            connection.process_data_events(time_limit=1)
            # Flush batch if timeout reached
            if batch and (time.time() - last_flush) > BATCH_TIMEOUT:
                flush_batch()
    except KeyboardInterrupt:
        print("Stopping consumer...")
    finally:
        # Final flush on exit
        flush_batch()
        cur.close()
        pg_conn.close()
        connection.close()

if __name__ == "__main__":
    main()
```

---

## 4. **How to Customize Batch Size**

- **Change** `BATCH_SIZE = 100` to any integer you want.
- **Change** `BATCH_TIMEOUT = 2.0` to control how long to wait before flushing a partial batch (in seconds).

---

## 5. **How it Works**

- Messages are consumed and added to a batch.
- When the batch reaches `BATCH_SIZE`, or `BATCH_TIMEOUT` seconds have passed since the last flush, the batch is inserted into Postgres using `COPY`.
- All messages in the batch are acknowledged only after a successful insert.
- If the insert fails, messages are **not acknowledged** and will be retried by RabbitMQ.

---

## 6. **Tips for Production**

- Use environment variables or a config file for settings.
- Add logging and error handling as needed.
- For very high throughput, consider running multiple consumer processes.

---