# multimaster-kubernetes-cluster


K8s Multi-Master Setup

Master nodes	
K8s-Master1-RNI-PRF - 10.0.0.41/161.189.77.7 -- master1-pop-cn-k8s

K8s-Master2-RNI-PRF - 10.0.2.160/52.83.51.73 -- master2-pop-cn-k8s

K8s-Master3-RNI-PRF - 10.0.2.26/52.83.245.114 -- master3-pop-cn-k8s

Master nodes Setup with HA	


Nginix HA Proxy for Master	
Nginix HA Proxy for Master - 161.189.16.54

K8s-HA-Cluster-Nginx2 - 52.82.59.177

Nginx HA with keep-alive configuration to ensure it is available.	


Worker Nodes	AMI id :: ami-0d1bb5cad9153f1f8
AMI Name :: K8s-Master-node2 
for worker node	Sample worker nodes to check the sanity of the cluster.	









RDS Databases

Create RDS DB from AWS console and create schemas and login users.	mysql -h rni-prf-db.cpxs5kpbdspw.rds.cn-northwest-1.amazonaws.com.cn-u pop_prf -P 3306 -p	RDS created.
User details and login shared.	


Migration of VDS schema	DB dump from RNI Preprod to be migrated to new schema for VDS application cluster to setup.	















NFS  Server Setup

NFS nodes	nfs-1-gluster - 10.0.1.19
nfs-2-gluster - 10.0.3.169	Create 2 new nodes for NFS setup with HA.	


HA enabling	root@nfs-1-gluster:~# gluster pool list
UUID                                    Hostname        State
ee0cf751-5f36-494b-abc4-194493978c35    nfs-2-gluster   Connected
db192a7b-c126-4321-9674-90b8ce304f7d    localhost       Connected	Setup and configure GlusterFS for HA	


Components addition	
Creation of specific partitions for volumes.

root@nfs-1-gluster:~# gluster volume list
content_files
current_files
lti_files
moodle_files
postlogin_files
prelogin_files

GlusterFS brick partition.

/dev/xvdc                       788G   69G  679G  10% /nfs_gluster

Creation of the specific partitions for all components.

/dev/xvdc                       788G   69G  679G  10% /nfs_gluster
nfs-1-gluster:/prelogin_files   788G   77G  679G  11% /prelogin
nfs-1-gluster:/postlogin_files  788G   77G  679G  11% /postlogin
nfs-1-gluster:/lti_files        788G   77G  679G  11% /netacad_lti
nfs-1-gluster:/moodle_files     788G   77G  679G  11% /moodle
nfs-1-gluster:/content_files    788G   77G  679G  11% /content
nfs-1-gluster:/current_files    788G   77G  679G  11% /current




Migration of Static files from China beta to new cluster	

All data is migrated.	


Client side configuration and mounting NFS	Refer:
https://wiki.cisco.com/display/NETACAD/High-Availability+NFS+Cluster+with+GlusterFS	Need to configure for client side while launching application on worker nodes	















       ElasticCache

Creation of 3 Redis clusters with 2 replicas and 1 master for both Content and session	
Prelogin:

prelogin-redis-cn-rni.9z9dep.ng.0001.cnw1.cache.amazonaws.com.cn:6379

Postlogin:

postlogin-redis-cn-rni.9z9dep.ng.0001.cnw1.cache.amazonaws.com.cn:6379

VDS:

vds-redis-cn-rni.9z9dep.ng.0001.cnw1.cache.amazonaws.com.cn:6379



Cache Size compared against GNI Prod to check trends and we have created t3.medium for

VDS and Prelogin and m4.large  for Postlogin.



Based on the trends of Perf test we can modify.

CLI command : run from k8s master1)

redis-cli -h vds-redis-cn-rni.9z9dep.ng.0001.cnw1.cache.amazonaws.com.cn INFO








 







   VDS with Mongo

