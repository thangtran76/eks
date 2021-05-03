# Amazon EKS Lab

## Introduction
Elastic Kubernetes Service (EKS) is a fully managed Kubernetes service from AWS. In this lab, you will work with the AWS command line interface and console, using command line utilities like eksctl and kubectl to launch an EKS cluster, provision a Kubernetes deployment and pod running instances of nginx, and create a LoadBalancer service to expose your application over the internet.

## Preparation for Bastion host

### Update AWS CLI
In the command line window, check the AWS CLI version:
aws --version
It should be an older version.
Download v2:
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
Unzip the file:
unzip awscliv2.zip
See where the current AWS CLI is installed:
which aws
It should be /usr/bin/aws.
Update it:
sudo ./aws/install --bin-dir /usr/bin --install-dir /usr/bin/aws-cli --update
Check the version of AWS CLI:
aws --version
It should now be updated.
Configure the CLI:

### Configure AWS CLI using your AWS Access Key ID and AWS Secret Access Key
*aws configure*

For AWS Access Key ID, paste in the access key ID you copied earlier.
For AWS Secret Access Key, paste in the secret access key you copied earlier.
For Default region name, enter us-east-1.
For Default output format, enter json.

### Install kubectl
1. Download kubectl:

    *curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl*

2. Apply execute permissions to the binary:

    *chmod +x ./kubectl*

3. Copy the binary to a directory in your path:

    *mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin*

4. Ensure kubectl is installed:

    *kubectl version --short --client*

### Install eksctl
1. Download eksctl:

    *curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp*

2. Move the extracted binary to /usr/bin:

    *sudo mv /tmp/eksctl /usr/bin*

3. Get the version of eksctl:

    *eksctl version*

4. See the options with eksctl:

    *eksctl help*

### Install Git
*sudo yum install -y git*

## Amazon EKS

### Provision an EKS Cluster
1. Provision an EKS cluster with three worker nodes in us-east-1:

    *eksctl create cluster --name dev --version 1.19 --region us-east-1 --nodegroup-name standard-workers --node-type t3.micro --nodes 2 --nodes-min 1 --nodes-max 4 --managed*

It will take 10–15 minutes since it's provisioning the control plane and worker nodes, attaching the worker nodes to the control plane, and creating the VPC, security group, and Auto Scaling group.

2. Check for the created cluster:

    *eksctl get cluster*

### Enable AWS CLI to connect to our cluster:
*aws eks update-kubeconfig --name dev --region us-east-1*

### Enable CloudWatch logging for cluster "dev" in "us-east-1":
*eksctl utils update-cluster-logging --enable-types=all --region=us-east-1 --cluster=dev --approve*

### Update Autoscaling Group
Follow these steps to modify the Min/Max Sizes of the Autoscaling Group

1. In your browser, log in to the AWS Management Console with the credentials provided on the lab instructions page. Make sure you are using the us-east-1 (N. Virginia) region.
2. Navigate to the EC2 service.
3. Click Autoscaling Groups in the left sidebar.
4. Select the Autoscaling Group that has been created by in the step to Provision an EKS Cluster.
5. Copy the Autoscaling Group name, we'll need it later.
6. Click Actions > Edit.
7. In the Edit details menu, configure the following settings: Min: 2, Max: 4, click Save.

### Clone Git repository
Clone Git repository to your Bastion host:
*git clone https://github.com/thangtran76/eks.git*

### Apply the autoscaling policies to Node Group role
1. Open the file cluster_autoscaler.yaml using Vim
2. Replace <AUTOSCALING_GROUP> with the Autoscaling Group name you copied earlier
3. Save and quit

### Update Cluster Autoscaler deployment before deploying to the namespace kube-system
1. Open the file cluster_autoscaler.yaml using Vim
2. Replace <AUTOSCALING_GROUP> with the Autoscaling Group name you copied earlier
3. Save and quit

### Deploy the Cluster Autoscaler
1.  Deploy the Cluster Autoscaler

    *kubectl apply -f ./cluster_autoscaler.yaml*

2. Check the cluster autoscaler logs

    *kubectl logs -f deployment/cluster-autoscaler -n kube-system*

3. Press Ctrl + C to exit the logs.

### Deploy the Nginx Deployment
1. Deploy the nginx deployment.

    *kubectl apply -f ./nginx.yaml*

2. Verify that the deployment was successful.

    *kubectl get deployment/nginx-scaleout*

3. View the pods

    *kubectl get pod*

4. View the nodes

    *kubectl get node*

### Create a service
1. Deploy the nginx deployment.

    *kubectl apply -f ./nginx-svc.yaml*

2. Verify that the deployment was successful.

    *kubectl get service*

### Scale the Nginx Deployment
1. View the ReplicaSets before scale

    *kubectl get rs*

2. Run kubectl scale command:

    *kubectl scale --replicas=10 deployment/nginx-scaleout*

3. Check the autoscaler logs again.

    *kubectl logs -f deployment/cluster-autoscaler -n kube-system*

## Clean up
### Delete Kubernetes components
1. Delete the service

    *kubectl delete -f nginx-svc.yaml*

2. Delete the cluster autoscaler deployments

    *kubectl delete -f cluster_autoscaler.yaml*

