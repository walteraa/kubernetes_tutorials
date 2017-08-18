# Federation
Federation is the way to make easier the use of multiple clusters. With a federation, you can do operations only one and it will be executed in all federation underlying clusters.

# Setting up a federation

To setup a federation, you'll need at least two clusters and the `kubefed` tools. For continue this tutorial, I recommend you do at least two cluster setup tutorials and the kubefed installation tutorial, if you didn't, you can do it now
- [Kubefed isntallation](https://github.com/walteraa/kubernetes_tutorials/#kubernetes-federation-administratorkubefed)
- [Setting up a gke cluster](../k8s-gke)
- [Setting up an on-premses cluster](../k8s-on-prem)
- [Setting up an Azure cluster](../k8s-azure)

Once you have the requirements to setup your federation, you can start creating your federation. For this tutorial, I'll create a federation containing all clusters listed above.

```
$ kubectl config get-contexts
CURRENT   NAME                CLUSTER                                       AUTHINFO                                       NAMESPACE
          az-europe-cluster   az-europe-cluster-az-europe-4fecb0mgmt        az-europe-cluster-az-europe-4fecb0mgmt-admin
*         gke-central         gke_corc-tutorial_us-central1-a_gke-central   gke_corc-tutorial_us-central1-a_gke-central
          on-prem             on-prem                                       on-prem
```

## Creating the managed DNS zone

The DNS zone is used to do cross-cluster service discovery. For this tutorial, I'll use google cloud DNS, but you can use Route53(from Amazon) or CoreDNS(from Core OS) as well. For create the managed DNS, I will use the google cloud SDK, which I teach how to install in [this step of gke tutorial](../k8s-gke#setting-up-google-cloud-sdk) using `federation.walteralves.me` as  DNS name(I will explain it forward).


```
$ gcloud dns managed-zones create federation \
              --description "CORC tutorial about kubernetes federation" \
              --dns-name federation.walteralves.me
```

