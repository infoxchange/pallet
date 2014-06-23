Pallet
======

###A standard for deploying containerized applications

[Docker] enables running applications regardless of the language, framework and
other underlying technologies they are using - via containers. However, thanks
to its flexibility, different containers have different deployment
instructions, requiring additional work to deploy and run.

This is a set of conventions to deploy Web applications via [Docker]
containers. It supports no-downtime upgrades and clustering, and is aimed to
follow [12factor] as close as possible.

The standard is a work in progress, so there is no stable version yet. The
specifications below can change without notice.

The Pallet specification is a contract between a deployment system (referred to
as Pallet hereafter) and an application represented by a Docker image.

Multiple independent instances of the application - environments - can be
running within Pallet at the same time.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119].

Application lifecycle
---------------------

The expected main function of the applications is sering user requests.
However, to facilitate releasing new versions of the application, four
distinct phases are recognized:

### Build

Each application is represented, to Pallet, as a Docker image, with tags
corresponding to versions. Building the image is thus outside the scope.
Typically, tasks such as fetching, installing and configuring dependencies,
compiling, building and minifying assets belong to this phase.

Pallet MAY assume the images are not changed on the container yard in use (i.e.
it MAY cache the downloaded images). Pallet assumes no version ordering, it is
up to the users to choose which versions to release.

### Deploy

The lifecycle of an application within Pallet begins with the deploy phase. The
application MUST support the deploy phase via running the `deploy` command.

Deploy phase is meant to prepare the environment for this particular version of
the application. This includes the tasks typically known as "migrations", for
exapmple, setting up the database structure.

Deploy phase SHALL be run at least once before serve phase of the particular
version.

Pallet MAY run the deploy phase in parallel, including at a different host,
with a serve phase previous or a current version of the application. Because of
this, the deploy phase SHOULD be idempotent with regards to the functioning of
serve phases of the current version.

Since existing serve phases of any previously released versions MAY be running
in parallel with the current version's deploy phase, the latter SHOULD preserve
the normal functioning of the old serve phases as well.

Pallet SHALL NOT run a deploy phase of two separate versions of the
applications in parallel. Pallet MAY run the deploy phase of the same version
in parallel on multiple hosts.

The deploy phase MUST exit with 0 to indicate success, and MUST exit with a
non-zero error code to indicate failure. Pallet MUST NOT run the serve phase of
a version that failed deploy, unless manually instructed to run the same
version again, in which case a deploy phase MUST be run again.

If Pallet ran more than one deploy phase of an application version in parallel,
and their success status was different, it MAY either abort the deployment or
proceed with running serve phases. Pallet SHOULD NOT forcefully terminate a
running deployment phase.

The deploy phase MUST NOT assume that any other version of itself was already
released in the environment it's running in. It also SHOULD run correctly on
top of any of the versions known to be ever released into the environment.

### Serve

Once an application is successfully deployed, it is expected to serve the
incoming HTTP requests for the user. It MAY also do additional processing as
well, such as running periodic tasks.

The serve phase is run via the `serve` command to the Docker image.

The application MUST listen for HTTP requests on port 8000 during this phase,
and Pallet MUST proxy the requests from the users to the application. In case
the application is exposed to the users via HTTPS, Pallet MUST still make
requests to the application in HTTP.

Pallet MAY run multiple serve phases of the same application version in
parallel, once at least one deploy phase of said version completed
successfully. Pallet MAY terminate serve phases at any time, however, at least
one of them SHOULD be running at any given time.

Once a new version of an application is deployed, Pallet MUST, within a
reasonable timeframe, terminate the serve phases or the previous version.

Terminating the serve phases SHOULD be done with a TERM signal, but Pallet MAY
send a KILL signal if the application afterwards or even instead of it.

### Test

Application SHOULD have a test suite. If it does, it MUST be run when `test`
command is passed to the Docker image. The command MUST exit with status 0 if
and only if the test suite passed.

Conventions on the test suite implementation, output, etc. are outside the
scope of the deployment standard. However, if a test system is provided, it
SHOULD follow the environment conventions below, for example, to provide the
services during the test.

Environment
-----------

Pallet MAY run different applications, and potentially different instances of
the same application independently of each other. Each instance is presented to
the users separately, and has no knowledge of any others.

Pallet MAY also provide external services, such as a database, a message queue,
or a mail transfer agent, to the applications.

Information about the instance configuration and required services is passed to
the application via environment variables.

### Instance configuration

Pallet MUST provide the following environment variables to all the application
commands it runs:

