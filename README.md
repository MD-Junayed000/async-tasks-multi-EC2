# Async Task Processing System with Flask, Celery, RabbitMQ & Pulumi-Driven AWS Deployment in Multi-EC2 Instances

This project demonstrates a production-ready asynchronous task processing system (i.e Deployed in EC2 using Pulumi explained in infra-branch) using:

* ✅ Flask (Web + API)
* ✅ Celery (Task Queue Executor)
* ✅ RabbitMQ (Message Broker)
* ✅ Redis (Result Backend)
* ✅ Flower (Monitoring UI)
* ✅ Docker (Multi-service orchestration)


>  **Objective**: Design a system that allows users to submit tasks asynchronously from a Flask API, execute them reliably in the background using Celery, queue tasks in RabbitMQ, and store task states and results in a backend db (redis).

---

##  System Architecture :
<img src="assets/implement.svg" alt="Implementation Diagram" width="1000">

***1. User Request via API :***

→ Sends request from UI (email, text reverse, sentiment).

***2. Flask App :***

→ Receives request and pushes task to Celery using delay()

→ Immediately responds to user (non-blocking).

***3. Celery Producer Role (📩 Task Sent to Broker):***

→  When .delay() is called in Flask,  celery sends the task to RabbitMQ (the message broker).

→ So, Flask acts as the Producer in this diagram.

***4. RabbitMQ Queue (Message Broker):***

→ Holds tasks in queue(s).

→ Forwards them to Celery Worker.

***5. Celery Workers:***

→ Continuously listens to RabbitMQ.

→ Pulls and executes tasks (email/text/sentiment).

→ Can run multiple processes (parallel).



***6. Redis (Result Backend):***

→ Stores task result/status (e.g., SUCCESS, FAILURE).

→ Flask can query result using ***AsyncResult.***

***7. UI Feedback:***

→ Task ID flashed in UI.

→ Success/failure notification shown based on result from Redis.

---
## Project Features :
* 📨 Async Email Sender with retry logic

* 🔁 Reverse Text Processor * 💬 Fake Sentiment Analyzer

* 🔁 Redis-based task result storage

* 📊 Live task monitoring via Flower

* 🧪 Task inspection via Redis

* 🖥️ UI with feedback using Flask + Bootstrap

* 🐳 Docker-based deployment


### 🔧 Components :

| Component    | Role                                 |
| ------------ | ------------------------------------ |
| **Flask**    | UI, task submission, status fetch    |
| **Celery**   | Task execution engine                |
| **RabbitMQ** | Message broker to queue tasks        |
| **Redis**    | Stores task status & results         |
| **Flower**   | Real-time task monitoring dashboard  |
| **Docker**   | Container orchestration for services |




###  Queues and Exchange in RabbitMQ

<img src="assets/Broker.svg" alt="Broker Diagram" width="700">

* Celery uses a direct exchange

* Tasks routed by name → bound to celery_see queue

* Each worker listens on that queue

>> In RabbitMQ:

* Producer = Flask (via Celery)

* Consumer = Celery Worker

* Exchange = Implicit direct exchange

* Queue = celery_see (task queue)

---

### Here, docker-compose.yml is  optimized with profiles so each instance only spins up its assigned service:(Use profiles to isolate services)

```bash
version: "3.8"
services:
  flask:
    build: .
    profiles: ["flask"]
    volumes:
      - .:/code
    ports:
      - "5000:5000"
    depends_on:
      - rabbitmq
      - redis

  rabbitmq:
    image: rabbitmq:3-management
    profiles: ["rabbitmq"]
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

  redis:
    image: redis
    profiles: ["redis"]
    ports:
      - "6379:6379"

  celery:
    build: .
    profiles: ["celery"]
    depends_on:
      rabbitmq:
        condition: service_healthy
    restart: always
    command: >
      sh -c "sleep 5 && celery -A app.tasks worker -Q celery_see --loglevel=info --concurrency=4"

  flower:
    build: .
    profiles: ["flower"]
    ports:
      - "5555:5555"
    depends_on:
      rabbitmq:
        condition: service_healthy
    restart: always
    command: >
      sh -c "sleep 10 && celery -A app.tasks flower --port=5555"

```


### 🔗 Celery Configuration (`async-tasks/app/celeryconfig.py`)

```python
# Multi-Instance Config
broker_url = 'amqp://guest:guest@<BROKER_IP>:5672//'
result_backend = 'redis://<REDIS_IP>:6379/0'

task_serializer = 'json'
result_serializer = 'json'
accept_content = ['json']
timezone = 'Asia/Dhaka'
task_default_queue = 'celery_see'
```

This is automatically patched in each instance during boot with correct IPs of RabbitMQ and Redis.

---



---



## 📁 Project Structure

```
async-tasks/
│
├── app/
│   ├── __init__.py          # Flask app factory
│   ├── celeryconfig.py      # Celery broker/backend configs
│   ├── routes.py            # Routes for UI + task handling
│   ├── tasks.py             # Celery tasks
│   ├── templates/           # Jinja2 HTML UI (index.html)
│   └── static/              # CSS, images, icons
│
├── Dockerfile               # Docker image setup
├── docker-compose.yml       # Multi-service Docker config
├── requirements.txt         # Python dependencies
├── run.py                   # Flask run entry
├── start.sh / start.ps1     # Quick Docker starter script

```

---






##  Concepts Implemented

* ✅ Direct exchange with `celery_see` queue
* ✅ Multi-worker concurrency with `--concurrency=4`
* ✅ Flask session flash for notifications
* ✅ Redis result tracking via `AsyncResult`
* ✅ Queue retry using `self.retry()`





## License

MIT License © 2025 [poridhi.io](https://poridhi.io)
