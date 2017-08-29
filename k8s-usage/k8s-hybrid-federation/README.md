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
=======

After the command above is completed successfully, this zone will appear in the [Google Cloud DNS panel](https://console.cloud.google.com/net-services/dns/zones). 

Once my federation is a hybrid federation, my DNS name(`federation.walteralves.me`) should be public, then I have been used the Name Servers generated from the Google Cloud DNS and configure it in the panel of the domain name registrar(which in my case was [GoDaddy](https://godaddy.com)) and put the Name Servers provided by Google DNS in the panel which in my case is

```
NS	federation	ns-cloud-d1.googledomains.com	1 hour
NS	federation	ns-cloud-d2.googledomains.com	1 hour
NS	federation	ns-cloud-d3.googledomains.com	1 hour
NS	federation	ns-cloud-d4.googledomains.com	1 hour
```

Once created and configured, you can use this DNS zone in your federation.

## Creating the federation control plane

The federation Control Plane(FCP) is the component which watches all underlying clusters to ensure that the current state of the resources created from the Federation Control Plane are as expected. The FCP should be configured in a cluster, in this tutorial I chose the `gke-central` as `host-cluster-context`(which is the cluster that has the FCP).

To create the FCP, we use the `kubefed` using the `gke-central` context in kubernetes configuration as showed in [setting up a federation](#setting-up-a-federation), the dns zone created before and `google-clouddns` as DNS provider

```
$ kubefed init federation --host-cluster-context=gke-central \
              --dns-zone-name=federation.walteralves.me. \
              --dns-provider=google-clouddns
```

Then the FCP will be created after this command is completed successfully and you will be able to see the federation context from kubectl

```
kubectl config get-contexts
CURRENT   NAME          CLUSTER                                       AUTHINFO                                      NAMESPACE
          on-prem       on-prem                                       on-prem                                       
          az-europe     az-europe-az-west-europe-4fecb0mgmt           az-europe-az-west-europe-4fecb0mgmt-admin     
          federation    federation                                    federation                                    
*         gke-central   gke_corc-tutorial_us-central1-a_gke-central   gke_corc-tutorial_us-central1-a_gke-central   
```

## Joining the clusters at federation

**_Notice: You should change your context to `federation` before execute the kubefed as I will execute below, but if you don't want change your context, you can use the param `--context=federation` to perform the 'kubefed` commands._**

If you want that the cluster make part of the federation, you need to use the `kubefed join` command. In production, the cluster that have the FCP, normally don't make part of the federation as a "worker cluster", but in our case, we will join the `gke-central` as a federated cluster

`kubefed join gke-central --host-cluster-context=gke-central`

The same for `on-prem` cluster

`kubefed join on-prem --host-cluster-context=gke-central`

In the case of `az-europe-cluster`, we need to do something before join it in the federation. The Azure clusters, normally comes with the RBAC(Role-Based Access Control) disabled, we need enable it first, then we will be able to join the Azure cluster in the federation.

**_Notice: I'lll not explain in details what is RBAC, but is important to understand what it is._**

First you need to access the azure master node using the #{HOST_IP}, which you can see in the Azure portal(dashboard)

```
$ ssh azureuser@#{HOST_IP}
azureuser@k8s-master-EF94B666-0:~$ sudo su
root@k8s-master-EF94B666-0:/home/azureuser# vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

You should open the `/etc/kubernetes/manifests/kube-apiserver.yaml` as root to add the flag `uthorization-mode=RBAC"` in the section `command` of `api-server`
```
- name: "kube-apiserver"
      image: "gcrio.azureedge.net/google_containers/hyperkube-amd64:v1.6.6"
      command:
        - "/hyperkube"
        - "apiserver"
        - "--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota"
        - "--address=0.0.0.0"
        - "--allow-privileged"
        - "--insecure-port=8080"
        - "--secure-port=443"
        - "--cloud-provider=azure"
        - "--cloud-config=/etc/kubernetes/azure.json"
        - "--service-cluster-ip-range=10.0.0.0/16"
        - "--etcd-servers=http://127.0.0.1:2379"
        - "--etcd-quorum-read=true"
        - "--advertise-address=10.240.255.15"
        - "--tls-cert-file=/etc/kubernetes/certs/apiserver.crt"
        - "--tls-private-key-file=/etc/kubernetes/certs/apiserver.key"
        - "--client-ca-file=/etc/kubernetes/certs/ca.crt"
        - "--service-account-key-file=/etc/kubernetes/certs/apiserver.key"
        - "--storage-backend=etcd2"
        - "--v=4"
        - "--authorization-mode=RBAC"   <----------
```

After that you should restart the kubelet service using `systemctl restart kubelet.service`, then the RBAC mode will be enable. But you will not be able to use the kubectl because the roles, so you need create a permissive-binding(which isn't recommended to used in production)

```
root@k8s-master-EF94B666-0:/home/azureuser# kubectl create clusterrolebinding permissive-binding \
                                            --clusterrole=cluster-admin \ 
                                            --user=admin \
                                            --user=kubelet \
                                            --user=kubeconfig \
                                            --user=client \
                                            --group=system:serviceaccounts
```

Now you are able to use the `kubectl` over the `az-europe` then you can join this cluster in the federation

`kubefed join az-europe-cluster --host-cluster-context=gke-central`


To check if everything is ok, you can list the clusters in the federation

```
kubectl get clusters
NAME                STATUS    AGE
az-europe-cluster   Ready     1m
gke-central         Ready     11m
on-prem             Ready     10m
```


Now you are able to create federated resources in your federation!! :grin:
