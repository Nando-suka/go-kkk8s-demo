# go-k8s-demo

A tiny Go HTTP API deployed to a local Kubernetes cluster using [Kind](https://kind.sigs.k8s.io/). Built as a hands-on playground for learning the **Go → Docker → Kubernetes** workflow, then automating it with GitHub Actions.

Every request returns JSON with the pod's hostname, an app version, and the current time. Because the hostname changes between replicas, you can actually see Kubernetes load-balancing traffic in front of your eyes.

## What you need

- **Go** 1.25 or newer
- **Docker** (running)
- **Kind** — install with `go install sigs.k8s.io/kind@v0.33.0`
- **kubectl**

> **Windows users:** Use **Git Bash** or **WSL**. Plain PowerShell works too, but the commands below assume a Unix-like shell.

---

## Run it on your machine

### 1. Build the image

```bash
docker build -t go-k8s-demo:v1 .
```

### 2. Create a local cluster

```bash
kind create cluster --name go-demo
```

### 3. Load the image into Kind

```bash
kind load docker-image go-k8s-demo:v1 --name go-demo
```

### 4. Deploy

```bash
kubectl apply -f k8s/
kubectl rollout status deployment/go-k8s-demo
```

### 5. Open it

Find the `NodePort` Kubernetes assigned:

```bash
kubectl get svc go-k8s-demo
```

Look at the `PORT(S)` column — you'll see something like `80:31234/TCP`. That `31234` is your port:

```bash
curl http://localhost:31234
```

> **Alternative:** If `NodePort` doesn't work on your setup, use port-forwarding instead:
> ```bash
> kubectl port-forward svc/go-k8s-demo 8080:80
> ```
> Then visit `http://localhost:8080`.

You should get a response like:

```json
{
  "message": "Hello from Kubernetes!",
  "hostname": "go-k8s-demo-7c9f8b6d4-abc12",
  "version": "v1",
  "time": "2025-01-15T10:30:00Z"
}
```

---

## Try a rolling update

Build a `v2` image, load it into Kind, and update the deployment:

```bash
docker build -t go-k8s-demo:v2 .
kind load docker-image go-k8s-demo:v2 --name go-demo
kubectl set image deployment/go-k8s-demo app=go-k8s-demo:v2
kubectl set env deployment/go-k8s-demo APP_VERSION=v2
kubectl rollout status deployment/go-k8s-demo
```

While it's rolling out, keep calling `curl`. You'll see `v1` and `v2` responses mixed for a few seconds with zero failed requests — that's a zero-downtime deployment in action.

## Scale it

```bash
kubectl scale deployment/go-k8s-demo --replicas=4
kubectl get pods -w
```

Hit the API again — now four pods share the load.

---

## CI/CD

The GitHub Actions workflow in `.github/workflows/ci.yaml` runs on every push to `main`. It:

- Runs `go test ./...`
- Builds the Docker image
- Spins up an ephemeral Kind cluster inside the runner
- Loads the image into that cluster
- Applies the manifests and waits for the rollout to finish
- Curls `/healthcheck` to confirm the app is actually serving traffic

You can watch it under the **Actions** tab in your repo. The cluster is torn down when the job ends — no leftover resources, no cloud bill.

## Cleanup

When you're done experimenting:

```bash
kind delete cluster --name go-demo
docker rmi go-k8s-demo:v1 go-k8s-demo:v2
```

That's it. Everything this project created lives inside that Kind cluster.

---

## Troubleshooting

### `kind load docker-image` seems to hang on Git Bash
It's not hanging — the confirmation line just doesn't always print. If `kubectl rollout status` succeeds afterwards, the image was loaded fine.

### Pods stuck in `ImagePullBackOff`
You probably forgot to run `kind load docker-image`. Verify the image is inside the node with:

```bash
docker exec -it go-demo-control-plane crictl images | grep go-k8s-demo
```

### Rebuilding the same tag doesn't take effect
Kind caches images by tag. Either use a new tag (`v2`, `v3`, …) or remove the old image from the node first:

```bash
docker exec -it go-demo-control-plane crictl rmi go-k8s-demo:v2
kind load docker-image go-k8s-demo:v2 --name go-demo
```

### `curl` gives "connection refused" on the NodePort
Give it a few seconds after `kubectl apply` — the pods need to pass their readiness probe first. Check their status with `kubectl get pods`.

---

## Why this exists

This started as a learning project to get hands-on with Go, Docker, and Kubernetes. Kind makes a real Kubernetes cluster run on your laptop without VMs or cloud accounts, which is perfect for experimenting. If you're learning DevOps and want a small, complete example of the whole loop — write code, build image, deploy, update, scale, automate — this is a solid starting point.

Feel free to fork it and break things. That's what it's for.