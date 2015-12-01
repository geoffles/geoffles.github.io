---
layout:	post
title:	"Getting started with Chef"
date:	2015-12-01 17:16:53 +0200
categories:	Chef
tags:	[chef, configuration, infrastructure]
---

I've been playing around with Chef for the past few weeks and have found that whilst there is a lot of documentation, it's been difficult to a short, concise document describing how to set up your own Chef server and what all the bits are.

* TOC
{:toc}

# The Chef Server

So the first comment here is that Chef Server is Linux only at this stage.

I've had to do this in CentOS, so this walkthrough will describe the process for that distro - not my choice, but hey, it works.

## Installation and Setup

1.  **Download** Chef Server from [Chef Downloads](https://downloads.chef.io)
2.  **Install** the package: `$ sudo rpm -Uvh ./chef-server-core-12.2.0-1.el7.x86_64.rpm`
3.  **Configure** Chef: `$ sudo chef-server-ctl reconfigure`
4.  A **User** for yourself, create: `$ sudo chef-server-ctl user-create user_name first_name last_name email password --filename FILE_NAME`
Note that the username **must** be all lowercase (and when logging in!).
5.  An **Organisation** for cookbooks, create: `$ sudo hef-server-ctl org-create short_name "full_organization_name" --association_user user_name --filename ORGANIZATION-validator.pem`
Use organisations to segregate access to cookbooks. Also note that you must associate your user with the organisation so that you can administer it.
6.  **Open Ports** (on CentOS): `$ sudo iptables -A IN_public_allow -p tcp -m tcp --dport 443 -j ACCEPT`

... And your barebones chef server is now running with a login, an organisation and hopefully accessible.

Note the *filename* arguments, as you will need these for your Chef workstation and clients. You can use something like: `~/bobby-tables.pem` and `~/acmecorp-organisation.pem` or the user and org pems respectively

## Notes

-  Chef will use the machines FQDN - so your clients/workstations will all need to use that host name since it will be on the certs.
-  Use the American spelling for organi**z**ation.

## Troubleshooting and diagnostics
-  Chef says that [SE Linux must be set to permissive mode](https://learn.chef.io/- install-and-manage-your-own-chef-server/linux/install-chef-server/install-chef-server-using-your-hardware/).
- Check that your port 443 is open. You can open ports on CentOS with:
`$ sudo iptables -A IN_public_allow -p tcp -m tcp --dport 443 -j ACCEPT`
- You can verify that your chef server is running with:
`$ sudo chef-server-ctl`

# The Chef Workstation

Since I do most of my work on windows, these are the directions I followed. Run this from power-shell, ideally use the shortcut that ChefDK creates for you on the Desktop/Start Menu.

## Setup and Configuration

1.	Download and Install the Chef DK from [Chef Downloads](https://downloads.chef.io)
2.	Run `knife configure`
3.	Supply a `knife.rb` path (default is good)
4.	Supply chef server URL, eg: `https://mychefserver/organisations/myorganisationshortname`
5.	Copy the pems to the location indicated by *knife configure*
	1.	You can use something WinSCP
	2.	The pem files are the user and org pem files from setting up the server.
6.	Run `knife ssl fetch` to download the chef server's certificate
7.	Run `knife ssl check` to verify the cert
	1.	Check the chef server FQDN, and the host you're connecting to. They must match
8.	Run `Knife client list` to verify that everything works

##  Troubleshooting
I had troubles with proxy servers. You can configure proxies for knife in your `knife.rb`. You can set `https_proxy` with your proxy server. 

If your chef server is local and you only need the proxy to download plugins and things from the internet, set your 'no_proxy' on your Chef Server.

If, like me, you had a real bad time with connecting your Corporate's proxy, you can make things easier with Fiddler which at least integrates into your windows proxy settings. If you need NT authentication as well on the proxy, you can use [Cntlm](http://cntlm.sourceforge.net/).

# The Chef Client

Chef advocates using `knife bootstrap`-ing, but I wanted to get a feel for what actually needed to be done and didn't feel like wrestling with WinRM problems.

## Manual Installation and Setup

1.	Install chef client
	2.	Target Folder `c:\chef`
3.	Create `client.rb` (example below) in `c:\chef`
4.	Copy `ORG-validation.pem` file to `c:\chef`
5.	Copy the trusted certs from your workstation
	1.	`~/.chef/trusted_cert`s -> `c:\chef\trusted_certs`
	2.	OR knife ssl fetch (requires knife config)
6.	Register node: `chef-client`

Here's a sample `client.rb`:

~~~
log_level :info
log_location STDOUT
chef_server_url 'https://chefserver/organizations/acmecorp'
validation_client_name 'acme-validator'
validation_key 'c:/chef/acme-validator.pem'
~~~

Note that no client.pem is required when first setting up the node. One will be generated to match the node name.

## Knife Boostrap

Client's can also be bootstrapped from a chef workstation. This is *theoretically* what is required , but I haven't done it personally yet.

From your chef Workstation:`
$ knife bootstrap windows winrm ADDRESS --winrm-user USER --winrm-password 'PASSWORD' --node-name node1 --run-list 'recipe[hello_chef_server]'`

Advice on enabling WinRM [here](http://linux.last-bastion.net/infocentre/how-to/winrm)

# Conclusion

The above instructions should be enough to get you going on setting up a Chef Server, Workstation, and Node. This was my experience when trying to get these things running, mileage may vary. Altogether though I have to say the Chef team seems to have put a lot of effort into making it pretty simple.

# Some References

- [https://learn.chef.io/install-and-manage-your-own-chef-server/linux/manage-a-node-on-your-chef-server/](https://learn.chef.io/install-and-manage-your-own-chef-server/linux/manage-a-node-on-your-chef-server/)
- [https://learn.chef.io/install-and-manage-your-own-chef-server/linux/install-chef-server/install-chef-server-using-your-hardware/](https://learn.chef.io/install-and-manage-your-own-chef-server/linux/install-chef-server/install-chef-server-using-your-hardware/)
- [https://docs.chef.io/install_server.html](https://docs.chef.io/install_server.html)
- [https://www.digitalocean.com/community/tutorials/how-to-set-up-a-chef-12-configuration-management-system-on-ubuntu-14-04-servers](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-chef-12-configuration-management-system-on-ubuntu-14-04-servers)
