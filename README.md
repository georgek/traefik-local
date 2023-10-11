# Traefik Local Reverse Proxy

A compose file for running a traefik as a local reverse proxy for web services. This
means you can run various web servers across various projects at the same time without
them clashing. 

The typical problem is each project will bring up a web server on port 8000 or
something. You can try changing ports, but then you have to remember which port
corresponds to which service.

This lets you give names to the services and not worry about ports.

Getting started is very easy, read on...

## Usage

Traefik itself will listen on port 80 so make sure this is available first (it usually
is).

Bring up Traefik:

```sh
docker compose up -d
```

Traefik will keep running; you don't need to keep doing this after you reboot.

Browse to http://traefik.localhost/ to see the Traefik dashboard.

## Test

To see if it's working correctly, run a temporary nginx container:

```sh
docker run --name test-nginx --network traefik --rm nginx
```

And browse to http://test-nginx.localhost/  You should see the nginx test page.

Notice that all we had to do here is make sure that container was connected to the
`traefik` network and we can connect to it by name. No ports were configured.

## Docker compose project set up

To make a docker compose project work you need to change only one or two things for each
project:

1. Each container you want to expose must be on the `traefik` network,
2. If the port cannot be auto-discovered (because either multiple ports are exposed, or
   a different port is used for the dev environment), you need to tell traefik the port
   to use.
   
These can both be achieved with a `compose.override.yaml` file (or
`docker-compose.override.yaml`) to override stuff in the regular compose file.

Assuming we have a project called `proj` with a service `www` we can do both 1 and 2 in
an override file like:

```yaml
services:
  www:
    labels:
      - "traefik.http.services.www-proj.loadbalancer.server.port=8000"
      - "traefik.docker.network=traefik"
    networks:
      - default
      - traefik

networks:
  traefik:
    external: true
```

Now you can connect to http://www.proj.localhost/

### Avoiding port clashes

If you have a project that opens specific ports in its `compose.yaml` then you can
disable this in your `compose.override.yaml` like so:

```yaml
services:
  web:
    ports: !reset []
```

You don't need open ports when using Traefik so this should work OK. You could also
change the upstream file to allow the ports to be set using environment variables.

If the upstream project includes an override file already then this is unfortunate. You
will have to supply your own override file manually:

```sh
docker compose -f compose.yaml -f compose.override.yaml -f my-overrides.yaml up -d
```

## How?

Traefik listens to the Docker daemon and discovers running containers automatically.
This project sets up Traefik with a default routing rule for each container: if it's
part of a docker compose set up the host is `service.project.localhost`, otherwise it is
`containername.localhost`.

Traefik uses labels on each container to configure itself further. This way you don't
have to have some big global config file. The combination of the default rule plus a
couple of extra rules is normally all you need.

## Customising

You'll need to read the [Traefik docs](https://doc.traefik.io/traefik/) for everything,
but to give a hint here's another common configuration via a label, you can configure a
different host name, or multiple host names:

```yaml
services:
  www:
    labels:
      - "traefik.http.routers.www-proj.rule=Host(`proj.localhost`) || Host(`othername.localhost`)"
      - "traefik.http.services.www-proj.loadbalancer.server.port=8000"
      - "traefik.docker.network=traefik"
    networks:
      - default
      - traefik

networks:
  traefik:
    external: true
```

## Troubleshooting

The above simple set up requires your OS to support a wildcard `localhost`.  This works
on most GNU/Linux distros like Ubuntu and (I think) MacOS.  It might not work on
Windows.  On Windows you can use `traefik.me` instead of `localhost`.  This does a real
DNS lookup that resolves to `127.0.0.1` for any subdomain.  See <http://traefik.me/> for
further info.

## Further reading

Before I formalised this set up I wrote a blog post about it
[here](https://georgek.github.io/blog/posts/multiple-web-projects-traefik/). You can
also find a link to the Hacker News discussion there with other ideas and options you
might like.
