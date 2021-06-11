
### 如何重置k8s master

```
kubelet reset
systemctl restart kubelet

kubeadm init --pod-network-cidr=192.168.0.0/24
```

### 如果重置k8s node,重新加入cluster

```
kubelet reset
systemctl restart kubelet

kubeadm join ip:6443 --token key \
    --discovery-token-ca-cert-hash sha256:hash-key
```

### cert证书问题

```
ERROR:
[root@labs-k8s-master ~]# kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
Unable to connect to the server: x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes”)
```

```
solution:
$ mkdir -p $HOME/.kube
$ cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ chown $(id -u):$(id -g) $HOME/.kube/config

```

### k8s node join时候出错

```
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
    [ERROR FileContent--proc-sys-net-bridge-bridge-nf-call-iptables]: /proc/sys/net/bridge/bridge-nf-call-iptables contents are not set to 1
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```

```
solution:
vi /etc/sysctl.conf
add net.bridge.bridge-nf-call-iptables = 1
sysctl -b
```