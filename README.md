# Build a Production-Ready RKE2 HA Cluster on AWS

<img width="230" alt="image" src="https://camo.githubusercontent.com/e0e6e05e3edcfa94bd0eb63a3c45a35110625bd53bef7ce2d314dcbc13837e5d/68747470733a2f2f646f63732e726b65322e696f2f696d672f6c6f676f2d686f72697a6f6e74616c2d726b65322e737667" />

## Table of Contents

- [Section 1: Introduction](https://github.com/pkthom/guide-aws-rke2-ha#section-1-introduction)
  - [What is RKE2?](https://github.com/pkthom/guide-aws-rke2-ha#what-is-rke2)
- [Section 2: Creating a Cluster (Manual)](https://github.com/pkthom/guide-aws-rke2-ha#section-2-creating-a-cluster-manual)
  - [Create a Security Group](https://github.com/pkthom/guide-aws-rke2-ha#create-a-security-group)
  - [Launch EC2 Instances](https://github.com/pkthom/guide-aws-rke2-ha#launch-ec2-instances)
  - [Set Up Server Nodes](https://github.com/pkthom/guide-aws-rke2-ha#set-up-server-nodes)
  - [Set Up Agent Nodes](https://github.com/pkthom/guide-aws-rke2-ha#set-up-agent-nodes)
  - [Connect to the Cluster From Your PC](https://github.com/pkthom/guide-aws-rke2-ha#connect-to-the-cluster-from-your-pc)
- [Section 3: Creating a Cluster (Terraform)](https://github.com/pkthom/guide-aws-rke2-ha#section-3-creating-a-cluster-terraform)
  - [What is Terraform?](https://github.com/pkthom/guide-aws-rke2-ha#what-is-terraform)
  - [Create an IAM User](https://github.com/pkthom/guide-aws-rke2-ha#create-an-iam-user)
  - [Install AWS CLI](https://github.com/pkthom/guide-aws-rke2-ha#install-aws-cli)
  - [Install Terraform with tfenv](https://github.com/pkthom/guide-aws-rke2-ha#install-terraform-with-tfenv)
  - [Create a Cluster with Terraform](https://github.com/pkthom/guide-aws-rke2-ha#create-a-cluster-with-terraform)
- [Section 4: Deploy a Test Nginx Page](https://github.com/pkthom/guide-aws-rke2-ha#deploy-a-test-nginx-page)
  - [How To Deploy](https://github.com/pkthom/guide-aws-rke2-ha#how-to-deploy)
  - [Install Amazon Cloud Provider](https://github.com/pkthom/guide-aws-rke2-ha#install-amazon-cloud-provider)
  - [Install NGINX Ingress Controller](https://github.com/pkthom/guide-aws-rke2-ha#install-nginx-ingress-controller)
  - [Deploy a Test Nginx](https://github.com/pkthom/guide-aws-rke2-ha#deploy-a-test-nginx)
- [Section 5: Rancher Setup](https://github.com/pkthom/guide-aws-rke2-ha#section-5-rancher-setup)
  - [Install Rancher](https://github.com/pkthom/guide-aws-rke2-ha#install-rancher)
  - [Import your cluster into Rancher](https://github.com/pkthom/guide-aws-rke2-ha#import-your-cluster-into-rancher)
  - [Node maintenance in Rancher](https://github.com/pkthom/guide-aws-rke2-ha#node-maintenance-in-rancher)
- [Section 6: Final Notes](https://github.com/pkthom/guide-aws-rke2-ha#section-6-final-notes)
  - [Final Comments and Precautions](https://github.com/pkthom/guide-aws-rke2-ha#%EF%B8%8F-final-comments-and-precautions)

# Section 1: Introduction

### What is RKE2?

https://docs.rke2.io/

https://github.com/rancher/rancher

https://github.com/rancher/rke2

https://ranchergovernment.com/products/rke2

# Section 2: Creating a Cluster (Manual)

### Create a Security Group

Create a security group with these inbound rules:

- Accept `All traffic` from this security group itselt
- Accept `All traffic` from your home IP address


### Launch EC2 Instances

Create 6 EC2 instances with these settings:

- OS: Ubuntu24.04
- Instance Type: t3.meduim
- Security Group: the one you just created above
- Volumes : 30 GiB

After creating the EC2 instances, name them:\
`server1`, `server2`, `server3`, `agent1`, `agent2`, `agent3`


### Set Up Server Nodes

Follow the official RKE2 guide: \
[https://docs.rke2.io/install/quickstart#server-node-installation](https://docs.rke2.io/install/quickstart#server-node-installation)

Set your private key permissions to 600:
```
chmod 600 mycluster.pem
```
SSH into `server1`:
```
ssh ubuntu@<SERVER1_PUBLIC_IP> -i Download/mycluster.pem
```

Install RKE2:
```
curl -sfL https://get.rke2.io | sh -
```

Create the config directory and file:
```
mkdir -p /etc/rancher/rke2/
vim /etc/rancher/rke2/config.yaml
```
Add this to config.yaml:
```
disable:
  - rke2-ingress-nginx
cloud-provider-name: aws
tls-san:
  - <SERVER1_PRIVATE_IP>
  - <SERVER1_PUBLIC_IP>
```

Enable and start the `rke2-server.service`:
```
systemctl enable rke2-server.service
systemctl start rke2-server.service
```

Repeat the above steps for `server2` and `server3`, but change config.yaml like this:
```
disable:
  - rke2-ingress-nginx
cloud-provider-name: aws
token: <SERVER1_CLUSTER_TOKEN>
server: https://<SERVER1_PRIVATE_IP>:9345
```

You can get `<SERVER1_CLUSTER_TOKEN>` on `server1`:
```
cat /var/lib/rancher/rke2/server/node-token
```

When all the `rke2-server.service` are running, check the cluster status:
```
/var/lib/rancher/rke2/bin/kubectl get nodes \
  --kubeconfig /etc/rancher/rke2/rke2.yaml
```

If all nodes are *Ready*, your servers are set up correctly.

### Set Up Agent Nodes

Follow the official RKE2 guide: \
[https://docs.rke2.io/install/quickstart#linux-agent-worker-node-installation](https://docs.rke2.io/install/quickstart#linux-agent-worker-node-installation)

SSH into each agent node:
```
ssh ubuntu@<AGENT_PUBLIC_IP> -i Download/mycluster.pem
sudo su -
```
Install RKE2 agent:
```
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
```
Create the config directory and file:
```
mkdir -p /etc/rancher/rke2/
vim /etc/rancher/rke2/config.yaml
```
Add this to `config.yaml` (change `node-name` accordingly):
```
token: <SERVER1_CLUSTER_TOKEN>
server: https://<SERVER1_PRIVATE_IP>:9345
cloud-provider-name: aws
```
Enable and start the `rke2-agent.service`:
```
systemctl enable rke2-agent.service
systemctl start rke2-agent.service
```
When all the `rke2-agent.service` is running, check cluster status again:
```
/var/lib/rancher/rke2/bin/kubectl get nodes \
  --kubeconfig /etc/rancher/rke2/rke2.yaml
```
You should see all 6 nodes in *Ready* status.

### Connect to the Cluster From Your PC

Install `kubectl` by following the official guide: \
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
chmod 600 config
```
Test the connection:
```
kubectl get nodes
```
You should see all your nodes listed as *Ready*.

Your RKE2 cluster is now ready to use.

# Section 3: Creating a Cluster (Terraform)

### What is Terraform?

https://developer.hashicorp.com/terraform

### Create an IAM User

Create an IAM user with `AdministratorAccess`.\
Save the `Access key` and `Secret access key` — you’ll need them later.

⚠️ Please be very careful not to leak these to anyone.\
Your account could be misused by others.

### Install AWS CLI

Launch a new EC2 instance.\
(Use the same settings as in [Launch EC2 instances](https://github.com/nojsvt/study/blob/main/rancher4.md#launch-ec2-instances))

SSH into the instance and install AWS CLI by following the official guide: \
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
Install the latest stable version (for example, 1.12.1):
```
tfenv install 1.12.1
tfenv use 1.12.1
```
Verify Terraform version:
```
terraform version
```

### Create a Cluster with Terraform

Follow the guide here:\
[https://github.com/pkthom/terraform-aws-rke2-ha](https://github.com/pkthom/terraform-aws-rke2-ha)

# Deploy a Test Nginx Page

### How To Deploy 

https://kubernetes.io/docs/concepts/services-networking/ingress/#what-is-ingress

### Install Amazon Cloud Provider

When deploying NGINX Ingress Controller with LoadBalancer service on AWS, an [ELB](https://aws.amazon.com/elasticloadbalancing/) is automatically created.\
To enable this, install AWS Cloud Provider.


Follow the official guide: \
[https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-clusters-in-rancher-setup/set-up-cloud-providers/amazon](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-clusters-in-rancher-setup/set-up-cloud-providers/amazon)

Create IAM Policies for server nodes:

1. Go to `IAM` -> `Policies` -> `Create Policy`
2. Select the `JSON` tab
3. Paste the Control Plane policy from https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-clusters-in-rancher-setup/set-up-cloud-providers/amazon#1-create-an-iam-role-and-attach-to-the-instances
4. Add `ec2:DescribeAvailabilityZones",` 
5. Click `Next` -> Name it `acl-server-policy` -> Click `Create policy`

Create IAM Policies for for agent nodes:

1. Go to `IAM` -> `Policies` -> `Create Policy`
2. Select the `JSON` tab
3. Paste the Worker Node policy from https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-clusters-in-rancher-setup/set-up-cloud-providers/amazon#1-create-an-iam-role-and-attach-to-the-instances
4. Add `ec2:DescribeAvailabilityZones` 
5. Click `Next` -> Name it `acl-agent-policy` -> Click `Create policy`

Create an IAM Role for server nodes:

1. Go to `IAM` > `Roles` > `Create role`
2. Select `AWS service` and `EC2` -> `Next`
3. Attach `acl-server-policy` -> `Next`
4. Name it `acl-server-role` -> `Create role`

Create an IAM Role for agent nodes:

1. Go to `IAM` > `Roles` > `Create role`
2. Select `AWS service` and `EC2` -> `Next`
3. Attach `acl-agent-policy` -> `Next`
4. Name it `acl-agent-role` -> `Create role`

Attach the IAM Roles to the EC2 instances

- Attach `acl-server-role` to all 3 server nodes.
- Attach `acl-agent-role` to all 3 agent nodes.

Tag Nodes and Subnets

Add the following tag to all 6 nodes and your subnet:
- Key: `kubernetes.io/cluster/mycluster`
- Value: `owned`

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
  --set controller.service.type=LoadBalancer
```
Check the new LoadBalancer service with external IP:
```
kubectl get svc -n ingress-nginx
```
You should also see the ELB in the AWS Console.

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
helm create my-nginx 
```
Enable Ingress in `my-nginx/values.yaml`:
```
ingress:
  enabled: true
  className: "nginx"
```

Install the Helm chart:
```
helm install my-nginx ./my-nginx -n test
```

Verify the deployment:
```
kubectl get pods -n test
```
```
kubectl get ingress -n test
```
You should see an Ingress resource associated with the external IP.

Add this to your local `/etc/hosts`
```
<EXTERNAL-IP>　chart-example.local
```
Now open your browser and visit:\
http://chart-example.local/

You should see the NGINX welcome page.


# Section 5: Rancher Setup

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
kubectl -n kube-system edit configmap rke2-coredns-rke2-coredns
```
Add this inside the `.:53 { ... }` block:
```
hosts {                                    
  <EXTERNAL-IP> rancher.my.org
  fallthrough
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
https://rancher.my.org/dashboard/home

2. Click `Import Existing` -> `Generic`

3. Enter `Cluster Name` -> Click `Create`

4. Copy the provided registration command and run it on your PC

After a short time, your cluster should appear in Rancher in the Active state.

### Node maintenance in Rancher

Run a looped curl request to `my-nginx`:
```
watch -n 0.5 curl -I http://chart-example.local
```
Check Where `my-nginx` pod is running:
```
kubectl get pods -n test -o wide
```
Note which node it's on.

Drain the Node from Rancher.

In the Rancher UI:

1. Go to `Home` -> your cluster -> `Nodes` 

2. Select the node with the `my-nginx` pod
3. Click `Cordon`

4. Click `Drain`

While the pod is rescheduled, you will see this in the curl output:
```
HTTP/1.1 503 Service Temporarily Unavailable
```
This indicates a short downtime occurred.

We should add redundancy to prevent downtime

Edit your Helm chart:
```
vi my-nginx/values.yaml
```
Change `replicaCount` from `1` to `2`.

Apply the change:
```
helm upgrade my-nginx ./my-nginx -n test
```
Verify pod distribution:
```
kubectl get pods -n test -o wide
```
You should see 2 pods on 2 different nodes.

Test again:
```
watch -n 0.5 curl -I http://chart-example.local
```
```
kubectl get pods -n test -o wide
```
Cordon and Drain one node with a pod.

You should not see any downtime in the curl output.

# Section 6: Final Notes

### ⚠️ Final Comments and Precautions

- AWS costs money. Delete all EC2 instances, Elastic IP, and IAM user if you don’t need them.
- Please be very careful not to leak IAM user's `Access key` and `Secret access key` to anyone. Your account could be misused by others. 
- If not needed, delete IAM user's `Access key`. If possible, delete the IAM user itself.
- Note: I’m not responsible for any issues caused by this course.


