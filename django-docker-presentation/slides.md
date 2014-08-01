<!-- .slide: class="title" -->
# Running Django on Docker
## a workflow and code

Alexey Kotlyarov • Danielle Madeley

![Infoxchange](Infoxchange_logo_SMALL.png)



<!-- .slide: data-background="https://farm4.staticflickr.com/3489/3866788424_3ee4a00967_b_d.jpg"-->
## The problem

<!-- .element: class="credit" -->
Pallets &copy; 2014 philHendley, CC-BY-NC-SA



<!-- .slide: data-background="https://farm1.staticflickr.com/207/459608453_4d9c18359b_o_d.jpg"-->
## A diverse world of applications

<!-- .element: class="credit" -->
4-in-a-box &copy; 2007 cloud_nine, CC-BY-NC-SA

Note:

You have:

* different dependencies between applications
* different *versions* of dependencies between applications
* different setups, i.e. choice of app server
* different package managers
* different Python versions
* compiled sources
* multiple languages
* specific deps you can't control with virtualenv



<!-- .slide: data-background="https://farm3.staticflickr.com/2866/12861161125_5f980a921d_b_d.jpg" class="bg-dark" -->
## Reproducible deployments

<!-- .element: class="credit" -->
Forklift bot &copy; 2014 Legozilla CC-BY-NC-SA

Note:

* Guaranteed identical releases to test, staging and production
* Ability to deploy while PyPI it down
* Rapid recovery after failure
* Rapid scaleout when required



<!-- .slide: data-background="http://i.ytimg.com/vi/Q5POuMHxW-0/maxresdefault.jpg"-->

Note:
* Docker is a framework for providing isolated containers on
  technology such as LXC
* They can be memory and CPU constrained using the same cgroup
  commands
* Containers are versioned and stored in a repository from where
  they can be pushed and pulled
* Containers are immutable!
* Containers can access external storage when configured to
  do so, enabling persistant storage
* Containers are assigned local IPs only routable from the host
  but docker will forward ports exposed ports only when
  configured to do so



<!-- .slide: data-background="#cfc"-->
![Layers](layers-diagram.svg)

Note:

* Docker containers are built up of layers
* You have your base system, application dependencies and
  application, all immutable.
* The files created by the running container will not exist when
  the container is cleaned up unless they were written to
  persistant storage.
* This means your containers start in a known good state
* You can share a container between different servers: test,
  staging, prod, and even start it on multiple servers to scale
  out



<!-- .slide: data-background="https://farm1.staticflickr.com/163/340528570_8c2cb842dd_b_d.jpg"-->
## But there's no standards!

<!-- .element: class="credit" -->
Standard Oil &copy; 2006 Greenlight Designs CC-BY-NC

Note:
* Docker does not provide a standardised interface
* What ports do I use, where do I store my data?
* How do I do maintenance tasks?
* We can borrow a lot of ideas here from 12factor, Heroku
  and Openshift



<!-- .slide: data-background="https://farm8.staticflickr.com/7394/13996065906_fddba4ec84_b_d.jpg" class="bg-white" -->
# Pallet
## An interface for Docker containers
### https://github.com/infoxchange/pallet

Note:
Pallet defines:
* standard ports for things like app servers
* standard internal mountpoints for external storage
* standard environment variables for services, i.e. postgres,
  memcache; and
* standard entrypoints, i.e. docker run deploy,
  docker run serve



## What's in **deploy**?
* database migrations
* loading fixtures
* install static content to static web server (CDN, Ceph, nginx,
  etc.)



## What's in **serve**?
* Start app server
* Starting supporting services, e.g. Celery



## Keep it lean

Note:
* We want to do as little as possible in these two steps,
  anything that can be done in the build should be done in the
  build.
* i.e. installing deps
* LESS -> CSS



<!-- .slide: data-background="#3F3F3F" -->
## $ docker build



<!-- .slide: data-background="#3F3F3F" -->
### Dockerfile
```
FROM debian/ubuntu/fedora/etc.
RUN apt-get -qq update && apt-get -qq install \
  git mercurial \
  python python-virtualenv python-pip \
  ...
```

Note:

Package installation step will be cached. For security updates, this needs to
be rebuilt from scratch, bringing in the latest updates.

Your choice of version control, extra packages, etc.


<!-- .slide: data-background="#3F3F3F" -->
```
RUN useradd -d /app -r app
WORKDIR /app
```

Note:

Docker isn't guaranteed to be isolated from `root` inside the container.


<!-- .slide: data-background="#3F3F3F" -->
```
ADD requirements.txt /app/requirements.txt
RUN virtualenv python_env && \
  pip install -r requirements.txt

ADD . /app
```

Note:

Python dependencies change less often than code, so caching them separately
allows to skip the slow download & install process.


<!-- .slide: data-background="#3F3F3F" -->
```
VOLUME ["/static", "/storage"]

RUN mkdir -p /static /storage && \
  chown -R app /static /storage
```

Note:

Volumes aren't there on build, so don't write anything there in Dockerfile.

When running the container, permissions aren't guaranteed, so `chown` again.
This requires not running as `app` from the start, unfortunately.


<!-- .slide: data-background="#3F3F3F" -->
```
RUN echo "__version__ = '`git describe`'" \
  > myapp/__version__.py

RUN ./invoke.sh install

ENTRYPOINT ["./invoke.sh"]

EXPOSE 8000
```



