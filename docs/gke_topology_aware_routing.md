# Topology-Aware Routing with GKE and Istio on GCP

## Introduction

Data transfer charges are often overlooked when operating Google Kubernetes Engine (GKE) clusters. Understanding these charges can help reduce costs while operating your workload on GKE at production scale. This document provides an overview of data transfer charges in GKE and how to use Istio for topology-aware routing to minimise these costs.

## Common Scenarios for Data Transfer Charges on GKE

Understanding general data transfer charges on Google Cloud Platform (GCP) will help you better understand the GKE networking costs. The [Overview of Data Transfer Costs for Common Architectures](https://cloud.google.com/blog/products/networking/understanding-data-transfer-costs-in-google-cloud) blog post will walk you through some of the scenarios for data transfer charges on GCP.

### Data Transfer Charges Across Worker Nodes and Control Plane

To better understand data transfer costs, we need to understand the different network topologies that GKE supports. The documentation page on [Creating a VPC for your GKE cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-vpc) provides you with a better understanding of those network topologies.

In any GKE network topology, worker nodes communicate with the control plane either through the public endpoint or through the GKE-managed network interfaces that are placed in the subnets you provide when you create the cluster. The route that worker nodes take to connect is determined by whether you have enabled or disabled the private endpoint for your cluster. Depending on how you have configured your GKE cluster and VPC, there will be data transfer out charges and NAT gateway data transfer charges. Even actions that originate from the Kubernetes API server, such as `kubectl exec` and `kubectl logs`, may result in cross-region data transfer charges. However, these charges should be negligible and should not drive up your data transfer costs.

### Data Transfer Charges Related to Kubernetes Services in a GKE Cluster

A significant driver for data transfer costs within Kubernetes clusters are calls to Kubernetes service objects. The data transfer costs when calling services occur in the following scenarios:

1. **DNS Lookups**: In a GKE cluster, two replicas of CoreDNS pods will be running by default. Hence, the DNS lookup may result in cross-region calls as the node from which DNS lookup is made and the node where the CoreDNS is running may be in different regions. However, this cost is very minimal, but there may be latency implications for traversing across regions.
2. **ClusterIP Services**: By default, services are exposed through ClusterIP, which distributes the traffic to the pods that may be spread across multiple regions, resulting in significant cross-region data transfer costs.

### Data Transfer Charges Related to Load Balancers in a GKE Cluster

Traffic from the load balancers (HTTP(S) Load Balancer, TCP/UDP Load Balancer) can result in significant cross-region charges within GKE clusters in addition to the regular load balancer charges.

GCP Load Balancer Controller supports provisioning Load Balancer in two traffic modes:

1. **Instance mode**
2. **IP mode**

By default, instance mode is used, which will incur cross-region data transfer costs as the traffic is routed using Kubernetes NodePort and ClusterIPs. One additional option to consider here is setting the `externalTrafficPolicy` to `Local`, which avoids creating ClusterIPs. However, this requires pods to be present in all nodes, which may result in imbalances in traffic routing.

### Data Transfer Charges Related to Calls Made to External Services from a GKE Cluster

Pods running inside the GKE cluster will often depend on connecting to GCP services such as Google Cloud Storage or services running outside of VPC. Calls to these services may result in NAT gateway charges or egress out charges. VPC Service Controls can be used to avoid NAT charges when communicating with GCP services. Even if VPC Service Controls are used, these may also incur cross-region charges unless region-specific endpoints are used in communicating with these services from within the GKE cluster.

## Best Practices to Address Data Transfer Costs

1. **Use VPC Service Controls**: To avoid NAT gateway charges, as data processing charges for VPC Service Controls are considerably lower than NAT gateway data processing charges. If possible, use region-aware VPC Service Controls to avoid cross-region charges.
2. **Use “IP mode” of Load Balancers**: To minimise cross-region data transfer costs.
3. **NodeLocal DNS Cache**: For clusters with large numbers of worker nodes, consider using the Kubernetes NodeLocal DNS Cache feature to reduce calls to CoreDNS.
4. **Optimise Image Sizes**: To reduce data transfer charges related to downloading images from the container registry. For example, consider building your images from Linux Alpine base images.

## Addressing Cross-Region Data Transfer Costs in GKE Clusters

Following the best practices specified in the earlier section will help reduce data transfer costs, but will not be able to address cross-region data transfer costs, which may be significant for your GKE clusters. For addressing cross-region data transfer costs, pods running in the cluster must be capable of performing topology-aware routing based on region.

There are two mechanisms that can help pods route to endpoints in their local region:

1. **Topology Aware Hints**: This is a Kubernetes feature currently in beta in v1.23 and will be available as part of GKE in the future. Using this feature will allow Kubernetes clusters to route to local region-specific endpoints. But this feature addresses only the inter-cluster communication and does not address where pods have to communicate with external entities. An additional factor to consider here is that some portion of traffic may still get routed cross-region even with Topology Aware Hints being enabled to ensure fair distribution of endpoints between regions.
2. **Service Meshes**: To control egress traffic from pods to use endpoints that are available in the local region. Istio is one such service mesh built on Envoy proxy that currently provides a topology-aware routing feature as part of its mesh implementation.

## Addressing Inter-Region Data Transfer Costs in GKE Clusters with Istio

In general, a service mesh sits on top of your Kubernetes infrastructure and is responsible for making communications between services over the network safe and reliable. Service mesh manages the network traffic between services. Istio is one of the many service mesh options available for GKE.

In this documentation, we will be using Istio because topology-aware routing is natively supported, which enables routing traffic to the pods or services within the same region. To learn more about Istio architecture and how to deploy it on GKE, please refer to the [Istio documentation](https://istio.io/latest/docs/setup/platform-setup/gke/).

### Setting Up Istio on GKE

For this demonstration, we need to create a three-node GKE cluster where the nodes span across multiple regions and then set up Istio for the cluster.

1. **Create a GKE Cluster**: Use the following command to create a GKE cluster.

```sh
gcloud container clusters create dto-analysis-k8scluster \
  --region us-west1 \
  --node-locations us-west1-a,us-west1-b,us-west1-c \
  --num-nodes 1 \
  --machine-type e2-standard-4
```

2. **Install Istio**: Follow the steps below to install Istio on the GKE cluster.

```sh
export ISTIO_VERSION="1.10.0"
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=${ISTIO_VERSION} sh -
cd istio-${ISTIO_VERSION}
sudo cp -v bin/istioctl /usr/local/bin/
istioctl version --remote=false
yes | istioctl install --set profile=demo
```

Once the cluster is created and Istio has been properly set up, we need to install our application onto the GKE cluster.

### Deploying the Application

1. **Create Kubernetes Manifest Files**:

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

2. **Deploy the Application**:

```sh
kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml
kubectl apply -f services.yaml
```

3. **Deploy a Test Container**:

```sh
kubectl run curl-debug --image=radial/busyboxplus:curl -l "type=testcontainer" -n octank-travel-ns -it --tty sh
# Run curl command in the test container to see which region the test container is running in
curl -H "Metadata-Flavor: Google" "http://169.254.169.254/computeMetadata/v1/instance/zone"
# Exit the test container
exit
```

4. **Expose the Test Container as a Service**:

```sh
kubectl expose deploy curl-debug -n octank-travel-ns --port=80 --target-port=8000
```

5. **Install a Test Script**:

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

The execution of the script should produce output showing calls to the backend services distributed across the regions using the default ClusterIP-based service call mechanism.

### Enabling Topology-Aware Routing with Istio

1. **Enable Istio for the Application**:

```yaml
# Update namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: octank-travel-ns
  labels:
    istio-injection: enabled
```

```sh
# Apply the changes
kubectl apply -f namespace.yaml
kubectl get po -n octank-travel-ns

# Restart the app pods to have the envoy side-car proxies injected
kubectl scale deploy zip-lookup-service-deployment -n octank-travel-ns --replicas=0
kubectl scale deploy zip-lookup-service-deployment -n octank-travel-ns --replicas=3

# Restart the test container pod
kubectl scale deploy curl-debug -n octank-travel-ns --replicas=0
kubectl scale deploy curl-debug -n octank-travel-ns --replicas=1

# Reinstall the test script in the newly created test container pod
kubectl exec -it --tty -n octank-travel-ns $(kubectl get pod -l "type=testcontainer" -n octank-travel-ns -o jsonpath='{.items[0].metadata.name}') sh
# Create a startup script
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
# Exit the test container
exit
```

2. **Create a Destination Rule Object**:

```sh
# Create a destination rule object
cat <<EOF>> destinationrule.yaml
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
EOF

# Deploy the object as part of the cluster
kubectl apply -f destinationrule.yaml
```

3. **Run the Test Again**:

```sh
# Log back in to the test container
kubectl exec -it --tty -n octank-travel-ns $(kubectl get pod -l "type=testcontainer" -n octank-travel-ns -o jsonpath='{.items[0].metadata.name}') sh
# Run the test
curl -H "Metadata-Flavor: Google" "http://169.254.169.254/computeMetadata/v1/instance/zone"
./test.sh
# Exit the test container
exit
```

The console output should show all the calls going to the same pod in the same region, avoiding cross-region data transfer charges.

### Addressing Cross-Region Data Transfer Costs with Istio When Calling GCP Services

Cross-region data transfer costs can occur when calling GCP services such as Google Cloud SQL, Google Cloud Pub/Sub, Google Cloud Storage, and others. For example, in the case of Google Cloud SQL, the primary database may be in a different region than nodes where the pods are running.

In such scenarios, Istio provides a service entry object that can be configured to intercept calls to the external GCP service and route to endpoints in the same region.

1. **Create a Google Cloud SQL Instance**:

Refer to the documentation on [Creating and Connecting to Cloud SQL](https://cloud.google.com/sql/docs/mysql/create-instance) for setting up a Cloud SQL instance.

2. **Define a Service Entry and Destination Rule Object**:

```sh
# Create a single file with the service entry and destination rule object
cat <<EOF> cloudsql-se.yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-svc-cloudsql
  namespace: octank-travel-ns
spec:
  hosts:
    - $PRIMARY_EP
  location: MESH_EXTERNAL
  ports:
  - number: 3305
    name: tcp
    protocol: tcp
  resolution: DNS
  endpoints:
    - address: $PRIMARY_EP
      locality: us-west1/us-west1-a
      ports:
        tcp: 3306
    - address: $READER_EP
      locality: us-west1/us-west1-b
      ports:
        tcp: 3306
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: external-svc-cloudsql-dr
  namespace: octank-travel-ns
spec:
  host: $PRIMARY_EP
  trafficPolicy:
    outlierDetection:
      consecutiveErrors: 7
      interval: 30s
      baseEjectionTime: 30s
EOF

kubectl apply -f cloudsql-se.yaml
```

3. **Monitor Envoy Proxy Logs**:

```sh
kubectl logs <<replace with podname running in us-west1-b>> -n octank-travel-ns -c istio-proxy -f
```

4. **Connect to the Cloud SQL Instance**:

```sh
echo $PRIMARY_EP
kubectl exec -it --tty -n octank-travel-ns <<replace with podname running in us-west1-b>> sh
curl <<replace with primary endpoint url>>:3305
exit
dig $READER_EP
```

You should be able to confirm in the logs that the connection happens to the reader endpoint though you are connecting to the primary endpoint URL from within the application.

## Conclusion

To recap, if your data transfer costs with GKE clusters are high, I recommend the following steps:

1. Ensure that you are following the best practices outlined in the section “Best practices to address data transfer costs”.
2. Enable VPC flows and create dashboards to monitor what are the key drivers of the data transfer costs.
3. Once key drivers are identified, evaluate if those drivers can be addressed by adopting a service mesh architecture. Then implement the service mesh to address the data transfer costs as outlined in this documentation. A key caveat to implementing service meshes is the additional complexity they bring to the overall architecture. Hence, complexity should be weighed against cost savings before making the decision.
4. One additional option we can consider once GKE support for Kubernetes 1.23 is available is that of Topology Aware Hints that can reduce inter-region data transfer costs for inter-cluster traffic and with much less complexity. To learn more, refer to the documentation [Topology Aware Hints](https://kubernetes.io/docs/concepts/services-networking/topology-aware-hints/).
5. When implementing topology-aware routing, it is important to have pods balanced across the regions using Topology Spread Constraints to avoid imbalances in the amount of traffic handled by each pod. To know more about Topology Spread Constraints, refer to [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/).

Data transfer costs can be a significant driver of costs within your GKE cluster. However, by following the best practices and strategies outlined in this documentation, you can significantly reduce these costs.
