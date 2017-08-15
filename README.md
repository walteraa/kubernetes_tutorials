# What is kubernetes

Kubernetes is a open-source platform used to automate deployment, scale and operate applications running in containers. :whale:

# Kubernetes client(kubectl)

The `kubectl` is a command line interface to operate kubernetes clusters. This tool make possible creates kubernetes resources(e.g.: Pods, Deployments, Services, DaemonSets, etc...) in a kubernetes cluster using commands operations, flags, and yaml files.
For more informations about `kubectl` you can read the official [documentation/user guide](https://kubernetes.io/docs/user-guide/kubectl-overview/).

# Installing kubectl

I'll show two ways to install the kubernetes cli, the first one is using a package manager and the other one getting the binary from official repository.

## Installing from package manager

You can install the `kubectl` using the package manager of your OS. In this case we are using `apt` as package manager.

- First of all you need install `apt-transport-https` which enables the usage of `'deb https://package distro main'` lines at `/etc/apt/source.list`

`$ apt-get update && apt-get install -y apt-transport-https`

- After that you should download the key used by apt to authenticate the package

`$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -`

- Then you need to add the official kubernetes repository in `/etc/apt/source.list`

```
$ cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

- And finally update the packages index and install the package

```
$ sudo apt-get update
$ apt-get install -y kubectl
```

## Standalone installation

For a standalone installation, you just need download the binary file, give right permission and move it to bin path.

- Downloading the binary from the official repository

`$ wget  https://storage.googleapis.com/kubernetes-release/release/(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl`

- Give the permissions to the binary and move it to bin path

```
$ chmod +x kubectl
$ sudo mv kubectl /usr/local/bin/kubectl
```

## Checking if the kubectl was installed properly

To check if the `kubectl` was installed, you can perform a command using kubectl(e.g.: help, version), for example

```
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"7", GitVersion:"v1.7.3", GitCommit:"2c2fe6e8278a5db2d15a013987b53968c743f2a1", GitTreeState:"clean", BuildDate:"2017-08-03T07:00:21Z", GoVersion:"go1.8.3", Compiler:"gc", Platform:"linux/amd64"}
```

# Kubernetes Federation "administrator"(kubefed)

The `kubefed` is a admin tool to coltrol a Kubernetes cluster federation. It is used to initialize/join/unjoin a federation in kubernetes. With a federation you can deploy, scale and/or operate applications cross clusters in the same federation.

## Standalone installation


For a standalone installation, you just need download the binary file, give right permission and move it to bin path.

- Downloading the binary from the official repository

`$ wget  https://storage.googleapis.com/kubernetes-release/release/(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubefed`

- Give the permissions to the binary and move it to bin path

```
$ chmod +x kubefed
$ sudo mv kubectl /usr/local/bin/kubefed
```

## Checking if the kubectl was installed properly

To check if the `kubefed` was installed, you can perform a command using kubefed(e.g.: help, version), for example

```
$ kubefed version
Client Veddrsion: version.Info{Major:"1", Minor:"7", GitVersion:"v1.7.3", GitCommit:"2c2fe6e8278a5db2d15a013987b53968c743f2a1", GitTreeState:"clean", BuildDate:"2017-08-03T07:00:21Z", GoVersion:"go1.8.3", Compiler:"gc", Platform:"linux/amd64"}
```


# Other tutorials
- [Setting up a Google cloud kubernetes cluster](k8s-gke)
- [Setting up an Azure kubernetes cluster](#)
- [Setting up an on-premises kubernetes cluster](k8s-on-prem)
- [Setting up a hybrid kubernetes federation](#)
- [Operating a kubernetes cluster](#)
- [Operating a kubernetes federation](#)

