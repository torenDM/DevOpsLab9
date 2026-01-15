import time
from flask import Flask
import redis
from prometheus_client import Counter, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)
cache = redis.Redis(host="redis", port=6379)

view_count = Counter("view_count", "Flask-Redis-App visit counter", ["service"])
SERVICE = "Flask-Redis-App"

def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr("hits")
        except redis.exceptions.ConnectionError:
            if retries == 0:
                raise
            retries -= 1
            time.sleep(1)

@app.route("/")
def hello():
    hits = get_hit_count()
    view_count.labels(service=SERVICE).inc()
    return f"Hello from DevOps Lab 8! Visits: {hits}\n"

@app.route("/metrics")
def metrics():
    data = generate_latest()
    return data, 200, {"Content-Type": CONTENT_TYPE_LATEST}

