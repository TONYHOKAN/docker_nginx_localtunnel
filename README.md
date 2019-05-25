# Background
I am a developer that building service that need call back and test in my local. One of the very useful service provider is [localtunnel.me](https://localtunnel.github.io/www/). But my service contain sensitive data, so use their service may not be good enough. 

Thanks that [localtunnel server](https://github.com/localtunnel/server) is open source that we can build and hold my own. 

I see the official doc do not describe detail.  Below is detail steps that i create with my own localtunnel

# Get Ready
Before start, we need to prepare below:

1. VPS, in my case i use [aws lightsail](https://aws.amazon.com/lightsail/). Of course, [vultr](https://www.vultr.com/), [linode](https://welcome.linode.com/) are also good choice.
2. Domain name for your VPS, just go any domain name seller to buy one like [Godaddy](https://www.godaddy.com/). We will setup wildcard subdomain.

# Let's Start

## 1. Set up in VPS

### Set up aws lightsail
In my case, i just start Ubuntu OS in aws lightsail, detail is skipped here, follow AWS guide is good enough. 

One thing **must** to do is we need allow all TCP going to aws. Just go lightsail console, Networking tab, Firewall section, add `All TCP` to the rule.

### Install softwares to VPS

1. SSH to server and become root user
```
ssh ubuntu@{your VPS IP}
sudo su
```

2. Install softwares 
```
# update apt
apt-get update 

# install git, docker
apt-get install -y git docker

# install docker-compose, ref: https://docs.docker.com/compose/install/
curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

```

## 2. Domain Setting
Base on your domain provider will have different setup. But in general, 2 things need to be set:
1. set Type `A` with Name `lt` point your `VPS IP`
2. set Type `A` with Name `*.lt` point your `VPS IP`

Remember you need wait at least 30 minutes that domain change take effect

## 3. Get SSL Cert 
Because my callback service need https, so i need SSL cert. I use [certbot](https://certbot.eff.org/) for free cert. Certbot will be expired every 3 months, so becareful remember to renew.

### Set up cerbot

Still use root in your VPS

```
# get certbot
git clone https://github.com/certbot/certbot

# cd to folder
cd /certbot

# start certbot domain check, you need replace with your domain
./certbot-auto certonly --manual -d *.lt.{YOUR_DOMAIN} -d lt.{YOUR_DOMAIN} --agree-tos --manual-public-ip-logging-ok --preferred-challenges dns-01 --server https://acme-v02.api.letsencrypt.org/directory

# you will see below, you need go back to your domain provider to setup 
--------------------------------------------------------------------
Please deploy a DNS TXT record under the name
_acme-challenge.lt.{YOUR_DOMAIN} with the following value:

MfXTzyYTiASihTo1t6gO-zr-v6A8wQBPOb2OehhY7AM

Before continuing, verify the record is deployed.
--------------------------------------------------------------------
Press Enter to Continue
--------------------------------------------------------------------
Please deploy a DNS TXT record under the name
_acme-challenge.lt.{YOUR_DOMAIN} with the following value:

MfXTzyYTiASihTo1t6gO-zr-v6A8wQBPOb2OehhY7AM

Before continuing, verify the record is deployed.
--------------------------------------------------------------------
Press Enter to Continue

# when done you will see successful message and find cert in path: /etc/letsencrypt/archive/lt.{YOUR_DOMAIN}

```

## 4. Set up Localtunnel Server

We need clone this project as template, and update some config to your own.

### Config 
```
# clone this project 
git clone https://github.com/TONYHOKAN/docker_nginx_localtunnel.git

# go to project folder
cd docker_nginx_localtunnel/

# copy .env_sample to .env and update DOMAIN key in .env to your own
cp .env_sample .env

# set up cert from certbot for nginx 
cp /etc/letsencrypt/archive/lt.{YOUR_DOMAIN}/fullchain.pem /ssl/server.crt
cp /etc/letsencrypt/archive/lt.{YOUR_DOMAIN}/privkey.pem /ssl/server.key 
```

### Start Localtunnel with Docker

We use docker to run our server
```
# build image
docker-compose build

# run the server in docker_nginx_localtunnel/ directory
docker-compose up

# after start, you will see message
Starting localtunnel_server ... done
Starting localtunnel_nginx  ... done
Attaching to localtunnel_server, localtunnel_nginx
localtunnel_server    | 2019-05-25T16:29:50.530Z koa-router defined route HEAD,GET /api/status
localtunnel_server    | 2019-05-25T16:29:50.533Z koa-router defined route HEAD,GET /api/tunnels/:id/status
localtunnel_server    | 2019-05-25T16:29:50.534Z koa:application use dispatch
localtunnel_server    | 2019-05-25T16:29:50.535Z koa:application use allowedMethods
localtunnel_server    | 2019-05-25T16:29:50.535Z koa:application use -
localtunnel_server    | 2019-05-25T16:29:50.536Z koa:application use -
localtunnel_server    | 2019-05-25T16:29:50.545Z localtunnel server listening on port: 3000

# try curl with your server
curl https://lt.{YOUR_DOMAIN}

# you will see message
localtunnel_nginx     | 210.6.196.42 - - [25/May/2019:16:33:52 +0000] "GET / HTTP/1.1" 404 169 "-" "curl/7.54.0"

# you are done!!

```

## 5. Test in Local
Everthing work fine until this step, you can start localtunnel in your local now.

```
# in your local, install localtunnel client 
npm install -g localtunnel

# start localtunnel client connection
lt -h https://lt.{YOUR_DOMAIN} -p 8080 --subdomain test

# you will see return new url
your url is: https://test.lt.{YOUR_DOMAIN}

# now everyone can go https://test.lt.{YOUR_DOMAIN} to access your local site
```