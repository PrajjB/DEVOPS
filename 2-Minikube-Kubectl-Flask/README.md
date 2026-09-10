# Exercise 2 – Flask App on Minikube

## Objective

Deploy a Python Flask application on Minikube by building a Docker image, creating a Kubernetes Deployment, exposing it through a NodePort Service, and accessing the application from the terminal.

## Environment

* OS: Windows 11 (WSL / Linux terminal for Docker driver commands)
* Kubernetes: v1.35.1
* Minikube: v1.38.1
* Container Runtime: Docker
* Application: Flask (Python 3.8)

## 1. Start Minikube

Start the local Kubernetes cluster:

```bash
minikube start
```

Minikube started successfully with Docker as the container driver, and `kubectl` was configured to use the Minikube cluster.

## 2. Create the Flask Application

Created `app.py`:

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def home():
    return "Hello from Flask on Kubernetes!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=15000)
```

## 3. Create the Dockerfile

Created `Dockerfile`:

```dockerfile
FROM python:3.8-slim
WORKDIR /app
COPY . /app
RUN pip install flask
CMD ["python", "app.py"]
```

## 4. Build the Docker Image Using Minikube's Docker Daemon

Pointed the local Docker CLI to Minikube's internal Docker daemon so the image is built directly inside the cluster:

```bash
eval $(minikube docker-env)
docker build -t flask-app .
```

The image `flask-app:latest` was built successfully inside Minikube's Docker environment.

## 5. Create the Kubernetes Deployment YAML

Created `flask-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-app
        image: flask-app:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 15000
```

`imagePullPolicy: Never` was used so Kubernetes uses the locally built image instead of trying to pull it from a remote registry.

## 6. Deploy the Application

```bash
kubectl apply -f flask-deployment.yaml
```

Output:

```text
deployment.apps/flask-app created
```

## 7. Verify the Deployment

```bash
kubectl get deployments
```

Output:

```text
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
flask-app    1/1     1            1           5s
```

The deployment reached the desired state with 1/1 replicas available.

## 8. Verify the Pods

```bash
kubectl get pods -l app=flask-app
```

Output:

```text
NAME                          READY   STATUS    RESTARTS   AGE
flask-app-3443a-213a          1/1     Running   0          5s
```

The Pod created by the Deployment reached the `Running` state.

## 9. Describe the Deployment

```bash
kubectl describe deployment flask-app
```

Output confirmed 1 desired / 1 available replica, `RollingUpdate` strategy, and a `ScalingReplicaSet` event showing the new ReplicaSet was scaled up successfully.

## 10. Check the Application Logs

```bash
kubectl logs flask-app-b8cd75b6f-tpdpr
```

Output:

```text
 * Serving Flask app 'app'
 * Debug mode: off
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:15000
 * Running on http://10.0.0.210:15000
Press CTRL+C to quit
```

The logs confirmed Flask was running and listening on port `15000` inside the container.

## 11. Check Existing Services

```bash
kubectl get services
```

Output:

```text
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   7d11h
```

No service existed yet for the Flask app, so the app could not be reached externally.

## 12. First Access Attempt (Failed)

```bash
curl http://127.0.0.1:15000
```

Output:

```text
curl: (7) Failed to connect to 127.0.0.1 port 15000 after 1 ms: Couldn't connect to server
```

This failed because Minikube runs in a virtual environment, and port `15000` was only exposed inside the container — not on the host.

## 13. Add a Service to the Deployment YAML

Updated `flask-deployment.yaml` to include a Service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-app
        image: flask-app:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 15000
---
apiVersion: v1
kind: Service
metadata:
  name: flask-app-service
spec:
  selector:
    app: flask-app
  ports:
  - port: 15000
    targetPort: 15000
  type: NodePort
```

Applied the updated file:

```bash
kubectl apply -f flask-deployment.yaml
```

Output:

```text
deployment.apps/flask-app unchanged
service/flask-app-service created
```

The mapping created was: **External Request (port 15000) → Service (port 15000) → Container (port 15000)**.

## 14. Get the Service URL

```bash
minikube service flask-app-service --url
```

Output:

```text
http://127.0.0.1:36157
❗  Because you are using a Docker driver on linux, the terminal needs to be open to run it.
```

The terminal was kept open to maintain the tunnel to the service.

## 15. Access the Application

In a new terminal window:

```bash
curl http://127.0.0.1:36157
```

Output:

```text
Hello from Flask on Kubernetes!
```

The Flask application was successfully accessed through the Minikube service tunnel.

## 16. Kubernetes Working

The flow for this exercise was:

```text
kubectl apply
   ↓
Kubernetes API Server
   ↓
etcd
   ↓
Scheduler
   ↓
Worker Node
   ↓
Kubelet
   ↓
Docker (Minikube's daemon)
   ↓
Flask Container
   ↓
NodePort Service
   ↓
minikube service --url (tunnel)
```

* `kubectl apply` submitted the Deployment and Service manifests to the API Server.
* `etcd` stored the desired cluster state.
* The Scheduler placed the Pod on the node.
* The Kubelet ensured the container stayed running.
* Docker (via Minikube's daemon) ran the locally built `flask-app` image.
* The NodePort Service exposed port `15000` from the container.
* `minikube service --url` created a tunnel from the host to the Service.

## 17. Commands Used

```bash
minikube start

eval $(minikube docker-env)
docker build -t flask-app .

kubectl apply -f flask-deployment.yaml

kubectl get deployments

kubectl get pods -l app=flask-app

kubectl describe deployment flask-app

kubectl logs flask-app-b8cd75b6f-tpdpr

kubectl get services

curl http://127.0.0.1:15000

kubectl apply -f flask-deployment.yaml

minikube service flask-app-service --url

curl http://127.0.0.1:36157
```

## 18. Result

The Flask application was successfully deployed on Minikube using a Deployment built from a locally built Docker image.

The Deployment reached `1/1` availability, the Pod ran successfully, and after adding a NodePort Service, the app was reachable through `minikube service --url` and confirmed working with `curl`, returning `Hello from Flask on Kubernetes!`.