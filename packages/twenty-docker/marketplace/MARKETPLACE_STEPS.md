# Step-by-Step Guide for GCP and AWS Marketplace Listing

This document details the latest technical, business, and administrative procedures required to list Twenty CRM on the Google Cloud (GCP) Marketplace and Amazon Web Services (AWS) Marketplace.

---

## Part 1: Google Cloud (GCP) Marketplace Integration

GCP Marketplace supports Kubernetes applications deployable via Helm charts using a **Deployer Container** model. The deployer container is a GKE Job that manages the installation of your Helm templates.

### Technical & Administrative Steps

#### Step 1.1: Business Enrollment & Project Setup
1. Enroll in the **Google Cloud Partner Advantage** program.
2. Create a dedicated publication project in the GCP Console (e.g., `twenty-marketplace-pub`).
3. Link a GCP Billing Account to the publishing project.
4. Enable the **Cloud Commerce Partner Procurement API** via the API Library.
5. Gain access to the **Google Cloud Producer Portal**.

#### Step 1.2: Build the Deployer Container Image
GCP Marketplace requires a deployer image wrapping your Helm chart.
1. Download Google’s [Marketplace Kubernetes App Tools (`mpdev`)](https://github.com/GoogleCloudPlatform/marketplace-k8s-app-tools):
   ```bash
   git clone https://github.com/GoogleCloudPlatform/marketplace-k8s-app-tools.git
   cd marketplace-k8s-app-tools
   ```
2. Create a deployer directory structure inside `packages/twenty-docker/marketplace/gcp-deployer/`:
   - `deployer/`: Contains your Helm chart packaged under a specific subdirectory.
   - `schema.yaml`: Defines configuration parameters that customers configure during installation.
   - `application.yaml`: Defines the GKE Application Custom Resource (CRD) descriptor.
   - `Dockerfile`: Wraps the Google Marketplace base deployer image.

**Example `schema.yaml`:**
```yaml
properties:
  server.ingress.hosts[0].host:
    type: string
    title: Host Domain
    description: Domain name for your CRM instance
    default: crm.example.com
  storage.type:
    type: string
    title: Storage Type
    description: Backend storage to use (local or s3)
    enum:
      - local
      - s3
    default: local
```

**Example `Dockerfile` for Deployer:**
```dockerfile
FROM gcr.io/cloud-marketplace-tools/k8s/deployer_helm:latest
COPY chart/twenty /data-chart/twenty
COPY schema.yaml /data/schema.yaml
```

3. Build and test the deployer image locally using `mpdev`:
   ```bash
   mpdev build
   mpdev doctor
   ```

#### Step 1.3: Push Containers to Artifact Registry
1. In your GCP Publisher project, create a public repository in **Artifact Registry**:
   ```bash
   gcloud artifacts repositories create twenty-marketplace \
     --repository-format=docker \
     --location=us-central1 \
     --description="GCP Marketplace Container Registry for Twenty"
   ```
2. Tag and push your application images (`twentycrm/twenty`, deployer image, internal databases) using GCP naming rules:
   ```bash
   # Tag deployer
   docker tag gcp-deployer:latest us-central1-docker.pkg.dev/twenty-marketplace-pub/twenty-marketplace/deployer:latest
   # Push deployer
   docker push us-central1-docker.pkg.dev/twenty-marketplace-pub/twenty-marketplace/deployer:latest
   ```

#### Step 1.4: Validation & Submission
1. Test GKE deployment using the `mpdev` test framework:
   ```bash
   mpdev install --deployer=us-central1-docker.pkg.dev/twenty-marketplace-pub/twenty-marketplace/deployer:latest
   ```
2. In the GCP Producer Portal, create a new product submission under **Kubernetes Apps**.
3. Link the Artifact Registry path of your deployer image.
4. Input marketing details (descriptions, logo, support URL, EULA).
5. Submit for automated vulnerability scans and technical review.

---

## Part 2: Amazon Web Services (AWS) Marketplace EKS Integration

AWS Marketplace distributes Kubernetes applications by hosting OCI-compliant Helm charts and container images directly inside **AWS Marketplace private ECR repositories**.

### Technical & Administrative Steps

#### Step 2.1: Register as an AWS Marketplace Seller
1. Go to the [AWS Marketplace Management Portal (AMMP)](https://aws.amazon.com/marketplace/management-portal).
2. Register as a Seller. Provide company tax profiles, bank accounts (necessary for identity verification), and support contacts.

#### Step 2.2: Prepare Container Repositories
AWS does not allow you to reference external public images (like Docker Hub) inside your Helm chart for marketplace products. All images must be hosted in AWS-provided ECR repositories.
1. Create a container product in the AMMP.
2. AWS will generate private ECR repository URIs specifically for your product version (e.g., `709825985650.dkr.ecr.us-east-1.amazonaws.com/your-org/twenty-app`).
3. Pull your official images and push them to these ECR targets:
   ```bash
   # Log in to AWS Marketplace ECR
   aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 709825985650.dkr.ecr.us-east-1.amazonaws.com
   
   # Retag and push Twenty server/worker container
   docker pull twentycrm/twenty:latest
   docker tag twentycrm/twenty:latest 709825985650.dkr.ecr.us-east-1.amazonaws.com/your-org/twenty-app:latest
   docker push 709825985650.dkr.ecr.us-east-1.amazonaws.com/your-org/twenty-app:latest
   ```

#### Step 2.3: Re-map Helm Chart Images
1. Update `packages/twenty-docker/helm/twenty/values.yaml` image tags to point to ECR paths:
   ```yaml
   image:
     repository: 709825985650.dkr.ecr.us-east-1.amazonaws.com/your-org/twenty-app
     tag: latest
   ```
2. Package the Helm chart:
   ```bash
   helm package packages/twenty-docker/helm/twenty --destination build/
   ```
3. Push the packaged Helm chart as an OCI artifact to ECR:
   ```bash
   helm push build/twenty-0.1.0.tgz oci://709825985650.dkr.ecr.us-east-1.amazonaws.com/your-org/twenty-chart
   ```

#### Step 2.4: Configure AMMP Listing
1. In AWS Seller Central, select **Add Container Delivery Option**.
2. Set delivery type to **Helm Chart**.
3. Reference the Helm OCI URI you pushed (`oci://709825985650.dkr.ecr...`).
4. Write clear usage/installation instructions for buyers:
   ```markdown
   ### Installation Instructions
   1. Authenticate Helm with AWS Marketplace ECR:
      `aws ecr get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin 709825985650.dkr.ecr.us-east-1.amazonaws.com`
   2. Deploy to your EKS cluster:
      `helm install my-twenty oci://709825985650.dkr.ecr.us-east-1.amazonaws.com/your-org/twenty-chart --version 0.1.0`
   ```

#### Step 2.5: Limited Visibility Testing & Publishing
1. Submit your draft listing. AWS Marketplace will compile your listing and publish it with **Limited** visibility.
2. AWS automatically runs static container scanning. You must fix any critical or high vulnerabilities identified.
3. Deploy the Helm chart to a test Amazon EKS cluster from the Limited preview page to verify it installs, connects, and starts successfully.
4. Approve the release. AWS will mark the listing as **Public**, making it searchable on the AWS Marketplace web portal.

---

## Part 3: Mandatory Pre-Publishing Checklist

Before submitting Twenty to either cloud portal, complete these tasks:

### 1. Security Compliance & Image Scanning
Cloud marketplaces will automatically reject container images containing critical or unresolved high security vulnerabilities.
* Run local scans using `trivy` before pushing images:
  ```bash
  trivy image twentycrm/twenty:latest
  ```
* Resolve all OS package security issues in the Dockerfiles before submission.

### 2. AWS IAM / Kubernetes Least Privilege
* Marketplaces reject pods running with cluster-admin roles. Ensure the Helm chart relies on standard ServiceAccounts with minimized RBAC permissions.
* AWS Marketplace requires integration with EKS **IAM Roles for Service Accounts (IRSA)** for cloud service access (e.g. S3 uploads) rather than hardcoded credentials.

### 3. Pricing Model Decision
* **BYOL (Bring Your Own License) / Free Tier (Recommended)**: The easiest listing format. Bypasses the need for any marketplace billing code integrations.
* **Metered Billing (Future State)**: Charges users based on active users or database size. Requires integrating the **AWS Marketplace Metering Service API** or the **GCP Service Control API** directly into the Twenty server application code to report usage periodically.
