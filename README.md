<div align="center">
  <img src="https://github.com/soravkumarsharma/Blood-Bank-Management-System-Infra/assets/77971771/8ef12cd0-4d48-4e3b-a7e3-11a9c52f4ef1" alt="logo" width="60" height="60">
  <h2>Deploy the BBMS on an Amazon EKS Cluster</h2>
  <h3>|| <ins>Setup Guide</ins> ||</h3>
</div>

### Prerequisites:
- Ubuntu Instance
  - Instance Type: ***t2.medium***
  - Root Volume: ***20GiB***
  - Region: ***us-east-1***
  - IAM Role:
    - Create IAM Role with Administrator policy, and attach that role with an EC2 Instance profile.
  
- Install <ins>**AWS CLI**</ins>, <ins>**eksctl**</ins>, <ins>**Kubectl**</ins>, <ins>**Kubectl argo rollouts**</ins> & <ins>**Helm**</ins>

### ***Step*** 1 : Updates the package list.
```
sudo apt update
```

### ***Step*** 2 : Fork & Clone this repository.
```
git clone https://github.com/soravkumarsharma/Blood-Bank-Management-System-Infra.git
```
### ***Step*** 3 : Change the directory.
```
cd Blood-Bank-Management-System-Infra
```

### ***Step*** 4 : Make a script file executable.
```
sudo chmod +x install.sh
```
### ***Step*** 5 : Run the Script.
```
./install.sh
```

### ***Step*** 6 : Check the AWS CLI package Version
```
aws --version | cut -d ' ' -f1 | cut -d '/' -f2
```

### ***Step*** 7 : Check the eksctl package Version
```
eksctl version
```

### ***Step*** 8 : Check the kubectl package Version
```
kubectl version --client | grep 'Client' | cut -d ' ' -f3
```
### ***Step*** 9 : Check the kubectl argo rollouts package Version
```
kubectl argo rollouts version | grep 'kubectl-argo-rollouts:' | cut -d ' ' -f2
```

### ***Step*** 10 : Check the Helm package Version
```
helm version | cut -d '"' -f2
```

### ***Step*** 11 : Create an IAM policy for the AWS Load Balancer Controller that allows it to make calls to AWS APIs on your behalf.
```
cd EKS-Cluster
```
```
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
```
### ***Step*** 12 : Retrieve the AWS account ID of the currently authenticated user.
```
ACC_ID=$(aws sts get-caller-identity | grep 'Account' | cut -d '"' -f4)
```
### ***Step*** 13 : Replace the Account ID.
```
sed -i "s/898855110204/${ACC_ID}/g" cluster.yml
```
### ***Step*** 14 : Create an EKS cluster. 
 - Note: wait for 10-15 minutes for the provisioning process to complete
```
eksctl create cluster -f cluster.yml
```
### ***Step*** 15 : Check all the worker nodes. 
```
kubectl get nodes
```
### ***Step*** 15 : Check Addons. 
```
kubectl get daemonset -n kube-system
```
### ***Step*** 16 : Add the eks-charts to the helm repository.
```
helm repo add eks https://aws.github.io/eks-charts
```
### ***Step*** 17 : Check the repo is added or not.
```
helm repo list
```
### ***Step*** 18 : Update your local repo to make sure that you have the most recent charts.
```
helm repo update eks
```
### ***Step*** 19 : Before installing the aws load balancer controller, first apply the crds.yml file.
```
kubectl apply -f crds.yml
```
### ***Step*** 20 : Install the aws-load-balancer-controller.
```
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=blood-bank-prod \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```
### ***Step*** 21 : Verify that the controller is installed.
```
helm list --all-namespaces
```
### ***Step*** 22 : Change the Directory.
```
cd ../Kubernetes/php
```
### ***Step*** 23 : Create an EFS.
- Retrieve the AWS VPC ID.
```
VPC_ID=$(aws eks describe-cluster \
    --name blood-bank-prod \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --output text)
```
- Retrieve the AWS VPC CIDR Range.
```
CIDR_RANGE=$(aws ec2 describe-vpcs \
    --vpc-ids $VPC_ID \
    --query "Vpcs[].CidrBlock" \
    --output text \
    --region us-east-1)
```
- Create Security Group for the EFS.
```
SG_ID=$(aws ec2 create-security-group \
    --group-name MyEfsSecurityGroup \
    --description "My EFS security group" \
    --vpc-id $VPC_ID \
    --output text)
```
- Add the Inbound rules into the Security Group.
```
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 2049 \
    --cidr $CIDR_RANGE
```
- Create the EFS.
```
FS_ID=$(aws efs create-file-system \
    --region us-east-1 \
    --performance-mode generalPurpose \
    --query 'FileSystemId' \
    --output text)
```
- Retrieve the Public Subnet ID.
```
aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query 'Subnets[?MapPublicIpOnLaunch==`true`].SubnetId' \
    --output text
```
- Mount the Public Subnet ID to the EFS.
```
aws efs create-mount-target \
    --file-system-id $FS_ID \
    --subnet-id subnet-EXAMPLEe2ba886490 \
    --security-groups $SG_ID
```
- Replace the old EFS ID with the new EFS ID.
```
sed -i "s/fs-0c3bf86e6fa1a57f6/${FS_ID}/g" pv.yml
```
- If you don't want to use Certificate Manger then remove the below annotation from the Ingress manifest.
```
  alb.ingress.kubernetes.io/certificate-arn: 
  alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
  alb.ingress.kubernetes.io/ssl-redirect: '443'
```
- Push the changes to the Git Repository.
```
git add .
git commit -m "manifest files modified"
git push origin main
```
- Change the Directory.
```
cd ../..
```
### ***Step*** 24 : Install ArgoCD.
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
- Expose the service of ArgoCD Server as a Classic Load Balancer.
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
- Retrieve the Load Balancer DNS.
```
kubectl get svc -n argocd
```
- Retrieve the ArgoCD Server initial password.
  - Login into the Server using the Load Balancer DNS url.
```
kubectl get secret/argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d; echo
```
### ***Step*** 25 : Install Argo Rollouts.
```
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```
- Before Apply the ApplicationSet manifest file to the kubernetes cluster, Firstly change the git repo url, and then Apply the manifest file.
```
kubectl apply -f ApplicationSet.yml
```
### ***Step*** 26 : Argo Rollouts Dashboard.
- Make sure that the inbound port 3100 is opened into the EC2 instance.
```
kubectl argo rollouts dashboard
```

![Untitled Diagram (1)](https://github.com/soravkumarsharma/Blood-Bank-Management-System/assets/77971771/4b1c1923-31b2-44cd-8840-ea6e0404269f)




