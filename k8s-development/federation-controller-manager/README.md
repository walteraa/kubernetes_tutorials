# Testing a code change in the Federation Controller Manager(FCM)

In this tutorial, I will show how to change FCM code and test it in a live cluster. For this tutorial, I will only add a Log entry in the controller and make this log appear in the pod logs.


## Creating a branch to add the changes

Once you have finished the tutorial [Setting up the kubernetes development environment](..), you just need create a branch in your repository

`$ cd $working_dir/kubernetes`
`$ git checkout -b adding-log-entry`


Now you can do changes safely.


