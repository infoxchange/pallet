## Forklift

Note:

* Forklift is a tool for running pallets
* Not specific to Docker (but mostly for Docker today)
* Define required services and Forklift will provide them
* Install PostgreSQL locally, or get a Docker image of it


## Forklift

```yaml
# myapp/forklift.yaml
services:
- postgres
- elasticsearch
```

```bash
forklift myapp serve
```


## Developing with Forklift

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


## Inspecting a container

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



## IXDjango - pallet configuration for Django

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

```
forklift --cleanroom myapp test
```

Note:

`test` command runs `./manage.py test` or whatever it is mapped to (Lettuce
tests, Nose, etc.)

Forklift `--cleanroom` flag starts every required services in a separate
container to ensure the tests don't interfere with any other running
application. Tests can even run in parallel, and the only problem would be
contending for port 8000 (not an issue in Docker).

We have tried to build a CI application to test other container while being a
containerised application itself, but ran into several weird bugs in early
Docker (hopefully fixed now?).
