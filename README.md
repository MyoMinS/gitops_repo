===========
AWS ESK ALB 
===========

curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

eksctl create iamserviceaccount   --cluster=eks-cluster-01   --namespace=kube-system   --name=aws-load-balancer-controller   --role-name AmazonEKSLoadBalancerControllerRole   --attach-policy-arn=arn:aws:iam::484907514740:policy/AWSLoadBalancerControllerIAMPolicy   --approve

helm install aws-load-balancer-controller eks/aws-load-balancer-controller   -n kube-system   --set clusterName=eks-cluster-01   --set serviceAccount.create=false   --set replicaCount=1   --set serviceAccount.name=aws-load-balancer-controller

`Fargate`

helm install aws-load-balancer-controller eks/aws-load-balancer-controller   -n kube-system   --set clusterName=eks-cluster-01   --set serviceAccount.create=false   --set replicaCount=1   --set serviceAccount.name=aws-load-balancer-controller --set vpcId=vpc-0832ec3aea47024b2


==========
Cluster
========

eksctl utils associate-iam-oidc-provider --region=ap-southeast-1 --cluster=eks-cluster-01 --approve

aws eks update-kubeconfig --name eks-cluster-01 --region ap-southeast-1

eksctl create cluster -f .\cluster.yaml 
eksctl delete cluster -f .\cluster.yaml 

eksctl upgrade cluster --config-file cluster.yaml 

eksctl scale nodegroup --cluster=eks-cluster-01 --nodes=1 spot

========
ECR Login
========

aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin 484907514740.dkr.ecr.ap-southeast-1.amazonaws.com

docker build -t 484907514740.dkr.ecr.ap-southeast-1.amazonaws.com/vote:prod .
docker push 484907514740.dkr.ecr.ap-southeast-1.amazonaws.com/vote:prod



========
EKS Node
========

eksctl create nodegroup -f cluster.yaml --include=spot

eksctl delete nodegroup --cluster=eks-cluster-01 --name=spot

eksctl delete nodegroup --cluster=eks-cluster-01 --name=spot --disable-eviction

eksctl drain nodegroup --cluster=eks-cluster-01 --name=spot

======
Add on
======
eksctl update addon --name=coredns --cluster=eks-cluster-01
eksctl update addon --name=kube-proxy --cluster=eks-cluster-01

========
Argocd
=======

aws eks create-pod-identity-association \
  --cluster-name eks-cluster-01 \
  --namespace argocd \
  --service-account argocd-image-updater-controller \
  --role-arn arn:aws:iam::484907514740:role/ArgocdImageUpdaterEKSPodIdentityRole


aws eks create-access-entry \
  --region ap-southeast-1 \
  --cluster-name eks-cluster-01 \
  --principal-arn arn:aws:iam::484907514740:role/ArgoCDCapabilityRole \
  --type STANDARD


argocd cluster add arn:aws:eks:ap-southeast-1:484907514740:cluster/eks-cluster-01 \
  --aws-cluster-name arn:aws:eks:ap-southeast-1:484907514740:cluster/eks-cluster-01 \
  --name in-cluster --project default


aws eks associate-access-policy \
  --region ap-southeast-1 \
  --cluster-name eks-cluster-01 \
  --principal-arn arn:aws:iam::484907514740:role/AmazonEKSCapabilityArgoCDRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster

export ARGOCD_SERVER=$(aws eks describe-capability \
  --cluster-name eks-cluster-01 \
  --capability-name eks-cluster-01-argocd \
  --query 'capability.configuration.argoCd.serverUrl' \
  --output text \
  --region ap-southeast-1 | sed 's|^https://||')

export ARGOCD_SERVER="1ce37fa79bd5267b1e47d5af21af4a4ae4cff62f2558d3ee0.eks-capabilities.ap-southeast-1.amazonaws.com"

export ARGOCD_AUTH_TOKEN="your-token-here"

export ARGOCD_OPTS="--grpc-web"


CLUSTER_ARN=$(aws eks describe-cluster --name eks-cluster-01 --query 'cluster.arn' --output text)

argocd cluster add $CLUSTER_ARN \
  --aws-cluster-name $CLUSTER_ARN \
  --name in-cluster \
  --project default

=====
Argocd Image updater
=====
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/config/install.yaml



=======
Secert
======

helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace \
  --set installCRDs=true

eksctl create iamserviceaccount   --cluster=eks-cluster-01   --namespace=prod  --name=aws-secret-controller   --role-name AmazonEKSSecretRole-prod   --attach-policy-arn=arn:aws:iam::aws:policy/AWSSecretsManagerClientReadOnlyAccess   --approve

`Manually trigger a secret refresh`
kubectl annotate externalsecret my-app-external-secret force-sync=$(date +%s) --overwrite


======
VPC
=====

subnets are tagged with at least the following:

kubernetes.io/cluster/<name> tag set to either shared or owned

kubernetes.io/role/internal-elb tag set to 1 for private subnets

kubernetes.io/role/elb tag set to 1 for public subnets# gitops_repo


====
VPA 
====
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler/
./hack/vpa-up.sh

