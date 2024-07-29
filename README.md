# Hosting a Dynamic Website on AWS Using Docker, EKS, and ECR

This project demonstrates how to host a dynamic website on AWS using Docker containers, an EKS cluster, and ECR. It includes creating AWS infrastructure, configuring Kubernetes resources, and deploying the application. The reference diagram and configuration files are available in the [GitHub repository](https://github.com/galkini/host-dynamic-website-eks).

## Prerequisites

1. **Create AWS Infrastructure**:
   - Set up a three-tier VPC with Internet Gateway, NAT Gateways, public and private route tables, and security groups.

2. **Docker Image and RDS Snapshot**:
   - Use an existing Docker image stored in ECR.
   - Use a database created with RDS snapshot from a previous project.

3. **GitHub Repositories**:
   - Create private repositories to store Kubernetes manifest files and EKS cluster configuration scripts.

4. **Install Tools**:
   - Install Chocolatey, kubectl, eksctl, and Helm on your PC.

5. **Create AWS Secrets Manager Secret**:
   - Store environment variables in AWS Secrets Manager.
   - Create an IAM policy to allow reading the secret.

## Steps to Deploy

### Step 1: Creating AWS Infrastructure

1. **Three-Tier VPC**:
   - Create a three-tier VPC with public and private subnets in two availability zones.

2. **Internet Gateway and NAT Gateway**:
   - Create an Internet Gateway for communication between VPC resources and the Internet.
   - Create NAT Gateways to allow private subnets to access the Internet.

3. **Security Groups**:
   - Create security groups to control inbound and outbound traffic to and from your resources.

### Step 2: Using Image from ECR and RDS Snapshot

1. **ECR Docker Image**:
   - Use a Docker image that is already stored in Amazon ECR.

2. **RDS Snapshot**:
   - Use an RDS snapshot to initialize the database for the website.

### Step 3: Configuring GitHub Repositories

1. **Create Repositories**:
   - Create private repositories in GitHub to store Kubernetes manifest files and EKS cluster configuration scripts.
   - Sync these repositories with your local PC.

### Step 4: Setting Up Tools

1. **Install Required Tools**:
   - Install Chocolatey, kubectl, eksctl, and Helm on your PC for managing Kubernetes resources.

### Step 5: AWS Secrets Manager and IAM Policies

1. **Create Secret**:
   - Create a secret in AWS Secrets Manager to store environment variables.

2. **IAM Policy for Secrets Manager**:
   - Create an IAM policy that allows access to the secret for the EKS cluster.

### Step 6: Creating EKS Cluster and IAM Roles

1. **EKS Cluster IAM Role**:
   - Create an IAM role for the EKS cluster.

2. **Create EKS Cluster**:
   - Create the EKS cluster and configure IAM access.

3. **Modify RDS Security Group**:
   - Update the RDS security group to allow access from the EKS cluster.

### Step 7: Setting Up Node Group and IAM Roles

1. **Worker Node IAM Role**:
   - Create an IAM role for the EKS worker nodes.

2. **Create Node Group**:
   - Create a node group with the required EC2 instances under EKS.

### Step 8: Kubernetes Manifest Files

1. **Create Namespace**:
   - Create a namespace to logically group all EKS resources.

    ```yaml
    $REGION = "us-east-1"
    $CLUSTER_NAME = "rentzone-cluster"

    # This command updates the kubeconfig file with the configuration necessary to connect to an AWS EKS cluster.
    aws eks --region $REGION update-kubeconfig --name $CLUSTER_NAME

    # List all contexts
    kubectl config get-contexts

    # Set the current context to the specified EKS cluster
    kubectl config use-context "arn:aws:eks:$REGION:$AWS_ACCOUNT_ID:cluster/$CLUSTER_NAME"

    # Create a namespace
    $NAMESPACE = "rentzone"
    kubectl create namespace $NAMESPACE

    # Permanently set the namespace for all subsequent kubectl commands
    kubectl config set-context --current --namespace=$NAMESPACE

    # Verify the namespace
    kubectl config view --minify | Select-String 'namespace:'
    ```

### Step 9: Secrets Store CSI Driver

1. **Install Secrets Store CSI Driver**:
   - Install the Secrets Store CSI Driver using Helm.

    ```sh
    helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
    helm repo update
    helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system
    ```

2. **Verify Installation**:
   - Verify that the Secrets Store CSI Driver has been deployed successfully.

    ```sh
    kubectl --namespace=kube-system get pods -l "app=secrets-store-csi-driver"
    ```

3. **Apply AWS Provider**:
   - Apply the AWS provider for the Secrets Store CSI Driver.

    ```sh
    kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
    ```

### Step 10: Configuring IAM OIDC Identity Provider

1. **Associate IAM OIDC Identity Provider**:

    ```sh
    eksctl utils associate-iam-oidc-provider --region=$REGION --cluster=$CLUSTER_NAME --approve
    ```

2. **Create IAM Service Account**:
    - Create a new IAM service account for the EKS cluster.

    ```sh
    eksctl create iamserviceaccount `
        --name $SERVICE_ACCOUNT_NAME `
        --namespace $NAMESPACE `
        --region $REGION `
        --cluster $CLUSTER_NAME `
        --attach-policy-arn "arn:aws:iam::$AWS_ACCOUNT_ID:policy/$SECRET_MANAGER_ACCESS_POLICY_NAME" `
        --approve `
        --override-existing-serviceaccounts
    ```

3. **Apply Secret Provider Class**:

    ```sh
    kubectl apply -f $SECRET_PROVIDER_CLASS_FILE_NAME
    ```

4. **Apply Deployment and Service Manifests**:

    ```sh
    kubectl apply -f $DEPLOYMENT_FILE_NAME
    kubectl apply -f $SERVICE_FILE_NAME
    ```

5. **Verify Deployment and Service**:
   - Verify that the pod is running and the service is created correctly.

    ```sh
    kubectl get pods --namespace $NAMESPACE
    kubectl get svc --namespace $NAMESPACE
    ```

### Kubernetes Manifest Files

#### secret-provider-class.yaml
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: rentzone-secret
  namespace: rentzone
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: <arn-of-the-secret-you-created-in-secret-manager>
        objectAlias: "app-secret"
```

#### deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rentzone-deployment
  namespace: rentzone
  labels:
    app: rentzone
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rentzone
  template:
    metadata:
      labels:
        app: rentzone
    spec:
      serviceAccountName: rentzone-service-account
      volumes:
      - name: secrets-store-inline
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "rentzone-secret"
      containers:
      - name: rentzone
        image: <the uri of your docker image>
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        volumeMounts:
        - name: secrets-store-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
```

#### service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: rentzone
  namespace: rentzone
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-internal: "false"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  ports:
    - name: web
      port: 80
  selector:
    app: rentzone
```

### Additional Steps

1. **Add TLS Listener to the Network Load Balancer**:
   - Configure TLS for secure communication using the Network Load Balancer.

2. **Add A Record in Route 53**:
   - Create an A record in Route 53 to point to the Network Load Balancer.

## Conclusion

By following the steps outlined above, this project successfully hosted a dynamic website on AWS using Docker, EKS, and ECR. The setup leverages various AWS services to ensure high availability, security, and efficient resource management
