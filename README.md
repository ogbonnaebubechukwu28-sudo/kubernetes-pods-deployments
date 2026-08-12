# Kubernetes Pods & Deployments with Minikube

### Student Information

Name: Ebubechukwu Ogbonna

---

## 1. Minikube Setup

I started a local Kubernetes cluster with:

```bash
minikube start
minikube status
```

This confirmed the control plane, kubelet, and API server were all running, with kubeconfig configured.

I then verified the node with:

```bash
kubectl get nodes
kubectl cluster-info
```

The node `minikube` was `Ready`, running Kubernetes version `v1.31.0`, with the control plane and CoreDNS reachable at `https://127.0.0.1:32776`.

Minikube's purpose is to run a lightweight, single-node Kubernetes cluster locally so you can develop and test Kubernetes workloads without needing a full cloud-based cluster.

## 2. Pod

I created a standalone Pod with:

```bash
kubectl run nginx-pod --image=nginx:latest
kubectl get pods
kubectl get pods -o wide
```

Once the `nginx:latest` image finished pulling, the Pod reached `Running` status with IP `10.244.0.3` on node `minikube`.

I inspected it with:

```bash
kubectl describe pod nginx-pod
kubectl get pod nginx-pod -o yaml
```

`kubectl describe` gave a detailed view of the Pod's spec, status, conditions, and event timeline (scheduling, image pull, container start) — the main tool for troubleshooting.

## 3. Deployment

I created `manifests/deployment.yaml` defining a Deployment named `web-app`, running a container named `nginx` from image `nginx:latest`, with 3 initial replicas.

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods
kubectl get pods -o wide
```

This created 3 Pods managed by a ReplicaSet (`web-app-c95765fd4`), unlike the standalone `nginx-pod`, which has no controller to recreate it if it fails. Using a Deployment means Kubernetes handles self-healing, scaling, and rolling updates automatically instead of me managing individual Pods by hand.

Running `kubectl get deployments` / `replicasets` / `pods` confirmed the hierarchy:

```
Deployment (web-app)
   ↓
ReplicaSet (web-app-c95765fd4)
   ↓
3 Pods
```

The ReplicaSet is the object that directly maintains the desired Pod count; if a Pod is deleted, the ReplicaSet creates a replacement.

## 4. Scaling

Starting from 3 replicas, I scaled up:

```bash
kubectl scale deployment web-app --replicas=5
kubectl get deployment
kubectl get pods
```

Then scaled down:

```bash
kubectl scale deployment web-app --replicas=2
kubectl get deployment
kubectl get pods
```

Pod count went **3 → 5 → 2**, confirmed each time with `kubectl get pods`. Scaling a Deployment is far easier than creating/deleting Pods manually — one command declares the desired count, and the ReplicaSet handles creating or terminating the exact Pods needed.

## 5. Self-Healing

With the Deployment at 2 replicas, I deleted one Pod directly:

```bash
kubectl delete pod web-app-c95765fd4-48fw2
kubectl get pods
```

Kubernetes immediately detected the Pod count had dropped below the desired state and created a new Pod (`web-app-c95765fd4-85hdp`) to restore it to 2/2 — without any manual action from me. This demonstrates **self-healing / desired-state reconciliation**: the ReplicaSet continuously reconciles the actual cluster state against the declared desired state.

## 6. Troubleshooting

I intentionally broke the Deployment by changing the image to an invalid tag:

```yaml
image: nginx:invalid
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods
kubectl describe pod <new-pod-name>
```

The new Pods showed status `ErrImagePull`, cycling to `ImagePullBackOff`. The Pod's events showed the exact cause:

```
Failed to pull image "nginx:invalid": Error response from daemon:
manifest for nginx:invalid not found: manifest unknown: manifest unknown
```

The `invalid` tag doesn't exist for the `nginx` image on Docker Hub, so the container runtime can't resolve or pull it — the container never gets created. I fixed this by restoring the image to `nginx:latest` and reapplying:

```bash
kubectl apply -f deployment.yaml
kubectl get pods
```

This rolled the Deployment back to a healthy state with all Pods running on the valid image.

---

## Clean Up

```bash
kubectl delete deployment web-app
kubectl delete pod nginx-pod
kubectl get pods
minikube stop
```
