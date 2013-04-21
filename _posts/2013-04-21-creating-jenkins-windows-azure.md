---
layout: post
title: Creating a Jenkins continuous integration on Windows Azure
tags : [jenkins, azure, iaas, vm]
---
{% include JB/setup %}

You can create a Jenkins VM very quickly with this approach.

* Create a storage account for your VMs if you don't have one.

* Create an image from VM Depot (choose Jenkins).

![](http://puu.sh/2DWSv.png)

* Choose the storage account where the image will be copied (the one you created in the first step).

![](http://puu.sh/2DWTs.png)

> This process will take approx 20 minutes (it will copy the base vhd image template to your account which could be a couple of GB)

* Once it finished copying, create the VM by clicking on NEW -> Virtual Machine -> From Gallery and choose the Jenkins image

![](http://puu.sh/2DV2Y.png)

* Give it a name and choose an Extra Small instance. That's enough to start with. You can scale up easily later, with one click. This should take between 5 to 7 minutes. 

![](http://puu.sh/2DX9H.png)

> If you want to ssh into the machine to install stuff, you can also upload your public key (you can create one with: `openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -subj "/CN=jenkins" -keyout jenkins.key  -out jenkins.cer`)

![](http://puu.sh/2DVbH.png)

* Once it finished, open endpoint 80 (takes 2 minutes max)

![](http://puu.sh/2DVjb.png)

* Now, you can browse the new Jenkins (http://yourjenkins.cloudapp.net)

![](http://puu.sh/2DVuK.png)

* Enter the app and use the following credentials (user: `user` and password: `bitnami`).

> More users can be added later or you can configure the GitHub OAuth plugin to authenticate with github accounts

## Create a job, configure GitHub and Windows Azure Web Sites repos

* Now let's install the GitHub plugin. Go to **Manage Jenkins** -> **Manage Plugins**. Install "GitHub Plugin" (if plugins are not there, keep refreshing and they should appear soon)

![](http://puu.sh/2DVFx.png)

* Click on "Install without restart"

![](http://puu.sh/2DVNo.png)

* Click on **Configure System**. Set Github user.name y user.email to something arbitrary like "build"

* Next, create a new job and choose "Build a free-style software project"

![](http://puu.sh/2DVRj.png)

* Click on Git radio button and add your GitHub repository.
IMPORTANT: use the git read only endpoint from GitHub which has this form `git://github.com/username/repo.git`. This is a quick way of doing it, the other option is to copy your SSH keys to the VM, but since you will always READ from GitHub, this is ok.

![](http://puu.sh/2DVW8.png)

* Then, add the azure repository using a URL like this `http://youruser:yourpassword@websitename.scm.azurewebsites.net/websitename.git`. Notice, that in order to get that endpoint you would need to have enable the "Local Git Repository" option in Windows Azure Web Sites. You can find the URL of the repo in the Azure Web Site dashboard.

![](http://puu.sh/2DW27.png)

* Click on **Advanced** and give it a name like "azure"

## Create a post build action to push to Windows Azure Web Sites

* Add a **Post-build** action of type **Git publisher**. Click on **Push Only If Build Succeeds**. Then, click on **Add branch** to configure the branch and target we want to deploy. Enter the following
    * branch: master
    * target: azure

![](http://puu.sh/2DW86.png)

## Configure the GitHub web hook to automatically build on push

* Make sure to check Build when a change is pushed to GitHub on the job

![](http://puu.sh/2DVSd.png)

* Go to your GitHub repo on github.com and click on Settings, then **Service Hooks** and look for "**Jenkins (Github Plugin)**". Use the following URL `http://yourjenkins.cloudapp.net/jenkins/github-webhook/` (replacing the DNS).

![](http://puu.sh/2DWaY.png)

* Push a change to github and you should see it will start building

![](http://puu.sh/2DWdp.png)

* If you go to the Azure Web Site you will also see that it pushed a new build

![](http://puu.sh/2DWhn.png)

## Customize the environment

Usually you will want to run tests and do a lot of things. For instance, in [Auth0](http://auth0.com) our job will run npm install, then run our unit and integration tests with mocha. If everything runs fine, it will push to the web site.

If you uploaded a public key you can SSH to the VM with:

    ssh -i jenkins.key -p AZURE_PORT user@yourjenkins.cloudapp.net

Where user is the user name you set when creating the VM on the Azure portal and AZURE_PORT is the one you will find in endpoints. Also, make sure to have the private key protected with `sudo chmod 600 jenkins.key`

You can install node.js like this

    sudo apt-get install python-software-properties python g++ make
    sudo add-apt-repository ppa:chris-lea/node.js
    sudo apt-get update
    sudo apt-get install nodejs

