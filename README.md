# Async Task Processing System with AWS Deployment

This project demonstrates a production-ready asynchronous task processing system (i.e Deployed in EC2 using Pulumi explained in infra-branch) using: 

 **Flask** as (Web + API), **Celery** as (Task Queue Executor) , **RabbitMQ** as (Message Broker) , **Redis** as (Result Backend) , **Flower** as (Monitoring UI) , **Docker** as (Multi-service orchestration)

>  **Objective**: Design a system that allows users to submit tasks asynchronously from a Flask API, execute them reliably in the background using Celery, queue tasks in RabbitMQ, and store task states and results in a backend db (redis).

---

##  System Architecture :
<img src="assets/Try-Page.svg" alt="Implementation Diagram" width="1000">

***1. User Request via API :***
 user interacts with the frontend UI to trigger a task—such as sending an email, reversing a text, or analyzing sentiment. This action sends an HTTP request to the Flask backend through a specific API endpoint.

***2.  Flask App: Receiving & Dispatching***
 Receives request and pushes task to Celery using delay().Flask responds to the user immediately with a confirmation message and a task_id, ensuring the experience remains fast and non-blocking.


***3. Celery as the Producer***
When .delay() is called, Celery acts as a task producer—serializing the task and sending it to the RabbitMQ message broker.

***4. RabbitMQ: Message Broker Layer***
RabbitMQ receives the serialized task and places it in a queue (commonly named celery_see).plays the role of a message router, managing queues and delivering tasks to any available consumer (Celery workers). It ensures decoupling between producers (Flask) and consumers (workers), allowing each part to scale independently.In these case:

<div align="center">
  <img src="assets/queue.svg" alt="Broker Diagram" width="700">
</div>

* Consumer = Celery Worker

* Exchange = Implicit direct exchange

* Queue = celery_see (task queue)


***5. Celery Workers: Task Execution Engine***

Celery workers continuously listen for tasks on the RabbitMQ queue. When a task becomes available, a worker pulls it, executes the defined function (like sending an email or reversing a string), and processes it in the background. With --concurrency enabled, workers can process multiple tasks in parallel, significantly improving throughput and responsiveness.



***6. Redis: Result Tracking Backend***

After a worker finishes processing a task, it stores the result and status (SUCCESS, FAILURE, or RETRY) in Redis. Redis serves as a fast, in-memory database that holds the task metadata under unique keys tied to the task_id. Flask can later query Redis using AsyncResult(task_id) to retrieve this data and update the user.

***7. UI Feedback:***

Finally, the frontend periodically polls the backend using the task_id to check the task’s status. Once Redis indicates that the task is complete, Flask fetches the result and returns it to the UI. The user sees a confirmation message or the processed output (e.g., reversed string or sentiment result). This feedback loop ensures the user is kept informed without blocking the main thread.


## Client Interaction Flow

<img src="assets/Flow.svg" alt="Implementation Diagram" width="1000">



###  docker-compose.yml is  optimized with profiles so each instance only spins up its assigned service:(Use profiles to isolate services)

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

* Direct exchange with `celery_see` queue
* Multi-worker concurrency with `--concurrency=4`
* Flask session flash for notifications
* Redis result tracking via `AsyncResult`
* Queue retry using `self.retry()`





## License

MIT License © 2025 [poridhi.io](https://poridhi.io)