3. Delete the nginx deployments

    *kubectl delete -f nginx.yaml*

### Delete AWS EKS cluster
*eksctl delete cluster dev*


## Explore
### 1. How CloudFormation launch an EKS Cluster
In the AWS Management Console, navigate to CloudFormation and take a look at what’s going on there.
Select the eksctl-dev-cluster stack (this is our control plane).
Click Events, so you can see all the resources that are being created.
We should then see another new stack being created — this one is our node group.
Once both stacks are complete, navigate to Elastic Kubernetes Service > Clusters.
Click the listed cluster.
Click the Compute tab, and then click the listed node group. There, we'll see the Kubernetes version, instance type, status, etc.
Click dev in the breadcrumb navigation link at the top of the screen.
Click the Networking tab, where we'll see the VPC, subnets, etc.
Click the Logging tab, where we'll see the control plane logging info.
The control plane is abstracted — we can only interact with it using the command line utilities or the console. It’s not an EC2 instance we can log into and start running Linux commands on.



curl "<LOAD_BALANCER_EXTERNAL_IP>"
The output should be the HTML for a default Nginx web page.
In a new browser tab, navigate to the same IP, where we should then see the same Nginx web page.
Test the High Availability Features of Your EKS Cluster
In the AWS console, on the EC2 instances page, select the three t3.micro instances.
Click Actions > Instance State > Stop.
In the dialog, click Yes, Stop.
After a few minutes, we should see EKS launching new instances to keep our service running.
In the CLI, check the status of our nodes:
kubectl get node
All the nodes should be down (i.e., display a NotReady status).
Check the pods:
kubectl get pod
We'll see a few different statuses — Terminating, Running, and Pending — because, as the instances shut down, EKS is trying to restart the pods.
Check the nodes again:
kubectl get node
We should see a new node, which we can identify by its age.
Wait a few minutes, and then check the nodes again:
kubectl get node
We should have one in a Ready state.
Check the pods again:
kubectl get pod
We should see a couple pods are now running as well.
Check the service status:
kubectl get service
Copy the external IP listed in the output.
Access the application using the load balancer, replacing <LOAD_BALANCER_EXTERNAL_IP> with the IP you just copied:
curl "<LOAD_BALANCER_EXTERNAL_IP>"
We should see the Nginx web page HTML again. (If you don't, wait a few more minutes.)
In a new browser tab, navigate to the same IP, where we should again see the Nginx web page.
In the CLI, delete everything:
eksctl delete cluster dev

Copy the content of asg-policy.json to your clipboard.
Switch to the AWS Management Console.
Navigate to the IAM service.
Click Roles in the left sidebar.
Type "node" in the search bar.
Click the name of the role that appears in the search results to open it.
Click + Add inline policy.
Click the JSON tab.
Delete the default text from the policy editor, and paste in the content of asg-policy.json you copied to your clipboard earlier.
Click Review policy.
Name the policy "CA".
Click Create policy.

## Monitoring

## Install Packages Manager Helm 3
Download the script:
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
Make the script executable:
chmod +x get_helm.sh
Run the script:
./get_helm.sh
 
Frequently used Kubernetes commands
Rolling restart of all pods:
kubectl -n service rollout restart deployment <name>
Make the script executable:
chmod +x get_helm.sh
Run the script:
./get_helm.sh

kubectl -n service rollout restart deployment <name>
# add grafana Helm repo
helm repo add grafana https://grafana.github.io/helm-charts


helm install prometheus prometheus-community/prometheus --namespace prometheus --set alertmanager.persistentVolume.storageClass="gp2" --set server.persistentVolume.storageClass="gp2"

Autoscaling an EKS Cluster
Introduction
Welcome to this hands-on AWS lab! In this lab, we will deploy the Cluster Autoscaler to an Elastic Container Service for Kubernetes (or EKS) cluster.
The Cluster Autoscaler is a default Kubernetes component that can scale either pods or nodes in a cluster. It automatically increases the size of an Auto Scaling group so that pods can continue to be placed successfully.
Connecting to the Lab
Open your terminal application, and run the following command. (Remember to replace <PUBLIC_IP> with the public IP you were provided on the lab instructions page.)
ssh cloud_user@<PUBLIC_IP>;
Enter your password at the prompt.
Configure the Cluster Autoscaler
Go back to your terminal application
List the contents of the home directory.
ls
Edit the cluster_autoscaler.yaml file.
vim cluster_autoscaler.yaml
Locate the <AUTOSCALING GROUP NAME>placeholder, and replace it with the autoscaling group name we found in the AWS Management Console.
Press Escape, then type :wq to quit the vim text editor.
Apply the IAM Policy to the Worker Node Group Role
List the contents of the asg-policy.json file.
cat asg-policy.json


Go back to the AWS Management Console.
Navigate to the EC2 service.
Click 3 Running Instances.
Go back to your terminal application.
Check the nodes.
kubectl get nodes
Conclusion
Congratulations, you've successfully completed this hands-on lab!


 

 
Install Packages Manager Helm 3
Download the script:
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
Make the script executable:
chmod +x get_helm.sh
Run the script:
./get_helm.sh
