# DevOps Kubernetes Lab

A simple, reproducible Kubernetes hands-on lab using Kind, Metrics Server, HPA, Prometheus and Grafana.

## Architecture

```text
                    Kind Cluster
                         |
          +--------------+--------------+
          |              |              |
     Control Plane    Worker 1       Worker 2
                         |
                  Nginx Deployment
                    /          \
                 Pod 1        Pod 2
                    \          /
                    ClusterIP
                         |
              +----------+----------+
              |                     |
        Metrics Server          Prometheus
              |                     |
             HPA                   Grafana
              |                     |
        Pod auto-scaling       Dashboard
```

## Requirements

- Windows 11
- Docker Desktop with WSL2 backend
- Ubuntu WSL
- Kind
- kubectl
- Helm
- About 8 GB RAM: use the lightweight settings in this repository.

## 1. Create the Kind cluster

Open **Windows CMD**:

```cmd
C:\kind\kind.exe create cluster --name devops-lab --config C:\devops-kubernetes-lab\kind-config.yaml
```

Verify:

```cmd
kubectl get nodes
```

Expected:

```text
devops-lab-control-plane
devops-lab-worker
devops-lab-worker2
```

## 2. Configure WSL kubeconfig

Open Ubuntu WSL:

```bash
export KUBECONFIG=/mnt/c/Users/USER/.kube/config
kubectl config current-context
kubectl get nodes
```

Expected context:

```text
kind-devops-lab
```

> Replace `USER` with your Windows username if required.

Do not use `sudo helm` in this lab.

## 3. Install Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Kind commonly needs insecure kubelet TLS:

```bash
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

Wait:

```bash
kubectl rollout status deployment/metrics-server -n kube-system --timeout=120s
```

Verify:

```bash
kubectl top nodes
```

## 4. Install Helm repository

Check Helm:

```bash
helm version
```

Add Prometheus Community:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

## 5. Install Prometheus + Grafana

From the repository root:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f monitoring/values.yaml
```

Check:

```bash
helm list -n monitoring
kubectl get pods -n monitoring
```

Wait for Grafana:

```bash
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=grafana \
  -n monitoring \
  --timeout=180s
```

## 6. Deploy Nginx

```bash
kubectl apply -f manifests/nginx-deployment.yaml
kubectl apply -f manifests/nginx-service.yaml
```

Verify:

```bash
kubectl get pods -o wide
kubectl get svc nginx-service
kubectl get endpointslices
```

## 7. Create HPA

```bash
kubectl apply -f manifests/nginx-hpa.yaml
```

Check:

```bash
kubectl get hpa
```

Initially the CPU target may show approximately:

```text
0%/50%
```

That is normal when there is no CPU load.

## 8. Access Prometheus

Open a second Ubuntu terminal.

Set kubeconfig again if needed:

```bash
export KUBECONFIG=/mnt/c/Users/USER/.kube/config
```

Run:

```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
```

Open:

```text
http://localhost:9090
```

Test:

```promql
up
```

## 9. Access Grafana

In another Ubuntu terminal:

```bash
export KUBECONFIG=/mnt/c/Users/USER/.kube/config
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Open:

```text
http://localhost:3000
```

Get the Grafana password:

```bash
kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

Login:

```text
Username: admin
Password: output from the command above
```

## 10. Prometheus datasource in Grafana

Because `kube-prometheus-stack` provisions Grafana, **do not manually install a Prometheus plugin**.

In Grafana:

```text
Connections
  -> Data sources
  -> Prometheus
```

The datasource should already exist.

If you create a datasource manually, the in-cluster URL is:

```text
http://monitoring-kube-prometheus-prometheus.monitoring:9090
```

## 11. Create the first Grafana visualization

Go to:

```text
Dashboards
  -> New
  -> New dashboard
  -> Add visualization
```

Select the Prometheus datasource.

Use:

```promql
sum by (pod) (
  rate(container_cpu_usage_seconds_total{
    namespace="default",
    pod=~"nginx-app-.*",
    container="nginx"
  }[1m])
)
```

Choose **Time series** and Apply.

## 12. Generate CPU load for HPA practice

For a simple controlled HPA lab, create a temporary CPU-burning Deployment:

```bash
kubectl create deployment cpu-demo --image=busybox:1.36 \
  -- /bin/sh -c 'while true; do :; done'
```

Set CPU request:

```bash
kubectl set resources deployment/cpu-demo --requests=cpu=100m
```

Scale it to two pods:

```bash
kubectl scale deployment/cpu-demo --replicas=2
```

Check:

```bash
kubectl top pods
```

This CPU demo is separate from the Nginx HPA. It is useful for learning CPU monitoring without changing the Nginx application.

Delete it when finished:

```bash
kubectl delete deployment cpu-demo
```

## 13. Useful Prometheus queries

### All targets

```promql
up
```

### Nginx CPU

```promql
sum by (pod) (
  rate(container_cpu_usage_seconds_total{
    namespace="default",
    pod=~"nginx-app-.*",
    container="nginx"
  }[1m])
)
```

### Kubernetes pod count

```promql
count(kube_pod_info)
```

### CPU usage by namespace

```promql
sum by (namespace) (
  rate(container_cpu_usage_seconds_total{
    container!="",
    image!=""
  }[5m])
)
```

## 14. Troubleshooting checklist

### Pods

```bash
kubectl get pods -A -o wide
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
```

### Events

```bash
kubectl get events -A --sort-by=.lastTimestamp
```

### Nodes

```bash
kubectl get nodes
kubectl describe node <node>
kubectl top nodes
```

### Deployment

```bash
kubectl get deployment
kubectl describe deployment <deployment>
kubectl rollout status deployment/<deployment>
kubectl rollout history deployment/<deployment>
```

### Service

```bash
kubectl get svc
kubectl describe svc <service>
kubectl get endpointslices
```

### HPA

```bash
kubectl get hpa
kubectl describe hpa nginx-hpa
```

### Monitoring

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl logs deployment/monitoring-grafana -n monitoring --tail=100
```

## 15. Stop port-forward

In the terminal running `kubectl port-forward`:

```text
Ctrl+C
```

This only stops the port-forward. It does not delete Grafana, Prometheus or the cluster.

## 16. Delete the lab

Delete applications:

```bash
kubectl delete -f manifests/
```

Delete monitoring:

```bash
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```

Delete the Kind cluster from Windows CMD:

```cmd
C:\kind\kind.exe delete cluster --name devops-lab
```

## Learning roadmap

1. Kind
2. Nodes
3. Deployment
4. ClusterIP Service
5. Metrics Server
6. HPA
7. Prometheus
8. Grafana
9. Grafana dashboards
10. CPU load
11. HPA scaling
12. AlertManager
13. Kubernetes troubleshooting
14. Ingress
15. Persistent Volumes
16. RBAC
17. NetworkPolicy
18. Production-style troubleshooting
