# Testing a change in kubectl code in a cluster

For this tutorial, I am assuming that you completed the [Setting up the kubernetes development environment](..) and one of [Cluster setup](../k8s-usage#other-tutorials) tutorials.

**Notice: This tutorial was made considering the kubernetes V1.7, in future versions it can be deprecated and useless.**

## Changing the code

I'll be using as example, the kubectl valid resource types information, that can be showed runing `kubectl get --help`. The file that we need change is named as `cmd.go` and can be found in `$working_dir/kubernetes/pkg/kubectl/cmd`.

To make some additional information appear when running `kubectl get --help` you should change the contant of variable `validResource`, which for now has the content below

```go
validResources = `Valid resource types include:

    * all
    * certificatesigningrequests (aka 'csr')
    * clusterrolebindings
    * clusterroles
    * clusters (valid only for federation apiservers)
    * componentstatuses (aka 'cs')
    * configmaps (aka 'cm')
    * controllerrevisions
    * cronjobs
    * customresourcedefinition (aka 'crd')
    * daemonsets (aka 'ds')
    * deployments (aka 'deploy')
    * endpoints (aka 'ep')
    * events (aka 'ev')
    * horizontalpodautoscalers (aka 'hpa')
    * ingresses (aka 'ing')
    * jobs
    * limitranges (aka 'limits')
    * namespaces (aka 'ns')
    * networkpolicies (aka 'netpol')
    * nodes (aka 'no')
    * persistentvolumeclaims (aka 'pvc')
    * persistentvolumes (aka 'pv')
    * poddisruptionbudgets (aka 'pdb')
    * podpreset
    * pods (aka 'po')
    * podsecuritypolicies (aka 'psp')
    * podtemplates
    * replicasets (aka 'rs')
    * replicationcontrollers (aka 'rc')
    * resourcequotas (aka 'quota')
    * rolebindings
    * roles
    * secrets
    * serviceaccounts (aka 'sa')
    * services (aka 'svc')
    * statefulsets
    * storageclasses
    `

```

Until the moment that I have developed this tutorial, there isn't a "aka" description for the resource `storageclasses`, then we'll put it beside `storageclass` and the new content should be like that

```go

validResources = `Valid resource types include:

    * all
    * certificatesigningrequests (aka 'csr')
    * clusterrolebindings
    * clusterroles
    * clusters (valid only for federation apiservers)
    * componentstatuses (aka 'cs')
    * configmaps (aka 'cm')
    * controllerrevisions
    * cronjobs
    * customresourcedefinition (aka 'crd')
    * daemonsets (aka 'ds')
    * deployments (aka 'deploy')
    * endpoints (aka 'ep')
    * events (aka 'ev')
    * horizontalpodautoscalers (aka 'hpa')
    * ingresses (aka 'ing')
    * jobs
    * limitranges (aka 'limits')
    * namespaces (aka 'ns')
    * networkpolicies (aka 'netpol')
    * nodes (aka 'no')
    * persistentvolumeclaims (aka 'pvc')
    * persistentvolumes (aka 'pv')
    * poddisruptionbudgets (aka 'pdb')
    * podpreset
    * pods (aka 'po')
    * podsecuritypolicies (aka 'psp')
    * podtemplates
    * replicasets (aka 'rs')
    * replicationcontrollers (aka 'rc')
    * resourcequotas (aka 'quota')
    * rolebindings
    * roles
    * secrets
    * serviceaccounts (aka 'sa')
    * services (aka 'svc')
    * statefulsets
    * storageclasses (aka 'sc')
    `
```

Ok, now we have a new kubectl version code.


## Building and testing the new kubectl binary version

To build the new version, you just need exec `make kubectl` command from `$working_dir/kubernetes` and wait the building process get finished. It will generate(if it doesnt't exists) the binary path, which you can find in `$working_dir/kubernetes/_output/bin` and the binary will be there.

Then, when you run `./kubectl get --help` from the binary path, you will see your changes there.


## Send your changes to upstream

To send your changes to upstream repository, you should follow the [Kubernetes Development guide](https://github.com/kubernetes/community/blob/master/contributors/devel/development.md).


