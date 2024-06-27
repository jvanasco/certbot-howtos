# HOWTO - Certbot + Nginx

I was an early adopter of nginx, and it has been my preferred webserver for nearly 2 decades.

As a longtime user, I don't want Certbot configuring my nginx files.

The nginx plugin also has issues with the following:

* it can not handle large installations (many domains) well
* it can not handle non-standard configuration files, such as the directives used in OpenResty (an nginx fork)

My preferred integration hinges on a few things:

## Centralized SSL Configuration

I define my core ssl directives in a shared macro file: `/etc/nginx/macros/ssl_core.conf`.

This file will contain all the ssl params shared across installs.

It is basically all the `ssl_*` directives output by https://ssl-config.mozilla.org/, with the exception of:

* ssl_certificate
* ssl_certificate_key

So an SSL integration for a configuration file will just look like:

    /etc/nginx/sites-enabled/com.example

    ## SSL - core
    include  /etc/nginx/macros/ssl_core.conf;

    ## SSL - this is our default
    ssl_certificate  /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key  /etc/letsencrypt/live/example.com/privkey.pem;

## Certbot Standalone Mode + Nginx Proxypass

I prefer to operate Certbot with "standalone" mode -- in which Certbot spins up it's own webserver and handles the entire procurement process within memory.  In order to do this without disrupting services, Certbot will need to run on an alternate port and we will need to proxypass traffic from port 80 onto it.

I accomplish this in a few steps:

I define my a shared proxypass in a macro: `/etc/nginx/macros/letsencrypt.conf`.  Every domain conf file will include this macro:

    location  /.well-known/acme-challenge  {
        if (!-f /etc/nginx/_flags/certbot-8111) {
            rewrite ^.*$ @letsencrypt_503 last;
        }
        proxy_set_header  X-Real-IP  $remote_addr;
        proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header  Host  $host;
        proxy_pass        http://127.0.0.1:8111;
    }

    location = @letsencrypt_503 {
        internal;
        return 503;
    }

Things to note here:

* the line `if (!-f /etc/nginx/_flags/certbot-811)` is a lightweight semaphore check to see if a file exists on disk. This is almost entirely done in memory due to several implementation details, and a popular technique to handle maintenance switches.  If the file does not exit, we do a final rewrite to `@letsencrypt_503`, which is a technically invalid and impossible URI.
* the location `@letsencrypt_503` can only be triggered by the above rewrite. This lets us return a 503 and also do any special logging or fileserving.

When I invoke Certbot, I use the `--pre-hook` and `--post-hook` to enable/disable that rewrite rule.  This means we will trigger that 503 if the backend service is not *allowed* to run.  

    certbot \
        renew \
        --standalone \
        --http-01-port=8111 \
        --pre-hook "touch /etc/nginx/_flags/certbot-8111" \
        --post-hook "rm /etc/nginx/_flags/certbot-8111"

In order to test the nginx integration, I just need to run a "fake server" on my alternate port that response to the `/.well-known/acme-challenge/{TOKEN}` requests.  I have open sourced one here: https://github.com/aptise/peter_sslers/blob/main/tools/fake_server.py

Assuming your `python` and `pip` bins are mapped to a recent version of Python3:

    cd ~
    mkdir nginx-tests
    cd nginx-tests
    curl -O https://github.com/aptise/peter_sslers/blob/main/tools/fake_server.py
    pip install pyramid
    python fake_server.py 8111

With that script running, I can make requests to `https://example.com/.well-known/acme-challenge/foo` and I should get a 200 response with "foo" -- or whatever other "challenge" I submit as the last part of the url.  The fake server will also add to the response headers: `X-Peter-SSLers: fakeserver`.  If I don't see either of those things, I need to debug the nginx logs.  

I will ensure this integration works correctly with the semaphore file (`/etc/nginx/_flags/certbot-8111`) both existing and missing.
