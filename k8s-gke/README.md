# Setting up google cloud sdk
*_Notice¹: I'm considering that you complete the tutorial "Setting up kubernetes" before start this tutorial._*

*_Notice²: This tutorial was developed over `Ubuntu 16.04`, maybe you'll need change this step depending on your OS_*

To setup your gke cluster, you need to get the Google cloud SDK. First of all, you should download the newest version of Google SDK(which was `165.0.0` when this tutorial was made) using the command below.

`wget https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-165.0.0-linux-x86_64.tar.gz`

Then you need extract the content and execute the installation script.

`tar -xvf google-cloud-sdk-165.0.0-linux-x86_64.tar.gz`

`./install.sh`

Make sure that the `gcloud` command is able.

```
$ which gcloud
#{PATH_OF_INSTALLATION}/google-cloud-sdk/bin/gcloud
```

Now you are able to use `gcloud` command.


# Creating your google cloud projet

Once you would logged in your google cloud account, go to [project creation section](https://console.cloud.google.com/projectcreate) in your console using browser, and create your project.
When you put the project name in the textfield, an ID will appear bellow showing a text similar to `Your project ID will be #{PROJECT_ID}`.
This `#{PROJECT_ID}` is important for the next step, then save this information. For this tutorial, I am using a project named `corc-tutorial`. 

# Authenticating and setting up your google cloud environment

To authenticate in your google cloud account, you should use the command 

`$ gcloud auth login`

Which will show an authentication URL, then you need open this URL in your browser and sign in your google account. The command will lock waiting for your sign in action and after you login successfully an instruction for setup your project ID will Appear. Now you need use the command to define the project that you will be using 

`gcloud config set project corc-tutorial`

# Creating GKE cluster

*_Notice: before starting this step, you will need activate the `Google Container Engine API` in the [GKE painel](https://console.cloud.google.com/apis/api/container.googleapis.com/overview). For this GKE cluster, I will use the `us-central1-a` as zone and a cluster containing 3 nodes._*

Now we will create our GKE cluster in `us-central1-a` zone using the command

```
$ gcloud container cluster create gke-central --zone=us-central1-a \
                                     --num-nodes=3 \
                                     --scopes "cloud-platform,storage-ro,service-control,service-management,https://www.googleapis.com/auth/ndev.clouddns.readwrite"
```

This command means that we are creating a cluster named `gke-central` at `us-central1-a` zone containing *3 nodes* and using the scopes listed in the parameter `--scopes`.

After completing the cluster setup, a cluster summary will appear.

# Connecting the GKE cluster with your kubernetes client(kubectl)

## Making more configuration
Before connect your GKE cluster in the kubernetes client, you need to make some configurations in the `gcloud` configuration referring to the way that the kubernetes connect to Google. Then, you need make `kubectl` uses cluster client cert or basic authentications, to do that you just need to execute

`$ gcloud config set container/use_client_certificate True`

or 

`$ export CLOUDSDK_CONTAINER_USE_CLIENT_CERTIFICATE=True`

(Or both to make sure that it is working)

## Now really connect kubectl to GKE cluster

The command used below can be obtained from the [Google Container Engine Panel](https://console.cloud.google.com/kubernetes/list) clicking in the *Connect* button

`$ gcloud container clusters get-credentials gke-central --zone us-central1-a --project corc-tutorial`

If the command is executed successfully, a confirmation about kubeconfig entry creation for `gke-central` will appear.

# Making sure that the configurations was created successfully

To make sure that everything until here works fine, we can run two commands

- Verifying if the cluster was added in kubernetes configuration

```
$ kubectl config get-contexts
CURRENT   NAME          CLUSTER                                       AUTHINFO                                      NAMESPACE
*         gke_corc-tutorial_us-central1-a_gke-central   gke_corc-tutorial_us-central1-a_gke-central   gke_corc-tutorial_us-central1-a_gke-central   
```

- Verifying if you are able to list resources from cluster

```
kubectl get pods --all-namespaces
NAMESPACE           NAME                                                    READY     STATUS    RESTARTS   AGE
kube-system         fluentd-gcp-v2.0-6vnpd                                  2/2       Running   0          1h
kube-system         fluentd-gcp-v2.0-hgjhw                                  2/2       Running   0          1h
kube-system         heapster-v1.3.0-4004699650-x156r                        2/2       Running   0          1h
kube-system         kube-dns-3664836949-ffg3c                               3/3       Running   0          1h
kube-system         kube-dns-3664836949-g24l8                               3/3       Running   0          1h
kube-system         kube-dns-autoscaler-2667913178-8mjvn                    1/1       Running   0          1h
kube-system         kube-proxy-gke-gke-central-default-pool-a83bdeac-d82l   1/1       Running   0          1h
kube-system         kube-proxy-gke-gke-central-default-pool-a83bdeac-jjb7   1/1       Running   0          1h
kube-system         kubernetes-dashboard-2917854236-qww30                   1/1       Running   0          1h
kube-system         l7-default-backend-1044750973-qpzn5                     1/1       Running   0          1h
```

# Making your life easier

Probably you realized that the name `gke_corc-tutorial_us-central1-a_gke-central` is a kinda large, somethimes you will need to use this name to get or create some resource from a specific context, but for our happiness, we can use the command below to make the context name smaller

`$ kubectl config rename-context gke_corc-tutorial_us-central1-a_gke-central gke-central`

Then you can check if the context name was changed

```
$ kubectl config get-contexts
CURRENT   NAME          CLUSTER                                       AUTHINFO                                      NAMESPACE
*         gke-central   gke_corc-tutorial_us-central1-a_gke-central   gke_corc-tutorial_us-central1-a_gke-central   
```


# Creating resources

You can read this tutorial about [Operating a kubernetes cluster](#)(TODO: create tutorial). 
