# Configuring AWS ALB Ingress Controller on Self-Hosted Canonical Kubernetes

---

## Prerequisites

- Running Kubernetes cluster (Canonical / snap)
- EC2 instances as Kubernetes nodes
- AWS CLI installed and configured
- `kubectl` access to the cluster
- Helm v3 installed
- IAM permissions to create:
  - IAM policies and roles
  - EC2 tags
  - Security group rules
  - ALBs and Target Groups


---
## Phase 1: Infrastructure Setup

### 1.1 Configure Environment Variables

```bash
export AWS_REGION="ap-southeast-2"
export CLUSTER_NAME="test-k8s-cluster"
export INSTANCE_ID="i-xxxxxxxxx"
export VPC_ID="vpc-xxxxxxxx"
```

### 1.2 Tag Subnets for ALB Discovery

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].[SubnetId,AvailabilityZone,CidrBlock]' \
  --output table
```

# Set at least two subnets in different AZs:
```bash
export SUBNET_1="subnet-xxxxxxxx"
export SUBNET_2="subnet-yyyyyyyy"
```

Tag them:
```bash
aws ec2 create-tags \
  --resources $SUBNET_1 $SUBNET_2 \
  --tags \
    Key=kubernetes.io/cluster/$CLUSTER_NAME,Value=shared \
    Key=kubernetes.io/role/elb,Value=1 \
  --region $AWS_REGION

# Verify:
aws ec2 describe-subnets \
  --subnet-ids $SUBNET_1 $SUBNET_2 \
  --query 'Subnets[*].[SubnetId,Tags[?Key==`kubernetes.io/role/elb`].Value|[0]]' \
  --output table
```

### 1.3 Tag EC2 Instances
```bash
# Tag ALL Kubernetes worker nodes
aws ec2 create-tags \
  --resources $INSTANCE_ID \
  --tags Key=kubernetes.io/cluster/$CLUSTER_NAME,Value=owned \
  --region $AWS_REGION
```

### 1.4 Configure Security Group
```bash
export SG_ID="sg-xxxxxxxx"  # Your security group ID 

# Add required ports to your security group:
# HTTP
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0 --region $AWS_REGION

# HTTPS
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID --protocol tcp --port 443 --cidr 0.0.0.0/0 --region $AWS_REGION

# NodePort range (CRITICAL)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID --protocol tcp --port 30000-32767 --cidr 0.0.0.0/0 --region $AWS_REGION
# Internal traffic
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID --protocol -1 --source-group $SG_ID --region $AWS_REGION
```

### 1.5 Configure Instance Metadata Service (IMDSv2)
The controller requires IMDSv2 access.

```bash
# Check current IMDS configuration:
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].MetadataOptions' \
  --region $AWS_REGION

# If not configured, set IMDS with the proper hop limit:
aws ec2 modify-instance-metadata-options \
  --instance-id $INSTANCE_ID \
  --http-tokens required \
  --http-put-response-hop-limit 2 \
  --region $AWS_REGION
```

### 1.6 Verify IMDSv2 from the Node
```bash
# IMDSv2 test
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```
Both commands should return your instance ID.

---

## Phase 2: IAM Configuration

### 2.1 Download IAM Policy
Download the official AWS Load Balancer Controller IAM policy:
```bash
curl -o alb-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```
### 2.2 Create IAM Policy
Download the official AWS Load Balancer Controller IAM policy:
```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://alb-iam-policy.json

# Save the policy ARN
export ALB_POLICY_ARN=$(aws iam list-policies \
  --query 'Policies[?PolicyName==`AWSLoadBalancerControllerIAMPolicy`].Arn' \
  --output text)


echo "Policy ARN: $ALB_POLICY_ARN"
```

### 2.3 Create IAM Role and Attach ALB controller policy
```bash
# Trust policy:
cat > ec2-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Create IAM role
aws iam create-role \
  --role-name K8sALBControllerRole \
  --assume-role-policy-document file://ec2-trust-policy.json

# Attach ALB controller policy to role
aws iam attach-role-policy \
  --role-name K8sALBControllerRole \
  --policy-arn $ALB_POLICY_ARN

# Attach EC2 read permissions (optional)
aws iam attach-role-policy \
  --role-name K8sALBControllerRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
```


### 2.4 Create Instance Profile and add role to profile
```bash 
# Create instance profile
aws iam create-instance-profile \
  --instance-profile-name K8sALBControllerInstanceProfile

# Add role to profile
aws iam add-role-to-instance-profile \
  --instance-profile-name K8sALBControllerInstanceProfile \
  --role-name K8sALBControllerRole

