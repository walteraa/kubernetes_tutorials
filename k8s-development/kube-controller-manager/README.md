# Testing a code change in the Kube Controller Manager(KCM)

In this tutorial, I will show how to change KCM code and test it in a live cluster. For this tutorial, I will only add a Log entry in the Namespace delete process and make this log appear in the pod logs.

## Creating a branch to add the changes

Once you have finished the tutorial [Setting up the kubernetes development environment](..), you just need create a branch in your repository

`$ cd $working_dir/kubernetes`
`$ git checkout -b adding-log-entry`


Now you can do changes safely.

## Changing the code

Now I will change the file `$working_dir/kubernetes/pkg/controller/namespaces_controller.go` and add a log in the method `syncNamespaceFromKey`, which is the method that ensures that the nasmespaces has been deleted successfully.

The log entry is 

`glog.Info("\n\n---> Our log here!!!\n\n")`


Now you can build the `kube-controller-manager` and a binary will be generated with the new change.

## Building the new version

To build the newest version of `kube-controller-manager` binary, you need only run the command `build/run.sh make kube-controller-manager` from the `$working_dir/kubernetes` root and wait while the code is compiled.

After the compile proccess has been finished, you can find the new binary version in the path `$working_dir/_output/bin`.

## Building and pushing the new kube-controller-manager docker image

### Configuring the repository environment

For this step you will need a docker image which will run the `kube-controller-manager` in your cluster, to manage all resources. For that you just need follow the steps below.

### Building the image

To build the image, you will need a simple Dockerfile to define your image over a `ubuntu-slim` source

```
FROM gcr.io/google_containers/ubuntu-slim:0.9

ADD kube-controller-manager /usr/bin

CMD ["cmod", "+x", "/usr/bin/kube-controller-manager"]
```

You should create this Dockerfile inside the same path where the `kube-controller-manager` is. Then you can execute the the `docker build` command  in the same path that you have defined your Dockerfile, using the repository name and tag where you will set in the kube-controller manifest.

`$ docker build -t walteraa/kube-controller-manager-amd64:v0.0.1 .`

Now you have a image prepared to be used in your cluster.

### Pushing the image

Once you have created the image, now you just need login in your docker account

`$ docker login`

(Provide your account credentials)

After successfully log-in your docker account, you just need push your image.

`$ docker push walteraa/kube-controller-manager-amd64:v0.0.1`

Now you can use your `kube-controller-manager` version.

## Configuring you cluster to use your kube-controller-manager version

First of all, you need log in the master of the cluster to make the changes below


`$ ssh ubuntu@10.11.4.189`

After that you need change the `kube-controller-manager` configuration in your cluster to make it always use your image version instead of the default version.Then you will need change the manifest file 

`# vim /etc/kubernetes/manifest/kube-controller-manager` 

Change the spec to use your image

```yaml
#(...)
spec:
  containers:
  - command:
    - kube-controller-manager
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --use-service-account-credentials=true
    - --leader-elect=true
    - --controllers=*,bootstrapsigner,tokencleaner
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --address=127.0.0.1
    image: docker.io/walteraa/kube-controller-manager-amd64:v0.0.1
#(...)
```

Restart the kubelet

`# systemctl restart kubelet.service`

Kill the current `kube-controller-manager` pod

```
ubuntu@corc-tutorial-master:~$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                           READY     STATUS    RESTARTS   AGE
default       federated-nginx-deployment-519985460-5bjhs     1/1       Running   0          18d
default       federated-nginx-deployment-519985460-ft240     1/1       Running   0          19d
default       federated-nginx-deployment-519985460-rtcr2     1/1       Running   0          18d
default       federated-nginx-deployment-519985460-s23kl     1/1       Running   0          21d
default       federated-nginx-deployment-519985460-v51q2     1/1       Running   0          18d
kube-system   calico-etcd-1fkw1                              1/1       Running   0          28d
kube-system   calico-node-1lkqt                              2/2       Running   6          28d
kube-system   calico-node-g911b                              2/2       Running   0          28d
kube-system   calico-node-n2rq8                              2/2       Running   0          28d
kube-system   calico-policy-controller-1727037546-n1qk0      1/1       Running   0          28d
kube-system   etcd-corc-tutorial-master                      1/1       Running   14         28d
kube-system   kube-apiserver-corc-tutorial-master            1/1       Running   4          28d
kube-system   kube-controller-manager-corc-tutorial-master   1/1       Running   2          16h
kube-system   kube-dns-2425271678-hx0x4                      3/3       Running   4          28d
kube-system   kube-proxy-5bvl6                               1/1       Running   0          28d
kube-system   kube-proxy-5kwc9                               1/1       Running   0          28d
kube-system   kube-proxy-m3kjd                               1/1       Running   2          28d
kube-system   kube-scheduler-corc-tutorial-master            1/1       Running   20         28d

ubuntu@corc-tutorial-master:~$ kubectl delete pod -n kube-system  kube-apiserver-corc-tutorial-master
pod "kube-apiserver-corc-tutorial-master" deleted
```

And finally ensure that the new version is being using the command below to check the image spec

`ubuntu@corc-tutorial-master:~$ kubectl get pod -n kube-system   kube-controller-manager-corc-tutorial-master -o yaml`

```yaml
#(...)
spec:
  containers:
  - command:
    - kube-controller-manager
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --use-service-account-credentials=true
    - --leader-elect=true
    - --controllers=*,bootstrapsigner,tokencleaner
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --address=127.0.0.1
    image: docker.io/walteraa/kube-controller-manager-amd64:v0.0.1
#(...)
```

Now your cluster is using your `kube-controller-manager` version

## Checking our log execution in the controller manager

Remember that Log that you have added in the code? Now we can check if it is being executed

First add a new namespace which you can delete right after

```
ubuntu@corc-tutorial-master:~$ kubectl create ns corc-tutorial
namespace "corc-tutorial" created
```

Now delete it

```
ubuntu@corc-tutorial-master:~$ kubectl delete ns corc-tutorial
namespace "corc-tutorial" deleted
```

And check if the log appear in your pod log

```
(...)
I0905 13:07:30.962466       1 namespace_controller.go:171] Namespace has been deleted corc-tutorial
I0905 13:07:30.962500       1 namespace_controller.go:172] 

---> Our log here!!!

ubuntu@corc-tutorial-master:~$ 
```

Now you are able to make and use changes in your kubernetes cluster controller manager. :smile: 
