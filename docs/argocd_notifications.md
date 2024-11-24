# Configuring Argo CD Notifications: Services, Templates, and Triggers

### Introduction

Argo CD Notifications enhances the Argo CD experience by providing real-time alerts based on application events. By configuring services, templates, and triggers, we can ensure our team stays informed about deployment statuses, failures, and other critical events. This guide offers a clear, step-by-step approach to setting up Argo CD Notifications, tailored for a technical audience familiar with Kubernetes and Argo CD.

### Prerequisites

Before we begin, ensure the following:

1. Kubernetes Cluster
2. Argo CD Installed is installed and configured
3. Argo CD Notifications is installed (we will be configuring this later)
4. Slack Token

### Architecture Diagram

![architecture diagram](./images/argocd-notifications-architecture-diagram.svg)

### Step-by-Step Configuration Guide

#### 1. Define Notification Services

Services determine where notifications are sent, such as Slack, email, or webhook endpoints. To configure a service, we'll need to update the `argocd-notifications-cm` ConfigMap.

**Example: Configuring Slack and Email Services**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  service.slack: |
    token: $slack-token
```

*Notes:*
- Replace `$slack_token` with the actual credentials or reference them securely using secrets.

#### 2. Create Notification Templates and Triggers

Templates define the content and format of the notifications. They can be customised for different services and event types.
Triggers specify the conditions under which notifications are sent. They link events to the appropriate templates and services.

**Example: Defining a Slack Template and Triggers**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  trigger.on-deleted: |
    - send: [app-deleted]
      when: app.metadata.deletionTimestamp != nil
      description: application has been deleted
  template.app-deleted: |
    message: |
      Application {{.app.metadata.name}} has been deleted.
    slack:
      attachemnts: |
        [{
          "title": "{{.app.metadata.name}}",
          "color": "#E96D76",
          "fields": [
          {
            "title": "Sync Status",
            "value": "{{.app.status.sync.status}}",
            "short": true
          },
          {
            "title": "Repository",
            "value": "{{.app.spec.source.repoURL}}",
            "short": true
          }]
        }]
      deliveryPolicy: Post
      groupingKey: "{{.app.status.sync.revision}}"
      notifyBroadcast: false
```

*Notes:*
- The `{{.app.metadata.name}}` and `{{.app.status.sync.status}}` placeholders will be replaced with actual application data at runtime.
- We can create multiple templates for different scenarios (e.g., failure, progress).
- Each trigger includes a `description`, the services to `send` notifications to, and the `template` to use.
- We can define multiple triggers based on different events and conditions.

#### 4. Apply the Configuration

After defining services, templates, and triggers, apply the ConfigMap to the Kubernetes cluster:

```bash
kubectl apply -f argocd-notifications-config.yaml
```

*Ensure that:*
- The `argocd-notifications-controller` is running and has access to the updated ConfigMap.
- Sensitive information is handled securely, possibly using Kubernetes Secrets.

#### 5. Verify the Setup

To confirm that notifications are functioning correctly:

1. **Trigger an Event**: For example, perform a sync operation on an application.
2. **Check Notification Channels**: Verify that the configured services (e.g., Slack channel, email inbox) receive the appropriate notifications.
3. **Review Logs**: Inspect the `argocd-notifications-controller` logs for any errors or confirmation messages.

### Conclusion

Setting up Argo CD Notifications with customised services, templates, and triggers enables our team to stay informed about critical application events seamlessly. By following this guide, we can enhance our deployment workflows, improve responsiveness to issues, and maintain a high level of operational awareness within our Kubernetes environments. Tailoring notifications to our organisation's needs ensures that the right information reaches the right people at the right time.