```

### 2.5 Attach IAM Role to EC2
```bash
aws ec2 associate-iam-instance-profile \
  --instance-id $INSTANCE_ID \
  --iam-instance-profile Name=K8sALBControllerInstanceProfile \
  --region $AWS_REGION
```

### 2.6 Verify IAM Role from EC2
```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```
Expected: K8sALBControllerRole


---

## Phase 3: Kubernetes Node Configuration
### 3.1 Add providerID to ALL Nodes
SSH to EACH Kubernetes node and run:
```bash
# Get instance metadata using IMDSv2
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/instance-id)

AZ=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/placement/availability-zone)


# Get node name
NODE_NAME=$(kubectl get nodes -o wide | grep $(hostname -I | awk '{print $1}') | awk '{print $1}')

echo "Instance ID: $INSTANCE_ID"
echo "Availability Zone: $AZ"
echo "Node Name: $NODE_NAME"

# Add providerID
kubectl patch node $NODE_NAME \
  -p "{\"spec\":{\"providerID\":\"aws:///$AZ/$INSTANCE_ID\"}}"
```

### 3.2 Verify on all nodes:
```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,PROVIDER_ID:.spec.providerID 

```

## Phase 4: Install cert-manager
```bash
# Download  cert-manager manifests
curl -L -o cert-manager.yaml https://github.com/cert-manager/cert-manager/releases/download/v1.14.6/cert-manager.yaml

# Apply cert-manager manifests
kubectl apply -f cert-manager.yaml

# Verify cert-manager pods:
kubectl get pods -n cert-manager
```
---

## Phase 5: Install AWS Load Balancer Controller
### 5.1 Add Helm Repository
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```
### 5.2 Install Controller
```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  -f aws-lb-controller-values.yaml
```
hostNetwork=true is required for self-hosted Kubernetes to allow controller pods to access IMDS.

### 5.3 Verify Controller
```bash
# Check deployment
kubectl get deployment -n kube-system aws-load-balancer-controller

# Check pods
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# Check logs for errors
kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=50
```

### 5.4 Create IngressClass
```bash
cat <<EOF > alb-ingress-class.yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: ingress.k8s.aws/alb
EOF

#Apply the manifest:
kubectl apply -f alb-ingress-class.yaml 

# Verify 
kubectl get ingressclass
```
---


## Phase 6: Deploy Test Application
### 6.1 Create Demo Application
```bash
# Create namespace
kubectl create namespace demo


# Create deployment and service
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: demo
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
EOF


# Apply the manifest:
kubectl apply -f deployment.yaml 

# Verify:
kubectl get pods -n demo
kubectl get svc -n demo


```


### 6.2 Create Ingress
```bash
cat <<EOF > ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: demo
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/load-balancer-name: demo-aws-elb
    alb.ingress.kubernetes.io/target-type: instance
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "30"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "2"
    alb.ingress.kubernetes.io/success-codes: "200-499"
    alb.ingress.kubernetes.io/tags: Environment=demo,Application=nginx
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
EOF 


# Apply the manifest:
kubectl apply -f ingress.yaml


# Verify:
kubectl get ingress -n demo
```

### 6.3 Verify ALB and Targets
```bash
# List ALBs
aws elbv2 describe-load-balancers \
  --region $AWS_REGION \
  --query 'LoadBalancers[*].{Name:LoadBalancerName,DNS:DNSName,State:State.Code}' \
  --output table


# Get target group
TG_ARN=$(aws elbv2 describe-target-groups \
  --region $AWS_REGION \
  --query 'TargetGroups[?contains(TargetGroupName, `k8s-demo-nginx`)].TargetGroupArn' \
  --output text)


# Check target health
aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN \
  --region $AWS_REGION

```



### 6.3 Test Application
```bash
# Get ALB DNS name
ALB_DNS=$(kubectl get ingress testing-nginx-ingress -n demo -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')


echo "ALB URL: http://$ALB_DNS"


# Test with curl
curl http://$ALB_DNS
```
Expected result: NGINX welcome page

---


## References
https://aws.amazon.com/blogs/containers/exposing-kubernetes-applications-part-2-aws-load-balancer-controller/

https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.7/deploy/installation/


https://kubernetes-sigs.github.io/aws-load-balancer-controller/v1.1/


https://okeyebereblessing.medium.com/configuring-an-ingress-controller-with-aws-load-balancer-in-kubernetes-kubeadm-2a0f625e9952


https://devopscube.com/aws-cloud-controller-manager/

https://medium.com/@anil.goyal0057/understanding-aws-alb-ingress-controller-in-kubernetes-end-to-end-request-flow-f8b8a7435559