Create worker nodes for VDS application	
Host details :: 10.0.0.92 / 161.189.79.103






Create Database and Redis

rni-prf-db.cpxs5kpbdspw.rds.cn-northwest-1.amazonaws.com.cn

popvds - Schema



Create MongoDB and Configurations	
Mongo Host:52.82.106.24

Version: Mongo 4.4.2




Create Application Pods	2 pods running	


Deployment of image

happy:9/latest	


Creation of User Accesses and Logs scripts on application.	Provided Access to Charles, Bon. Josh and Aravind.	








High Availability Kubernetes Cluster
Kubernetes 1.14 introduced an new feature for dynamically adding master nodes to a cluster. This prevents the need to copy certificates and keys among nodes relieving additional orchestration and complexity in the bootstrapping process.
Architecture of Kubernetes HA




Master Node: Each master node in a multi-master environment run its’ own copy of Kube API server. This can be used for load balancing among the master nodes. Master node also runs its copy of the etcd database, which stores all the data of cluster. In addition to API server and etcd database, the master node also runs k8s controller manager, which handles replication and scheduler, which schedules pods to nodes.

Worker Node: Like single master in the multi-master cluster also the worker runs their own component mainly orchestrating pods in the Kubernetes cluster.

Reference:  https://kubernetes.io/docs/concepts/overview/components/            



How It Works
kubeadm init initializes a Kubernetes cluster by standing up a single master. After running the command, you end up with the following.



This represents the default. Through additional configuration,  kubeadm init can behave differently, such as reusing an existing etcd cluster.

Conventionally, after installing a CNI plugin, users copy PKI information across 2 more master nodes and run a kubeadm command to add new control plane nodes. This results in a 3 node control plane.

1.14 introduced the --upload-certs flag to the kubeadm init command.It has the following impact.

Creates an encryption key on the host.

Encrypts certificates and keys with the encryption key.

Adds encrypted data to the kubeadm-certs secret in the kube-system namespace.

Links the kubeadm-certs secret to a kubeadm token with a 1 hour TTL.

Taking the example above, if we run the following command, it will result in the new diagram below.

kubeadm init --experimental-upload-certs


m ini


When the kubeadm token expires, so does the kubeadm-certs secret. Also, whenever the init phase upload-certs is run, a new encryption key is created. Ensuring that if kube-apiserver is compromised during the adding of a master node, the secret is encrypted and meaningless to the attacker. This example demonstrates running --experimental-upload-certs during cluster bootstrap. It is also possible to tap into phases, using kubeadm init phase upload-certs to achieve the above on an existing master. This will be detailed in the walkthrough below.

Lastly, it is important to note the token generated during upload-certs is only a proxy used to bind a TTL to the kubeadm-cert secret. You still need a conventional kubeadm token to join a control plane. This is the same token you would need to join a worker.

To add another control plane (master) node, a user can run the following command.

kubeadm join ${API_SERVER_PROXY_IP}:${API_SERVER_PROXY_PORT} \
    --experimental-control-plane \
    --certificate-key=${ENCRYPTION_KEY} \
    --token ${KUBEADM_TOKEN} \
    --discovery-token-ca-cert-hash ${APISERVER_CA_CERT_HASH}
How to Create HA LB?
→  Using Active-Passive HA for NGINX Plus on AWS Using Elastic IP Addresses


Walkthrough: Creating the HA Control Plane
This walkthrough will guide you to creating a new Kubernetes cluster with a 3 node control plane. It will demonstrate joining a control plane right after bootstrap and how to add another control plane node after the bootstrap tokens have expired.

Create 4 hosts (EC2,vms, baremetal, etc).

These hosts will be referred to as loadbalancer, master1, master2, and master3.

master hosts may need 2 vCPUs and 2GB of RAM available. You can get around these requirements during testing by ignoring pre-flight checks. See the kubeadm documentation for more details.

Record the host IPs for later use.

