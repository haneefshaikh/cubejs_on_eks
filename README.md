# CUBEJS DEPLOYMENT on AWS EKS CLUSTER

### 1.  INSTALL OR UPDATE EKSCTL ON LINUX.

a. Download and extract the latest release of `eksctl` with the following command.

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```
b. Move the extracted binary to `/usr/local/bin`.
```
sudo mv /tmp/eksctl /usr/local/bin
```
c. Test that your installation was successful with the following command.
```
eksctl version
```

b. Install kubectl binary for Kubernetes Client to communicate with the

Kubernetes API Server.
```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
kubectl version --client
```

c. Create first EKS cluster

`eks-cube-cluster.yaml`
```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eks-cube-test
  region: us-east-1

nodeGroups:
  - name: eks-cube-test-ng-01
    instanceType: t2.medium
    desiredCapacity: 2
    volumeSize: 20
    ssh:
      allow: true # will use ~/.ssh/id_rsa.pub as the default ssh key
      publicKeyPath: ~/keys/id_rsa.pub

  - name: eks-cube-test-ng-02
    instanceType: t2.small
    desiredCapacity: 2
    volumeSize: 20
    ssh:
      publicKeyPath: ~/keys/id_rsa.pub
```

d. Next, run this command:
```
eksctl create cluster -f eks-cube-cluster.yaml
```

e. to get the kube-config to access the EKS cluster
```
aws eks --region us-east-1 update-kubeconfig --name eks-cube-test
-
Added new context
arn:aws:eks:us-east-1:509002973204:cluster/eks-cube-dev to
/Users/haneefshaikh/.kube/config
try accessing the EKS cluster
```
```
kubectl get nodes
-
NAME STATUS ROLES AGE VERSION
ip-192-168-17-221.ec2.internal Ready <none> 5m49s
v1.24.9-eks-49d8fe8
ip-192-168-27-243.ec2.internal Ready <none> 4m53s
v1.24.9-eks-49d8fe8
ip-192-168-44-137.ec2.internal Ready <none> 4m52s
v1.24.9-eks-49d8fe8
ip-192-168-52-244.ec2.internal Ready <none> 5m49s
v1.24.9-eks-49d8fe8
```

## 2. TO CREATE AN IAM OIDC IDENTITY PROVIDER FOR YOUR CLUSTER WITH EKSCTL


2.1. Determine whether you have an existing IAM OIDC provider for your cluster.
Retrieve your cluster's OIDC provider ID and store it in a variable.
```
oidc_id=$(aws eks describe-cluster --name eks-cube-test --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```
2.2. Determine whether an IAM OIDC provider with your cluster's ID is already in your
account.
```
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```
2.3. If output is returned, then you already have an IAM OIDC provider for your
cluster and you can skip the next step. If no output is returned, then you must
create an IAM OIDC provider for your cluster.

2.4. Create an IAM OIDC identity provider for your cluster with the following command. Replace eks-cube-dev with your own value.
```
eksctl utils associate-iam-oidc-provider --cluster eks-cube-test --approve
```
2.5. Check again whether an IAM OIDC provider with your cluster's ID is created in your account.
```
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
-
2EB6AC4AC3AD029AF7FA879A18FF5188”
```

### 3. CREATE A SERVICE ACCOUNT POLICY, AND ROLE-BASED ACCESS CONTROL (RBAC) POLICIES


1. To download an IAM policy that allows the AWS Load Balancer Controller to make
calls to AWS APIs on your behalf, run the following command:
```
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load￾balancer-controller/v2.4.4/docs/install/iam_policy.json
-
% Total % Received % Xferd Average Speed Time
Time Time Current
Dload Upload Total
Spent Left Speed
100 7617 100 7617 0 0 12351 0 --:--:----:--:-- --:--:-- 12486
```

2. To create an IAM policy using the policy that you downloaded in step 1, run the
following command:
```
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
-
{
"Policy": {
"PolicyName": "AWSLoadBalancerControllerIAMPolicy",
"PolicyId": "ANPAXNAXRAAKPJ6254U2T",
"Arn":
"arn:aws:iam::509002973204:policy/AWSLoadBalancerController
IAMPolicy",
"Path": "/",
"DefaultVersionId": "v1",
"AttachmentCount": 0,
"PermissionsBoundaryUsageCount": 0,
"IsAttachable": true,
"CreateDate": "2023-02-03T12:16:03+00:00",
"UpdateDate": "2023-02-03T12:16:03+00:00"
} }
```
3. To create a service account named aws-load-balancer-controller in the kube-system
namespace for the AWS Load Balancer Controller, run the following command:
```
eksctl create iamserviceaccount \
--cluster=eks-cube-test \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::509002973204:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--approve
```

4. To verify that the new service role is created, run one of the following commands:
```
eksctl get iamserviceaccount --cluster eks-cube-dev --name aws-load-balancer-controller --namespace kube-system
-
NAMESPACE NAME ROLE ARN
kube-system aws-load-balancer-controller
arn:aws:iam::509002973204:role/eksctl-eks-cube-dev-addon-iamserviceaccount-Role1-1GFOB2NYF7MFA
```
or
```
kubectl get serviceaccount aws-load-balancer-controller --namespace kube-system
-
NAME SECRETS AGE
aws-load-balancer-controller 0 26s
```


### 4.  INSTALL THE AWS LOAD BALANCER CONTROLLER USING HELM   

( NOTE:- THE ALB INGRESS CONTROLLER IS NOW THE AWS LOAD BALANCER CONTROLLER) 

1. Run below command to check helm version, IF NOT INSTALL
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
			
chmod 700 get_helm.sh
			
./get_helm.sh

helm version
-
version.BuildInfo{Version:"v3.11.0",
GitCommit:"472c5736ab01133de504a826bd9ee12cbe4e7904",
GitTreeState:"clean", GoVersion:"go1.19.5"}
```
1. To add the Amazon EKS chart repo to Helm, run the following command:
```
helm repo add eks https://aws.github.io/eks-charts
-
"eks" has been added to your repositories

helm repo list
-
NAME URL
eks https://aws.github.io/eks-charts
```
2. To install the TargetGroupBinding custom resource definitions (CRDs), run the
following command:
```
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
-
customresourcedefinition.apiextensions.k8s.io/ingressclassparams.elbv2.k8s.aws created
customresourcedefinition.apiextensions.k8s.io/targetgroupbindings.elbv2.k8s.aws created
```
3. To install the Helm chart, run the following command:

NOTE : here make sure you add EKS VPC ID
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
--set clusterName=eks-cube-dev \
--set serviceAccount.create=false \
--set region=us-east-1 \
--set vpcId=vpc-0d2199a1e22977239 \
--set serviceAccount.name=aws-load-balancer-controller \
-n kube-system
-
NAME: aws-load-balancer-controller
LAST DEPLOYED: Fri Feb 3 18:01:26 2023
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
AWS Load Balancer controller installed!
```

### 5. AMAZON EFS CSI DYNAMIC PROVISIONING

1. Download the IAM policy document 
```
curl -S https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/v1.2.0/docs/iam-policy-example.json -o iam-policy.json
```
2. Create an IAM policy 
```
aws iam create-policy \ 
--policy-name EFSCSIControllerIAMPolicy \ 
--policy-document file://iam-policy.json 
```
3. Create a Kubernetes service account 
```
eksctl create iamserviceaccount \ 
--cluster=eks-cube-test \  
--region=us-east-2 \ 
--namespace=kube-system \ 
--name=efs-csi-controller-sa \ 
--override-existing-serviceaccounts \ 
--attach-policy-arn=arn:aws:iam::<AWS account ID>:policy/EFSCSIControllerIAMPolicy \ 
--approve
```
4. To verify that the new service role is created, run one of the following commands:
```
eksctl get iamserviceaccount --cluster eks-cube-test --name efs-csi-controller-sa --namespace kube-system
```
or
```
kubectl get serviceaccount --namespace kube-system efs-csi-controller-sa
```	
5. Now install AWS EFS Storage Controller driver. 
```
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver

helm repo update

helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
--namespace kube-system \
--set image.repository=602401143452.dkr.ecr.us-west-2.amazonaws.com/eks/aws-efs-csi-driver \
--set controller.serviceAccount.create=false \
--set controller.serviceAccount.name=efs-csi-controller-sa
```		
6. To verify that aws-efs-csi-driver has started, run:
```
kubectl get pod -n kube-system -l "app.kubernetes.io/name=aws-efs-csi-driver,app.kubernetes.io/instance=aws-efs-csi-driver"
```
7. To create iamserviceaccount.
```
eksctl create iamserviceaccount \
--cluster=eks-cube-test \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::509002973204:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--approve
```