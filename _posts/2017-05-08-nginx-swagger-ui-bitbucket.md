---
layout:	post
title:	"Nginx Proxy Settings for Bitbucket in Swagger UI"
date:	2017-05-08 12:10:29 +0200
categories:	Nginx
tags:	[Nginx, Swagger, Proxy]
---

***Abstract:** Provide a proxy for bitbucket through nginx as a workaround for swagger UI and bitbucket.*

# Introduction

[Swagger](http://swagger.io/) is a great tool for documenting your REST interfaces. Arguably the best feature of Swagger is that you can define the documentation in text based format, such as YAML, since this allows you to check-in and diff your specifications.

Swagger UI and Swagger Editor are super easy to install too! Simply install [docker](http://docker.com), and then run the containers ([here](https://hub.docker.com/r/swaggerapi/swagger-editor/) and [here](https://hub.docker.com/r/swaggerapi/swagger-ui/)).

There are a couple of catches though...

1.  As great as swagger editor is, it's a pain to version the docs since they only live in the browser's local storage.
2.  Whilst swagger UI lets you inject a URL for the specs to show, CORS on modern browsers means the repository for your swagger files needs to add your swagger UI in the allowed origins list - tricky to do on bitbucket...

But, fear not for there are solutions to both.

#  Versioned Swagger

[Visual Studio Code](https://code.visualstudio.com/) has a [plugin](https://marketplace.visualstudio.com/items?itemName=Arjun.swagger-viewer) that lets you preview swagger in editor. Done.

#  Swagger Viewer

This one is a little more tricky.

Start a swagger UI container: `docker run -p 80:8080 swaggerapi/swagger-ui`

Now you want to reconfigure nginx on the container to get around the CORS problem, but first some background.

Swagger UI lets you input the document to display in the UI with query on your swagger UI URL. For example: `http://localhost:8080?url=http://someotherhost/myswagger.yml`. This works well until the browser blocks your request because `someotherhost` isn't issuing the following headers:

-  `Access-Control-Allow-Origin`: `http://localhost:8080` 
-  `Vary`: `origin`

(Many webservers allow `*` on the request origins, but browsers might ignore that.)

This is becuase the document is loaded by Swagger running in the browser using an XHR request. Since it's cross origin, it must confirm to cross origin requirements with the server.

If you have access to the webserver configuration that is serving your documents, then you can simply add the access control headers and be done with it, however it's a little more tricky when it's a black-box system, like for example BitBucket, where you can't control the headers.

A novel solution however is to use a webserver where you *can* add said headers, and use that to fetch the documents on your behalf. Since the Swagger UI container uses Ngnix, the simplest is to get *that* server to do the proxying for you. 

That's not the end of the story though - there are some other contraints. First of all, your BitBucket might be serving on `https` whilst your modest Swagger UI is plain old `http`. So you'll want to set up the forwarding headers for `https` in the proxy config as well.

The next problem is that BitBucket has access control to the source code. If you're in a trusted environment, you can simply enable anonymous read access to your source code - they're probably "public" APIs anyway. You can set this in BitBucket with an option on the repository

So now that we've got all that out the way, we're ready to jump through the (hopefully) last hoop - the proxy settings.

You can "logon" to the container with a good old: `docker exec -it <container name/id> /bin/sh`.

You can meander over to `/etc/nginx` and note the existance of 1 &times; `nginx.conf`. Simply pop the below snippet in the `server` section before the `location / {` section using `vi` and then reload your config with a `nginx -s reload`. Voile, should now be able to proxy.

```
location /projects {
      proxy_pass https://mybitbucketurl;

      proxy_set_header        Host $host;
      proxy_set_header        X-Real-IP $remote_addr;
      proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header        X-Forwarded-Proto $scheme;

      proxy_set_header accept "text/plain";
    }
```

You may be asking *"But why /projects?"*. The short answer is because it's available. The `projects/` comes immediately after the hostname on the BitBucket file URLs and we can then exploit that to identify BitBucket bound requests in Nginx and foward them on. You could of course place the proxy anywhere, but this seems least intrusive.

You may also be asking *"Why the `accept` header?"*. Turns out that swagger foolishly asked for JSON which BitBucket willingly oblidges... only it serves an array of strings for the lines of the file instead of the JSON model of the swagger. With the `text/plain` it then sends the raw YAML.

You can now access your Swagger documents live from BitBucket through Swagger UI like so:

`http://localhost:8080?url=http://localhost:8080/projects/XYZ/repos/browse/to/my/swagger.yml%3Fat=refs%2Fhead%2Fmaster%26raw`

It's important to remember the `%26raw` which is a url encoded `&raw`, otherwise it loads the HTML page containing the file, instead of the file itself.

# Conclusion

You can now access the live content of your versioned Swagger files straight from BitBucket in Swagger UI - super neato. Happy swaggering. 