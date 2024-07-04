# Implementing Topology-Aware Routing in Amazon EKS with Istio

## Introduction

Data transfer charges are often overlooked when operating Amazon Elastic Kubernetes Service (Amazon EKS) clusters. Understanding these charges can help reduce costs while operating your workload on Amazon EKS at production scale. This document provides a comprehensive guide on implementing topology-aware routing using Istio to manage and reduce data transfer costs in EKS clusters.

## Common Scenarios for Data Transfer Charges on EKS

Understanding general data transfer charges on AWS will help you better understand the EKS networking costs. The [Overview of Data Transfer Costs for Common Architectures](https://aws.amazon.com/blogs/architecture/overview-of-data-transfer-costs-for-common-architectures/) blog post provides insights into various scenarios for data transfer charges on AWS.

### Data Transfer Charges Across Worker Nodes and Control Plane

In any EKS network topology, worker nodes communicate with the control plane either through the public endpoint or through EKS-managed elastic network interfaces placed in the subnets provided during cluster creation. The route taken by worker nodes is determined by whether the private endpoint for your cluster is enabled or disabled. Depending on your EKS cluster and VPC configuration, there will be data transfer out charges and NAT gateway data transfer charges. Even actions originating from the Kubernetes API server, such as `kubectl exec` and `kubectl logs`, may result in cross-AZ data transfer charges, though these should be negligible.

### Data Transfer Charges Related to Kubernetes Services in an EKS Cluster

A significant driver for data transfer costs within Kubernetes clusters are calls to Kubernetes service objects. These costs occur in the following scenarios:

- **CoreDNS Pods**: By default, two replicas of CoreDNS pods run in an EKS cluster, potentially resulting in cross-AZ calls during DNS lookups.
- **ClusterIP Services**: Services exposed through ClusterIP distribute traffic to pods across multiple Availability Zones, leading to significant cross-AZ data transfer costs.

### Data Transfer Charges Related to Load Balancers in an EKS Cluster

Traffic from load balancers (ALB/NLB/CLB) can result in significant cross-AZ charges within EKS clusters, in addition to regular load balancer charges. AWS Load Balancer Controller supports provisioning load balancers in two traffic modes: Instance mode and IP mode. By default, instance mode is used, which incurs cross-AZ data transfer costs as traffic is routed using Kubernetes NodePort and ClusterIPs.

### Data Transfer Charges Related to Calls Made to External Services from an EKS Cluster

Pods running inside the EKS cluster often connect to AWS services such as Amazon S3 or services running outside of the VPC. Calls to these services may result in NAT gateway charges or egress out charges. VPC endpoints can be used to avoid NAT charges when communicating with AWS services. However, these endpoints may also incur cross-AZ charges unless Availability Zone-specific VPC endpoints are used.

## Best Practices to Address Data Transfer Costs

1. **Use VPC Endpoints**: Avoid NAT gateway charges by using VPC endpoints, which have lower data processing charges.
2. **Use IP Mode for Load Balancers**: Minimise cross-AZ data transfer costs by using the IP mode of ALB and NLB load balancers.
3. **Kubernetes NodeLocal Caches**: For clusters with many worker nodes, consider using Kubernetes NodeLocal caches to reduce calls to CoreDNS.
4. **Optimise Image Sizes**: Reduce data transfer charges related to downloading images from the container registry by using smaller base images, such as Linux Alpine.

## Addressing Cross-AZ Data Transfer Costs in EKS Clusters

To address cross-AZ data transfer costs, pods running in the cluster must be capable of performing topology-aware routing based on Availability Zone. Two mechanisms can help:

1. **Topology Aware Hints**: A Kubernetes feature currently in beta (v1.23) that allows routing to local Availability Zone-specific endpoints.
2. **Service Meshes**: Use service meshes like Istio to control egress traffic from pods to use endpoints available in the local Availability Zone.

## Implementing Topology-Aware Routing with Istio

### Prerequisites

- An EKS cluster with nodes spanning multiple Availability Zones.
- Istio installed on the EKS cluster.

### Step-by-Step Implementation

1. **Create an EKS Cluster**: Use the following configuration to create a three-node EKS cluster.

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: dto-analysis-k8scluster
  region: us-west-2
  version: "1.21"
  managedNodeGroups:
    - name: appservers
      instanceType: t3.xlarge
      desiredCapacity: 3
      minSize: 1
      maxSize: 4
      labels: { role: appservers }
      privateNetworking: true
      volumeSize: 8
      iam:
        withAddonPolicies:
          imageBuilder: true
          autoScaler: true
          xRay: true
          cloudWatch: true
          albIngress: true
          ssh:
            enableSsm: true
```

Create the cluster using the above configuration:

```sh
eksctl create cluster -f eksdtoanalysis.yaml
```

2. **Install Istio**:

```sh
export ISTIO_VERSION="1.10.0"
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=${ISTIO_VERSION} sh -
cd istio-${ISTIO_VERSION}
sudo cp -v bin/istioctl /usr/local/bin/
istioctl version --remote=false
yes | istioctl install --set profile=demo
```

3. **Deploy the Application**:

Create the necessary Kubernetes manifest files and deploy them:

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: octank-travel-ns
```

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zip-lookup-service-deployment
  namespace: octank-travel-ns
  labels:
  app: zip-lookup-service
  spec:
    replicas: 3
    selector:
     matchLabels:
      app: zip-lookup-service
    template:
      metadata:
        labels:
          app: zip-lookup-service
      spec:
        containers:
          - name: zip-lookup-service
            image: public.ecr.aws/m3x5o2v9/istio-tpr-app:latest
            ports:
              - containerPort: 8080
                protocol: TCP
            resources:
              requests:
                cpu: "250m"
            env:
              - name: XRAY_URL
                value: "xray-service.default:2000"
              - name: APPNAME_NAME
                value: "zip-service.octank-travel"
              - name: ZIPCODE
                value: "94582"
              - name: HTHRESHOLD
                value: "1"
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.labels['app']
            livenessProbe:
              httpGet:
                path: /health
                port: 8080
              initialDelaySeconds: 5
              periodSeconds: 5
```
```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: zip-lookup-service-local
  namespace: octank-travel-ns
spec:
  selector:
    app: zip-lookup-service
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```

Deploy the manifests:

```sh
kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml
kubectl apply -f services.yaml
```

4. **Validate Inter-AZ Calls**:

Deploy a test container and validate inter-AZ calls:

```sh
kubectl run curl-debug --image=radial/busyboxplus:curl -l "type=testcontainer" -n octank-travel-ns -it --tty sh
curl -s 169.254.169.254/latest/meta-data/placement/availability-zone
exit
kubectl expose deploy curl-debug -n octank-travel-ns --port=80 --target-port=8000
```

Create and execute a test script:

```sh
kubectl exec -it --tty -n octank-travel-ns $(kubectl get pod -l "type=testcontainer" -n octank-travel-ns -o jsonpath='{.items[0].metadata.name}') sh
cat <<EOF>> test.sh
n=1
while [ \$n -le 5 ]
do
  curl -s zip-lookup-service-local.octank-travel-ns
  sleep 1
  echo "---"
  n=\$(( n+1 ))
done
EOF
chmod +x test.sh
./test.sh
exit
```

5. **Enable Istio for the Application**:

Enable sidecar injection and restart the pods:

```yaml
# Update namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: octank-travel-ns
  labels:
    istio-injection: enabled
```

Apply the changes and restart the pods:

```sh
kubectl apply -f namespace.yaml
kubectl scale deploy zip-lookup-service-deployment -n octank-travel-ns --replicas=0
kubectl scale deploy zip-lookup-service-deployment -n octank-travel-ns --replicas=3
kubectl scale deploy curl-debug -n octank-travel-ns --replicas=0
kubectl scale deploy curl-debug -n octank-travel-ns --replicas=1
```

6. **Create a Destination Rule**:

Create a destination rule to enable topology-aware routing:

```yaml
# destinationrule.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: zip-lookup-service-local
  namespace: octank-travel-ns
spec:
  host: zip-lookup-service-local
  trafficPolicy:
    outlierDetection:
      consecutiveErrors: 7
      interval: 30s
      baseEjectionTime: 30s
```

Deploy the destination rule:

```sh
kubectl apply -f destinationrule.yaml
```

7. **Validate Topology-Aware Routing**:

Log back into the test container and run the test script:

```sh
kubectl exec -it --tty -n octank-travel-ns $(kubectl get pod -l "type=testcontainer" -n octank-travel-ns -o jsonpath='{.items[0].metadata.name}') sh
./test.sh
exit
```

The console output should show all calls going to the same pod in the same Availability Zone.

## Conclusion

By following the best practices and implementing topology-aware routing with Istio, you can significantly reduce data transfer costs in your EKS clusters. While service meshes add complexity, the cost savings can be substantial. Evaluate the trade-offs and consider adopting these strategies to optimise your EKS operations.

## Souce

- [Addressing latency and data transfer costs on EKS](https://aws.amazon.com/blogs/containers/addressing-latency-and-data-transfer-costs-on-eks-using-istio)
