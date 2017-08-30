# Setting up the kubernetes development environment

This tutorial will consider that you are using Linux as OS, github as version control system and bash as shell.

## Installing go 

For this tutorial, I am using GO version `1.8.3` and I have used `$HOME/.programs/` path as the place where I will install go files 

First you need download golang binaries

```bash
$ wget https://storage.googleapis.com/golang/go1.8.3.linux-amd64.tar.gz
```

After finish the download, you should extract the go file to the path which you chose

```bash
$ tar -C $HOME/.programs/go -xzf go1.8.3.linux-amd64.tar.gz
```

Now you have a go version installed in your computer, but you'll need set the environment variables

```bash
$ export GOROOT=$HOME/.programs/go
$ export export PATH=$PATH:$GOROOT/bin
```

After that you will be able to execute go command from your shell

```bash
$ go version
go version go1.8.3 linux/amd64
```


Ok, now you will setup the go workspace environment variable, which in my case was created in `$HOME` path

```bash
$ mkdir go go/bin go/pkg go /src 
```

And set the variable `$GOPATH` 

```bash
$ export GOPATH=$HOME/go
```
**Notice: You will need add the export lines in you .bashrc to use it permanently.**

Now you have your go environment configured

```bash
$ go env GOPATH GOROOT
$HOME/go
$HOME/.programs/go
```

## Configuring the kubernetes variables

To use the kubernetes repository in your development workspace, you will need two variables: `$working_dir` and `$user`. The first one is the path where your kubernetes repository will be cloned, the second one is your **github user** which is **walteraa** in my case.

```bash
$ export working_dir=$GOPATH/src/k8s.io
$ export user=walteraa
```
**Notice: this variables should be added in the .bashrc file as well**

Oh, don't forget creating the workin_dir path

```bash
$ mkdir -p $working_dir
```


## Creating(forking) your kubernetes repository

1. Visit [https://github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)
2. Click Fork button (top right) to establish a cloud-based fork.

## Cloning and branching your kubernetes repository

First of all, let's clone the repository from local computer

`$ git clone https://github.com/$user/kubernetes.git $working_dir`

And add the upstream remote

```bash
$ cd $working_dir/kubernetes
$ git remote add upstream https://github.com/kubernetes/kubernetes.git
```

Making sure that you'll never push to upstream 

`$ git remote set-url --push upstream no_push`

Update your local from remote upstream

```bash
$ git fetch upstream
$ git checkout master
$ git rebase upstream/master
```

Now you can create your feature's branches and develop.


## Testing the code

To test code, you'll need have the etcd installed locally

`$ ./hack/install-etcd.sh`

Running pre-submissions verification

`$ make verify`

Running update scripts

`$ make update`

Running all unity tests

`$ make test`

Running integratin tests

`$ make test-integration`

Running end-to-end tests

`$ make test-e2e`


**Notice: All commands above are executed inside `$working_dir/kubernetes` path.**

## Building the code

Building the project for the computer OS platforms

`$ make `

Building for all platforms

`$ make cross`

**Notice: All commands above are executed inside `$working_dir/kubernetes` path.**

All binaries will be generated inside the `$workind_dir/kubernetes/_output/bin` path, which is a link for `$workind_dir/kubernetes/_output/bin/local/${so}/${archtecture}`, for example, in my case was generated in `$workind_dir/kubernetes/_output/bin/local/linux/amd64`, because I am running from a Ubuntu amd64.
