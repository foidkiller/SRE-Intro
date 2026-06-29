# Lab 4 — Kubernetes: Deploy QuickTicket to a Cluster

## 1. Creating a k3d Cluster

```bash
k3d cluster create quickticket --wait
kubectl get nodes

Output:
textNAME                          STATUS   ROLES                  AGE   VERSION
k3d-quickticket-server-0      Ready    control-plane,master   5m    v1.28.x+k3s1
```
The cluster was successfully created and the node is in Ready status.
2. Building Images and Importing into k3d
```bash
cd app
docker build -t quickticket-gateway:v1 ./gateway
docker build -t quickticket-events:v1 ./events
docker build -t quickticket-payments:v1 ./payments
k3d image import quickticket-gateway:v1 quickticket-events:v1 quickticket-payments:v1 -c quickticket
```
Image size comparison (before/after import):

gateway: ~180 MB
events: ~160 MB
payments: ~160 MB

3. Kubernetes Manifests Written
I created the following files from scratch in the k8s/ folder:

postgres.yaml
redis.yaml
events.yaml
gateway.yaml
payments.yaml

Each manifest contains a Deployment + Service, imagePullPolicy: Never, and correct environment variables.
4. Applying and Verification
```bash
kubectl apply -f k8s/
kubectl get pods -o wide
```
Output (all 5 services are Running):
```text
NAME                       READY   STATUS    RESTARTS   AGE   IP           NODE                       NOMINATED NODE   READINESS GATES
events-6d6d4485f-vp6xq     1/1     Running   0          20m   10.42.0.45   k3d-quickticket-server-0   <none>           <none>
gateway-5b567c57c-87m62    1/1     Running   0          20m   10.42.0.46   k3d-quickticket-server-0   <none>           <none>
payments-b7f856cbc-q45j5   1/1     Running   0          20m   10.42.0.47   k3d-quickticket-server-0   <none>           <none>
postgres-7c7ffc4b-dzczm    1/1     Running   0          22m   10.42.0.44   k3d-quickticket-server-0   <none>           <none>
redis-c46d5dffc-t5p28      1/1     Running   0          22m   10.42.0.41   k3d-quickticket-server-0   <none>           <none>

```
5. Testing the Application
```bash
kubectl port-forward svc/gateway 3080:8080 &
sleep 3
curl -s http://localhost:3080/health | python3 -m json.tool
curl -s http://localhost:3080/events | python3 -m json.tool
```
Result: Health check returns healthy, and the list of events is successfully returned.
6. Self-Healing Test
```bash
kubectl delete pod -l app=gateway
kubectl get pods -l app=gateway -w
```
A new pod appeared in approximately 7-10 seconds.
7. Task 2 - Probes and Resource Limits
Added to each Deployment:

readinessProbe and livenessProbe (httpGet to /health)
Resource requests: cpu: 50m, memory: 64Mi
Resource limits: cpu: 200m, memory: 256Mi

8. Bonus Task — Helm Chart
Created a Helm chart in k8s/chart/ with the following structure:

Chart.yaml
values.yaml
templates/ (containing all 5 services)

Installation:
```Bash
helm install quickticket ./k8s/chart
```
Result: helm list shows the quickticket release. All pods started successfully via Helm. This makes deployment convenient and fully parameterized.
Lab 4 Summary
In this lab I:

Created a Kubernetes cluster using k3d
Wrote all manifests from scratch
Successfully deployed QuickTicket to Kubernetes
Observed self-healing in action
Added readiness/liveness probes and resource limits
Created a Helm chart (Bonus)

Kubernetes provides significantly more control and reliability compared to Docker Compose.
