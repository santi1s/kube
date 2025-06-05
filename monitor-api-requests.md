# Monitoring Kubernetes API Server Requests in Kind Clusters

This guide demonstrates multiple methods to monitor API server requests in a Kind (Kubernetes in Docker) cluster.

## Prerequisites

- Kind installed
- kubectl configured
- Docker running

## Method 1: Checking API Server Logs Directly

### Access API server logs from the Kind control plane container:

```bash
# Get the name of your Kind cluster (default is "kind")
kind get clusters

# Access the control plane container logs
docker logs kind-control-plane 2>&1 | grep kube-apiserver

# Follow API server logs in real-time
docker logs -f kind-control-plane 2>&1 | grep kube-apiserver

# Filter for specific patterns (e.g., POST requests)
docker logs kind-control-plane 2>&1 | grep kube-apiserver | grep POST

# Get more detailed logs with timestamps
docker exec kind-control-plane journalctl -u kubelet -f | grep apiserver
```

### Direct access to API server logs inside the container:

```bash
# Enter the control plane container
docker exec -it kind-control-plane bash

# Inside the container, check API server logs
cat /var/log/pods/kube-system_kube-apiserver-*/kube-apiserver/*.log

# Or use crictl to check container logs
crictl ps | grep kube-apiserver
crictl logs <container-id>
```

## Method 2: Using kubectl with Increased Verbosity

### Monitor API requests by increasing kubectl verbosity:

```bash
# Level 6 shows HTTP request headers
kubectl get pods -v=6

# Level 7 shows HTTP request headers and response headers
kubectl get pods -v=7

# Level 8 shows HTTP request and response bodies
kubectl get pods -v=8

# Level 9 shows HTTP request/response bodies and curl commands
kubectl get pods -v=9

# Example: Watch pod creation with high verbosity
kubectl run test-pod --image=nginx -v=8

# Monitor all API calls for a specific operation
kubectl apply -f deployment.yaml -v=8
```

## Method 3: Enable Audit Logging in Kind

### Create a Kind cluster with audit logging enabled:

1. Create an audit policy file:

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all requests at the Metadata level
  - level: Metadata
    omitStages:
      - RequestReceived
  # Log pod creation/deletion at RequestResponse level
  - level: RequestResponse
    omitStages:
      - RequestReceived
    resources:
      - group: ""
        resources: ["pods"]
    verbs: ["create", "delete"]
  # Log full details for secrets operations
  - level: RequestResponse
    omitStages:
      - RequestReceived
    resources:
      - group: ""
        resources: ["secrets"]
```

2. Create a Kind cluster configuration:

```yaml
# kind-audit-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: ClusterConfiguration
        apiServer:
          extraArgs:
            audit-log-path: /var/log/kubernetes/audit.log
            audit-policy-file: /etc/kubernetes/audit-policy.yaml
            audit-log-maxage: "30"
            audit-log-maxbackup: "3"
            audit-log-maxsize: "100"
          extraVolumes:
            - name: audit-policy
              hostPath: /etc/kubernetes/audit-policy.yaml
              mountPath: /etc/kubernetes/audit-policy.yaml
              readOnly: true
            - name: audit-log
              hostPath: /var/log/kubernetes/audit.log
              mountPath: /var/log/kubernetes/audit.log
    extraMounts:
      - hostPath: ./audit-policy.yaml
        containerPath: /etc/kubernetes/audit-policy.yaml
```

3. Create the cluster:

```bash
kind create cluster --config kind-audit-config.yaml
```

4. Access audit logs:

```bash
# View audit logs
docker exec kind-control-plane cat /var/log/kubernetes/audit.log | jq .

# Follow audit logs in real-time
docker exec kind-control-plane tail -f /var/log/kubernetes/audit.log | jq .

# Filter for specific resources
docker exec kind-control-plane cat /var/log/kubernetes/audit.log | jq 'select(.objectRef.resource=="pods")'
```

## Method 4: Using a Proxy to Monitor Requests

### Set up kubectl proxy to monitor API requests:

```bash
# Start kubectl proxy with verbose logging
kubectl proxy -v=8

# In another terminal, make requests through the proxy
curl http://localhost:8001/api/v1/namespaces/default/pods

# The proxy terminal will show all API requests and responses
```

## Method 5: Check API Server Metrics

### Access API server metrics (if metrics-server is installed):

```bash
# Install metrics-server for Kind
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch metrics-server for Kind
kubectl patch -n kube-system deployment metrics-server --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Check API server metrics endpoint
kubectl get --raw /metrics | grep apiserver_request

# Get specific metrics
kubectl get --raw /metrics | grep apiserver_request_total
kubectl get --raw /metrics | grep apiserver_request_duration_seconds
```

## Method 6: Using tcpdump for Network Monitoring

### Monitor API server network traffic:

```bash
# Inside the Kind control plane container
docker exec -it kind-control-plane bash

# Install tcpdump
apt-get update && apt-get install -y tcpdump

# Capture traffic on port 6443 (API server)
tcpdump -i any -nn port 6443 -A

# Save capture to file for analysis
tcpdump -i any -nn port 6443 -w /tmp/api-traffic.pcap
```

## Quick Monitoring Script

Create a simple monitoring script:

```bash
#!/bin/bash
# monitor-api.sh

echo "Starting API server monitoring..."
echo "Press Ctrl+C to stop"
echo ""

# Monitor in multiple terminals or use tmux
echo "=== Real-time API Server Logs ==="
docker logs -f kind-control-plane 2>&1 | grep -E "kube-apiserver.*HTTP" &

echo ""
echo "=== High-level API Requests ==="
while true; do
    kubectl get events --all-namespaces --watch-only -v=6 2>&1 | grep -E "GET|POST|PUT|DELETE|PATCH"
done
```

## Tips for Effective Monitoring

1. **Filter by User**: Look for specific users in audit logs:
   ```bash
   docker exec kind-control-plane cat /var/log/kubernetes/audit.log | jq 'select(.user.username=="kubernetes-admin")'
   ```

2. **Monitor Specific Resources**: Focus on particular resources:
   ```bash
   kubectl get pods -w -v=7 2>&1 | grep -E "GET|POST|PUT|DELETE|PATCH.*pods"
   ```

3. **Check Rate Limiting**: Monitor for rate limit issues:
   ```bash
   docker logs kind-control-plane 2>&1 | grep -i "rate limit"
   ```

4. **Performance Monitoring**: Check request latencies:
   ```bash
   kubectl get --raw /metrics | grep apiserver_request_duration_seconds
   ```

## Common Use Cases

### Debug Authentication Issues
```bash
kubectl get pods -v=8 2>&1 | grep -E "401|403|authentication|authorization"
```

### Monitor ConfigMap/Secret Access
```bash
docker exec kind-control-plane cat /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.resource=="secrets" or .objectRef.resource=="configmaps")'
```

### Track Pod Lifecycle
```bash
kubectl get events -w | grep -E "Created|Started|Killing|Deleted"
```

## Cleanup

To remove the Kind cluster:
```bash
kind delete cluster
```