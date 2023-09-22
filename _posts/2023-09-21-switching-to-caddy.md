---
title: Switching to Caddy
date: 2023-09-21 11:34:00 +03:00
categories: [Technology, Server]
tags: [caddy]
---

## Backstory

This entire chronicle happened since I mentioned that I was installing *nginx* to my *Proxmox* server. A friend told me to switch to *Caddy* instead, and I was like, "why?". I didn't really get the reason to switch to *Caddy*, but I did decide to give it a shot since with its *Caddyfile* feature it seemed perfect for development environments.

well, this made me look further into what Caddy can do, and I found out that it can do a lot more than what I thought.

## What is Caddy?

**Caddy** is an OSS web server written in Go. At least, that's what it is at its core.

Caddy is very extensible, and it can easily be configured to replace many moving parts of a web server. Even out of the box, it replaces whatever HTTPS certificate solution you were previously using, by providing certificates from *Let's Encrypt* or *ZeroSSL* by default. It can even provide self-signed certificates for local development with its own root authority (which you obviously need to add to your client's trust store).

## How do we extend Caddy?

Before getting too deep, let's talk about the basics.

### How to install modules

You can install modules by going to the [*Caddy* downloads page](https://Caddyserver.com/download) and downloading a binary with the modules you wish to use.
While this initially made me worry about automatic updates, I found out that this is not an issue as Caddy updates itself with the according binary for the modules you have installed.

### Automatic HTTPS

As mentioned, *Caddy* provides a default ACME certificate authority, but you can easily change that to something such as *Cloudflare*.

So, how do we do that?

As you might have noticed in the previous section, *Caddy* provides modules for many DNS providers (look for `dns.provider.*`). Simply enable the one you wish to use and download the binary.

After that, you can start using it by adding the following to your `Caddyfile`:

{: file="Caddyfile" }

```py
{
    # If you wish to explicitly use a self-signed
    # certificate uncomment the line below and comment
    # the acme_dns line
    #
    # local_certs
    acme_dns cloudflare {env.CF_API_TOKEN}
}
```

This will set it as the default DNS provider for all domains.

If you wish to configure this per domain instead, you can do this instead:

{: file="Caddyfile" }

```py
example.com {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }
}
```

> Note: I am using environment variables to set the *API token* for *Cloudflare*. You will see how I have done that later in this post.
{: .prompt-info }

### Dynamic DNS

Now if your reaction was like mine, you are probably equally shocked and excited about the fact *Caddy* can provide dynamic DNS records. Out of the box, *Caddy* does not come with the ability to do this, but again, that's what modules are for.

You will need the [*dynamic_dns* module](https://caddyserver.com/docs/modules/dynamic_dns). Additionally, remember the *DNS provider* module you chose earlier for automatic HTTPS? That same module also gives us the DNS provider information here.

After that, you can use a config such as this:

{: file="Caddyfile" }

```py
{
    dynamic_dns {
        provider cloudflare {env.CF_API_TOKEN}
        domains {
            example.com ddns # This will update 'ddns.example.com'
            domain.com @ ddns # This will update 'domain.com' and 'ddns.domain.com'
        }
        ip_source upnp # You may specify multiple sources. They will be tried in order
        ip_source simple_http https://icanhazip.com
        ip_source simple_http https://api64.ipify.org
        ip_source interface eth0
        check_interval 5m
        versions ipv4 ipv6 # Update both A and AAAA records
    }
}
```

## Ok, cool but how do I switch?

Alright, I get it. Enough of what it can do. How do you go from *nginx* to *Caddy*?

I will show you a couple examples from *nginx* and the way I have achieved them in *Caddy*.

First of all, *Caddy* has no such thing as `sites-available` or `sites-enabled`. Instead, it has a `Caddyfile`. Though, I assume you do not wish to have all of your domains in one file. That's why we will be using the `import` directive.

{: file="Caddyfile" }

```py
{
    ... # Contents of Caddyfile
}

import Example
```

Simple as that. This will import directives defined in the `Example` Caddyfile.

Now, how do we create the sites themselves? I will show you a practical example, that being my `Gitea` config file.

Let's start with my *nginx* config file:

{: file="Gitea.conf" }

```nginx
server {
    server_name git.aesth.dev;
    client_max_body_size 10M;
    location / {
        proxy_pass http://10.10.10.101:3000;
    }

    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    ssl_certificate         /etc/ssl/certs/key.pem;
    ssl_certificate_key     /etc/ssl/private/key.pem;
}
```

Simple enough, right? Well, as you are about to see, this can become even easier using *Caddy*.

{: file="Gitea" }

```py
git.aesth.dev {
        encode gzip
        reverse_proxy 10.10.10.101:3000

        request_body {
                max_size 10MB
        }
}
```

That's it. This will also handle HTTPS automagically using the options we specified earlier in the global config. The way this is set up for me is that `git.aesth.dev`{: .filepath} is simply a `CNAME` record to `ddns.aesth.dev`{: .filepath} which I have also shown you how to configure earlier.

### Ok, now how about more "complex" configurations?

#### File Server

Take this *nginx* configuration for my old website:

{: file="aesth.conf" }

```nginx
server {
        root /var/www/aesth;
        index index.html;
        server_name aesth.dev;

        location / {
                try_files $uri $uri/ =404;
        }

        location /reboot {
                alias /var/www/reboot;
        }

        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        ssl_certificate /etc/ssl/certs/aesth.pem;
        ssl_certificate_key /etc/ssl/private/aesth.pem;
}
```

This is a bit more complicated to achieve in *Caddy*, as it requires handling of trailing slashes.

{: file="aesth" }

```py
aesth.dev {
    encode gzip
    redir /reboot /reboot/
    root * /var/www/aesth
    try_files {path} /index.html
    file_server
}

aesth.dev/reboot/ {
    encode gzip
    root * /var/www/reboot
    try_files {path} /index.html
    file_server
}
```

This will handle `aesth.dev/reboot/`{: .filepath} and redirect `aesth.dev/reboot`{: .filepath} to it.

The `redir` directive is quite powerful, so you can even do things such as `redir /reboot /reboot/?{query}` if you wish to preserve the URL query string.

There is a lot to cover here, but the vast majority of configurations can be translated to *Caddy* without too much trouble. You can see more info about this on the [*Caddy* community wiki](https://caddy.community/t/tutorial-convert-a-nginx-conf-to-caddy-config/18853)

#### Running as a service

Now, you will probably want to run *Caddy* as a service. This section will also show you how I configured my environment variables.

Simply add the following file to `/etc/systemd/system/`{: .filepath}:

{: file="caddy.service" }

```sh
# caddy.service
#
# For using Caddy with a config file.
#
# Make sure the ExecStart and ExecReload commands are correct
# for your installation.
#
# See https://caddyserver.com/docs/install for instructions.
#
# WARNING: This service does not use the --resume flag, so if you
# use the API to make changes, they will be overwritten by the
# Caddyfile next time the service is restarted. If you intend to
# use Caddy's API to configure it, add the --resume flag to the
# `caddy run` command or use the caddy-api.service file instead.

[Unit]
Description=Caddy
Documentation=https://caddyserver.com/docs/
After=network.target network-online.target
Requires=network-online.target

[Service]
EnvironmentFile=/etc/caddy/env
Type=notify
User=caddy
Group=caddy
ExecStart=/usr/bin/caddy run --environ --config /etc/caddy/Caddyfile
ExecReload=/usr/bin/caddy reload --config /etc/caddy/Caddyfile --force
TimeoutStopSec=5s
LimitNOFILE=1048576
LimitNPROC=512
PrivateTmp=true
ProtectSystem=full
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

Then simply add the following to `/etc/caddy/env`{: .filepath}:

{: file="env" }

```sh
CF_API_TOKEN=YOUR_API_TOKEN
```

The resulting directory structure for `/etc/caddy/`{: .filepath} looks like this in my case:

```tree
/etc/caddy
├── Caddyfile
├── env
├── Gitea
└── ...
```

And my `Caddy` binary is located at `/usr/bin/caddy`{: .filepath}.

## Conclusion

I hope this post has also made you consider switching to *Caddy*, or at least made you think about it. In the end *Caddy* has replaced handling HTTPS and my old cron job which handled running [*cloudflare-ddns*](https://github.com/timothymiller/cloudflare-ddns) with a simple nicely contained solution.
