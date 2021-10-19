# calico-cloud-legacy-server-policy
Creating network policy for legacy servers



## Install Docker on the legacy server
Check Docker Status
```
sudo systemctl status docker
```

Enable Docker runtime
```
sudo systemctl enable docker
```

Start the Docker Runtime
```
sudo systemctl start docker
```

```
sudo /usr/sbin/usermod -aG docker $USER
```

```
sudo chown $USER:docker /var/run/docker.sock
```


This command is outlined in the quickstart guide: <br/>
https://www.suse.com/products/suse-rancher/get-started/


## Install Docker on EC2 instances

Install Docker on each AWS EC2 instance as per the below workflow: <br/>
https://docs.docker.com/engine/install/ubuntu/

```
sudo su
```

```
apt-get update
```
```
apt-get install \
   apt-transport-https \
   ca-certificates \
   curl \
   gnupg \
   lsb-release
```

Add Docker’s official GPG key:
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

```
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## Install Docker Engine

```
apt-get update
```
```
apt-get install docker-ce docker-ce-cli containerd.io
```

## Step 1: Pulling images from Quay.io

If pulling images directly from quay.io/tigera, you will likely want to use the credentials provided to you by your Tigera support representative. <br/>
If using a private registry, use your private registry credentials instead.

```
kubectl create secret generic tigera-pull-secret \
    --type=kubernetes.io/dockerconfigjson -n tigera-operator \
    --from-file=.dockerconfigjson=<path/to/pull/secret>
```

## Step 2: Download and extract the binary

This step requires Docker, but it can be run from any machine with Docker installed. <br/>
It doesn’t have to be the host you will run it on (i.e your laptop is fine).<br/>
<br/>
Use the following command to download the cnx-node image.

```
docker pull quay.io/tigera/cnx-node:v3.10.0
docker pull cnx-node:
```

Confirm that the image has loaded by typing docker images.

```
REPOSITORY       TAG           IMAGE ID       CREATED         SIZE
quay.io/tigera/cnx-node      v3.10.0        e07d59b0eb8a   2 minutes ago   42MB
```

Create a temporary cnx-node container.

```
docker create --name container quay.io/tigera/cnx-node:v3.10.0
```

## Step X: Install kubectl binary with curl on Linux
https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux <br/>
<br/>
Download the latest release with the command:
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Validate the binary (optional)
```
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
```

Validate the kubectl binary against the checksum file:
```
echo "$(<kubectl.sha256) kubectl" | sha256sum --check
```

Install kubectl
```
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

If you do not have root access on the target system, you can still install kubectl to the ~/.local/bin directory:
```
chmod +x kubectl
mkdir -p ~/.local/bin/kubectl
mv ./kubectl ~/.local/bin/kubectl
# and then add ~/.local/bin/kubectl to $PATH
```

Test to ensure the version you installed is up-to-date:
```
kubectl version --client
```

## Download the Kubeconfig file from Rancher UI

Create a file and paste the contents of the downloaded Kubeconfig.yaml manifest:
```
vi kubeconfig.yaml
```

```
KUBECONFIG=kubeconfig.yaml
```

```
export KUBECONFIG=kubeconfig.yaml kubectl get nodes
```

You should now be able to see your 3 nodes (if the docker install command was used on each EC2 instance):
```
kubectl get nodes
```

Confirm all environmental variables are configured correctly:
```
env
```

Connect your cluster to Calico Cloud:
```
curl https://installer.calicocloud.io/YOUR-ACCOUNT_install.sh | bash
```

