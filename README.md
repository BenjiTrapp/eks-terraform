[![Checkov](https://github.com/BenjiTrapp/eks-terraform/actions/workflows/checkov.yml/badge.svg)](https://github.com/BenjiTrapp/eks-terraform/actions/workflows/checkov.yml)
[![terraform-lint](https://github.com/BenjiTrapp/eks-terraform/actions/workflows/blank.yml/badge.svg)](https://github.com/BenjiTrapp/eks-terraform/actions/workflows/blank.yml)

# eks-terraform

Some Demo to play a little with Amazon EKS and k8s and here comes the obligatory disclaimer:

**I'm not responsible for any bill created by the usage of the stuff from this repository. Use the [AWS Price Calculator](https://calculator.aws/#/) for a rough estimation of the costs you might get billed for**

### Getting Started with Amazon EKS using Terraform

### Amazon CLI

You can get the Amazon CLI on [Docker-Hub](https://hub.docker.com/r/amazon/aws-cli) <br/>
We'll need the Amazon CLI to gather information so we can build our Terraform file.

```
# Run Amazon CLI
docker run -it --rm -v ${PWD}:/work -w /work --entrypoint /bin/sh amazon/aws-cli:2.0.43

# some handy tools :)
yum install -y jq gzip nano tar git unzip wget

```

### Login to Amazon

```
# Access your "My Security Credentials" section in your profile. 
# Create an access key

aws configure

Default region name: eu-central-1
Default output format: json
```

### Terraform CLI 

```
# Get Terraform

curl -o /tmp/terraform.zip -LO https://releases.hashicorp.com/terraform/0.13.1/terraform_0.13.1_linux_amd64.zip
unzip /tmp/terraform.zip
chmod +x terraform && mv terraform /usr/local/bin/
terraform
```

### Terraform Amazon Kubernetes Provider 

Documentation on all the Kubernetes fields for terraform [here](https://www.terraform.io/docs/providers/aws/r/eks_cluster.html)

```
cd kubernetes/cloud/amazon/terraform

terraform init

terraform plan
terraform apply

```

### Lets see what we deployed

```
# grab our EKS config
aws eks update-kubeconfig --name getting-started-eks --region eu-central-1

# Get kubectl

curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl

kubectl get nodes
kubectl get deploy
kubectl get pods
kubectl get svc


```

### Clean up 

```
terraform destroy
```


## Getting Started with EKS

### Amazon CLI

```

# Run Amazon CLI
docker run -it --rm -v ${PWD}:/work -w /work --entrypoint /bin/sh amazon/aws-cli:2.0.17

cd ./kubernetes/cloud/amazon

yum install jq gzip vim tar git
```

### Login to AWS

https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html

```
# Access your "My Security Credentials" section in your profile. 
# Create an access key

aws configure

# Regions
https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html

```


### Deploy Cluster with AWS CLI

You can deploy a cluster using multiple ways.  </br>
We will cover the two fundamental ways.

1) AWS CLI https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html
2) EKS CLI (newer) https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html


### AWS CLI

Kubernetes needs a service account to manage our Kubernetes cluster <br/>
In AWS this is an IAM role <br/>
Lets create one! <br/>

Follow "Create your Amazon EKS cluster IAM role" [here](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html) <br/>

```

# create our role for EKS
role_arn=$(aws iam create-role --role-name getting-started-eks-role --assume-role-policy-document file://policies/assume-policy.json | jq .Role.Arn | sed s/\"//g)
aws iam attach-role-policy --role-name getting-started-eks-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

# create the cluster VPC

curl https://amazon-eks.s3.us-west-2.amazonaws.com/cloudformation/2020-05-08/amazon-eks-vpc-sample.yaml -o vpc.yaml
aws cloudformation deploy --template-file vpc.yaml --stack-name getting-started-eks

# grab your stack details 
aws cloudformation list-stack-resources --stack-name getting-started-eks > stack.json

# create our cluster

aws eks create-cluster \
--name getting-started-eks \
--role-arn $role_arn \
--resources-vpc-config subnetIds=subnet-063efe1fa0c5d4913,subnet-06f91e563755e2077,subnet-0824d16f8536b3681,securityGroupIds=sg-0960d3a116ba912e1,endpointPublicAccess=true,endpointPrivateAccess=false

aws eks list-clusters
aws eks describe-cluster --name getting-started-eks
```


### Get a kubeconfig for our clu#ster

```

aws eks update-kubeconfig --name getting-started-eks --region ap-southeast-2

#grab the config if you want it
cp ~/.kube/config .

curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl && mv ./kubectl /usr/local/bin/kubectl

```

### Add nodes to our cluster

```

# create our role for nodes
role_arn=$(aws iam create-role --role-name getting-started-eks-role-nodes --assume-role-policy-document file://policies/assume-node-policy.json | jq .Role.Arn | sed s/\"//g)

aws iam attach-role-policy --role-name getting-started-eks-role-nodes --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name getting-started-eks-role-nodes --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
aws iam attach-role-policy --role-name getting-started-eks-role-nodes --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

```
More details on node permissions [here](https://docs.aws.amazon.com/eks/latest/userguide/worker_node_IAM_role.html)


More details on instance types to choose from [here](https://aws.amazon.com/ec2/instance-types/)

```
aws eks create-nodegroup \
--cluster-name getting-started-eks \
--nodegroup-name test \
--node-role $role_arn \
--subnets subnet-0ec47e6ae964a233f \
--disk-size 200 \
--scaling-config minSize=1,maxSize=2,desiredSize=1 \
--instance-types t2.small
```

### EKS CTL example

```
# Install EKS CTL
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
mv /tmp/eksctl /usr/local/bin

# Create SSH key for Node access (if you need it)
yum install openssh
mkdir -p ~/.ssh/
PASSPHRASE="mysuperstrongpassword"
ssh-keygen -t rsa -b 4096 -N "${PASSPHRASE}" -C "your_email@example.com" -q -f  ~/.ssh/id_rsa
chmod 400 ~/.ssh/id_rsa*


eksctl create cluster --name getting-started-eks \
--region eu-central-1 \
--version 1.23 \
--managed \
--node-type t2.small \
--nodes 1 \
--node-volume-size 200 \
--ssh-access \
--ssh-public-key=~/.ssh/id_rsa.pub \

```
### Create some sample containers

```
cd ../..

kubectl create ns example-app

# lets create some resources.
kubectl apply -n example-app -f secrets/secret.yaml
kubectl apply -n example-app -f configmaps/configmap.yaml
kubectl apply -n example-app -f deployments/deployment.yaml

# remember to change the `type: LoadBalancer`
kubectl apply -n example-app -f services/service.yaml

```
### Cleanup 

```

eksctl delete cluster --name getting-started-eks-1

aws eks delete-nodegroup --cluster-name getting-started-eks --nodegroup-name test
aws eks delete-cluster --name getting-started-eks

aws iam detach-role-policy --role-name getting-started-eks-role --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
aws iam delete-role --role-name getting-started-eks-role

aws iam detach-role-policy --role-name getting-started-eks-role-nodes --policy-arn  arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam detach-role-policy --role-name getting-started-eks-role-nodes --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
aws iam detach-role-policy --role-name getting-started-eks-role-nodes --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

aws iam delete-role --role-name getting-started-eks-role-nodes

aws cloudformation delete-stack --stack-name getting-started-eks
```
