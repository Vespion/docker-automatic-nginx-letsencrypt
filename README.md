# Automatic reverse proxying with Docker and nginx

This repository contains the `docker-compose.yml` file used to bring up
a collection of containers that will provide automatic reverse proxying
and SSL termination for other docker containers.

More details will be added here in due course. For now, the full process
is described in [this blog post](https://www.chameth.com/2016/05/21/docker-automatic-nginx-proxy.html).

## Hosting static content

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

