## 1. **Install Requests**

```bash
pip install requests
```

---

## 2. **Utility Module: `rabbitmq_stats.py`**

Place this in your Django app (e.g., `myapp/utils/rabbitmq_stats.py`):

```python
import requests
from requests.auth import HTTPBasicAuth

# --- CONFIGURABLE PARAMETERS ---
RABBITMQ_API_URL = "http://localhost:15672/api"
RABBITMQ_USER = "user"
RABBITMQ_PASS = "pass"
VHOST = "/"  # Change if you use a different vhost

def get_queue_stats(queue_name):
    """
    Returns stats for a specific RabbitMQ queue.
    """
    url = f"{RABBITMQ_API_URL}/queues/{requests.utils.quote(VHOST, safe='')}/{queue_name}"
    resp = requests.get(url, auth=HTTPBasicAuth(RABBITMQ_USER, RABBITMQ_PASS))
    if resp.status_code != 200:
        raise Exception(f"Failed to get queue stats: {resp.status_code} {resp.text}")
    data = resp.json()
    return {
        "messages_ready": data.get("messages_ready"),  # messages waiting to be delivered
        "messages_unacknowledged": data.get("messages_unacknowledged"),  # delivered but not acked
        "messages_total": data.get("messages"),  # total messages in queue
        "consumers": data.get("consumers"),
        "message_stats": data.get("message_stats", {}),
        "state": data.get("state"),
        "name": data.get("name"),
        "vhost": data.get("vhost"),
    }

def get_overview():
    """
    Returns general RabbitMQ stats.
    """
    url = f"{RABBITMQ_API_URL}/overview"
    resp = requests.get(url, auth=HTTPBasicAuth(RABBITMQ_USER, RABBITMQ_PASS))
    if resp.status_code != 200:
        raise Exception(f"Failed to get overview: {resp.status_code} {resp.text}")
    return resp.json()
```

---

## 3. **Sample Django View**

Here’s a simple Django view that uses the above utility to display queue stats:

```python
# myapp/views.py
from django.http import JsonResponse
from .utils.rabbitmq_stats import get_queue_stats, get_overview

def rabbitmq_queue_stats(request, queue_name):
    try:
        stats = get_queue_stats(queue_name)
        return JsonResponse(stats)
    except Exception as e:
        return JsonResponse({"error": str(e)}, status=500)

def rabbitmq_overview(request):
    try:
        stats = get_overview()
        return JsonResponse(stats)
    except Exception as e:
        return JsonResponse({"error": str(e)}, status=500)
```

---

## 4. **Sample Django URL Patterns**

```python
# myapp/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path("rabbitmq/queue/<str:queue_name>/", views.rabbitmq_queue_stats, name="rabbitmq_queue_stats"),
    path("rabbitmq/overview/", views.rabbitmq_overview, name="rabbitmq_overview"),
]
```

---

## 5. **What Metrics Can You Visualize?**

- **messages_ready**: Messages waiting to be delivered to consumers.
- **messages_unacknowledged**: Messages delivered but not yet acknowledged.
- **messages_total**: Total messages in the queue.
- **consumers**: Number of consumers attached to the queue.
- **message_stats**: Includes counts for `publish`, `deliver_get`, `ack`, etc.
- **state**: Queue state (e.g., running, idle).
- **overview**: System-wide stats (connections, channels, queues, message rates, etc).

---

## 6. **Security Note**

- The management API must be enabled (`rabbitmq:3-management` image does this by default).
- Make sure the API is not exposed to the public internet without authentication.

---

## 7. **Example Output**

A call to `/rabbitmq/queue/my_queue/` might return:

```json
{
  "messages_ready": 10,
  "messages_unacknowledged": 2,
  "messages_total": 12,
  "consumers": 1,
  "message_stats": {
    "publish": 1000,
    "deliver_get": 998,
    "ack": 996
  },
  "state": "running",
  "name": "my_queue",
  "vhost": "/"
}
```

---