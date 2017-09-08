# Testing a code change Kubernetes

> Important: Here we use the Kube Controller Manager as example, but this also works for `api-server`, `scheduler` and any other component that is deployed as a `pod` in Kubernetes.

In this tutorial, I will show how to change Kuberentes code and test it in a live cluster. For this tutorial, we use the Kubernetes Controller Manager (KCM) as example. I will only add a Log entry in the Namespace deletion process and make this log appear in the pod logs.

## Creating a branch to add the changes

Once you have finished the tutorial [Setting up the kubernetes development environment](..), you just need create a branch in your repository

`$ cd $working_dir/kubernetes`
`$ git checkout -b adding-log-entry`

Now you can do changes safely.

## Changing the code

Now, I will change the file `$working_dir/kubernetes/pkg/controller/namespaces_controller.go` and add a log in the method `syncNamespaceFromKey`, which is the method that ensures namespaces has been deleted successfully.

The log entry is

`glog.Info("\n\n---> Our log here!!!\n\n")`


## Building the new version

Now you can build the `kube-controller-manager`. To do so, you only need to run the command `build/run.sh make kube-controller-manager` from the `$working_dir/kubernetes` root and wait while the code is compiled.

After the compile proccess has been finished, you can find the new binary version in the path `$working_dir/_output/bin`.

## Building and pushing the new kube-controller-manager docker image

### Configuring the repository environment

For this step you will need a docker image which will run the `kube-controller-manager` in your cluster, to manage all resources. The next subtopics go through this.

### Building the image

To build the image, you will need a simple Dockerfile to define your image over a `ubuntu-slim` source

```bash
FROM gcr.io/google_containers/ubuntu-slim:0.9

ADD kube-controller-manager /usr/bin
```

You should create this Dockerfile in the same location where the `kube-controller-manager` binary is. Then, execute the `docker build` command inside this directory, using the repository name and tag where you will set in the kube-controller manifest. For example:

`$ docker build -t walteraa/kube-controller-manager-amd64:v0.0.1 .`

Now you have an image prepared to be used in your cluster.

### Pushing the image

Once you have created the image, you just need login into your docker account and push your image.

```bash
 docker login`
# (Provide your account credentials)
$ docker push walteraa/kube-controller-manager-amd64:v0.0.1
```

Now you can use your `kube-controller-manager` version.

## Configuring your cluster to use your kube-controller-manager version

First of all, you need ssh to the master of the cluster to make the changes below.

`$ ssh user@master-ip`

Once you're in, you need to change the manifest that declares how the `kube-controller-manager` should be executed, making sure it points to your image. Open for edtion the following file:

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

`systemctl restart kubelet.service`

Kill the current `kube-controller-manager` pod

```bash
# Look for it by listing pods in kube-system namespace
$ kubectl get pods -n kube-system
NAMESPACE     NAME                                           READY     STATUS    RESTARTS   AGE
...
kube-system   kube-controller-manager-corc-tutorial-master   1/1       Running   2          16h
...

# Delete it
$ kubectl delete pod -n kube-system  kube-controller-manager-corc-tutorial-master
pod "kube-controller-manager-corc-tutorial-master" deleted
```

And finally ensure that the new version is being using the command below to check the image spec:

`$ kubectl get pod -n kube-system   kube-controller-manager-corc-tutorial-master -o yaml`

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

If you see the name of your image, it means that your cluster is using your `kube-controller-manager` version.

## Checking our log execution in the controller manager

Remember that Log that you have added in the code? Now we can check if it is being executed.

First add a new namespace which you can delete right after.

```bash
# Add a namespace
$ kubectl create ns corc-tutorial
namespace "corc-tutorial" created
# And delete it
$ kubectl delete ns corc-tutorial
namespace "corc-tutorial" deleted
```

And check if the log appears in your pod log

```bash
kubectl logs -n kube-system kube-controller-manager-corc-tutorial-master
(...)
I0905 13:07:30.962466       1 namespace_controller.go:171] Namespace has been deleted corc-tutorial
I0905 13:07:30.962500       1 namespace_controller.go:172] 

---> Our log here!!!

(...)
```

Now you are able to make and use changes in your kubernetes cluster controller manager. :smile:
