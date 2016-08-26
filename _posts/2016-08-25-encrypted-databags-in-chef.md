---
layout:	post
title:	"Encrypted Data Bags in Chef"
date:	2016-08-25 14:14:00 +0200
categories:	Chef
tags:	[chef, databags]
---

Theres a little more to using encrypted data bags with Chef than meets the eye.

This post is using chef zero and Windows, although Linux shouldn't be too different. 

* TOC
{:toc}

# Prerequisites

 - ChefDK
 - Somewhere to provision machines

 That's about it really - This post doesn't cover provisioning, but I have another post which does [here](http://pleasefixme).

# Creating a data bags

You can use knife to setup your encrypted data bags

## Knife Zero

If you're like me and prefer to develop against chef-zero, you can use knife-zero with the `-z` option load stuff into your data bags. IF you need any configs (which sadly it doesn't seem you can pass in via command line), you can just write a `knife.rb` and point your knife to that config with the `-c <my file>` option. I use `knifezero.rb` as the filename since it's now clear that that's the config for my knife zero.

## Creating and loading the data into the data bag   

{% highlight powershell %}
knife data bag create -z -c ".\knifezero.rb" secret_secrets --secret-file ".\my_secrets_key"
{% endhighlight %}

{% highlight powershell %}
knife data bag from file -z -c ".\knifezero.rb" secret_secrets ".\secrets_bag.json" --secret-file ".\my_secrets_key"
{% endhighlight %}

The bag looks like:

{% highlight json %}
{
  "id":"secret_user",
  "user":"secretuser",
  "pass":"mysupersecretpassword"
}
{% endhighlight %}

The `knifezero.rb` looks like:

{% highlight ruby %}
data_bag_encrypt_version 2
{% endhighlight %}

This sets the encryption scheme to version 2.

# Creating and managing your secret key (on windows)

## Creating a key

You can simply use the command below froma Chef Development Kit Powershell session. ChefDK has an embedded OpenSSL which it uses.

{% highlight powershell %}
openssl rand -base64 512 | tr -d "\r\n" > "c:\path\to\my\secret_key"
{% endhighlight %}

Keep this key safe and secret - no checking into VC or public access, etc. 

## Copying the key 

{% highlight ruby %}
machine 'mymachine' do
	machine_options the_machine_options
	admin true    
	recipe 'my_cookbook::target_recipe'
	files 'c:\\secrets\\secret_key' => {
		local_path: 'c:\\path\\to\\my\\secret_key'
	}
end
{% endhighlight %}


# Version mismatches

Knife defaults to version 1 encrypted data bags where as chef client defaults to version 2. Your knife.rb sets knife to upgrade to version 2 encrypted data bags.

If you have a version mismatch, Chef infuriatingly reports "Bad Decrypt" and suggests that maybe you have a bad key.

If you get such an error and you're pretty sure that the key is in fact correct, chances are you actually have a version mismatch on your 



