# Quotes App on OpenShift with PostgreSQL

A complete beginner tutorial to deploy a 3-tier web application on AppuIO OpenShift:
- **Frontend** — React app that displays quotes
- **Backend** — Python REST API
- **Database** — PostgreSQL

---

## What you need

- AppuIO account with `oc` CLI logged in
- Docker Desktop
- Git + GitHub account
- GitHub Container Registry access (ghcr.io)

---

## Step 1 — Clone the repos

```bash
mkdir quotes-app && cd quotes-app
git clone https://github.com/redhat-developer-demos/quotesweb.git
git clone https://github.com/redhat-developer-demos/qotd-python.git
```

---

## Step 2 — Set up the project folder

Create a `quotes-postgres` folder and copy the files from the cloned repo:

```bash
mkdir quotes-postgres
mkdir quotes-postgres/k8s
cp qotd-python/Dockerfile quotes-postgres/Dockerfile
cp qotd-python/app.py quotes-postgres/app.py
```

All files from here on go into `quotes-postgres/`.

---

## Step 3 — Update the Python backend for PostgreSQL

### 3.1 Replace the contents of `quotes-postgres/app.py` (was copied in Step 2)

```python
import flask
from flask import jsonify, make_response
from flask_cors import CORS
import random
import socket
import os
import psycopg2

app = flask.Flask(__name__)
CORS(app)

def get_db():
    return psycopg2.connect(
        host=os.environ.get("DB_SERVICE_NAME"),
        database=os.environ.get("DB_NAME"),
        user=os.environ.get("DB_USER"),
        password=os.environ.get("DB_PASSWORD")
    )


def fetch_quotes():
    conn = get_db()
    cur = conn.cursor()
    cur.execute("SELECT id, quotation, author FROM quotes")
    rows = cur.fetchall()
    cur.close()
    conn.close()
    h = socket.gethostname()
    return [{"id": r[0], "quotation": r[1], "author": r[2], "hostname": h} for r in rows]

@app.route("/")
def home():
    return make_response("qotd-postgres", 200)

@app.route("/version")
def version():
    return make_response("v1", 200)

@app.route("/quotes")
def getQuotes():
    return jsonify(fetch_quotes())

@app.route("/quotes/random")
def getRandom():
    return jsonify(random.choice(fetch_quotes()))

@app.route("/quotes/<int:id>")
def getById(id):
    conn = get_db()
    cur = conn.cursor()
    cur.execute("SELECT id, quotation, author FROM quotes WHERE id = %s", (id,))
    r = cur.fetchone()
    cur.close()
    conn.close()
    return jsonify({"id": r[0], "quotation": r[1], "author": r[2], "hostname": socket.gethostname()})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=10000)
```

### 3.2 `quotes-postgres/requirements.txt`

```
flask>=2.2.2
flask-cors
gunicorn==20.0.4
psycopg2-binary
```

### 3.3 Build and push

```bash
cd quotes-postgres
docker build -t ghcr.io/<your-github-username>/quotes-postgres:v1 .
docker login ghcr.io -u <your-github-username>
docker push ghcr.io/<your-github-username>/quotes-postgres:v1
```

---

## Step 4 — Deploy the backend on OpenShift

Create `quotes-postgres/k8s/quotes-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quotes
spec:
  replicas: 1
  selector:
    matchLabels:
      app: quotes
  template:
    metadata:
      labels:
        app: quotes
        app.kubernetes.io/part-of: quotes-app
    spec:
      imagePullSecrets:
        - name: ghcr-secret
      containers:
        - name: quotes
          image: ghcr.io/<your-github-username>/quotes-postgres:latest
          ports:
            - containerPort: 10000
          env:
            - name: DB_SERVICE_NAME
              value: postgres
            - name: DB_NAME
              value: quotesdb
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
```

Create a secret to pull the image from ghcr.io (skip if you already have a `ghcr-secret` from a previous deployment):

```bash
oc create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<your-github-username> \
  --docker-password=<your-github-PAT>
```

Apply the deployment:

```bash
cd quotes-postgres/k8s
oc apply -f quotes-deployment.yaml
oc create service clusterip quotes --tcp=10000:10000
oc create route edge quotes --service=quotes --port=10000
```

---

## Step 5 — Deploy the frontend

```bash
oc create -f quotesweb-deployment.yaml
oc create -f quotesweb-service.yaml
oc create route edge quotesweb --service=quotesweb --port=3000-tcp
oc label deployment quotesweb app.kubernetes.io/part-of=quotes-app
```

---

## Step 6 — Deploy PostgreSQL

### 6.1 `quotes-postgres/k8s/postgres-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-volume
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

### 6.2 `quotes-postgres/k8s/postgres-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
data:
  POSTGRES_PASSWORD: YWRtaW4=   # admin
  POSTGRES_USER: YWRtaW4=       # admin
  POSTGRES_DB: cXVvdGVzZGI=     # quotesdb
```

> To encode your own values: `echo -n "yourvalue" | base64`

### 6.3 `quotes-postgres/k8s/postgres-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        app.kubernetes.io/part-of: quotes-app
    spec:
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-volume
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_USER
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_DB
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
```

### 6.4 Apply everything

```bash
cd quotes-postgres/k8s
oc apply -f postgres-pvc.yaml
oc apply -f postgres-secret.yaml
oc apply -f postgres-deployment.yaml
oc get pods   # wait until postgres pod is Running
kubectl expose deployment postgres --port=5432
```

---

## Step 7 — Create the table and load data

### 7.1 Create the table

```bash
cd quotes-postgres/k8s
kubectl exec deploy/postgres -- psql -U admin -d quotesdb -c "
CREATE TABLE quotes (
  id SERIAL PRIMARY KEY,
  quotation VARCHAR(1024),
  author VARCHAR(255)
);"
```

### 7.2 Load quotes

Create a file `quotes.csv` in `quotes-postgres/k8s/` with pipe-separated values:

```
1|Knowledge is power.|Francis Bacon
2|Life is really simple, but we insist on making it complicated.|Confucius
3|This above all, to thine own self be true.|William Shakespeare
4|Anyone who has ever made anything of importance was disciplined.|Andrew Hendrixson
5|The only way to do great work is to love what you do.|Steve Jobs
```

Copy and import:

```bash
cd quotes-postgres/k8s
export PODNAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}' | grep postgres)

kubectl cp ./quotes.csv $PODNAME:/tmp/quotes.csv

kubectl exec deploy/postgres -- psql -U admin -d quotesdb -c "
COPY quotes(id, quotation, author)
FROM '/tmp/quotes.csv'
DELIMITER '|';"
```

Verify:

```bash
kubectl exec deploy/postgres -- psql -U admin -d quotesdb -c "SELECT COUNT(*) FROM quotes;"
```

---

## Step 8 — Test the app

Open the frontend URL:

```
https://quotesweb-<namespace>.apps.exoscale-ch-gva-2-0.appuio.cloud
```

Enter the backend URL in the text box:

```
https://quotes-<namespace>.apps.exoscale-ch-gva-2-0.appuio.cloud/quotes/random
```

Click **Start** — quotes from PostgreSQL should appear.

---

## Step 9 — Scale the backend

```bash
kubectl scale deployment quotes --replicas=3
kubectl get pods
```

The hostname in the quote response will rotate between the 3 pods, showing load balancing.