<!-- .slide: data-background="#3F3F3F" class="small" -->
### invoke.sh
```bash
# default parameters
: ${APP_USER:=app}
: ${WEB_CONCURRENCY:=1}
export WEB_CONCURRENCY

if [ "x$(whoami)" != "x$APP_USER" ]; then
    # ensure we own our storage
    chown -R "$APP_USER" /static /storage

    # Call back into ourselves as the app user
    exec sudo -sE -u "$APP_USER" -- "$@"
else
    . ./startenv
    case "$1" in
        deploy)
            shift 1  # consume command from $@
            ./manage.py migrate "$@"
            ;;

        serve)
            gunicorn -w "$WEB_CONCURRENCY" \
              -b 0.0.0.0:8000 "${APP}.wsgi:application"
            ;;

        *)
            ./manage.py "$@"
            ;;
    esac
fi
```



## Django settings.py



<!-- .slide: data-background="#3F3F3F" -->
```python
from dj_database_url import parse
DATABASES = {
    'default': parse(os.environ['DB_DEFAULT_URL']),
}
```


<!-- .slide: data-background="#3F3F3F" -->
```python
# Logging is complex
LOGGING['handlers']['logstash'] = {
    'level': 'DEBUG' if DEBUG else 'INFO',
    'class': 'logging.handlers.SysLogHandler',
    'address': (os.environ['SYSLOG_SERVER'],
                int(os.environ['SYSLOG_PORT']))
    'socktype': socket.SOCK_STREAM \
                    if os.environ['SYSLOG_PROTO'] == 'tcp' \
                    else socket.SOCK_DGRAM,
}
```


<!-- .slide: data-background="#3F3F3F" -->
```python
# Trust our nginx server
USE_X_FORWARDED_HOST = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
MY_SITE_DOMAIN = os.environ.get('SITE_DOMAIN')
if MY_SITE_DOMAIN:
    ALLOWED_HOSTS = (MY_SITE_DOMAIN,)
```



## Running the container



<!-- .slide: data-background="#3F3F3F" -->
```
docker run \
  -p 8000:8000 \
  -e
DB_DEFAULT_URL=postgres://myapp:pass@db3:5432/myapp_db
\
  -e SITE_DOMAIN=myapp-staging.company.com \
  -e SITE_PROTO=https \
  -e ENVIRONMENT=staging \
  -e
ELASTICSEARCH_URLS=http://es-server-01:9200/myapp_index
\
  -v /mnt/docker-storage/myapp:/storage \
  -h WHY_ARE_YOU_STILL_READING_THIS \
  myapp \
  serve
```



# Urgh!



# Forklift
### a tool for loading pallets

Note:

* Forklift is a tool for running pallets
* Not specific to Docker (but mostly for Docker today)
* Define required services and Forklift will provide them
* Install PostgreSQL locally, or get a Docker image of it



<!-- .slide: data-background="#3F3F3F" -->
### myapp/forklift.yaml
```yaml
services:
- postgres
- elasticsearch
```



<!-- .slide: data-background="#3F3F3F" -->
### $ forklift myapp serve



## Developing with Forklift



<!-- .slide: data-background="#3F3F3F" -->
### $ forklift ./invoke.sh serve

### $ forklift ./manage.py test

Note:

Building container is slow, especially when you want a very tight feedback loop
(debugging). But since the application just needs the environment variables,
Forklift can help!

Caveats:
* Virtual environment is still needed.
* No isolation from OS packages - you need everything installed, ideally the
  same version.



## Poking around inside containers

(aka troubleshooting)



<!-- .slide: data-background="#3F3F3F" -->
#### $ forklift --mount-root /tmp/myapp myapp sshd

Note:

When things go wrong _only in Docker_, you want to reproduce the problem as
closely as possible on the development machine. To do that, you need to run
interesting commands inside the container, modify the filesystem, etc.

Tricks done by Forklift:
* Bind mount container filesystem to a given location
* Install SSH with your public keys authorised inside the container
* More magic to make it work properly

This might also be the only option if you don't want or cannot run the
application directly on your host. (PHP, I'm looking at you.)

In the future, `cgexec` might replace some of the nastiness here.



## Extending Forklift

Note:

Custom services are easy to add (e.g. MySQL).

Can be extended to run pallets on OpenShift or Heroku?



# IXDjango
### pallet configuration for Django

```bash
pip install IXDjango
```

```python
from ixdjango.docker_settings import *
```

github.com/infoxchange/ixdjango

Note:

Reads the required environment variables and provides the default database,
logging, static settings, etc.

Also provides `manage.py deploy`.



## Continuous integration



<!-- .slide: data-background="#3F3F3F" -->
```
forklift --cleanroom myapp test
```

Note:

`test` command runs `./manage.py test` or whatever it is mapped to
via your entrypoint (Lettuce tests, Nose, etc.)

Forklift `--cleanroom` flag starts every required services in a separate
container to ensure the tests don't interfere with any other running
application. Tests can even run in parallel, and the only problem would be
contending for port 8000 (not an issue in Docker).

We have tried to build a CI application to test other container while being a
containerised application itself, but ran into several weird bugs in early
Docker (hopefully fixed now?).



## Legacy applications

Note:

Docker is ideal for doing nasty things...

...reproducibly

* mod_python
* Apache 1.3

Please don't do this, unless you have to

Docker makes it possible to decommission legacy platforms while reducing the
attack area (load balancer in front helps).

The legacy applications get all the benefits of immutable releases, stable
deployments, etc.
