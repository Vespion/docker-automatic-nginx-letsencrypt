# Automatic reverse proxying with Docker and nginx

This repository contains the `docker-compose.yml` file used to bring up
a collection of containers that will provide automatic reverse proxying
and SSL termination for other docker containers.

More details will be added here in due course. For now, the full process
is described in [this blog post](https://www.chameth.com/2016/05/21/docker-automatic-nginx-proxy).

## Getting started

This repository contains configuration for four different methods of obtaining
Let's Encrypt certificates:

 * **`docker-compose.override.generic.yml`** - a generic solution for obtaining
   certificates using DNS entries. You must supply a
   [Dehydrated](https://github.com/lukas2511/dehydrated) hook that will add and
   remove DNS entries as needed.
 * **`docker-compose.override.lexicon.yml`** - uses the
   [Lexicon](https://github.com/AnalogJ/lexicon) library to perform DNS updates
   for major cloud DNS providers.
 * **`docker-compose.override.mydnshost.yml`** - uses the
   [MyDNSHost](https://mydnshost.co.uk) API to perform DNS updates for domains
   hosted there
 * **`docker-compose.override.http.yml`** - performs a HTTP challenge instead of
   using DNS, saving the response to disk so it can be served by Nginx.

To get started:

 1. Copy the relevant docker-compose.override.\*.yml file to
    docker-compose.override.yml
 2. Change any settings (auth token, staging environment, etc)
 3. Run `docker-compose up -d`

If you have existing containers with the appropriate labels, the certificates
will be requested for them straight away, and proxy rules added. To launch
a new container and have it be proxied, add the following labels:

    com.chameth.proxy=<port>
    com.chameth.proxy.protocol=<protocol> # defaults to http
    com.chameth.vhost=<primary vhost>,<secondary vhost>,<...>

For example:

    docker run \
        --label com.chameth.proxy=80 \
        --label com.chameth.vhost=example.domain.com \
        tutum/hello-world

It may take a minute or two for the certificate to be obtained and for
Nginx to be reconfigured. You can see output from the various tools
by running `docker-compose logs -f`.

## Advanced / Tips and Tricks

### Adding extra config to Nginx

Out of the box, the Nginx server will only handle HTTPS requests,
with a very minimal config. The [extra](extra/) directory contains
some additional configuration snippets which may potentially be
useful.

Once you have the services running, you can copy additional config
using the cp command:

    docker cp file.conf autoproxy_nginx:/etc/nginx/conf.d/

The following config files are available in the extra directory:

 * [hsts.conf](extra/hsts.conf) - enables HTTP Strict Transport Security for
   all HTTPS hosts. HSTS tells browsers that they should only ever request
   pages on that domain over HTTPS.
 * [redirect-http.conf](extra/redirect-http.conf) - adds a default HTTP
   server that redirects all traffic to HTTPS.
 * [security.conf](extra/security.conf) - enables some security best
   practices: stops Nginx reporting its version, and adds headers to
   help mitigate clickjacking, content type hijacking, and XSS.
 * [ssl.conf](extra/ssl.conf) - adds extra SSL configuration options to
   disable old protocols and ciphers, enable stapling, etc. This will prevent
   access from older browsers and operating systems!

### Hosting static content

If you're serving static content, it's not desirable to have lots of
instances of nginx running just to handle requests from the proxy.

I recommend using [GoStatic](https://github.com/PierreZ/goStatic) to
host static content. This is a very small image that runs a very
small Go binary to serve the files. You can use it in a docker-compose
file like so:

```yaml
---
version: '2'

services:
  www:
    image: pierrezemb/gostatic:latest
    command:
      - --forceHTTP
    labels:
      com.chameth.vhost: 'example.com'
      com.chameth.proxy: '8043'
    volumes:
      - ./www:/srv/http
```

