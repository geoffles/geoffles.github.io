---
layout:	post
title:	"Running a docker registry in AWS"
date:	2016-03-29 11:45:00 +0200
categories:	Development
tags:	[AWS, docker]
---

The easiest way to do this is to run a registry docker container. The docker hub image unfortunately (but understandably) has no certificates for TLS, but it's not difficult to add the certs.

The main reason to have a private registry is the same as having private code repos. You don't always want to share everything.

## Set Up Docker

### With Ubuntu

Basically you can follow the steps [here](https://docs.docker.com/engine/installation/linux/ubuntulinux/) for setting up docker engine.

In a nutshell:

- Add the docker repo
- Apt-Get install docker

### With Amazon Linux

`yum install docker`

It's worth noting that this is also not the lastest version of docker, but should be good enough. 

### A Note about MTUs

This one baffles me a bit, but the default MTU settings on AWS machines interferes with docker. Set your MTU to 1500:

`sudo ifconfig eth0 mtu 1500 up` 
 
## The Registry

There the intructions [here](https://github.com/docker/distribution/blob/master/docs/deploying.md)
are pretty good.

You'll want to set up a secure registry because it seems to be simpler than running without security (and you should anyway to make sure you're talking to the correct registry).

To do that:

1. Clone the repo `git clone https://github.com/docker/distribution.git` (repo name is "*distribution*")
2. Create a certs directory
3. Generate a cert using something like OpenSSL (example below).
4. Run a `docker build`

You can do this on the AWS instance as well. 

```
openssl req -config /path/to/openssl.cnf -newkey rsa:2048 -nodes -keyout /path/to/distribution/certs/domain.key -x509 -days 365 -out /path/to/distribution/certs/domain.crt
```
**Note:** The `Common Name` is also the FQDN which must match the hostname for your registry otherwise clients won't trust the machine. 

You will want to install the `domain.crt` on whichever workstations you'll want to work from.

You can then run the registry with:

```
docker run -d -p 5000:5000 my-secure-registry
```

## Certs

You'll need to install the certs on your workstations if you want to develop images on your workstations and push to the registry.

You can follow the docker official guide [here](https://docs.docker.com/registry/insecure/#docker-still-complains-about-the-certificate-when-using-authentication). This should be good enough for your AWS machines, but unfortunatley isn't so good for boot2docker (ironically).

The steps I'm about to share for boot2docker steps don't survive a reboot, but if they're scripted you can just re-run the script using docker-machine. [This](https://github.com/boot2docker/boot2docker/issues/347) post claims you can script it into your bootlocal.sh, but I haven't tried that since I don't often reboot my docker machine.

**Hostnames:** You might also want your script to add a host entry if your domain is using some funky routing.  
 

### Step 1: Docker Cert

`/etc/docker/certs.d/mydomain.com/ca.crt`

The guide place claims you should include the port. Don't do that.

### Step 2: Machine CA

#### Boot2Docker
1. Copy your CA cert somewher
2. Link your CA cert to the SSL certs dir
3. Create a hash link to the cert
4. Add your cert to the CA list

For Example:

First copy the CA to your docker-machine:

```
docker-machine scp /path/to/distribution/certs/domain.crt/ default:/home/docker/ca.crt
```

You can SSH into the dockermachine with:

```
docker-machine ssh default 'sh'
```

Or alternatively push a script, or connect using ssh - any SSH option is fine. 

```sh
cp myCA.crt /var/lib/boot2docker/certs/mydomain/ca.crt
CA_HASH=$(openssl x509 -hash -in /var/lib/boot2docker/certs/mydomain/ca.crt | head -n 1)

ln -s /var/lib/boot2docker/certs/mydomain/ca.crt /etc/ssl/certs/domain.pem
ln -s /var/lib/boot2docker/certs/mydomain/ca.crt /etc/ssl/certs/$CA_HASH.0

cat /var/lib/boot2docker/certs/mydomain/ca.crt | sudo tee -a /etc/ssl/certs/ca-certificates.crt
```


An `ls -l` of your `/etc/ssl/certs/` should now have the following entries:

-  `xxxxxxxxx.0 -> /var/lib/boot2docker/certs/mydomain/ca.crt`
-  `mydomain.pem -> /var/lib/boot2docker/certs/mydomain/ca.crt`

Where `xxxxxxxxx` is the hash of your CA cert. Your ca-certificates.crt file should also contain your CA cert.


## Conclusion

You should now have a private docker registry that you can securely access. You can push images to the registry by tagging them:

```
docker tag <Image Name> my.docker.registry:5000/<image-name>
docker push my.docker.registry:5000/<image-name>
```