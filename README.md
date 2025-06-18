# Build a Production-Ready RKE2 HA Cluster on AWS

<img width="230" alt="image" src="https://camo.githubusercontent.com/e0e6e05e3edcfa94bd0eb63a3c45a35110625bd53bef7ce2d314dcbc13837e5d/68747470733a2f2f646f63732e726b65322e696f2f696d672f6c6f676f2d686f72697a6f6e74616c2d726b65322e737667" />

## Table of Contents

- [Section 1: Introduction](https://github.com/pkthom/guide-aws-rke2-ha#section-1-introduction)
  - [Course Overview](https://github.com/pkthom/guide-aws-rke2-ha/blob/main/README.md#course-overview)
- [Section 2: Creating a Cluster](https://github.com/pkthom/guide-aws-rke2-ha/blob/main/README.md#section-2-creating-a-cluster)
  - [Prepare Terraform Resources](https://github.com/pkthom/guide-aws-rke2-ha/blob/main/README.md#prepare-terraform-resources)
  - [Install AWS CLI](https://github.com/pkthom/guide-aws-rke2-ha#install-aws-cli)
  - [Install Terraform with tfenv](https://github.com/pkthom/guide-aws-rke2-ha#install-terraform-with-tfenv)
  - [Create a Cluster with Terraform](https://github.com/pkthom/guide-aws-rke2-ha#create-a-cluster-with-terraform)
  - [Connect to the Cluster From Your PC](https://github.com/pkthom/guide-aws-rke2-ha/blob/main/README.md#connect-to-the-cluster-from-your-pc)
- [Section 3: Deploy a Test Nginx](https://github.com/pkthom/guide-aws-rke2-ha/blob/main/README.md#section-3-deploy-a-test-nginx)
  - [Install Amazon Cloud Provider](https://github.com/pkthom/guide-aws-rke2-ha#install-amazon-cloud-provider)
  - [Install NGINX Ingress Controller](https://github.com/pkthom/guide-aws-rke2-ha#install-nginx-ingress-controller)
  - [Deploy a Test Nginx](https://github.com/pkthom/guide-aws-rke2-ha#deploy-a-test-nginx)
- [Section 4: Rancher Setup](https://github.com/pkthom/guide-aws-rke2-ha#section-4-rancher-setup)
  - [Install Rancher](https://github.com/pkthom/guide-aws-rke2-ha#install-rancher)
  - [Import your cluster into Rancher](https://github.com/pkthom/guide-aws-rke2-ha#import-your-cluster-into-rancher)
  - [Node maintenance in Rancher](https://github.com/pkthom/guide-aws-rke2-ha#node-maintenance-in-rancher)
- [Section 5: Final Notes](https://github.com/pkthom/guide-aws-rke2-ha#section-5-final-notes)
  - [Final Comments and Precautions](https://github.com/pkthom/guide-aws-rke2-ha#%EF%B8%8F-final-comments-and-precautions)

# Section 1: Introduction

### Course Overview 

https://kubernetes.io/docs/concepts/services-networking/ingress/#what-is-ingress

# Section 2: Creating a Cluster 

### Prepare Terraform Resources

1. Create a key pair `mycluster`

Set your private key permissions to 600:
```
chmod 600 Downloads/mycluster.pem
```

2. Allocate an Elastic IP

3. Create a security group with these inbound rules:

- Accept `All traffic` from this security group itselt
- Accept `All traffic` from your home IP address

4. Create an IAM User

Create an IAM user with `AdministratorAccess`.\
Save the `Access key` and `Secret access key` — you’ll need them later.

⚠️ Please be very careful not to leak these to anyone.\
Your account could be misused by others.

### Install AWS CLI

Launch a new EC2 instance.
- Amazon Machine Image (AMI): Ubuntu 24.04
- Instance type: t3.medium
- Key pair name: The one you created
- Firewall (security groups): The one you created

SSH into the EC2 instance and install AWS CLI by following the official guide: \
[https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#getting-started-install-instructions](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#getting-started-install-instructions)
```
sudo apt update && sudo apt install -y unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```
Then configure it with your `Access key` and `Secret access key`
```
aws configure
```

### Install Terraform with tfenv

Install [tfenv](https://github.com/tfutils/tfenv) by following the official guide: \
[https://github.com/tfutils/tfenv#manual](https://github.com/tfutils/tfenv#manual)
```
git clone --depth=1 https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bash_profile
source ~/.bash_profile
```
Verify tfenv is installed:
```
tfenv
```

List available Terraform versions:
```
tfenv list-remote
```
Install the latest stable version (for example, 1.12.2):
```
tfenv install 1.12.2
tfenv use 1.12.2
```
Verify Terraform version:
```
terraform version
```

### Create a Cluster with Terraform

Follow the guide here:\
[https://github.com/pkthom/terraform-aws-rke2-ha](https://github.com/pkthom/terraform-aws-rke2-ha)

### Connect to the Cluster From Your PC

Install `kubectl` to your local PC by following the official guide: \
[https://kubernetes.io/docs/tasks/tools](https://kubernetes.io/docs/tasks/tools)

After installing, create a Kubernetes config directory:
```
mkdir -p ~/.kube
cd ~/.kube
```
Run this on `server1`:
```
cat /etc/rancher/rke2/rke2.yaml
```
Copy the output and paste it into your local `~/.kube/config`. 

Replace `127.0.0.1` in the `server:` line with the public IP of `server1`.

Set the file permission:
```
chmod 600 .kube/config
```
Test the connection:
```
kubectl get nodes
```
You should see all your nodes listed as *Ready*.

Your RKE2 cluster is now ready to use.

# Section 3: Deploy a Test Nginx

### Install Amazon Cloud Provider

Install Helm by the following installation guide:\
https://helm.sh/docs/intro/install/

When deploying NGINX Ingress Controller with LoadBalancer service on AWS, an [ELB](https://aws.amazon.com/elasticloadbalancing/) is automatically created.\
To enable this, install AWS Cloud Provider.

Follow the official guide: \
[https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-clusters-in-rancher-setup/set-up-cloud-providers/amazon](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-clusters-in-rancher-setup/set-up-cloud-providers/amazon)

In the official guide, [1. Create an IAM Role and attach to the instances](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-clusters-in-rancher-setup/set-up-cloud-providers/amazon#1-create-an-iam-role-and-attach-to-the-instances) and [2. Configure the ClusterID](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-clusters-in-rancher-setup/set-up-cloud-providers/amazon#2-configure-the-clusterid) are already done by Terraform.

Start from [Helm Chart Installation from CLI](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-clusters-in-rancher-setup/set-up-cloud-providers/amazon#helm-chart-installation-from-cli)

Add the Helm repository for Amazon Cloud Provider:
```
helm repo add aws-cloud-controller-manager https://kubernetes.github.io/cloud-provider-aws
helm repo update
```
Create `values.yaml` and paste the content from the guide.

Then install Amazon Cloud Provider:
```
helm upgrade --install aws-cloud-controller-manager aws-cloud-controller-manager/aws-cloud-controller-manager --values values.yaml -n kube-system
```
Verify that it is running:
```
kubectl get pods -n kube-system | grep aws
```
You should see the `aws-cloud-controller-manager` running.

### Install NGINX Ingress Controller

Install [NGINX Ingress Controller](https://docs.nginx.com/nginx-ingress-controller/) with LoadBalancer service type following the official guide:\
[https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx](https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx)
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.progressDeadlineSeconds=60
```
Check the new LoadBalancer service with external IP:
```
kubectl get svc -n ingress-nginx
```
You should also see the `ELB` and `a new security group` for it in the AWS Console.

Add new rules to the security group to allow HTTP and HTTPS traffic from the ELB.

For example, this means the ELB will access your cluster on ports 30802 and 30884.
```
pkthom@MacBook-Pro ~ % kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP                            PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.43.17.212    abc.ap-southeast-1.elb.amazonaws.com   80:30802/TCP,443:30884/TCP   65m
```
![image](https://github.com/user-attachments/assets/19184dd3-00d2-47eb-ba5a-7e69cdbe58dd)

So add 2 rules to the security group
- Accept `30802` from the newly created ELB security group
- Accept `30884` from the newly created ELB security group

ELBs are highly available, spanning multiple Availability Zones.

Check that the EXTERNAL-IP is public:
```
dig +short <EXTERNAL-IP>
```
This confirms that your Ingress is publicly accessible from anywhere in the world.

Check the IngressClass:
```
kubectl get ingressclass
```
It should show an IngressClass named `nginx`.

To use this Ingress Controller, specify `className: "nginx"` in your Ingress resource.

### Deploy a Test Nginx

Create a namespace and a Helm chart:
```
kubectl create namespace test
helm create test-nginx 
```
Enable Ingress in `test-nginx/values.yaml`:
```
ingress:
  enabled: true
  className: "nginx"
```

Install the Helm chart:
```
helm install test-nginx ./test-nginx -n test
```

Verify the deployment:
```
kubectl get pods -n test
```
Verify the ingress:
```
kubectl get ingress -n test
```
You should see an Ingress resource associated with the external IP.

Add this to your local `/etc/hosts`
```
<EXTERNAL-IP>　chart-example.local
```
Now open your browser and visit:\
[http://chart-example.local/](http://chart-example.local/)

You should see the NGINX welcome page.


# Section 4: Rancher Setup

### Install Rancher

Install [Rancher](https://www.rancher.com/) with Helm following the official guide:\
[https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster#install-the-rancher-helm-chart](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster#install-the-rancher-helm-chart)

Install cert-manager:
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.17/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace
```
Verify the installation:
```
kubectl get pods --namespace cert-manager
```
Add the Rancher Helm Chart Repository:
```
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
```
Create a Namespace for Rancher:
```
kubectl create namespace cattle-system
```
Add this to `/etc/hosts` in your PC:
```
<EXTERNAL-IP> rancher.my.org
```
Install Rancher:
```
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.my.org \
  --set bootstrapPassword=admin \
  --set ingress.ingressClassName=nginx \
  --set ingress.tls.source=rancher
```
Check if Rancher was rolled out successfully:
```
kubectl -n cattle-system rollout status deploy/rancher
```
Expected output:
```
deployment "rancher" successfully rolled out
```

Verify the ingress:
```
kubectl get ingress -n cattle-system
```

Verify the TLS certificate has been created:
```
kubectl get certificates -A
```
Ping Rancher endpoint to confirm:
```
curl -k https://rancher.my.org/ping
```
Expected output:
```
pong
```
Access the Rancher UI:

[https://rancher.my.org/](https://rancher.my.org/)

### Import your cluster into Rancher

Edit CoreDNS configmap:
```
kubectl -n kube-system get configmap rke2-coredns-rke2-coredns -o yaml > coredns-cm.yaml
vi coredns-cm.yaml
kubectl apply -f coredns-cm.yaml
```
Add this inside the `.:53 { ... }` block:
```
hosts {                                    
  <EXTERNAL-IP> rancher.my.org
}
```
Restart CoreDNS:
```
kubectl -n kube-system rollout restart deployment rke2-coredns-rke2-coredns
```
Test DNS Resolution:
```
kubectl exec -it rancher-58768d4f48-cxxrr -n cattle-system -- getent hosts rancher.my.org
```
Expected output:
```
<EXTERNAL-IP> rancher.my.org
```
Check if Rancher is responding:
```
kubectl exec -it rancher-58768d4f48-cxxrr -n cattle-system -- sh
curl -k https://rancher.my.org/ping
```
Expected response: `pong`

Import the Cluster into Rancher

1. Open Rancher in your browser:\
[https://rancher.my.org/dashboard/home](https://rancher.my.org/dashboard/home)

2. Click `Import Existing` -> `Generic`

3. Enter `Cluster Name` -> Click `Create`

4. Copy the provided registration command and run it on your PC

After a short time, your cluster should appear in Rancher in the Active state.


# Section 5: Final Notes

### ⚠️ Final Comments and Precautions

- AWS costs money. Delete all EC2 instances, Elastic IP, and IAM user if you don’t need them.
- Please be very careful not to leak IAM user's `Access key` and `Secret access key` to anyone. Your account could be misused by others. 
- If not needed, delete IAM user's `Access key`. If possible, delete the IAM user itself.
- Note: I’m not responsible for any issues caused by this course.


