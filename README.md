# Redis Standalone StatefulSet with Persistent Storage and Metrics

This setup creates a standalone Redis deployment using a Kubernetes StatefulSet. Each Pod has its own persistent volume for data storage, ensuring data durability and isolation. Additionally, a sidecar container (Redis Exporter) provides metrics on a dedicated service for observability.

## Key Features

- **Namespace Isolation**:  
  All resources are deployed within a dedicated namespace (`redis-standalone`), providing clear organization and scoping.
  
- **Stable Network Identities**:  
  A headless Service (`redis-standalone-headless`) ensures that each Pod in the StatefulSet has a stable DNS address. This is crucial for reliable access to specific Redis Pods.
  
- **StatefulSet with volumeClaimTemplates**:  
  Using `volumeClaimTemplates`, each Redis Pod is assigned its own PersistentVolumeClaim (PVC), ensuring unique, dedicated storage for each replica. This architecture guarantees that:
  - Each Pod’s data persists through restarts.
  - Data is isolated per Pod, preventing interference between replicas.
  
- **Scalable from the Start**:  
  By default, the StatefulSet runs with 3 replicas. However, you can easily adjust `replicas` to a higher number (e.g., 50). Kubernetes will then create additional Pods, each with its own PVC.
  
- **Sidecar for Metrics**:  
  A Redis Exporter container runs alongside Redis in each Pod. A separate Service (`redis-standalone-metrics`) exposes the metrics endpoint on port 9121. Monitoring tools like Prometheus can scrape this endpoint to track Redis performance and health.
  
- **Separation of Concerns**:  
  The Redis Pods do not form a cluster; each replica is standalone. This simplifies the configuration while maintaining durability and scalability. If clustering is needed, additional configuration and tooling would be required.

## Components Overview

1. **Namespace (`redis-standalone`)**:  
   Provides a dedicated environment for all Redis resources.

2. **Headless Service (`redis-standalone-headless`)**:  
   Ensures stable, DNS-based Pod naming within the StatefulSet. Each Pod will be accessible at a hostname like `redis-standalone-0.redis-standalone-headless`.

3. **Metrics Service (`redis-standalone-metrics`)**:  
   Makes Redis metrics accessible inside the cluster at `redis-standalone-metrics.redis-standalone.svc.cluster.local:9121`.

4. **StatefulSet (`redis-standalone`)**:  
   Deploys the Redis Pods with:
   - One Redis container per Pod.
   - One Redis Exporter container per Pod.
   - Independent storage per Pod, provisioned from `volumeClaimTemplates`.

5. **PVCs from `volumeClaimTemplates`**:  
   Automatically generated PVCs for each Pod, ensuring each replica has a unique, persistent data store.

## How It Works

- On creation, the StatefulSet starts `redis-standalone-0`, `redis-standalone-1`, `redis-standalone-2`, and so forth (incrementing by one for each replica).
- Each Pod is associated with its own PVC for data persistence.
- Redis runs with `--appendonly yes` and `--dir /data`, ensuring data is written to the PVC-backed storage.
- The Redis Exporter container scrapes Redis metrics at `localhost:6379` and exposes them on `9121`.
- The `redis-standalone-metrics` Service provides a stable endpoint for metrics scraping.

## Scaling Up or Down

- To increase replicas (e.g., to 50), update the StatefulSet `replicas: 50`.
- Kubernetes creates more Pods (`redis-standalone-3`, `redis-standalone-4`, …, `redis-standalone-49`), each with its own PVC.
- Scaling down removes Pods but does not automatically delete PVCs, preserving data. This enables you to potentially restore to a previous state if needed.

## Considerations

- **Non-clustered**: Each Redis Pod runs independently. They do not share state or cluster awareness. For a clustered Redis setup, additional configuration is required.
- **Storage Class**: The manifest references `longhorn` as the storage class. Ensure `longhorn` or another suitable storage class is available in your cluster.
- **Monitoring Integration**: Connect the `redis-standalone-metrics` Service to a Prometheus instance or another monitoring tool to gain insights into Redis performance and resource usage.

---

This configuration is a robust starting point for running standalone Redis instances at scale on Kubernetes, with each replica maintaining its own dedicated persistent storage and offering metrics for monitoring.
