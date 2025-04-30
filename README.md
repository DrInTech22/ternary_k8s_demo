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
- All our pods are up in their respective namespaces
```
ubuntu@ip-172-31-2-101:~/ternary_k8s_demo/manifests$ kubectl get pods -n team-ecommerce
NAME                                     READY   STATUS    RESTARTS   AGE
cartservice-c4876d7f5-z7jkd              1/1     Running   0          8m8s
checkoutservice-575bb454f4-79pw6         1/1     Running   0          8m8s
frontend-5c9cfd9fd4-wskmh                1/1     Running   0          8m8s
productcatalogservice-5cb8759967-h8x8g   1/1     Running   0          8m8s
ubuntu@ip-172-31-2-101:~/ternary_k8s_demo/manifests$ kubectl get pods -n team-platform
NAME                                     READY   STATUS    RESTARTS   AGE
currencyservice-7988bdd9c5-jh96j         1/1     Running   0          8m23s
recommendationservice-58f6656765-g9mds   1/1     Running   0          8m22s
redis-cart-7fff569b8b-zxd7m              1/1     Running   0          8m22s
shippingservice-c46bbdc84-kzh9x          1/1     Running   0          8m22s
ubuntu@ip-172-31-2-101:~/ternary_k8s_demo/manifests$ kubectl get pods -n team-marketing
NAME                            READY   STATUS    RESTARTS   AGE
adservice-d8959797f-fz94x       1/1     Running   0          8m35s
emailservice-78d6cc6549-k6phg   1/1     Running   0          8m35s
ubuntu@ip-172-31-2-101:~/ternary_k8s_demo/manifests$ kubectl get pods -n team-finance
NAME                              READY   STATUS    RESTARTS   AGE
paymentservice-75d4497fcc-sgn74   1/1     Running   0          8m45s
```

- Cloudwatch logging doesn't need to be enabled just container insights needed.

- Next step is to install the CloudWatch agent using the Amazon CloudWatch Observability EKS add-on. The  `vpc-cni` addon needs OIDC to receive required IAM permissions. We'll  install OIDC, create service account with IAM role for Cloud Watch Agent and configure the add-on to use the service account. 
  - create an OIDC provider: `eksctl utils associate-iam-oidc-provider --cluster ternary-demo --approve`
  - Create the agent service account and attach an IAM role with the CloudWatchAgentServerPolicy policy.The agent service account to assumes the role using OIDC.
```
eksctl create iamserviceaccount \
  --name cloudwatch-agent \
  --namespace amazon-cloudwatch --cluster ternary-demo \
  --role-name cloudwatch-agent-role \
  --attach-policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
  --role-only \
  --approve
```
  - Install the add-on and specify the service account role. Replace 111122223333 with your account ID.
```
aws eks create-addon --addon-name amazon-cloudwatch-observability --cluster-name ternary-demo --service-account-role-arn arn:aws:iam::111122223333:role/cloudwatch-agent-role
```
  - You can confirm that container Insights is enabled from the CloudWatch Dashboard in the console.
Note: You can get your account ID using `aws sts get-caller-identity`.

- Note that there are different options to install and enable container insights on EKS.

- Now let's configure AWS Cost and Usage Report (CUR). Ternary uses the report to ingest our AWS spend.
  - Search `Cost and Usage Report` on the console and navigate to the page.
  - Click on Create report and enter the name of the report `Ternary-demo-report`.
  - ensure you tick the following settings:
    - Include resource IDs: Yes
    - Automatically refresh your Cost & Usage Report: Yes
  - leave the data refresh settings as default and click next.
  - On this page we'll configure the delivery options for the report. Click on `configure`, set the bucket name `ternary-demo-bucket` and tick the default policy option. Then click save.
  <!image will be inserted here>
  - Set the path prefix as `demo-report`
  - Scroll down and click `next`
  - Review the settings, note the bucket name and path prefix, then click `create report`.
  Note: It takes about 24 hours for AWS to dump the first report in the S3 bucket.

- Configure rightsizing recommendations and Cost Split Allocation Data. Rightsizing recommendations allows ternary to suggest optimal resource configurations for our workloads. The cost allocation data categorizes break spending by teams, applications, or other dimensions based on Kubernetes labels and cloud provider tags.
  - Search `Cost Explorer` and navigate to the page. 
  - On the left navigation pane, go to the `Preferences and Settings` section. Select  `Cost Management Preferences`. 
  - On the general tab, enable the split cost allocation data for EKS and select Cloud Watch container insights for allocating the cost.
  - Also enable the rightsizing recommendations. Then click "save preferences".

- Configure AWS Role for Ternary CUR access.
  Ternary will access the CUR by assuming a role created in the account containing the S3 bucket and CUR. Ternary simplifies this process for us with a CloudFormation template.
```
aws cloudformation create-stack \
  --stack-name ternary-access-role \
  --template-url https://raw.githubusercontent.com/TernaryInc/ternary-onb-permissions/master/aws/payer-account/ternary-payer-account-cfn.json \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=CurBucketArnParameter,ParameterValue=arn:aws:s3:::ternary-demo-bucket \
    ParameterKey=ServiceAccountParameter,ParameterValue=ternary-service-account@yourproject.iam.gserviceaccount.com \
    ParameterKey=ServiceAccountUniqueIDParameter,ParameterValue=123456789012345678901
```

- 





