# Kubernetes on-premises cluster

A kubernetes on-premises cluster is a cluster which doesn't have a cloud provider to operate the cluster. In a nutshell, we can consider as an on-premises cluster, a cluster that isn't controller by a cloud manager(e. g.: Google cloud, AWS, Azure, etc...).

In this tutorial we'll use a cluster hosted in an OpenStack environment but don't controlled by the OpenStack Cloud provider.

# Cluster nodes

This cluster is composed for 1 master(`corc-tutorial-master`) and 2 nodes(`corc-tutorial-node1` and `corc-tutorial-node2`) having the configuration below 

```
NAME                    OS              IP ADDRESS       vCPU    DISK    RAM
corc-tutorial-master    ubuntu-16.04	10.11.4.189       2       40     2GB
corc-tutorial-node1     ubuntu-16.04	10.11.4.113       1       20     1GB
corc-tutorial-node2     ubuntu-16.04	10.11.4.199       1       20     1GB

```

# Setting up the master

In this tutorial, the kubernetes master needs the follow packages to work fine: docker, kubeadm(to initialize the cluster), kubelet(which is the kubernetes agent), kubectl and kubernetes-cni(a kubernetes network plugin).

Consider that to the steps below, we are connected in the `corc-tutorial-master` over SSH.

`$ ssh -i my_key.pem ubuntu@10.11.4.189`

## Installing the packages

- As root, you should install `apt-transport-http` to enable the usage of `deb` lines at `/etc/source.list`

`root@corc-tutorial-master:/home/ubuntu# apt-get update && apt-get install -y apt-transport-https`

- Now is necessary download the key used by apt to authenticate the package

`root@corc-tutorial-master:/home/ubuntu# curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -`

- Then you should add the official kubernete repository in `/etc/apt/source.list`

```
$ cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

- Update the index

`root@corc-tutorial-master:/home/ubuntu# apt-get update`

- And finally install the packages

`root@corc-tutorial-master:/home/ubuntu# apt-get install docker.io kubelet kubeadm kubectl kubernetes-cni`

## Initializing the kubernetes master

- Now you are able to use the `kubeadm` to initialize the kubernetes cluster(master)

`ubuntu@corc-tutorial-master:~$ sudo kubeadm init`

After use the command above, some instructions will appear, you should follow this instructions as below

```
ubuntu@corc-tutorial-master:~$ mkdir -p $HOME/.kube
ubuntu@corc-tutorial-master:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
ubuntu@corc-tutorial-master:~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

A command similar to the below will be showed in the same instructions

`kubeadmin join --token #{CLUSTER_TOKEN} 10.11.4.189:6443`

Save this command, it will be used to join nodes in your cluster.

- Now you need to enable the networking and network policy in kubernetes across the cluster, for it we are using `calico`

`ubuntu@corc-tutorial-master:~$ kubectl apply -f http://docs.projectcalico.org/v2.3/getting-started/kubernetes/installation/hosted/kubeadm/1.6/calico.yaml`

- Ok, now you can check if everything happened good

```
ubuntu@corc-tutorial-master:~$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                           READY     STATUS    RESTARTS   AGE
kube-system   calico-etcd-1fkw1                              1/1       Running   0          2m
kube-system   calico-node-n2rq8                              2/2       Running   0          2m
kube-system   calico-policy-controller-1727037546-n1qk0      1/1       Running   0          2m
kube-system   etcd-corc-tutorial-master                      1/1       Running   0          10m
kube-system   kube-apiserver-corc-tutorial-master            1/1       Running   0          10m
kube-system   kube-controller-manager-corc-tutorial-master   1/1       Running   1          15m
kube-system   kube-dns-2425271678-hx0x4                      3/3       Running   1          15m
kube-system   kube-proxy-m3kjd                               1/1       Running   0          
kube-system   kube-scheduler-corc-tutorial-master            1/1       Running   0          15m

```

Now you can join a node in this cluster using `kubedm join` command with cluster's token and `IP:PORT`.

# Joining a node in the cluster

For the steps below, I will only use one of the nodes listed in the section [Cluster nodes](#), to join the second node, you only need to do this steps on the other one. I choose `corc-tutorial-node1`, so all commands above will be executed in this host connected over SSH.

`$ ssh -i my_key.pem ubuntu@10.11.4.113`

## Installing the packages

This step is exactly the same as in [the master](#), so you just need repeat it in each node.

## Joining the node using kubeadm

- Is very simple to join in a cluster, you just need use the command displayed in the moment that you create the cluster(master initialization)

`kubeadmin join --token #{CLUSTER_TOKEN} 10.11.4.189:6443`

- If you don't remember the cluster's token, you can list it using the `kubeadm`

```
ubuntu@corc-tutorial-master:~$ sudo kubeadm token list
TOKEN                     TTL         EXPIRES   USAGES                   DESCRIPTION
#{SOME_TOKEN}          <forever>   <never>   authentication,signing   The default bootstrap token generated by 'kubeadm init'.
```

# Checking if the nodes are joined successfully

To check if you have success in your setup, you just need the `kubectl`. Using the `kubectl get nodes` command in master, you will be able to list all nodes joined in this cluster

```
ubuntu@corc-tutorial-master:~$ kubectl get nodes
NAME                   STATUS    AGE       VERSION
corc-tutorial-master   Ready     48m        v1.7.3
corc-tutorial-node1    Ready     15m        v1.7.3
corc-tutorial-node2    Ready     5m         v1.7.3
```

Now you are able to manage your cluster using `kubectl`. For using `kubectl` from your local computer, you just need a copy of the `$HOME/.kube/config` file in your computer and make sure that **master is accessible from your computer**.