Setup the Load Balancer
In this section, we’ll run a simple NGINX load balancer to provide a single endpoint for our control plane.

SSH to the loadbalancer host.

Create the directory /etc/nginx.

mkdir /etc/nginx


Add and edit the file /etc/nginx/nginx.conf.


vi /etc/nginx/nginx.conf


Inside the file, add the following configuration.
events { }
stream {
    upstream stream_backend {
        least_conn;
        # REPLACE WITH master1 IP
        server 172.31.11.242:6443 max_fails=1 fail_timeout=3s;
        # REPLACE WITH master2 IP
        server 172.31.7.31:6443 max_fails=1 fail_timeout=3s;
         # REPLACE WITH master3 IP
        server 172.31.26.190:6443 max_fails=1 fail_timeout=3s;
    }
    server {
        listen        6443;
        proxy_pass    stream_backend;
        proxy_timeout 3s;
        proxy_connect_timeout 1s;
    }
}


Alter each line above with a REPLACE comment above it.

Start NGINX.   

docker run --name proxy \
    -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
    -p 6443:6443 \
    -d nginx
Note: For HA for this process we can use the docker-compose file, which can be deployed on docker swarm.

Verify you can reach NGINX at its address.

curl 172.31.11.139
output:

curl: (52) Empty reply from server
Install Kubernetes Binaries on Master Hosts
Complete all pre-requisites and installation on master nodes.



1)#Get the Docker gpg key:
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
2)#Add the Docker repository:
   sudo add-apt-repository    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
3)#Get the Kubernetes gpg key:
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
4)#Add the Kubernetes repository:
  cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
  deb https://apt.kubernetes.io/ kubernetes-xenial main
  EOF
5)#Update your packages:
  sudo apt-get update
6)#Install Docker, kubelet, kubeadm, and kubectl:
  sudo apt-get install -y docker-ce=18.06.1~ce~3-0~ubuntu kubelet=1.14.0-00 kubeadm=1.14.0-00 kubectl=1.14.0-00
7)#Hold them at the current version:
  sudo apt-mark hold docker-ce kubelet kubeadm kubectl
8)#Add the iptables rule to sysctl.conf:
  echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
9)#Enable iptables immediately:
  sudo sysctl -p
10)swapoff -a


       Reference:- https://kubernetes.io/docs/setup/independent/install-kubeadm.

Initialize the Cluster and Examine Certificates
SSH to the master0 host.

Create the directory /etc/kubernetes/kubeadm

mkdir /etc/kubernetes/kubeadm


Create and edit the file /etc/kubernetes/kubeadm/kubeadm-config.yaml.

vi /etc/kubernetes/kubeadm/kubeadm-config.yaml

Add the following configuration.

apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: stable
# REPLACE with `loadbalancer` IP
controlPlaneEndpoint: "172.31.11.139:6443"
networking:  
  podSubnet: "10.0.0.0/18"  
  serviceSubnet: "172.31.0.0/16"


Alter the line with a REPLACE comment above it.

Initialize the cluster with upload-certs and config specified.

kubeadm init \
    --config=/etc/kubernetes/kubeadm/kubeadm-config.yaml \
    --experimental-upload-certs


Record the output regarding joining control plane nodes for later use.

output:

You can now join any number of the control-plane node running the following command on each as root:
  kubeadm join 172.31.11.139:6443 --token 3c691q.chi5w4hgguodt6bi \
    --discovery-token-ca-cert-hash sha256:818c83c5c0721fd2d7d120bd06b1d1d0fd5e634dcaa44626bac77e2c637e4850 \
    --experimental-control-plane \
     --certificate-key f5c405e3158eac678494ea433a3b485a14176603d1b67fdcec811735595bbdd4
Note the certificate-key that enables decrypting the kubeadm certs secret.

As your user, run the recommended kubeconfig commands for kubectl access.

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


Examine the kubeadm-cert secret in the kube-system namespace.

