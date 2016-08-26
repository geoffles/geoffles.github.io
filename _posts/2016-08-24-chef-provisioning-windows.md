---
layout:	post
title:	"Chef Provisioning for Windows"
date:	2016-08-24 09:43:00 +0200
categories:	Chef
tags:	[chef, windows, provisioning]
---

Chef provisioning allows you to use chef to allocate machines. This is particularly handy for cloud platforms or would like to script machine allocation for virtual machines.

This specific post uses Windows Provisioning for AWS (Amazon Web Services).

* TOC
{:toc}

# Prerequisites

-  An AWS Account
-  ChefDK
-  A Windows Machine

The windows machine is rumoured to be required since Chef uses WinRM to talk to the Windows box and alledgedly doesn't work so well on Linux (which is sad).

# Overview

The key parts are:

-  Reference AWS SDK
-  Define the machine options for provisioning
    -  Configure your AWS user script
-  Define a `machine` Chef resource

The AWS SDK allow chef to connect to AWS and launch an EC2 instance as well as acquire information about the instance. The machine options specify things like what image to use, the security group, etc. Finally, your machine resource kicks all of this off.

By defining a machine resource, Chef will internally keep a node cache of the resource.

# The `machine_options`

You going to need to set the following in `bootstrap_options`:

-  `key_name` -  The key against which the machine will be secured (AWS EC2 Key-Pair)       
-  `instance_type` -  The grade of the EC2 instance (eg: t2.micro)
-  `image_id` -  The AMI Id for the EC2 instance
-  `subnet_id` -  The ID of the subnet into which the EC2 instance will be allocated 
-  `security_group_ids` -  Array of the IDs for the security groups that the EC2 instance will belong to
-  `user_data` -  For Windows, this needs to do a couple of things:
    -  Enable WinRM
    -  Open WinRM on the firewall
    -  For AWS, you might need to add a hostname to your hosts file for your Chef server

You're also going to need to set the following:

-  `is_windows: true` - So that Chef knows it's windows 
-  `transport_address_location: :private_ip` - Mileage may vary, but you should rather set up infrastructure to administer the instance via private IP so that you're better protected against unauthorised access. Basically this means a VPN into the VPC.
-  `winrm_username: 'Administrator'` - Chef will decrypt the Admin password using the key-pair, so just specify the username.
-  `aws_tags:` - I reccommend setting tags like, which cook book allocated the machine, maybe things like cost-centre, etc. Your machine name can be set in the `machine` resource

# The actual code

Heres the good bit. The code below *should* allocate a machine once you set the Subnet and Security Group IDs: 

{% highlight ruby %}
require 'chef/provisioning/aws_driver'

with_driver 'aws' do
  the_machine_options = {
    bootstrap_options: {
      key_name:           'my-ec2-keypair',
      instance_type:      't2.micro',
      image_id:           'ami-2979185a', #Windows Server 2012r2
      subnet_id:          'subnet-xxxxx',
      security_group_ids: ['sg-xxxxx'],
      user_data: <<-END
<script>
winrm quickconfig -q & winrm set winrm/config/winrs @{MaxMemoryPerShellMB="300"} & winrm set winrm/config @{MaxTimeoutms="1800000"} & winrm set winrm/config/service @{AllowUnencrypted="true"} & winrm set winrm/config/service/auth @{Basic="true"}
</script>
<powershell>
netsh advfirewall firewall add rule name="WinRM in" protocol=TCP dir=in profile=any localport=5985 remoteip=any localip=any action=allow
Add-Content c:\\Windows\\System32\\drivers\\etc\\hosts "1.2.3.4	whereischef"
</powershell>
END
    },
    is_windows: true,
    transport_address_location: :private_ip,
    winrm_username: 'Administrator',
    aws_tags: {
      Project:     'Project Gemini',
      Environment: 'Dev',
      Network:     'Private'
    }
  }

  machine 'myEC2Instance' do
    machine_options the_machine_options
    admin true
    recipe 'mycookbook::myrecipe'
  end
