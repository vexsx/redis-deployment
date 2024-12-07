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

### Additional Notes:

1. **Storage Class Selection**:
   - The `volumeClaimTemplates` use `storageClassName: longhorn`, which is the default storage class as per your `kubectl get storageclasses.storage.k8s.io` output.
   - **Longhorn** supports dynamic provisioning and volume expansion, allowing you to scale storage as needed without manual intervention.

2. **Allow Volume Expansion**:
   - The `allowVolumeExpansion: true` field in `volumeClaimTemplates` enables resizing of PVCs if your storage requirements grow beyond the initial 5Gi.

3. **Security Context**:
   - Running containers as non-root (`runAsUser: 1000`) and setting `fsGroup: 1000` ensures that the Redis process has the necessary permissions to read/write to the mounted volumes without requiring root privileges, enhancing security.

4. **Resource Management**:
   - Defining `resources.requests` and `resources.limits` ensures that each container has guaranteed resources and is capped to prevent overconsumption, maintaining cluster stability.

5. **Metrics Integration**:
   - The Redis Exporter sidecar provides valuable metrics that can be scraped by Prometheus for monitoring Redis performance and health.
   - Ensure your Prometheus configuration includes the `redis-standalone-metrics` service as a scrape target.

6. **Scaling**:
   - Scaling the StatefulSet to a higher number of replicas (e.g., 50) will create additional Pods (`redis-standalone-3` to `redis-standalone-49`), each with its own PVC and configuration.
   - Since this setup does not configure Redis clustering, each Redis instance operates independently, suitable for scenarios where multiple Redis instances are required without intercommunication.

7. **ConfigMap Updates**:
   - When updating the `redis.conf` in the ConfigMap, perform a rolling update to ensure all Pods pick up the new configuration:
     ```bash
     kubectl apply -f redis-standalone.yaml
     kubectl rollout restart statefulset redis-standalone -n redis-standalone
     ```

8. **Backup Strategy**:
   - Regular backups are crucial for data durability. Consider integrating backup solutions like Velero or custom scripts to snapshot PVCs and store backups externally.

By following this manifest and the accompanying best practices, you can deploy a robust, scalable, and secure standalone Redis setup on your Kubernetes cluster.
