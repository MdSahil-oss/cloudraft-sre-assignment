# Repository structure

```bash
.
├── application.yaml  # ArgoCD configuration
├── kind-cluster.yaml # Kind K8s configuration
├── metrics-app       # Helm chart, contains application & ingress both
│   ├── charts
│   ├── Chart.yaml
│   ├── templates
│   │   ├── deployment.yaml
│   │   ├── _helpers.tpl
│   │   ├── hpa.yaml
│   │   ├── ingress.yaml
│   │   ├── NOTES.txt
│   │   ├── secret.yaml
│   │   ├── serviceaccount.yaml
│   │   ├── service.yaml
│   │   └── tests
│   │       └── test-connection.yaml
│   └── values.yaml
└── README.md
```

# Deployment steps

## Start Cluster

```bash
kind create cluster --config ./kind-cluster.yaml
```

## Install NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.6/deploy/static/provider/kind/deploy.yaml
```

Wait for the controller to be ready:

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=Ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

## Install Argo CD

Create namespace `argocd`

```bash
kubectl create namespace argocd
```

Then install argocd

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait

```bash
kubectl -n argocd wait --for=condition=Ready pods --all --timeout=120s
```

## Create ArgoCD Application resource

```bash
kubectl apply -f application.yaml
```

Wait for the application to be ready:

```bash
kubectl wait --namespace default \
  --for=condition=Ready pod \
  --selector=app.kubernetes.io/name=metrics-app \
  --timeout=90s
```

## Check

Add domain to local host

```bash
echo "127.0.0.1 metrics-app.local" | sudo tee -a /etc/hosts
```

check the application behavior

```bash
for i in $(seq 0 20)
do
time curl http://metrics-app.local/counter
done
```

# Observation

Response to each call

```bash
for i in $(seq 0 20); do time curl http://metrics-app.local/counter; done
Counter value: 1
real    0m0.011s
user    0m0.000s
sys     0m0.005s
Counter value: 2
real    0m0.163s
user    0m0.003s
sys     0m0.003s
Counter value: 3
real    0m0.101s
user    0m0.005s
sys     0m0.000s
Counter value: 4
real    0m0.300s
user    0m0.000s
sys     0m0.005s
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx</center>
</body>
</html>

real    0m0.200s
user    0m0.005s
sys     0m0.000s
...
```

I noted following things when I sent requests to `counter` endpoint:

- First a few (3-5) requests `/counter` endpoints handles smoothly
- Later on server crashes and K8s restart the pod.

# Issue

The application crashes after a few requests

# Debug

I debugged the application by following methods

First checked the application logs using:

```bash
kubectl logs -n default POD_NAME --previous
```

and found no useful log regarding application crash and found `Debug mode: off` so enabled it by putting `FLASK_DEBUG=1` in Pod config of deployment as enviroment variable.

Yet I didn't get any useful log so I had to copy container workdir `/app` to my system by following command to debug application code.

```bash
kubectl cp POD_NAME:/app PATH/TO/ -c metrics-app
```

And in the application code I found `collector.py` file which does followings by executing script in `resources.dat`:

- Continuously allocates large blocks/objects of memory (500 MB per iteration of the outer loop) into global_memory.
- Never releases memory because the bytearrays are stored in a global list, This prevents those objects from being garbage-collected.
- Will eventually consume all available resource RAM, leading to performance issues, crashes, or out-of-memory (OOM) errors.

# Solution

To fix this memory-leaking Python code, the core issue is that large bytearray objects are continuously created and stored in a global list (global_memory) that never gets cleared, causing unbounded memory growth.

I think one of the solutions would be:

- Limit Memory Usage, So upon exceeding limit it removes old Objects of global_memory then put new ones.
  ```python
  global_memory = deque(maxlen=3)
  ...
  ```
  Also maange the size of allocating resources to pods (CPU & Memory) as per usage.
