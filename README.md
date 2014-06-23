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

Application lifecycle
---------------------

The expected main function of the applications is sering user requests.
However, to facilitate releasing new versions of the application, three
distinct actions are recognized:

* Build
* Test
* Deploy
* Serve

###Build

Entirely application-specific phase, run outside of the deployment system.
Tasks here include fetching dependencies, compilation, building and minifying
assets.

###Test

Run the application test suite. Testing is outside the scope of the deployment
standard, however, the conventions, such as the service configuration, still
apply.

###Deploy

Prepare the environment for this particular version of the application. This
includes the tasks typically known as "migrations", for exapmple, setting up
the database structure.

###Serve

The main loop of the application - serving Web requests to the users.

Services
--------

Applications are expected to depend on a number of external services, for
example, a database, a key-value store or an email server.

Specification
-------------

The Pallet specification is a contract between a deployment system (referred to
as Pallet hereafter) and an application represented by a Docker image.

Multiple independent instances of the application - environments - can be
running within Pallet at the same time.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119].

### Building the application

Building the application is entirely outside the scope of Pallet. Each
application is represented, to Pallet, as a Docker image, with tags
corresponding to versions. Pallet MAY assume the images are not changed on the
container yard in use (i.e. it MAY cache the downloaded images). Pallet assumes
no version ordering, it is up to the users to choose which versions to release.

### Deploying the application

The lifecycle of an application within Pallet begins with the deploy phase. The
application MUST support the deploy phase via running the `deploy` command.

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
and their success status was different, it MAY either terminate the deployment
or proceed with running serve phases. Pallet SHOULD NOT forcefully terminate a
running deployment phase.

The deploy phase MUST NOT assume that any other version of itself was already
released in the environment it's running in. It also SHOULD run correctly on
top of any of the versions known to be ever released into the environment.

### Serving requests

Once an application is successfully deployed, the serve phase can be run. The
application is expected to serve the incoming Web requests for the users at
this phase. It MAY choose to do additional processing as well.

The serve phase is run via the `serve` command to the Docker image.

Pallet MAY run multiple serve phases of the same application version in
parallel, once at least one deploy phase of said version completed
successfully. Pallet MAY terminate serve phases at any time, however, at least
one of them SHOULD be running at any given time.

### Testing the application

Application SHOULD have a test suite. If it does, it MUST be run when `test`
command is passed to the Docker image. The command MUST exit with status 0 if
and only if the test suite passed.

Frequently asked questions
--------------------------

Tools
-----

* [Forklift](https://github.com/infoxchange/docker-forklift) - a tool for
  developing containerized applications.

[Docker]: https://www.docker.com/
[12factor]: http://12factor.net/
[RFC 2119]: http://www.ietf.org/rfc/rfc2119