* `SITE_PROTOCOL`, `SITE_DOMAIN` - respectively, the URL scheme (such as `http`
  or `https` and domain, as reachable by the application users. As Pallet is
  proxying the requests to the application, it SHOULD NOT trust the HTTP `Host`
  header. If the application is exposed to the users via HTTPS, it still MUST
  accept HTTP and not HTTPS, relying on Pallet proxy.
* `ENVIRONMENT` - a string identifying the instance configuration. This is set
  up by Pallet users, and is only expected to be understood by the application.
  Typical use can be controlling the performance vs. logging output for test
  and production instances.

### Services

Pallet MAY provide external services to the applications. It is up to the
Pallet users to determine which services to provide to which applications, if
at all. However, if a service deemed required by an application is not
provided, the application behavior is unspecified.

For each type of service, Pallet MUST provide the specified environment
variables to all phases of the application. If the provided environment does
not point to a properly configured service, the application behavior is
unspecified.

#### PostgreSQL

Environment variable: `DB_DEFAULT_URL`, a
[DJ-Database-URL](https://github.com/kennethreitz/dj-database-url)-style URL
specifying the database information.

The specified URL MUST include credentials sufficient to connect to the
database. All operations on the database, including modifying the schema,
SHOULD be permitted as well. The application MUST NOT assume superuser access
to the whole database server, or any access to other databases on the same
server.

#### Elasticsearch

Environment variables:

* `ELASTICSEARCH_URLS`: a pipe (`|`)-separated list of Elasticsearch node URLs.
* `ELASTICSEARCH_INDEX_NAME`: the index name to use.

All specified URLs MUST belong to the same cluster. The application SHOULD
balance the requests uniformly across the URLs, and retry the failed requests
on a different URL from a set.

If the requests to Elasticsearch are proxied, all operations, including write,
index creation and deletion SHOULD be allowed.

The application MUST NOT make requests to any indices other than the specified
one. It MAY, however, create, delete or modify any indices having the specified
name as a prefix, for atomizing long-running index operations.

#### Memcache

Environment variables:

* `MEMCACHE_HOSTS`: a pipe (`|`)-separated list of Memcache hosts.
* `MEMCACHE_PREFIX`: a prefix to use for keys stored by the application.

#### Mail transfer agent (SMTP)

Environment variables: `EMAIL_HOST`, `EMAIL_PORT` - an address of an SMTP
server.

The SMTP server provided MUST NOT require authentication.

#### HTTP proxy

Environment variable: `HTTP_PROXY` - an URL for the HTTP proxy.

Connecting to arbitrary hosts from the application MAY be restricted by Pallet,
in which case an HTTP proxy service SHOULD be provided.

#### Syslog

Environment variables: `SYSLOG_SERVER`, `SYSLOG_HOST`, `SYSLOG_PROTO`: host,
port and protocol (`tcp` or `udp`) to connect to a Syslog instance.

The application SHOULD prefer logging events to the Syslog service rather than
to the console.

### Storage

Applications typically want to store the files, such as user uploads, and
persist them between versions. For this purpose, Pallet MUST mount a filesystem
to `/storage` inside the Docker container. The same filesystem MUST be mounted
to all the phases, including ones running in parallel and on different hosts.
The filesystem MUST be writable. The application MUST NOT expect particular
permissions (and is expected to set them up itself).

### Static file serving

Applications typically want to serve static content to the users, for which
aggressive caching is desirable and going through the application server itself
is overkill. For this purpose, Pallet provides a mechanism to optimize serving
static files.

Pallet MUST mount a filesystem to `/static` inside the Docker container during
all phases, and upon receiving a request for a URL inside `/static`, MUST serve
the corresponding file out of that filesystem.

When responding to the request, Pallet SHOULD consider the static resources
immutable, and advertise this to the clients by setting appropriate cache
headers.

The application SHOULD assume the clients can have an older version of static
resources if they are overwritten, and instead SHOULD rely on versioning or
hashing in the file names to ensure clients have correct version of resources.

As with storage, the same filesystem MUST be mounted to all the phases,
including different hosts. Pallet MAY mount either an old version's static
filesystem or a clean new one to a new version. As with storage, the
application MUST NOT expect particular permissions on any files in it.

If a new static filesystem is used for the new version, then files from the old
version SHOULD be available while any serve phases from the old version are
running. Files from the new version MUST be available as soon as at least one
new serve phase is running.

Docker parameters
-----------------

Pallet MUST run the Docker containers using the initial user and the entrypoint
specified in the application image. Port 8000 MUST be forwarded to the outside
for the purposes of sending the application requests. Any additional ports
defined in the image SHOULD NOT be forwarded. Volumes (except storage and
static) defined in the image SHOULD NOT be mounted.

Tools
-----

* [Forklift](https://github.com/infoxchange/docker-forklift) - a tool for
  developing containerized applications.

[Docker]: https://www.docker.com/
[12factor]: http://12factor.net/
[RFC 2119]: http://www.ietf.org/rfc/rfc2119
