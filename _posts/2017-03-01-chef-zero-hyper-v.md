---
layout:	post
title:	"Setting up a Chef-Zero Environment using Hyper-V"
categories:	Chef
date:	2017-03-01 18:14:32+0200
tags:	[chef, hyper-v]
---

***Abstract:** An offline chef development environment using Chef-Zero and Hyper-V*

* TOC
{:toc}

# Introduction

I've been playing around with setting up a Chef development environment on my PC that is a little more representative of Chef in a production environment. Whilst Test Kitchen is probably best for *testing* your cookbook, it doesn't work so well for things like demos; teaching people how chef actually works; and Test Kitchen just seems slow to cycle.

What I've done then it set up a minimal enironment on my Windows machine using Hyper-V and Debian. This is mostly useful for demonstrating things like `knife bootstrap` without relying on the internet.

# Hyper-V

The tasks here are as follows:

1.  Setup Hyper-V
1.  Download your OS (I used Debian netinst)
1.  Create an Internal Virtual Switch  for your Machine (I called mine DebianNAT)
1.  Configure NAT for internet access
1.  Create a VM

Hyper-V seems to work really nicely with the OS booting fast, and obviously being a native Windows feature.

I used *Debian netinst* and configured an internal adapter. 

## Configuring the NAT

You can connect the VM to the internet to download packages by setting up NAT on that adapter. From Powershell:

{% highlight powershell %}
new-netnat -name DebianNAT -InternalIPInterfaceAddressPrefix 169.254.0.0/16
{% endhighlight %}

Where `DebianNAT` is the network adaptor in Hyper-V and the `169.254.0.0/16` is the matching range for the adaptor.

So, you should now have a Hyper-V Debian VM, ready to install.

# Debian

To set up the Debian, you need to do the following:

1. Configure the networking and users (in the boot setup)
1. Install Open SSH Server
1. Install and configure sudo

I set my user as `chef-user`

# Networking

You'll either need to setup DHCP on your host machine, or simply use a static IP on the machine using the `DebianNAT` adaptor IP as the gateway. You can then your DNS to something lilke Google's `8.8.8.8`. I had some firewall issues, and disabled it while I installed the packages on the VM. I found that you also need to reset your DNS server after booting post-install.

If going static ip, be sure to chosoe an IP different to the virtual adapter's IP address but in the same range. Eg, if your adapter is `169.254.10.1`, then use something like `169.254.10.10`. In this case, Debian will have the IP address `169.254.10.10` and a Gateway IP of `169.254.10.1`.

# SSH and Sudo

Install with `apt-get install openssh-server sudo` (Don't forget you need to do this as root).

Next, use `visudo` to add chef-user to the `sudoers` list and not require a password. You'll need to add another entry for chef-user that looks something like this:

```
# User privilege specification

root    ALL=(ALL:ALL) ALL

chef-user ALL=(ALL) NOPASSWD: ALL
```

You should be able to SSH into your machine now using the Debian OS IP address. Eg, with `putty`: `putty chef-user@169.254.10.10`.

It also wouldn't hurt to setup a key-based login at this point.

# Chef

So now you have an environment. 

**Protip:** Now is a good time to set a checkpoint on the VM so you can easily roll-back.

Now that you're ready to do the Chef stuff, here's what we'll be doing:

1.  Setting up knife and Chef-Zero server
1.  Faking the installer script for knife bootstrap
1.  Bootstrapping the machine

## Chef-Zero

Chef-Zero is basically a light-weight chef server you can run on your local machine. 

In our casae, you start it with a simple: `chef-zero -H 169.254.10.1`, where the IP matches your DebianNAT adapter IP. This is so that the server can recieve calls from your VM.

Now that that's up, you can create a chef repo folder and configure your knife. Create a directory and call `chef generate repo`. Then in that directory create a directory called `.chef` and place the following content in `knife.rb`:

```
chef_repo = File.join(File.dirname(__FILE__), "..")

chef_server_url "http://169.254.10.1:8889"
node_name       "knifezero"
client_key      File.join(File.dirname(__FILE__), "knifezero.pem")
cookbook_path   "#{chef_repo}/cookbooks"
cache_type      "BasicFile"
cache_options   :path => "#{chef_repo}/checksums"
```

Once again, match IP addresses.

Still in `.chef`, now create your key using `ssh-keygen -f knifezero.pem`. Chef-Zero doesn't use the key, but `knife` needs a valid key to send in the request.

Back in your chef repo dir, you should be able test the connection with: `knife node list`. It won't list anything yet, but shouldn't error either.

## Faking the install

To start with, we're faking the install because we'd like to have a demo safe machine that mimics the install experience relatively closely.

To do this, you need to "cache" three things:

1.  Installer script
1.  Chef Metadata
1.  Chef Omnibus deb packages

The installer script is a hacked copy of this: `https://omnitruck.chef.io/install.sh`.

The script calculates which metadata file to download, which in my case was: `https://omnitruck-direct.chef.io/stable/chef/metadata?v=12.18.31&p=debian&pv=8.7&m=x86_64`. I then downloaded the deb package pointed to in the meta data file.

I then cached these in a static IIS site. I had to make the following policy changes in IIS to serve the items:

 -  Add MIME type `.deb:application/octet-stream`
 -  Add "Allow" rules to request filtering for the following types: `.sh`, `.deb`

Now for the hack part.

First change the metadata to point to your now newly served `.deb` file. The host should be the same ip as your Chef-Zero server, eg: `http://169.254.10.1/chef_12.18.31-1_amd64.deb`.

Next you need to hack the install.sh to point to the metadata: Search for `metadata_url=` and set the vallue to L `"http://169.254.10.1/metadata.txt"`, assuming you sasved your metadata as metadata.txt. 

And now you should be ready to bootstrap.

## Bootstrapping

This is a mostly "normal" bootstrap operation, with the one exception that you need the additional option `--bootstrap-url http://169.254.10.1/install.sh`. That's it, you should now be able to bootstrap your VM. The full command is:

`knife bootstrap 169.254.10.10 --bootstrap-url http://169.254.10.1/install.sh -x chef-user -P password -N debchef --sudo -V`

Where the paramters are as follows:

-  `169.254.10.10` - The VM OS' IP address
-  `--bootstrap-url http://169.254.10.1/install.sh` - Tells knife to download and run this script instead of the normal one
-  `-x chef-user` - The SSH user to login as
-  `-P password` - The user's password. You can replace this argument with your key (-i identityfile).
-  `-N debchef` - The new node name on your Chef-Zero server
-  `--sudo` - Tells knife to sudo the commands
-  `-V`(optional) - Verbose output

# Epilogue

Now you're ready to create your bookbooks and bootstrap your new VM against them.

You can of course set the run list during the bootstrap using `-r 'recipe[mycookbook],role[myrole]` added to your bootstrap command.

Note that you'll need to upload these cookbooks and roles to the Chef-Zero server first. You can do this with knife. Eg: `knife role from file myrole.json` and `knife cookbook upload mycookbook`. You also upload environments.

You might find it useful to script these operations as well.

So there you have it, it's a pseudo Chef environment thats a little closer to what you might be using in production.

# Conclusion

Just to recap the why, this is useful for offline, solo development, or demos. The environment is highly predicatable, scriptable, and comparatively fast. Reverting to a known state is dead simple: Revert your VM checkpoint and restart your Chef-Zero.

In practice, for production I would strongly recommend you setup a proper pipeline, ideally using [Chef Automate](https://www.chef.io/automate/), or Jenkins. This allows you to enforce things like kitchen tests and cookbook versioning.

