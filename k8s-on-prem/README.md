# Kubernetes on-premises cluster

A kubernetes on-premises cluster is a cluster which doesn't have a cloud provider to operate the cluster. In a nutshell, we can consider as an on-premises cluster, a cluster that isn't controller by a cloud manager(e. g.: Google cloud, AWS, Azure, etc...).

In this tutorial we'll use a cluster hosted in an OpenStack environment but don't controlled for the OpenStack Cloud provider.

# Cluster nodes

This cluster is composed for 1 master and 2 nodes having the configuration below 

```
NAME                    OS              IP ADDRESS       vCPU    DISK    RAM
corc-tutorial-master    ubuntu-16.04	10.11.4.189       2       40     2GB
corc-tutorial-node1     ubuntu-16.04	10.11.4.113       1       20     1GB
corc-tutorial-node2     ubuntu-16.04	10.11.4.199       1       20     1GB

```


