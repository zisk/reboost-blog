Title: Let's Encrypt for Nginx Behind CloudFlare
Date: 2016-08-16 20:56

I decided to try out [CloudFlare](https://www.cloudflare.com/) on this blog to get a better understanding of how it works. The site was already protected with an SSL certificate from (the very awesome) [Let's Encrypt](https://letsencrypt.org/) service. CloudFlare also gives you a free SSL when you route your site through them, but I wanted to ensure that traffic was fully encrypted end-to-end. The Let's Encrypt Auto script could no longer verify the site via direct IP connection since it is masked by CloudFlare. Luckily the script is also able to verify ownership by looking at a special page it generates when using the [webroot](http://letsencrypt.readthedocs.io/en/latest/using.html#webroot) option. I just had to make sure the files were accessible properly.

I am using NGINX to serve the content from [Ghost](https://ghost.org/). I didn't feel like digging into Ghost to figure out if it would be possible to serve the transient files that the script creates. Luckily, this task is easy with NGINX.

Here's the relevant snippet before the change:

```
server {
        listen 443 ssl;
        server_name blog.reboost.net;
        ssl_certificate /etc/letsencrypt/live/blog.reboost.net/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/blog.reboost.net/privkey.pem;


        location / {
                proxy_set_header  X-Real-IP $remote_addr;
                proxy_set_header   Host    $http_host;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_pass        http://127.0.0.1:2368;

        }

}

```

The auto renew script creates folder called .well-known (so `http://example.com/.well-known/<unique renewal key>`).

Simply adding a new more specific location tag will do the trick:

```

server {
        listen 443 ssl;
        server_name blog.reboost.net;
        ssl_certificate /etc/letsencrypt/live/blog.reboost.net/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/blog.reboost.net/privkey.pem;

        location /.well-known {
                root /var/www/;
        }

        location / {
                proxy_set_header  X-Real-IP $remote_addr;
                proxy_set_header   Host    $http_host;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_pass        http://127.0.0.1:2368;

        }

}

```

Now, the following command can be run to renew any certificates on the server:

``` bash 
letsencrypt-auto renew --webroot --webroot-path /var/www
```
