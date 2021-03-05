# Kubernetes-Multi-Node-Cluster-Over-AWS-Cloud
#
* Here we will see how to create ```Kubernetes-Multi-Node-Cluster-Over-AWS-Cloud```. First we have to launch ec2 instances on AWS Cloud one as ```Master``` others as```Slaves```.
* We can use ```t2.micro``` Instance Type which comes under free-tier. 

> After the instances are launched, Now we can now setup our Cluster on them.

**Master Node Setup**
* First we need to install ```docker```. If we are using ```Amazon Linux 2 AMI``` we can install docker using
```
yum install docker
```
We can enable docker so everytime after we start instance it will always be in started state.
```
 systemctl enable docker --now
```

* Now we need to create a yum repository for Kubernetes
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```
> Now we are ready to install ```kubeadm```, ```kubelet```, ```kubectl```.
> Link for the Kubeadm Installation Guide: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
* kubeadm: the command to bootstrap the cluster.

* kubelet: the component that runs on all of the machines in your cluster and does things like starting pods and containers.

* kubectl: the command line util to talk to your cluster.

```
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
> After installing now we can enable kubelet. So if you restart the OS then by default the kubelet will be enabled.
```
systemctl enable --now kubelet
```
> Now we have to pull the images of different types of resources using kubeadm.
```
 kubeadm config images pull
 ```
 Kuberenets has to run differmt process behind the scene. So, to install these resources it uses containers and in the containers we will be having the required resources eg.
- kube-apiserver:v1.20.2
- kube-controller-manager:v1.20.2
- kube-scheduler:v1.20.2
- kube-proxy:v1.20.2
- pause:3.2
- etcd:3.4.13-0
- coredns:1.7.0

* Now we need to change the driver in the docker. By default docker uses cgroupfs driver. So we need to change it to systemd driver.
```
cd /etc/docker

vi daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```
> Restarting docker service
```
systemctl restart docker

```
> Now after restarting docker we check the driver ```docker info | grep Driver```. It will get updated to ```systemd```

Now we can proceed further with installing ```iproute-tc``` require to set the routing path.
```
yum install iproute-tc
```

> Now we need to set the bridge routing to 1 using 
```
echo "1" > /proc/sys/net/bridge/bridge-nf-call-iptables
```
* Now we are all set to initialize the master with respective pod-network-cidr. We can initialize using
```
kubeadm init --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=NumCPU --ignore-preflight-errors=Mem
```

- --pod-network-cidr=10.244.0.0/16: containers will be launched within this range of network.
- --ignore-preflight-errors=NumCPU: As we are using t2.micro i.e. 1GB RAM and 1 CPU so to ignore the errors of CPU we are using this.
- --ignore-preflight-errors=Mem: We have only 1GB RAM so to ignore errors on Mem we are using this.

> After initializing master it will give you commands to run. Below are the commands we need to run
```
 mkdir -p $HOME/.kube
 sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 sudo chown $(id -u):$(id -g) $HOME/.kube/config
 ```
 > After running this it kubetnetes creates a .kube from the above first command and inside that we need to create a config file (configuration file) i.e from second command. Now to change the owner permission we are using chown so that it will change the owner permission of config file inside .kube.
 
> Now if you run ```kubectl get pods``` you will find that you don't have any resources in the default namespace. 
```
kubectl get pods --all-namespaces

kubectl status kubelet
```
* To create Token for slave/worker nodes so that they can join to Master using
```
kubeadm token create  --print-join-command
```

If we run ```kubectl get nodes``` we will see that master node will show status of ```Not Ready```.
> To make this ready we need to make the ```overlay connection```. We need to use ```flannel plugin```. 
- Flannel is a plugin that gives a facility of overlay network.
```
kubectl apply  -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
Now we can see that our master node gets ```Ready```.

**Slave/Worker Node Setup**

> We have follow same steps to setup slaves/worker Nodes
* Installing docker and enabling docker service
```
yum install docker -y
systemctl enable docker --now
```

* After that we need to create a repository of kubernetes ```vi /etc/yum.repos.d/kubernetes.repo``` and write the command in that file
```
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
```
* Now we can proceed to install ```kubeadm```, ```kubectl```, ```kubelet```.
```
 yum install -y  kubectl kubelet  kubeadm  --disableexcludes=kubernetes
 systemctl enable kubelet --now
 ```
 * Now we need to pull the images using kubeadm ```kubeadm config images pull```
 
 Now we need to change the driver in the docker. By default docker uses cgroupfs driver. So we need to change it to systemd driver.
```
cd /etc/docker

vi daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```
> Restarting docker service
```
systemctl restart docker

```
> Now after restarting docker we check the driver ```docker info | grep Driver```. It will get updated to ```systemd```

* Now we can proceed further with installing ```iproute-tc``` require to set the routing path.
```
yum install iproute-tc
```

> Now we need to set the bridge routing to 1 using 
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

```
> Now we are ready with our slave node to join to master node. To Join we have to use the token generated from master and use ```kubeadm join``` command.
> For Example token generated will look like
```
kubeadm join 172.31.40.209:6443 --token 3joamn.y8asi8hw2jb41xx5 --discovery-token-ca-cert-hash sha256:86dd3714b1168b8de0abcb63300a458baf838860ff9c6f53bd85707853ff4db7
```

> Now go to master and then run ```kubectl get nodes``` you will find that slave is now connected to master.
#

* We can now launch as many slave nodes in same way and connect it to the Cluster.

