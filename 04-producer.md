Absolutely! Here’s a **simple, robust Python RabbitMQ producer** that sends JSON messages to a queue. This example is easy to adapt and can be run as a script or imported as a function.

---

## 1. **Install Dependencies**

```bash
pip install pika
```

---

## 2. **Producer Example**

```python
import pika
import json
import time

# --- CONFIGURABLE PARAMETERS ---
RABBITMQ_URL = "amqp://user:pass@localhost:5672/"
QUEUE_NAME = "my_queue"

def send_message(data):
    """Send a single message to RabbitMQ."""
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
    # Example: send 100 messages
    for i in range(100):
        msg = {"id": i, "value": f"value_{i}"}
        send_message(msg)
        print(f"Sent: {msg}")
        time.sleep(0.01)  # Optional: slow down for demonstration
```

---

## 3. **How it Works**

- **Connects** to RabbitMQ using the provided URL.
- **Declares** the queue (safe to call repeatedly).
- **Sends** a JSON-encoded message with `delivery_mode=2` (persistent).
- **Closes** the connection after each message (for high throughput, you can keep the connection open and send many messages in a loop).

---

## 4. **Tips for Production**

- For **high throughput**, open the connection once, send all messages, then close.
- Use **environment variables** for configuration in real deployments.
- Add **error handling** as needed.

---

## 5. **Reusable Producer Class (Optional)**

If you want to send many messages efficiently:

```python
class RabbitProducer:
    def __init__(self, url, queue):
        self.connection = pika.BlockingConnection(pika.URLParameters(url))
        self.channel = self.connection.channel()
        self.channel.queue_declare(queue=queue, durable=True)
        self.queue = queue

    def send(self, data):
        self.channel.basic_publish(
            exchange='',
            routing_key=self.queue,
            body=json.dumps(data),
            properties=pika.BasicProperties(delivery_mode=2)
        )

    def close(self):
        self.connection.close()

if __name__ == "__main__":
    producer = RabbitProducer(RABBITMQ_URL, QUEUE_NAME)
    for i in range(100):
        producer.send({"id": i, "value": f"value_{i}"})
        print(f"Sent: {i}")
    producer.close()
```

---