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
