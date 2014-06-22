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

Frequently asked questions
--------------------------

Tools
-----

* [Forklift](https://github.com/infoxchange/docker-forklift) - a tool for
  developing containerized applications.

[Docker]: https://www.docker.com/
[12factor]: http://12factor.net/
