# Exercise 1 – Hello Pod

## Objective

Deploy an Nginx container as a Kubernetes Pod using Minikube, expose it through a NodePort Service, and access the application via a web browser.

## Environment

* OS: Windows 11
* Kubernetes: v1.35.1
* Minikube: v1.38.1
* Container Runtime: Docker
* Application: Nginx

## 1. Start Minikube

Launch the local Kubernetes cluster:

```bash
minikube start
```

Minikube started successfully with Docker as the container driver, and `kubectl` was automatically configured to point to the Minikube cluster.

## 2. Create the Pod

Create a Pod named `hello-k8s` running the Nginx image:

```bash
kubectl run hello-k8s --image=nginx --port=80
```

Output:

```text
pod/hello-k8s created
```

## 3. Verify the Pod

Check the Pod's status:

```bash
kubectl get pods
```

The Pod reached the `Running` state successfully:

```text
NAME        READY   STATUS    RESTARTS
hello-k8s   1/1     Running   0
```

The `1/1` value confirms that the Nginx container is ready and running.

## 4. Expose the Pod

Expose the Pod via a NodePort Service:

```bash
kubectl expose pod hello-k8s --type=NodePort --port=80
```

Output:

```text
service/hello-k8s exposed
```

The Pod is now reachable through the Kubernetes Service.

## 5. Access the Application

Launch the Service through Minikube:

```bash
minikube service hello-k8s
```

Minikube set up a tunnel to the Service and opened the application directly in the browser.

The Nginx welcome page will load successfully.

## 6. How Kubernetes Handled This

The overall flow for creating the Pod looked like this:

```text
kubectl
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
Container Runtime
   ↓
Nginx Container
```

* `kubectl` submits the Pod creation request to the Kubernetes API Server.
* `etcd` stores and maintains the cluster's state.
* The Scheduler assigns the Pod to a suitable node.
* The Kubelet on that node makes sure the Pod stays running.
* The container runtime spins up the Nginx container.
* Nginx runs and listens on port `80`.
* The NodePort Service exposes the Pod for external access.

## 7. Commands Used

```bash
minikube start

kubectl run hello-k8s --image=nginx --port=80

kubectl get pods

kubectl expose pod hello-k8s --type=NodePort --port=80

minikube service hello-k8s
```

## 8. Result

The Nginx application was deployed successfully as a Kubernetes Pod on Minikube.

The Pod reached the `Running` state, was exposed through a NodePort Service, and the Nginx welcome page was accessed successfully in the browser.