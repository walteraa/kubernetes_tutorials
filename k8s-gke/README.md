# Setting up google cloud sdk

*_Notice: This tutorial was developed over `Ubuntu 16.04`, maybe you'll need change this step depending on your OS_*

To setup your gke cluster, you need to get the Google cloud SDK. First of all, you should download the newest version of Google SDK(which was `165.0.0` when this tutorial was made) using the command below.

`wget https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-165.0.0-linux-x86_64.tar.gz`

Then you need extract the content and execute the installation script.

`tar -xvf google-cloud-sdk-165.0.0-linux-x86_64.tar.gz`

`./install.sh`

Make sure that the `gcloud` command is able.

```
which gcloud
${PATH_OF_INSTALLATION}/google-cloud-sdk/bin/gcloud
```

Now you are able to use `gcloud` command.


# Creating your google cloud projet

Once you would logged in your google cloud account, go to [project creation section](https://console.cloud.google.com/projectcreate) in your console using browser, and create your project.
When you put the project name in the textfield, an ID will appear bellow showing a text similar to `Your project ID will be ${PROJECT_ID}`.
This `${PROJECT_ID}` is important for the next step, then save this information.

# Authenticating and setting up your google cloud environment

To authenticate in your google cloud account, you should use the command 

`gcloud auth login`

Which will show an authentication URL, then you need open this URL in your browser and sign in your google account. The command will lock waiting for your sign in action and after you login successfully an instruction for setup your project ID will Appear. Now you need use the command to define the project that you will be using 

`gcloud config set project ${PROJECT_ID}`


