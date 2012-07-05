---
layout: post
title: Creating a SharePoint 2010 virtual machine in Windows Azure
tags : [azure, Sharepoint 2010]
---
{% include JB/setup %}

During the last couple of days I've recorded a couple of [screencast for Auth10](http://auth10.com/howitworks) demonstrating how easy is to setup SharePoint 2010 to accept Google and ADFS identities. I decided to give a try to [Windows Azure Virtual Machines](https://www.windowsazure.com/en-us/home/features/virtual-machines/) and I have to say that I am very satisfied. I had a SharePoint vm running in less than 30 minutes.

In this article I will go through the steps needed to create the SharePoint VM and then will compare my experience running the same VM in AWS for this use case. 

###How to create a Virtual Machine running SharePoint 2010 in Windows Azure

I wished there was a prepopulated image syspreped with SharePoint installed, but that wasn't the case. So I had to start from a "Windows 2008 R2" image.

_Disclaimer: Virtual Machines is a "preview" feature in Windows Azure today and this tutorial is not intended to have a production VM. It's more about, having a SharePoint VM up and running quickly. If you want to create a farm and start from vhds created on Hyper-V on premises, I recommend this post from [Steve Peschka: Creating an Azure Persistent VM for an Isolated SharePoint Farm](http://blogs.technet.com/b/speschka/archive/2012/06/17/creating-an-azure-persistent-vm-for-an-isolated-sharepoint-farm.aspx)_

1. Create a new virtual machine by choosing the Windows Server 2008 R2 image. I chose the "June" release, the "Small" size and an arbitrary dns name like "sp-auth10.cloudapp.net".
**Important**: don't try to run SharePoint in an Extra Small instance because it's way too heavy for such configuration

![](http://puu.sh/FRxO)

2. The provisioning will take 5 to 10 minutes and will create the VM and a 30GB VHD with the OS.

3. When you are done, connect to the VM via remote desktop (using the "Connect" button)

4. Download SharePoint Enterprise trial from <http://www.microsoft.com/en-us/download/details.aspx?id=16631> and install it. The installation process is smooth if you choose all the defaults. It will also create a SQL Express instance in the VM, so no need to worry about it.

5. DONE. Enjoy SharePoint :). If you have the budget to leave the machine turned on go ahead (small instance 24x30 = $0.115 * 720 = 82USD / month + 30gb storage), otherwise read the next section.

###How to turn on and off Virtual Machines using scripts and save some $

As I said at the beginning, I needed the VMs to record some [screencasts for Auth10](http://auth10.com) and do some testing. So no need to have the VM running 24x7 and I didn't want to pay for that either. I though that shutting down the VM from the management portal would do the trick, but unfortunately that's not the case.

So I've learnt the trick from [Michael Washam blog](http://michaelwasham.com/2012/06/18/importing-and-exporting-virtual-machine-settings/). It basically goes like this:

**Export VM definition to a file**

    Export-AzureVM dns-name vm-name -Path c:\sharepointvm.xml

**Delete VM (but not the vhd)**

    Remove-AzureVM dns-name vm-name
    Remove-AzureDeployment -ServiceName dns-name -Slot Production -Force

**When you want to start it again, import VM definition from a file**

    Import-AzureVM -Path 'c:\sharepointvm.xml' | New-AzureVM -ServiceName dns-name -location "East US"

>Note: I tried to achieve the same using the Azure CLI (`azure vm create-from vm.json`) from a Mac but it does not work. Apparently there is something wrong on the json fields expected in create-from. I opened the issue in [Github](https://github.com/WindowsAzure/azure-sdk-for-node/issues/254).

###Pricing

Windows Azure is cheaper than AWS for this usecase. When I created VM in AWS from a SharePoint AMI six months ago, I was forced to use a Large instance (not sure if that's still the case today).

	Large VM in AWS = $0.46/hr
	Small VM in Windows Azure = $0.115/hr
	Storage in AWS = $0.10 / GB
	Storage in Windows Azure = $0.125

So let's say you use the VM couple of days per week 

**Compute cost per month**

	8 hs x 2 days x 4 weeks = 64 hs => 29.44 USD in AWS
	8 hs x 2 days x 4 weeks = 64 hs =>  7.36 USD in Windows Azure

**Storage cost per month**

	30GB = 0.10 * 30 = 3 USD in AWS
	30GB = 0.125 * 30 = 3.75 USD in Windows Azure

Storage transactions are depreciable in this case

###Performance

I haven't done any real measurement, all I can say is that Extra Small instance in Azure is a no-go. However, the small instance was good enough to do what I had to do. The Large VM on AWS was equally good. For the purpose of my tests it didn't make a difference.

###Turning it on/off

In terms of fleixibilty, AWS provides the ability to turn on and off the VM without having to "remove it" like we explained above. The time it takes to have the machine up and running after being turned off (or removed) is almost the same, with AWS being a little bit faster and easier (one-click vs scripts). It's in the 5 to 10 minutes range.

###SID

Another reason why I tried Windows Azure was because last week when I turned the AWS VM on, I realized there was some issue with the database and the NETWORK SERVICE user. I guess it was an issue with the SIDs changed. Not sure what caused that, but suddenly the VM was broken. Not sure if that will happen in Windows Azure, but that was annoying.

###DNS name

Unless you use an elastic IP in AWS, every time you turn on the machine you get a different DNS so there is no way to have a fixed DNS name mapped to the VM. On the other hand, in Windows Azure you get always the same DNS name so you can create a CNAME record that works consistently. You could loose the DNS name if someone else provision a service under that DNS while your machine was turned off... but if you choose an obscure name I don't see that happening (unless someone hates you :)

**UPDATE**: Actually, it is even better. You can use Remove-AzureDeployment to remove the VM from the Cloud Service while keeping the dns name reserved for you. Meaning you can reserve CNAMEs effectively while turning on and off the VM (something that would cost around 7 USD per month using elasti IPs if you use it couple of days a week). Thanks [Michael Wood](http://mvwood.com/) for pointing this out.

##Conclusion

Next time I need to build a VM running Windows to test something out, I will probably go with Windows Azure Virtual Machines for the reasons explained above. There are still some rough edges and it's in Preview state, but in general I am satisfied with the results.