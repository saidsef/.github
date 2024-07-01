# Kubernetes Topology Aware Routing Documentation

## Introduction

In the ever-evolving landscape of cloud-native applications, ensuring optimal performance and resource utilisation is paramount. Kubernetes Topology Aware Routing is a sophisticated feature designed to enhance network efficiency by routing traffic based on the physical or logical topology of the cluster. This capability not only reduces latency but also optimises resource usage, leading to improved application performance and cost savings.

## Advantages of Topology Aware Routing

1. **Reduced Latency**: By routing traffic to the nearest available endpoints, Topology Aware Routing minimises network hops, thereby reducing latency and improving response times.
2. **Enhanced Resource Utilisation**: Efficient traffic routing ensures that resources are utilised optimally, preventing overloading of specific nodes and promoting balanced workloads.
3. **Improved Fault Tolerance**: By understanding the topology, Kubernetes can make more informed decisions about routing, enhancing the system's resilience to failures.
4. **Cost Efficiency**: Optimised routing can lead to reduced data transfer costs, especially in multi-zone or multi-region deployments.

## Implementation

### Prerequisites

- Kubernetes cluster version 1.23 or later.
- `kube-proxy` configured in IPVS mode.
- Nodes labelled with topology keys such as `topology.kubernetes.io/zone` and `topology.kubernetes.io/region`.

### Step-by-Step Guide

1. **Enable Topology Aware Routing Feature Gate**:
Ensure that the `TopologyAwareHints` feature gate is enabled in your Kubernetes cluster. This can be done by adding the following flag to the API server and the kubelet configuration:

```yaml
--feature-gates=TopologyAwareHints=true
```

2. **Label Nodes with Topology Information**:
Nodes should be labelled with appropriate topology keys. For example:

```bash
kubectl label nodes <node-name> topology.kubernetes.io/zone=<zone-name>
kubectl label nodes <node-name> topology.kubernetes.io/region=<region-name>
```

3. **Configure Services for Topology Aware Routing**:
When creating a Service, set the `topologyKeys` field to specify the preferred topology keys for routing. For example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
    ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "topology.kubernetes.io/zone"
    - "topology.kubernetes.io/region"
    - "*"
```

In this configuration, traffic will be routed first within the same zone, then within the same region, and finally to any available endpoint if no closer ones are found.

## Validation

### Verifying Node Labels

Ensure that nodes are correctly labelled with the topology information:

```bash
kubectl get nodes --show-labels
```

### Testing Service Routing

1. **Deploy the Application**:
Deploy your application and the configured Service.

```bash
kubectl apply -f my-app-deployment.yaml
kubectl apply -f my-service.yaml
```

2. **Generate Traffic**:
Use a tool like `curl` or `wrk` to generate traffic to the Service and observe the routing behaviour.

```bash
curl http://<service-ip>:<port>
```

3. **Monitor Traffic Distribution**:
Use Kubernetes metrics and logs to monitor how traffic is being distributed across different nodes and zones. Tools like Prometheus and Grafana can be particularly useful for visualising this data.

```bash
kubectl logs -l app=my-app
```

### Enabling Topology Aware Routing

> Prior to Kubernetes 1.27, this behavior was controlled using the `service.kubernetes.io/topology-aware-hints` annotation.

You can enable Topology Aware Routing for a Service by setting the `service.kubernetes.io/topology-mode` annotation to `Auto`. When there are enough endpoints available in each zone, Topology Hints will be populated on EndpointSlices to allocate individual endpoints to specific zones, resulting in traffic being routed closer to where it originated from.

## Conclusion

Kubernetes Topology Aware Routing is a powerful feature that can significantly enhance the performance and efficiency of your applications. By intelligently routing traffic based on the cluster's topology, it ensures reduced latency, better resource utilisation, and improved fault tolerance. Implementing this feature requires careful configuration and validation, but the benefits it brings to your cloud-native infrastructure are well worth the effort.

For further details and advanced configurations, refer to the official Kubernetes documentation on [Topology Aware Routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/).