kubectl get secrets -n kube-system kubeadm-certs -o yaml


You should see certs for etcd, kube-apiserver, and service accounts.

Under ownerReferences, examine the name.

name: bootstrap-token-cwb9ra
This correlates to an ephemeral kubeadm token. When that token expires, so does this secret.

List available tokens with kubeadm.

kubeadm token list
output:

TOKEN                     TTL       EXPIRES                USAGES                   DESCRIPTION                                       
cwb9ra.gegoj2eqddaf3yps   1h        2019-03-26T19:38:18Z   <none>                   Proxy for managing TTL for the kubeadm-certs secret
nmiqmn.yls76lcyxg2wt36c   23h       2019-03-27T17:38:18Z   authentication,signing   <none>                                              
Note that cwb9ra is the owner reference in the above step. This is not a join token, instead a proxy that enables ttl on kubeadm-certs. We still need to use the nmiqmn token when joining.

Install calico CNI-plugin with a pod CIDR matching the podSubnet configured above.

kubectl apply -f https://gist.githubusercontent.com/joshrosso/ed1f5ea5a2f47d86f536e9eee3f1a2c2/raw/dfd95b9230fb3f75543706f3a95989964f36b154/calico-3.5.yaml

Verify 1 node is Ready.

kubectl get nodes
output:

NAME       STATUS   ROLES    AGE   VERSION
master1     Ready    master  79m   v1.14.0


Verify kube-system pods are Running.

kubectl get pods -n kube-system
output:

NAME                            READY   STATUS    RESTARTS   AGE
calico-node-mphpw                1/1     Running   0          58m
coredns-fb8b8dccf-c6s9q          1/1     Running   0          80m
coredns-fb8b8dccf-mxzrm          1/1     Running   0          80m
etcd-master1                     1/1     Running   0          79m
kube-apiserver-master1           1/1     Running   0          79m
kube-controller-manager-master1  1/1     Running   0          79m
kube-proxy-dpxhx                 1/1     Running   0          80m
kube-scheduler-master1           1/1     Running   0          79m
Add the Second Master
SSH to the master1 host.

Run the recorded join command from the previous section.

kubeadm join 172.31.11.139:6443 --token 3c691q.chi5w4hgguodt6bi \ 
   --discovery-token-ca-cert-hash sha256:818c83c5c0721fd2d7d120bd06b1d1d0fd5e634dcaa44626bac77e2c637e4850 \
   --experimental-control-plane \
   --certificate-key f5c405e3158eac678494ea433a3b485a14176603d1b67fdcec811735595bbdd4


After completion, verify there are now 2 nodes.

kubectl get nodes
output:

NAME      STATUS   ROLES    AGE   VERSION
master1   Ready    master   22m   v1.14.0
master2   Ready    master   34s   v1.14.0
Verify new pods have been created.

kubectl get pods -n kube-system
output:

NAME                            READY   STATUS    RESTARTS   AGE
calico-node-cq5nt                1/1     Running   0          60s
calico-node-spn5w                1/1     Running   0          13m
coredns-fb8b8dccf-r9sc8          1/1     Running   0          22m
coredns-fb8b8dccf-wlcm4          1/1     Running   0          22m
etcd-master1                     1/1     Running   0          21m
etcd-master2                     1/1     Running   0          59s
kube-apiserver-master1           1/1     Running   0          21m
kube-apiserver-master2           1/1     Running   0          59s
kube-controller-manager-master1  1/1     Running   0          21m
kube-controller-manager-master2  1/1     Running   0          60s
kube-proxy-tflhf                 1/1     Running   0          60s
kube-proxy-vthjr                 1/1     Running   0          22m
kube-scheduler-master1           1/1     Running   0          22m
kube-scheduler-master2           1/1     Running   0          59s
Add the Third Master with New Tokens
This section joins the third and final master. However, we will first delete all existing kubeadm tokens. This approach demonstrates how you could add masters when the Kubernetes cluster is already running.

