---
layout:	post
title:	"Deploying Windows Services with Chef"
categories:	Chef 
date:	2016-01-21 08:10:25+0200
tags:	[chef, deployment]
---

Deploying Windows services with chef can be a little tricky. The [`service`](https://docs.chef.io/resource_service.html) resource doesn't actually know how to create services in Windows, and surprisingly, niether does [`windows_service`](https://docs.chef.io/resource_windows_service.html).

Fortunately, this is not a deal breaker since creating services is actually pretty easy using a variety of approaches, and the simplest being `sc create`.

So our task breakdown is:

1.  Create the service if it does not exist.
1.  Stop the service
1.  Fetch the updated service files
1.  Start the service

# Create the service

An [`execute`](https://docs.chef.io/execute.html) resource can run our `sc` command. In order to check if the service exists first, we can use `::Win32::Service.exists?` in the guard block for the execute. You will also need to `require 'win32/service'.

So, it might look like this:

<pre>
  <code class="ruby">
execute "Create FooService" do
    command "sc create \"FooService\" binPath= \"c:/Service/FooService.exe\""
    not_if {::Win32::Service.exists?("FooService")}
end
  </code>
</pre>
Voile!

# Stop the service

This one is straight forward with the `service` or `windows_service` resource:

<pre>
  <code class="ruby">
windows_service "FooService" do
    action :stop
end
  </code>
</pre>


# Fetch the updated service files

In my case I am using my build server to "distribute" the services in a folder structure as a build step, which I then simply zip up, and can download as a build artefact.

This means I have two steps to "fetch": Download, Unpack.

The download step is done with `remote_file`:

<pre>
  <code class="ruby">
remote_file "c:/temp/Package.zip" do
    source "http://buildserver/lastest/package.zip"
    mode '0775'
    only_if {::File.directory?('C:/temp/') }
    action :create
end
  </code>
</pre>

You can of course use `role` and `environment` attributes to inject the URL for the package.

I've decided to use [7-Zip](www.7-zip.org/download.html) to do my unpacking, which I run with another execute resource:
 
<pre>
  <code class="ruby">
execute 'UnpackServices' do
    command '7z x C:/temp/Package.zip -o"C:/Service" -y'
    environment(
        'Path' => 'c:/"Program Files"/7-Zip' 
    )
    subscribes :run, 'remote_file[c:/temp/Package.zip]', :immediately
end
  </code>
</pre>

Note that I'm subscribing to the `remote_file[c:/temp/Package.zip`. This is because the `remote_file` resource seems to execute asynchronously, so I need to wait for that to finish before trying to unpack it.

# Start the service

Starting the service is straight forward. You can either use a new service resource (just provide a different name), or you can notify the existing service resource from your `UnpackServices`:

<pre>
  <code class="ruby">
notifies :start, 'windows_service[FooService]', :immediately
  </code>
</pre>

# Conclusion

And thats the nuts and bolts of it.

Theres a few things going on here, and I've templatised this into a way that makes it easy to extend without changing the recipe (as in, add new services), but that's for another post.