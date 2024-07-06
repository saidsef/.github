# Access GCP from AWS using Workload Identity Federation

## Prerequisites
Before proceeding with this guide, ensure you have the following:
- Access to both Google Cloud Platform (GCP) and Amazon Web Services (AWS) accounts.
- Installed and configured `gcloud` CLI.
- Installed and configured AWS CLI.
- Basic understanding of IAM roles and policies in both GCP and AWS.

## Introduction
Service account keys in GCP have long been a security concern. These keys can be hardcoded in source code, inadvertently pushed to public repositories, shared among users, and often lack expiration dates, making them a potential security risk. Traditionally, accessing GCP resources from outside the platform, such as in a multi-cloud environment or for SaaS providers running infrastructure on AWS, necessitated the use of these insecure service account keys.

Google has introduced a new service called Workload Identity Federation, currently in Beta, to address these issues. This service aims to eliminate the need for service account keys by providing ephemeral, short-lived credentials to access GCP services and resources from outside GCP. This guide will focus on configuring AWS applications to authenticate securely to GCP without the need for service account keys.

## Pros and Cons
### Pros
- **Enhanced Security**: Eliminates the risk associated with long-lived service account keys.
- **Simplified Management**: Reduces the complexity of key rotation and management.
- **Ephemeral Credentials**: Provides short-lived credentials, reducing the window of opportunity for misuse.

### Cons
- **Beta Service**: As the service is currently in Beta, it may have limited support and potential instability.
- **Initial Setup Complexity**: Requires initial configuration and understanding of both GCP and AWS IAM policies.

## Workload Identity Federation Overview
Workload Identity Federation consists of two main components: workload identity pools and workload identity providers.

- **Workload Identity Pools**: Logical containers for external identities (e.g., AWS roles, Azure managed identities).
- **Workload Identity Providers**: Entities containing metadata about the relationship between the external identity provider and GCP.

Additionally, attribute mappings are used to secure tokens by attaching metadata to the external identity token, ensuring that only approved external identities can use the tokens during the federation process.

## Accessing GCP from AWS
This section outlines the steps to configure AWS applications to authenticate to GCP without using service account keys. We will use a fictional application called “GCP bakery” running on an EC2 instance in AWS that copies files from a GCS bucket for analysis.

### Step 1: GCP Service Account Creation
Create a GCP service account that the AWS application will use to access GCS objects.

```bash
gcloud iam service-accounts create $SERVICE_ACCOUNT_ID \
--description="$DESCRIPTION" \
--display-name="$DISPLAY_NAME"
```

Example:
```bash
gcloud iam service-accounts create gcp-bakery \
--description="AWS application that copies GCS objects." \
--display-name="gcp-bakery"
```

Assign the Storage Object Viewer role to the GCP bucket for the newly created service account.

### Step 2: AWS IAM Role Creation
Create an AWS IAM role that the GCP bakery application will assume while running on an EC2 instance.

```bash
aws iam create-role --role-name $ROLE-NAME \
--assume-role-policy-document $FILE-LOCATION \
--description "$DESCRIPTION"
```

Example:
```bash
aws iam create-role --role-name "gcp-bakery" \
--assume-role-policy-document "/Users/bakery/assume.json" \
--description "Allows EC2 instances to copy GCS objects"
```

Attach an IAM policy to the newly created IAM role.

```bash
aws iam put-role-policy --role-name $ROLE-NAME \
--policy-name $EXAMPLE-POLICY \
--policy-document $FILE-LOCATION
```

Example:
```bash
aws iam put-role-policy --role-name "gcp-bakery" \
--policy-name "gcp-bakery-policy" \
--policy-document "/Users/bakery/role.json"
```

### Step 3: Workload Identity Pool Creation
Create a workload identity pool in GCP.

```bash
gcloud beta iam workload-identity-pools create $POOL-ID \
--location="global" \
--description="$DESCRIPTION" \
--display-name="DISPLAY-NAME"
```

Example:
```bash
gcloud beta iam workload-identity-pools create gcpbakerypool \
--location="global" \
--description="Workload identity pool for GCP bakery." \
--display-name="GCP bakery Pool"
```

### Step 4: Workload Identity Provider Creation
Create an AWS identity provider in GCP.

```bash
gcloud beta iam workload-identity-pools providers create-aws $PROVIDER_NAME \
--location="global" \
--workload-identity-pool="$VALUE_FROM_STEP_3" \
--display-name="$DISPLAY_NAME" --description="$DESCRIPTION" \
--attribute-condition="$EXAMPLE_CONDITION" \
--account-id=$AWS_ACCOUNT_ID
```

Example:
```bash
gcloud beta iam workload-identity-pools providers create-aws gcpbakeryprovide \
--location="global" \
--workload-identity-pool="gcpbakerypool" \
--display-name="GCP bakery AWS Provider" \
--description="The Identity Provider for GCP bakery" \
--attribute-condition="'arn:aws:sts::123456789012:assumed-role/gcp-bakery' == attribute.aws_role" \
--account-id=123456789012
```

### Step 5: GCP Service Account Impersonation Binding
Bind the IAM role `roles/iam.workloadIdentityUser` to the GCP service account.

```bash
gcloud iam service-accounts add-iam-policy-binding $workload_sa_email \
--role roles/iam.workloadIdentityUser \
--member "principalSet://iam.googleapis.com/projects/$gcp_project_number/locations/global/workloadIdentityPools/$workload_id/attribute.aws_role/arn:aws:sts::${aws_account_id}:assumed-role/$role_name"
```

Example:
```bash
gcloud iam service-accounts add-iam-policy-binding gcp-bakery@example-project.iam.gserviceaccount.com \
--role "roles/iam.workloadIdentityUser" \
--member "principalSet://iam.googleapis.com/projects/123456789012/locations/global/workloadIdentityPools/gcpbakerypool/attribute.aws_role/arn:aws:sts::123456789012:assumed-role/gcp-bakery"
```

### Step 6: Enable Required GCP Services
Enable the Google Security Token Service and the IAM credentials service in your GCP project.

```bash
gcloud services enable sts.googleapis.com
gcloud services enable iamcredentials.googleapis.com
```

## Workload Identity Federation Token Flow
To begin the token exchange process, generate temporary security credentials for AWS using the `AssumeRole` API. Using the credentials returned by `AssumeRole`, create a `GetCallerIdentity` token. Exchange this token for a GCP federated access token via an HTTP POST call to the STS URL: `https://sts.googleapis.com/v1beta/token`.

The GCP STS returns a federated access token, which can be exchanged for a service account access token via an HTTP POST to the Cloud IAM URL: `https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/$SA-NAME@$PROJECT-ID.iam.gserviceaccount.com:generateAccessToken`.

With the service account access token, the GCP bakery application can make calls to the GCS API without needing a static downloaded service account key.

Example:
```bash
curl -H "Authorization: Bearer $OAUTH_TOKEN" "https://storage.googleapis.com/storage/v1/b/$BUCKETNAME/o"
```

## Conclusion
Workload Identity Federation is a significant advancement for multi-cloud organisations using AWS and GCP. It mitigates the risks associated with static service account keys and simplifies credential management. By leveraging this service, organisations can enhance security, streamline operations, and ensure secure communication between AWS and GCP.
