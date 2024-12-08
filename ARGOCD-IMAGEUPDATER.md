Here's a comprehensive **`README.md`** that combines all the steps for setting up Argo CD Image Updater with ECR and EKS, tailored to your configuration.

---

# **Argo CD Image Updater with ECR and EKS**

This guide walks you through configuring **Argo CD Image Updater** to automatically update your Kubernetes applications deployed on **Amazon EKS** with images stored in **Amazon ECR**. The images are tagged using GitHub commit hashes for traceability.

## **Prerequisites**

1. **Amazon EKS Cluster** set up and running.
2. **Argo CD** installed on the EKS cluster.
3. **Argo CD Image Updater** installed.
4. **AWS CLI** configured with appropriate permissions.
5. **Amazon ECR** repository to store your images.
6. **GitHub Actions Workflow** to build and push images with commit hash tags.

---

## **Step 1: Install Argo CD Image Updater**

Install Argo CD Image Updater in the `argocd` namespace:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/main/manifests/install.yaml
```

Verify the installation:

```bash
kubectl get pods -n argocd
```

---

## **Step 2: Create an ECR Pull Secret**

To allow Argo CD Image Updater to pull images from ECR, create a Kubernetes pull secret.

### **Generate the Secret**

```bash
kubectl -n argocd create secret docker-registry ecr-secret \
  --docker-server=<aws-account-id>.dkr.ecr.<region>.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region <region>)"
```

Replace the placeholders:

- **`<aws-account-id>`**: Your 12-digit AWS account ID.
- **`<region>`**: Your AWS region (e.g., `us-east-2`).

### **Verify the Secret**

```bash
kubectl get secret ecr-secret -n argocd
```

---

## **Step 3: Configure Argo CD Application**

Create an Argo CD application manifest to deploy your application and enable automatic image updates.

### **`application.yaml`**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flask-app
  namespace: argocd
  annotations:
   ##ARGOCD Image Updater config
   argocd-image-updater.argoproj.io/image-list: alias=149536489720.dkr.ecr.us-east-2.amazonaws.com/python-flask-app
   argocd-image-updater.argoproj.io/alias.update-strategy: latest
   argocd-image-updater.argoproj.io/alias.allow-tags: '^[a-f0-9]{7}$' # Matches commit hash-based tags (e.g., abc1234)
   argocd-image-updater.argoproj.io/alias.pull-interval: 1m           # Checks for updates every minute
   argocd-image-updater.argoproj.io/alias.pull-secret: ecr-secret     # ECR pull secret
   
spec:
  project: default
  source:
    repoURL: 'https://github.com/MGK91/gitops-sample'
    targetRevision: HEAD
    path: flask-app/  # Path to the Helm chart in your repository
    helm:
      valueFiles:
        - values.yaml  # You can specify a custom values file if needed
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default  # Namespace to deploy the app to
  syncPolicy:
    automated:
      prune: true   # Automatically remove resources no longer tracked by Helm chart
      selfHeal: true  # Automatically sync the app if the state is changed manually
    syncOptions:
      - CreateNamespace=true  # Automatically create the namespace if it doesn't exist
```

Replace the placeholders:

- **`<aws-account-id>`**: Your AWS account ID.
- **`<region>`**: Your AWS region.
- **`<your-org>/<your-repo>`**: Your GitHub organization/repository.
- **`my-container`**: The container name in your Kubernetes `Deployment`.

---

## **Step 4: GitHub Actions Workflow to Push Images**

Create a GitHub Actions workflow (`.github/workflows/deploy.yml`) to build and push Docker images to ECR with the commit hash as the tag.

### **`deploy.yml`**

```yaml
name: Build and Deploy to ECR

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-2

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build, Tag, and Push Image to ECR
        run: |
          IMAGE_TAG=$(git rev-parse --short HEAD)
          aws_account_id=<accountid>
          region=<region>
          repository=<ecr-private-repo>

          docker build -t $aws_account_id.dkr.ecr.$region.amazonaws.com/$repository:$IMAGE_TAG .
          docker push $aws_account_id.dkr.ecr.$region.amazonaws.com/$repository:$IMAGE_TAG
```

### **Secrets Required in GitHub**

Add the following secrets to your GitHub repository:

- **`AWS_ACCESS_KEY_ID`**
- **`AWS_SECRET_ACCESS_KEY`**

---

## **Step 5: Deploy the Application**

Apply the Argo CD application manifest:

```bash
kubectl apply -f application.yaml -n argocd
```

---

## **Step 6: Verify Image Updates**

1. Check the Argo CD Image Updater logs to verify it detects new images:

    ```bash
    kubectl logs -n argocd deployment/argocd-image-updater
    ```

2. In the Argo CD UI, ensure the application reflects the latest commit-hash-based image.

---

## **Summary**

This guide sets up:

1. **Argo CD Image Updater** for automatic image updates.
2. **ECR Pull Secret** for secure authentication.
3. **GitHub Actions** to build and push images tagged with commit hashes.
4. **Argo CD Application** with annotations for dynamic image updates.

Now, every time you push to the `main` branch, the GitHub Actions workflow will build a new image, push it to ECR, and Argo CD Image Updater will automatically deploy the latest image to your EKS cluster.


