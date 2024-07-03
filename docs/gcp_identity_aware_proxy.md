# Google Cloud Platform (GCP) Identity-Aware Proxy (IAP) Documentation

## Overview

Google Cloud Platform's Identity-Aware Proxy (IAP) provides a secure method to control access to your applications and VMs. By leveraging IAP, you can ensure that only authenticated and authorised users can access your resources, thereby enhancing security and compliance.

## Key Features

- **Centralised Access Control**: Manage access to applications and VMs from a single interface.
- **Granular Permissions**: Define access policies based on user identity and group membership.
- **Secure Connectivity**: Protect your resources from unauthorised access and potential threats.

## IAP for SSH

IAP for SSH allows you to securely connect to your VM instances without needing an external IP address. This reduces the attack surface and enhances security by routing SSH traffic through Google's infrastructure.

## Setting Up IAP

### Prerequisites

1. **GCP Project**: Ensure you have a GCP project set up.
2. **Billing Enabled**: Verify that billing is enabled for your project.
3. **IAM Roles**: Assign the necessary IAM roles (`IAP-Secured Tunnel User`, `Compute Viewer`, and `IAP-Secured Web App User`) to the users who will access the resources.

### Steps to Enable IAP

1. **Enable APIs**:
- Navigate to the GCP Console.
- Enable the "Identity-Aware Proxy" API.
- Enable the "Compute Engine" API.

2. **Configure OAuth Consent Screen**:
- Go to the "APIs & Services" > "OAuth consent screen".
- Configure the consent screen with the necessary details.

3. **Set Up IAP**:
- Navigate to the "Identity-Aware Proxy" section in the GCP Console.
- Select the resources (App Engine, Compute Engine, etc.) you wish to protect.
- Click "Enable IAP".

4. **Configure Access Policies**:
- Go to the "IAM & Admin" > "IAM".
- Add members and assign the `IAP-Secured Web App User` role to users who need access.

### Setting Up IAP for SSH

1. **Install gcloud CLI**:
- Ensure you have the `gcloud` CLI installed and configured.

2. **Create SSH Key**:
- Generate an SSH key pair if you do not already have one.

3. **Connect via IAP**:
- Use the following command to connect to your VM:
```bash
gcloud compute ssh [INSTANCE_NAME] --zone [ZONE] --tunnel-through-iap
```

## Validating IAP Setup

1. **Access Application**:
- Attempt to access the application or VM through the IAP-secured URL.
- Ensure you are prompted for authentication.

2. **Check Logs**:
- Navigate to "Logging" in the GCP Console.
- Verify that access logs reflect the correct user access and any denied attempts.

3. **Test SSH Connection**:
- Use the `gcloud compute ssh` command to connect to your VM.
- Confirm that the connection is established successfully through IAP.

## Conclusion

By implementing GCP's Identity-Aware Proxy, you can significantly enhance the security of your applications and VM instances. Follow the outlined steps to set up and validate IAP, ensuring that only authorised users have access to your critical resources.

For further details, refer to the [GCP IAP Documentation](https://cloud.google.com/iap/docs).
