# Instruction: Deploying the ToDo App to Kubernetes

## Prerequisites

- A running Kubernetes cluster and `kubectl` configured to talk to it.
- All manifests in `.infrastructure/` target the `mateapp` namespace.

## Deployment steps

1. Create the namespace (if not already present):

   ```
   kubectl apply -f .infrastructure/namespace.yml
   ```

2. Apply the Deployment:

   ```
   kubectl apply -f .infrastructure/deployment.yml
   ```

3. Apply the Horizontal Pod Autoscaler:

   ```
   kubectl apply -f .infrastructure/hpa.yml
   ```

4. Apply the Service (ClusterIP and/or NodePort, depending on how you want to expose the app):

   ```
   kubectl apply -f .infrastructure/clusterIp.yml
   kubectl apply -f .infrastructure/nodeport.yml
   ```

5. Verify everything is running:

   ```
   kubectl get pods -n mateapp
   kubectl get deployment -n mateapp
   kubectl get hpa -n mateapp
   kubectl get svc -n mateapp
   ```

## Resource requests and limits

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

- **Requests** (`250m` CPU, `64Mi` memory) reflect what the app actually needs to run comfortably at idle/low load, based on it being a lightweight Django app. These values are also what the HPA uses as the baseline to calculate CPU/memory utilization percentage — setting them too high would make the HPA under-scale, too low would make it over-scale.
- **Limits** (`500m` CPU, `128Mi` memory) are set to 2x the requests, giving the container headroom to handle short bursts of traffic without being OOMKilled or CPU-throttled too aggressively, while still protecting the node from a single misbehaving pod consuming unbounded resources.

## HPA configuration

```yaml
minReplicas: 2
maxReplicas: 5
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
```

- **minReplicas: 2** ensures the app stays available even if one pod is down or being restarted (rolling updates, node issues), avoiding downtime from a single point of failure.
- **maxReplicas: 5** caps the scale-out to a reasonable ceiling for this workload size, preventing runaway scaling from consuming excessive cluster resources during traffic spikes or misbehaving clients.
- **70% average utilization** (both CPU and memory) is a common middle-ground threshold: low enough to react before pods are overloaded, high enough to avoid scaling on noise/minor fluctuations. Scaling on **both CPU and memory** means the HPA reacts to whichever resource becomes the bottleneck first, since this app's load pattern isn't known to be purely CPU- or memory-bound.

## RollingUpdate strategy

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

- **RollingUpdate** (instead of `Recreate`) ensures zero-downtime deployments — old pods are only terminated as new ones become ready.
- **maxUnavailable: 1** allows at most one pod to be taken down at a time during an update, so with a minimum of 2 replicas at least one pod is always serving traffic.
- **maxSurge: 1** allows at most one extra pod to be created above the desired replica count during the rollout, keeping resource usage from spiking too much while still speeding up the rollout compared to `maxSurge: 0`.

## Accessing the app

- **Within the cluster**: other services can reach the app via the ClusterIP Service at `todoapp.mateapp.svc.cluster.local:80`.
- **From outside the cluster**: the NodePort Service exposes the app on port `30080` of any cluster node, e.g. `http://<node-ip>:30080`.