Once connected to Calico Cloud, you can see the new Calico deployment in your managed cluster view within the Rancher UI
![Screenshot 2021-09-27 at 13 16 11](https://user-images.githubusercontent.com/82048393/134906279-072f9da1-e21f-4c49-874f-21b9b90c7b0d.png)

## Building Calico Policies

The initial policy that comes packed with RKE clusters is an allow-all for the cattle-fleet-system namespace. <br/>
As you can see in the Calico Cloud web UI, it is placed in the 'default' tier - because Tiers is a unique CRD for Calico Cloud & Enterprise.

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-allow-all
  namespace: cattle-fleet-system
spec:
  podSelector: {}
  egress:
    - {}
  ingress:
    - {}
  policyTypes:
    - Ingress
    - Egress
```

![Screenshot 2021-09-27 at 13 54 12](https://user-images.githubusercontent.com/82048393/134911983-75be370a-c2d5-4224-b5f3-ea0abdb40926.png)









## Introducing a test application - "Storefront":
If your cluster does not have applications, you can use the following storefront application:
```
kubectl apply -f https://installer.calicocloud.io/storefront-demo.yaml
```
Create the Product Tier:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/product.yaml
```  


## Securing EKS hosts:

Automatically register your nodes as Host Endpoints (HEPS). To enable automatic host endpoints, edit the default KubeControllersConfiguration instance, and set spec.controllers.node.hostEndpoint.autoCreate to true:

```
kubectl patch kubecontrollersconfiguration default --patch='{"spec": {"controllers": {"node": {"hostEndpoint": {"autoCreate": "Enabled"}}}}}'
```

Add the label kubernetes-host to all nodes and their host endpoints:
```
kubectl label nodes --all kubernetes-host=  
```
This tutorial assumes that you already have a tier called 'aws-nodes' in Calico Cloud:  
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/node-tier.yaml
```
Once the tier is created, Build 3 policies for each scenario: 
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/etcd.yaml
```
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/master.yaml
```
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/worker.yaml
```

#### Label based on node purpose
To select a specific set of host endpoints (and their corresponding Kubernetes nodes), use a policy selector that selects a label unique to that set of host endpoints. For example, if we want to add the label environment=dev to nodes named node1 and node2:

```
kubectl label node ip-10-0-1-165 environment=master
kubectl label node ip-10-0-1-167 environment=worker
kubectl label node ip-10-0-1-227 environment=etcd
```

## Dynamic Packet Capture:

Check that there are no packet captures in this directory  
```
ls *pcap
```
A Packet Capture resource (PacketCapture) represents captured live traffic for debugging microservices and application interaction inside a Kubernetes cluster.</br>
https://docs.tigera.io/reference/calicoctl/captured-packets  
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/workloads/packet-capture.yaml
```
Confirm this is now running:  
```  
kubectl get packetcapture -n storefront
```
Once the capture is created, you can delete the collector:
```
kubectl delete -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/workloads/packet-capture.yaml
```
#### Install a Calicoctl plugin  
Use the following command to download the calicoctl binary:</br>
https://docs.tigera.io/maintenance/clis/calicoctl/install#install-calicoctl-as-a-kubectl-plugin-on-a-single-host
``` 
curl -o kubectl-calico -O -L  https://docs.tigera.io/download/binaries/v3.7.0/calicoctl
``` 
Set the file to be executable.
``` 
chmod +x kubectl-calico
```
Verify the plugin works:
``` 
./kubectl-calico -h
``` 
#### Move the packet capture
```
./kubectl-calico captured-packets copy storefront-capture -n storefront
``` 
Check that the packet captures are now created:
```
ls *pcap
```
#### Install TSHARK and troubleshoot per pod 
Use Yum To Search For The Package That Installs Tshark:</br>
https://www.question-defense.com/2010/03/07/install-tshark-on-centos-linux-using-the-yum-package-manager
```  
sudo yum install wireshark
```  
```  
tshark -r frontend-75875cb97c-2fkt2_enib222096b242.pcap -2 -R dns | grep microservice1
``` 
```  
tshark -r frontend-75875cb97c-2fkt2_enib222096b242.pcap -2 -R dns | grep microservice2
```  





## Wireguard In-Transit Encryption:

To begin, you will need a Kubernetes cluster with WireGuard installed on the host operating system.</br>
https://www.wireguard.com/install/ <br/>
<br/>
Installing wireguard on an Ubuntu EC2 instance
```
sudo apt install wireguard
```
Enable WireGuard encryption across all the nodes using the following command:
```
kubectl patch felixconfiguration default --type='merge' -p '{"spec":{"wireguardEnabled":true}}'
```
To verify that the nodes are configured for WireGuard encryption:
```
kubectl get node ip-192-168-30-158.eu-west-1.compute.internal -o yaml | grep Wireguard
```
Show how this has applied to traffic in-transit:
```
sudo wg show
```
