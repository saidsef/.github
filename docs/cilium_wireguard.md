# Installing Cilium with Kubernetes and Enabling WireGuard Encryption

## Introduction

In today's digital landscape, ensuring the security and integrity of network communications is paramount. Cilium, a powerful networking and security solution for Kubernetes, offers advanced features to enhance the security posture of your Kubernetes clusters. This documentation provides a comprehensive guide for installing Cilium v1.15 or later on Kubernetes clusters with version 1.26 or higher, and enabling WireGuard encryption to secure your network traffic. By following this guide, your organisation can achieve robust network security, safeguarding sensitive data and maintaining compliance with industry standards.

## Prerequisites

Before proceeding with the installation, ensure that the following prerequisites are met:

1. **Kubernetes Cluster**: A running Kubernetes cluster with version >= 1.26 or higher.
2. **kubectl**: The Kubernetes command-line tool installed and configured to interact with your cluster.
3. **Helm**: The package manager for Kubernetes, installed and configured.
4. (Optional) **Cilium CLI** The [Cilium command-line]((https://github.com/cilium/cilium/releases)) interface installed >= v1.15.0

## Step-by-Step Installation Guide

### Step 1: Add the Cilium Helm Repository

First, add the Cilium Helm repository to your Helm configuration:

```sh
helm repo add cilium https://helm.cilium.io/
helm repo update
```

### Step 2: Install Cilium

Next, install Cilium using Helm. This command will deploy Cilium v1.15 into your Kubernetes cluster:

```sh
helm install cilium cilium/cilium --version 1.15.0 \
--namespace kube-system \
--set kubeProxyReplacement=strict \
--set autoDirectNodeRoutes=true \
--set encryption.enabled=true \
--set encryption.type=wireguard
```

### Step 3: Verify the Installation

To ensure that Cilium has been installed correctly, check the status of the Cilium pods:

```sh
kubectl get pods -n kube-system -l k8s-app=cilium
```

All Cilium pods should be in the `Running` state. Additionally, you can verify the Cilium status by running:

```sh
kubectl exec -n kube-system $(kubectl get pods -n kube-system -l k8s-app=cilium -o jsonpath='{.items[0].metadata.name}') -- cilium status
```

### Step 4: Enable WireGuard Encryption

WireGuard is a modern, high-performance VPN protocol that provides state-of-the-art encryption. To enable WireGuard encryption in Cilium, ensure that the `encryption.type` parameter is set to `wireguard` during the Helm installation (as shown in Step 2).

You can verify that WireGuard encryption is enabled by checking the Cilium configuration:

```sh
kubectl exec -n kube-system $(kubectl get pods -n kube-system -l k8s-app=cilium -o jsonpath='{.items[0].metadata.name}') -- cilium config | grep Encryption
```

The output should indicate that WireGuard encryption is enabled.

## Conclusion

By following this guide, you have successfully installed Cilium v1.15 on your Kubernetes cluster and enabled WireGuard encryption. This setup not only enhances the security of your network communications but also ensures that your organisation is well-equipped to handle the evolving security landscape. The integration of Cilium with WireGuard provides a robust, scalable, and secure networking solution, empowering your technical teams to focus on delivering value while maintaining the highest standards of security.

For further information and advanced configurations, please refer to the [Cilium documentation](https://docs.cilium.io/).