On an existing master, list all tokens.

kubeadm token list


Delete all existing tokens.

kubeadm token delete cwb9ra.gegoj2eqddaf3yps
kubeadm token delete nmiqmn.yls76lcyxg2wt36c
Now the previously recorded join command will not work as the kubeadm-certs secret has expired and been deleted, the encryption key is no longer valid, and the join token will not work.

Create a new token with a 10 minute TTL.

kubeadm token create --ttl 10m --print-join-command
Or
kubeadm token create --ttl=0 --print-join-command


output:

kubeadm join 172.31.11.139:6443 \
    --token xaw58o.0fjg0xp0ohpucwhr \
    --discovery-token-ca-cert-hash sha256:818c83c5c0721fd2d7d120bd06b1d1d0fd5e634dcaa44626bac77e2c637e4850 
Note the IP above must reflect your loadbalancer host.

Run the upload-certs phase of kubeadm init.

kubeadm init phase upload-certs --experimental-upload-certs
output:

[upload-certs] Storing the certificates in ConfigMap "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
9555b74008f24687eb964bd90a164ecb5760a89481d9c55a77c129b7db438168
SSH to the master3 host.

Use the outputs from the previous steps to run a control-plane join command.

kubeadm join 172.31.11.139:6443 \
    --experimental-control-plane \
    --certificate-key f5c405e3158eac678494ea433a3b485a14176603d1b67fdcec811735595bbdd4 \
    --token xaw58o.0fjg0xp0ohpucwhr \
    --discovery-token-ca-cert-hash sha256:818c83c5c0721fd2d7d120bd06b1d1d0fd5e634dcaa44626bac77e2c637e4850

After completion, verify there are now 3 nodes.

kubectl get nodes
output:

NAME      STATUS   ROLES    AGE     VERSION
master1   Ready    master   50m     v1.14.0
master2   Ready    master   28m     v1.14.0
master3   Ready    master   3m30s   v1.14.0
Add the Worker Nodes
SSH to the worker1 host.

Run the recorded join command from the recorded section.

kubeadm join 172.31.11.139:6443 --token 3c691q.chi5w4hgguodt6bi \
    --discovery-token-ca-cert-hash sha256:818c83c5c0721fd2d7d120bd06b1d1d0fd5e634dcaa44626bac77e2c637e4850


After completion, verify the worker node.

kubectl get nodes
output:

NAME      STATUS   ROLES    AGE     VERSION
master1   Ready    master   50m     v1.14.0
master2   Ready    master   28m     v1.14.0
master3   Ready    master   3m30s   v1.14.0
worker1   Ready    <none>   1m      v1.14.0


Deploy the Test application:
Reference: https://www.linode.com/docs/kubernetes/how-to-deploy-nginx-on-a-kubernetes-cluster/

Test the different failure scenarios:
Stop the host master2 & observe  the behavior of "watch kubectl get nodes"

Start the host master2 & observe the behavior of "watch kubectl get nodes".

Access the application continuously using script & observed the behavior.
Repeat the steps 1-3 for different master & worker nodes.


References:
1) https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/

2) https://medium.com/velotio-perspectives/demystifying-high-availability-in-kubernetes-using-kubeadm-3d83ed8c458b



how to create a highly available (HA) active‑passive deployment of NGINX Plus in the Amazon Web Services (AWS) cloud. It combines the keepalived‑based solution for high availability (provided by NGINX for on‑premises HA deployments) with the AWS Elastic IP address feature.

K8s-Master1-RNI-PRF - 10.0.0.41/161.189.77.7 -- master1-pop-cn-k8s
   K8s-Master2-RNI-PRF - 10.0.2.160/52.83.51.73 -- master2-pop-cn-k8s
   K8s-Master3-RNI-PRF - 10.0.2.26/52.83.245.114 -- master3-pop-cn-k8s
