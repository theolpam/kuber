# Application FastAPI avec Redis

Ce projet configure une application FastAPI interagissant avec Redis à l'aide de Docker et Kubernetes.

## Contenu

- [Dockerfile](#dockerfile)
- [App.py](#apppy)
- [Deps.txt](#depstxt)
- [Deployment FastAPI](#deployment-fastapi)
- [Deployment Redis](#deployment-redis)
- [Service FastAPI](#service-fastapi)

## Dockerfile

Le Dockerfile pour créer l'image Docker de l'application.

```Dockerfile
## Utiliser l'image de base Python
FROM python:3.11-slim

## Définir le répertoire de travail dans le conteneur
WORKDIR /myapp

## Copier les fichiers locaux app.py et deps.txt dans le répertoire du conteneur /myapp
COPY app.py deps.txt /myapp/

## Installer les dépendances
RUN pip install --no-cache-dir -r deps.txt

## Exposer le port 8080 vers l'extérieur
EXPOSE 8080

## Commande pour exécuter l'application
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8080"]
```

## App.py

Le fichier principal de l'application.

```python
from fastapi import FastAPI
import redis
import os

app = FastAPI()

REDIS_SERVER = os.getenv("REDIS_SERVER", "localhost")
redis_conn = redis.StrictRedis(host=REDIS_SERVER, port=6379, db=0)

@app.get("/")
def health_check():
    return {"status": "OK"}

@app.get("/redis_data/{key}")
def get_data(key: str):
    value = redis_conn.get(key)
    if value:
        return {"key": key, "value": value.decode('utf-8')}
    else:
        return {"key": key, "value": "Key not found"}

@app.post("/set_data/{key}/{value}")
def set_data(key: str, value: str):
    redis_conn.set(key, value)
    return {"message": "Data set successfully", "key": key, "value": value}
```

## Deps.txt

Les dépendances de l'application.

```plaintext
fastapi==0.95
redis==5.0.1
uvicorn==0.24.0
```

## Deployment FastAPI

Configuration Kubernetes pour le déploiement de l'application FastAPI.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-fastapi-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp-fastapi
  template:
    metadata:
      labels:
        app: myapp-fastapi
    spec:
      containers:
      - name: myapp-fastapi
        image: myrepo/python-redis:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-fastapi-service
spec:
  selector:
    app: myapp-fastapi
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
```

## Deployment Redis

Configuration Kubernetes pour le déploiement de Redis.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-redis-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp-redis
  template:
    metadata:
      labels:
        app: myapp-redis
    spec:
      containers:
      - name: myapp-redis
        image: redis:alpine
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-redis-service
spec:
  selector:
    app: myapp-redis
  ports:
  - port: 6379
    targetPort: 6379
  type: ClusterIP
```

## Service FastAPI

Configuration Kubernetes pour le service de l'application FastAPI.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-fastapi-service
spec:
  selector:
    app: myapp-fastapi
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
```

---
