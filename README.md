# ternary_k8s_demo

Steps
- Install and configure awscli `sudo snap install aws-cli --classic`
- Install Helm ` sudo snap install helm --classic`
- Install kubectl and eksctl 
  - kubectl v1.32: 
    - Install kubectl: `curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.32.0/2024-12-20/bin/linux/amd64/kubectl`
    - execute: `chmod +x ./kubectl`
    - `mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH`
  - eksctl: script to install it
    ```
    # for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
    ARCH=amd64
    PLATFORM=$(uname -s)_$ARCH

    curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

    # (Optional) Verify checksum
    curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

    tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

    sudo mv /tmp/eksctl /usr/local/bin
    ```

- The user account used to create eks cluster should have the following permissions:
    AWS Service:	    Access Level
    CloudFormation:	    Full Access
    EC2:	            Full: Tagging Limited: List, Read, Write
    EC2 Auto Scaling:	Limited: List, Write
    EKS:	            Full Access
    IAM	Limited:        List, Read, Write, Permissions Management
    Systems Manager	    Limited: List, Read

- create the cluster with eksctl
```
eksctl create cluster \
  --name ternary-demo \
  --region us-east-1 \
  --nodegroup-name demo-nodes \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 4 \
  --managed
```

Next Step is to 
- Update KubeConfig to communicate with the newly created cluster: `aws eks update-kubeconfig --region us-east-1 --name ternary-demo`

- Deploy the microservices application to the cluster
We will clone the repo and apply the kubernetes manifests to deploy the applications to our EKS Cluster.
```
# Clone the repo
git clone https://github.com/drintech22/ternary_k8s_demo.git
cd ternary_k8s_demo/manifests

# Create namespaces and apply all manifests 
kubectl create namespace team-platform && kubectl create namespace team-marketing && kubectl create namespace team-ecommerce && kubectl create namespace team-finance
kubectl apply -f .
```

- Cloudwatch logging doesn't need to be enabled just container insights needed.

- The  `vpc-cni` addon needs OIDC to receive required IAM permissions. Next step is to install OIDC, create service account with IAM role for Cloud Watch Agent attached and configure the add-on to use the service account. 

- Note that there are different options to install and enable container insights on EKS.
- To enable container insights, we'll use the Amazon CloudWatch Observability EKS add-on using IAM service account role.
- Navigate to the `Billing and Cost Management` page and enable 
