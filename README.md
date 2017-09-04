# Kubernetes tutorials

This tutorial is splited in two purpuses: Usage and Development. The first one is focused in using and creating kibernetes clusters and federation in On-prem, GKE and Azure environments. The second one is focused in kubernetes development and test your own code in a live (on-premises)cluster.

## Usage tutorials

In this tutorial you can find instructions about how to download the tools and setup clusters at GKE, Azure and on-prem environments.

* [Clients installations](k8s-usage)
* Setting-up clusters
  * [Azure cluster setup](k8s-usage/k8s-azure)
  * [GKE cluster setup](k8s-usage/k8s-gke)
  * [On-premises cluster setup](k8s-usage/k8s-on-prem)
* [Hybrid federation setup](k8s-usage/k8s-hybrid-federation)


## Development tutorials

In this section, you'll find instructions about how change kubernetes code and test it in a cluster. For this section, I will use `kubectl` and `federation-controller-manager` as examples.

* [Setting up the development environment](k8s-development)
* Building and testing kubernetes clients
  * [kubectl (WIP)](k8s-development/kubectl)
  * [kubefed (TODO)](#)
* Building and testing kubernetes component in a live cluster
  * [kube-controller-manager (WIP)](k8s-development/kube-controller-manager)
  * [federation-controller-manager (WIP)]  
  * [federation-apiserver (TODO)](#)
  * [kube-apiserver (TODO)](#)

