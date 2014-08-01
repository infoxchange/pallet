## docker build



<!-- .slide: data-background="#3F3F3F" -->
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



## TODO: invoke.sh goes here



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
```bash
forklift myapp serve
```



## Developing with Forklift



<!-- .slide: data-background="#3F3F3F" -->
```bash
forklift ./invoke.sh serve
forklift ./manage.py test
```

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
```
forklift --mount-root /tmp/myapp myapp sshd
```

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
