## Centos7下安装k8s(CNI插件为flannel)
## Install Prerequisites on ALL (Worker and Master) Nodes
Let's remove any old versions of Docker if they exist:

```shell
sudo yum remove docker \
                  docker-common \
                  docker-selinux \
                  docker-engine
```

And let's (re)install a fresh copy of Docker:

`yum install docker`

We will also need to install the latest release of kubectl, which is used to control Kubernetes. The instructions are straight from https://kubernetes.io/docs/tasks/tools/install-kubectl/

`curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl`

Make it executable:

`chmod +x ./kubectl`

And pop it into the PATH:

`sudo mv ./kubectl /usr/local/bin/kubectl`

## Install kubelet and kubeadm on ALL (Worker and Master) Nodes
This is straight from https://kubernetes.io/docs/setup/independent/install-kubeadm/

```shell
sudo cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
sudo setenforce 0
sudo yum install -y kubelet kubeadm
sudo systemctl enable kubelet && systemctl start kubelet
```

Typically, you would need to install the CNI packages, but they're already installed in this case.

# Configure Kubernetes Master
On the master node, we want to run:

`sudo kubeadm init --pod-network-cidr=192.168.0.0/24`

The `--pod-network-cidr=192.168.0.0/24` option is a requirement for Flannel - don't change that network address!

Save the command it gives you to join nodes in the cluster, but we don't want to do that just yet. You should see a message like 

```
You can now join any number of machines by running the following on each node as root:

  kubeadm join --token <token> <IP>:6443
```

Start the cluster as a normal user. This part, I realized, was pretty important as it doesn't like to play well when you do it as root.

```shell
sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
```

# Install Flannel for the Pod Network (On Master Node)
We need to install the pod network before the cluster can come up. As such we want to install the latest yaml file that flannel provides. Most installations will use the following:

```shell
kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
```
At this point, give it a minute, and have a look at the status of the cluster. run `kubectl get pods --all-namespaces` and see what it comes back with. If everything shows running, then you're in business! Otherwise, if you notice errors like:

```
NAMESPACE     NAME                                                    READY     STATUS              RESTARTS   AGE
...
kube-system   kube-flannel-ds-knq4b                                   1/2       Error               5          3m
...
```

or

```
NAMESPACE     NAME                                                    READY     STATUS              RESTARTS   AGE
...
kube-system   kube-flannel-ds-knq4b                                   1/2       CrashLoopBackOff    5          5m
...
```

If this is the case, you will need to run the RBAC module as well:

```shell
kubectl create -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel-rbac.yml
```

# Add Worker Nodes to the Cluster
Upto this point, we haven't really touched the worker nodes (other than installing the prerequisites), but now you can join the worker nodes by running the command that was given to us when we created the cluster

```shell
sudo kubeadm join --token <token> <ip>:6443
```

We'll see more services spinning up on other services:

`kubectl get pods --all-namespaces`

```
NAMESPACE     NAME                                                    READY     STATUS    RESTARTS   AGE
...
kube-system   kube-flannel-ds-fldtn                                   0/2       Pending   0          3s
...
kube-system   kube-proxy-c8s32                                        0/1       Pending   0          3s
```

And to confirm, when we do a `kubectl get nodes`, we should see something like:

```
NAME                            STATUS    AGE       VERSION
server1                         Ready     46m       v1.7.0
server2                         Ready     3m        v1.7.0
server3                         Ready     2m        v1.7.0
```

# Running Workloads on the Master Node
By default, no workloads will run on the master node. You usually want this in a production environment. In my case, since I'm using it for development and testing, I want to allow containers to run on the master node as well. This is done by a process called "tainting" the host.

On the master, we can run the command `kubectl taint nodes --all node-role.kubernetes.io/master-` and allow the master to run workloads as well.