end
{% endhighlight %}

For convenience, heres the user script on it's own:
{% highlight xml %}
<script>
winrm quickconfig -q & winrm set winrm/config/winrs @{MaxMemoryPerShellMB="300"} & winrm set winrm/config @{MaxTimeoutms="1800000"} & winrm set winrm/config/service @{AllowUnencrypted="true"} & winrm set winrm/config/service/auth @{Basic="true"}
</script>
<powershell>
netsh advfirewall firewall add rule name="WinRM in" protocol=TCP dir=in profile=any localport=5985 remoteip=any localip=any action=allow
Add-Content c:\\Windows\\System32\\drivers\\etc\\hosts "1.2.3.4	whereischef"
</powershell>
{% endhighlight %}

# Discussion

There are some things to talk about. 

Firstly, it's worth noting that Chef is not focusing on Provisioning at this stage and do not officially support it for enterprise use - it works, but sometimes there are a few small surprises (Woohoo!).

You can pretty much copy the user data script as is, and either update or remove the hosts file addition. Also of note, is that it includes a standard windows "shell" script and a powershell script. As mentioned, these will together enable and configure WinRM and open WinRM on the firewall so that Chef can remotely administer the machine. Unfortunately you're also limited by what is supported by Chef at this stage.

As mentioned, the password for the machine is decrypted via the AWS SDK - which means that at no point is the administrator password ever exposed which is really nice.

There are certain elements which are of course just my personal style - like keeping the machine options as a separate variable. You can also define a runlist instead of just a recipe for the machine - see [Chef Doc](https://docs.chef.io/resource_machine.html). 

# Running the recipe

In practice, you will have a *provisioning node* which is a machine that has chef client and "one time runs" provisioning recipes against itself.

For this to happen you will also need to set up your AWS Credentials. You can use the command `aws configure`, however I prefer just setting the environment variables manually since chef doesn't alway pick up the folder `.aws` in your home folder properly. The environment variables you must set are:

-  `AWS_SECRET_ACCESS_KEY` 
-  `AWS_ACCESS_KEY_ID`
-  `AWS_DEFAULT_REGION`

You can get these from your AWS account.

**Note:** The above details are the keys to your AWS castle and therefore your credit card! Do not keep those details in a publicly accessible place, version control, or anywhere not absolutely neccessary. ***Read up on the AWS guidelines for how to manage your access keys!***

So, once you're all set up, you then just simply:

{% highlight powershell %}
chef-client -o 'recipe[mycookbook::myrecipe]'
{% endhighlight %}

And voilà, magic happens.

# Chef-zero

You can of course also run this against `chef-zero` which is my preferred mechanism for development:

{% highlight powershell %}
chef-client -z -o 'recipe[mycookbook::myrecipe]'
{% endhighlight %}

Chef-zero of course runs by default on port 8889 which may not work so well for some firewalls, since the provisioned machine needs to dial back to your chef server in order to execute it's `chef-client` run and download cookbooks etc. In this case, you can tell chef which hostname and port to use! You will also need to advise your recipe of a resolvable hostname for your chef-server prior to defining your machine resource.

{% highlight ruby %}
if Chef::Config[:chef_server_url].include?('localhost')
  with_chef_server(
    format('http://%s:%s', '1.2.3.4', '80'), # your machine's ip or hostname and the port you're running chef-zero on
    client_name:          Chef::Config[:node_name],
    signing_key_filename: Chef::Config[:client_key]
  )
{% endhighlight %}

And then you adjust your `chef-client` run as follows:

{% highlight powershell %}
chef-client -z --chef-zero-host 1.2.3.4 --chef-zero-port 80 -o 'recipe[mycookbook::myrecipe]'
{% endhighlight %}

#  Conclusion

The above should provide you with a barebones solution for bootstrapping a windows machine on AWS.

From there you can, as Chef says, automate the impossible, or should I say, the improbable.